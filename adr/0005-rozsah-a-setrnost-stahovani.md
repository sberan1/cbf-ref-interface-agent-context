# 0005 – Rozsah stahování a šetrnost vůči ČBF

- **Status:** Accepted (parametry zvolené uživatelem; TTL obnovy jsou výchozí hodnoty k ladění)
- **Datum:** 2026-10-14
- **Související:** [ADR 0001](0001-cbf-data-ingestion-strategy.md) (zpřesňuje bod 5 a otevřenou otázku intervalů), [ADR 0003](0003-sync-fronta-rabbitmq.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Kontext

Uživatel chce integrovat zhruba 20 sezón zpět a ukládat všechno. ČBF žádá šetrné používání exportu, jinak hrozí blokace. Uživatel zvolil tempo **1 request / 2–3 s** a **v2 JSON primárně, v1 XML doplňkově**.

## Rozhodnutí

### Rozsah

- **Sezóny:** aktuální + 20 předchozích (`CBF_BOOTSTRAP_SEASONS=20`), k 2026-10 tedy 2006/07–2026/27. Rok sezóny je hodnota parametru `season` / `s`. Předpoklad: je to rok začátku sezóny (neověřeno). Aktuální sezóna se od srpna počítá z data, jde přepsat `CBF_CURRENT_SEASON_YEAR`.
- **Oblasti:** `CBF_AREAS=0-14` (0 = celostátní, 1–14 regiony včetně 4, 9–11, které klienti neznají). Discovery zkouší 1–20.
- **Pohlaví:** obě (`female=0|1`).
- **Hierarchie:** `v2.divs` (sezóna × oblast × pohlaví) → `v2.phases` → pro fázi `v2.games` + `v2.table` + `v1.sched` → pro zápas `v2.game`, pro tým v soutěži `v2.team` + `v2.teamplayers`. K tomu `v1.divs` jednou za sezónu (flagy soutěží) a `v2.seasonList`. `v1.table` se stahuje jen v discovery (duplicita `v2.table`).

### Šetrnost

- Mezi requesty je pauza **2000 ms + náhodně 0–1000 ms** (`CBF_MIN_INTERVAL_MS`, `CBF_JITTER_MS`) a v jednu chvíli běží **jen jeden request** (single active consumer, ADR 0003).
- Na **403 / 429 / 503** se všechny requesty zastaví na **15 min** (`CBF_BLOCK_PAUSE_MS`) a zpráva jde do retry.
- Retry 30 s / 5 min / 30 min, maximálně 6 pokusů, pak dead queue. 404 se uloží jako prázdný zdroj a neopakuje se.
- User-Agent identifikuje projekt (`cbf-ref-interface/0.1 (+https://github.com/sberan1/cbf-ref-interface)`) a jde změnit přes `CBF_USER_AGENT`.
- Uživatelské requesty aplikace nikdy nevolají ČBF (ADR 0001 bod 4).

### Obnova dat (`ingest.fetch_state.next_due_at`)

| Zdroj (aktuální sezóna) | Výchozí obnova |
| --- | --- |
| divs, phases, games, table, v1 sched | 20 h (běh `daily`) |
| seasonList, v1 divs | 24 h |
| game detail | podle stavu zápasu: odehraný a starší než 3 dny → nikdy; v okně −3/+14 dní → 6 h; dál v budoucnu → 7 dní; bez data nebo neodehraný → 3 dny |
| team | 7 dní |
| teamplayers | 3 dny |

- **Historické sezóny** (rok < aktuální) se považují za neměnné: stáhnou se jednou (`next_due_at = NULL`). Znovu jen přes `--force`, který se u rekurzivní úlohy propaguje na celý podstrom.
- **Plány:** `bootstrap` jednorázově, `daily` 1× denně (cron), `window` po několika hodinách (cron, detail zápasu se znovu stáhne, pokud je starší než `CBF_WINDOW_MAX_AGE_HOURS=12`).
- **Priority:** window > daily > bootstrap. Bootstrap jde od nejnovější sezóny, takže nedávná historie je použitelná nejdřív.

### Odhad objemu (neověřeno)

Řádově 10–25 tis. requestů na sezónu, většinou detaily zápasů. Za 21 sezón tedy 200–500 tis. requestů. Při průměru 2,5 s to je zhruba **6–15 dní nepřetržitého běhu**. Přesnější číslo dá discovery a první sezóna bootstrapu (`npm run cbf -- stats`).

## Zvažované alternativy

- **1 request / 1 s:** bootstrap 2–3× rychlejší, ale vyšší riziko blokace. Uživatel zvolil pomalejší variantu.
- **Jen v2:** méně requestů, ale bez kontroly delegací proti dokumentovanému v1 a bez flagů soutěží.
- **Window přes detail zápasu každé 3 h:** pro tisíce zápasů v okně příliš mnoho requestů. Pokud discovery potvrdí, že `games.php` obsahuje delegace, přejít na obnovu seznamu zápasů aktivních fází (1 request na fázi).

## Důsledky

- \+ Zátěž ČBF odpovídá jednomu pomalému klientovi a každý request je auditovaný v `fetch_log`.
- − Bootstrap trvá dny až týdny. Je přerušitelný a navazuje (freshness v `fetch_state`).
- − Data aktuální sezóny jsou zpožděná podle frekvence `daily` / `window`.

## Otevřené otázky

- Ověřit význam parametru `season` (rok začátku?) a dostupnost starých sezón ve v2 i v1. Pokud v2 staré sezóny nemá, použít pro ně v1.
- Má `games.php` delegace? Pak zlevnit window (viz výše).
- Vyřadit z `daily` ukončené soutěže (`competition.flags.finished`).
- Oficiální žádost ČBF o přístup / domluva na tempu (ADR 0001).
