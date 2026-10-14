# 0003 – Fronta sync jobů: RabbitMQ

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [ADR 0001](0001-cbf-data-ingestion-strategy.md), [ADR 0005](0005-rozsah-a-setrnost-stahovani.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Kontext

Podle ADR 0001 jde všechno přes sync joby. Bootstrap 20 sezón je řádově statisíce requestů (odhad v ADR 0005), při 1 requestu za 2–3 s tedy dny až týdny běhu. Fronta proto musí:

- přežít restart a navázat,
- dodržet **jeden globální limit rychlosti** vůči ČBF bez ohledu na počet workerů,
- umět **zpožděné retry** (5xx, timeouty, blokace),
- dát **přednost** inkrementálnímu syncu aktuální sezóny před bootstrapem staré historie,
- odkládat neopravitelné zprávy stranou, aby šly prohlédnout.

Uživatel na otázku, jak řadit joby, odpověděl „klidně naimplementuj RMQ nebo Kafku“. Výběr mezi nimi tím delegoval.

## Rozhodnutí

**RabbitMQ 4** (classic queues), klient `amqplib`. Topologie (`src/lib/server/cbf/queue/broker.ts`):

| Fronta | Vlastnosti | Účel |
| --- | --- | --- |
| `cbf.fetch` | `x-max-priority=10`, `x-single-active-consumer`, prefetch 1 | HTTP requesty na ČBF, vždy jen jeden aktivní consumer → globální rate limit |
| `cbf.normalize` | `x-max-priority=10`, prefetch 4 (`CBF_NORMALIZE_PREFETCH`) | parsování raw payloadu → normalizované tabulky, expanze potomků |
| `<q>.retry.30s`, `.5m`, `.30m` | `x-message-ttl`, dead-letter zpět do `<q>` | zpožděné retry (pokus 1 → 30 s, 2 → 5 min, 3+ → 30 min) |
| `<q>.dead` | – | zprávy po `CBF_MAX_ATTEMPTS` (6) pokusech nebo s trvalou chybou; zpět do fronty přes `npm run cbf -- requeue-dead` |

- Publikuje se přes výchozí exchange přímo do front s publisher confirms a persistentními zprávami.
- Doručení je **at-least-once**, handlery jsou idempotentní (upserty, deduplikace hashem, `fetch_state`).
- **Zdrojem pravdy je DB, ne fronta.** O tom, co je stažené a kdy se má obnovit, rozhoduje `ingest.fetch_state`. Když se fronta ztratí, stačí znovu spustit `bootstrap`/`daily`: čerstvé zdroje se přeskočí bez HTTP requestu a jen se znovu rozbalí na potomky.
- Priority: manual 10 > window 9 > daily 8 > bootstrap 7…1 (novější sezóna = vyšší priorita).
- Worker při ztrátě spojení skončí a vnější smyčka ho po 5 s spustí znovu (`runWorkerForever`). Nepotvrzené zprávy RabbitMQ doručí znovu.

## Zvažované alternativy

- **Kafka:** je to log, ne task queue. Nemá ack a zpoždění na úrovni jednotlivé zprávy ani priority. Retry se řeší dalšími topicy a vlastní logikou. Globální rate limit přes partitions a consumer groups je neohrabaný a provoz (KRaft, retence) je pro pár set tisíc zpráv zbytečně těžký. Hlavní výhodu Kafky, replay historie, už pokrývá raw vrstva v Postgresu (`renormalize`).
- **Fronta v Postgresu** (`FOR UPDATE SKIP LOCKED`) nebo **pg-boss:** nejméně komponent, transakčně svázané s daty. Uživatel ale chtěl dedikovaný broker. Navíc RabbitMQ management UI dává dobrý přehled o průběhu několikadenního bootstrapu. Kdyby se RabbitMQ ukázal jako zbytečná režie, přechod je levný: pipeline závisí jen na rozhraní `TaskSink`.

## Důsledky

- \+ Single active consumer drží šetrnost vůči ČBF i při víc instancích workeru (další instance jsou hot standby).
- \+ Retry a parkování chyb bez vlastního kódu pro plánování.
- − Další služba v `compose.yaml` (`rabbitmq:4-management`, porty 5672 a 15672).
- − Argumenty front nejdou později změnit redeklarací. Změna (např. jiné TTL) znamená smazat frontu a znovu ji vytvořit; obsah se dá obnovit spuštěním `bootstrap`/`daily`.
- − Retry tiery jsou pevné (30 s / 5 min / 30 min). Jemnější backoff by vyžadoval další fronty.

## Otevřené otázky

- Monitoring a alerty (např. rostoucí `cbf.fetch.dead`, dlouhodobě pozastavený fetch kvůli blokaci).
