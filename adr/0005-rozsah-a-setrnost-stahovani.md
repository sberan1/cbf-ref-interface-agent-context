# 0005 – Rozsah stahování a šetrnost vůči ČBF

- **Status:** Accepted (parametry zvolené uživatelem; TTL obnovy jsou výchozí hodnoty k ladění)
- **Datum:** 2026-10-14 (upraveno týž den po discovery)
- **Související:** [ADR 0001](0001-cbf-data-ingestion-strategy.md) (zpřesňuje bod 5 a otevřenou otázku intervalů), [ADR 0003](0003-sync-fronta-rabbitmq.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Kontext

Uživatel chce integrovat zhruba 20 sezón zpět a ukládat všechno. ČBF žádá šetrné používání exportu, jinak hrozí blokace. Uživatel zvolil tempo **1 request / 2–3 s** a **v2 JSON primárně, v1 XML doplňkově**.

## Rozhodnutí

### Rozsah

- **Sezóny:** aktuální + 20 předchozích (`CBF_BOOTSTRAP_SEASONS=20`), k 2026-10 tedy 2006/07–2026/27.
  - Rok sezóny je hodnota parametru `season` / `s` = rok začátku. Ověřeno: `season=2012` → 2012/13.
  - `seasonList.php` má 35 sezón od 1992/93.
  - Aktuální sezóna se od srpna počítá z data a jde přepsat `CBF_CURRENT_SEASON_YEAR`. Pozor: ČBF vede 2026/27 jako aktuální už od 2026-05-01 (`date_of_begin`).
- **Oblasti:** `CBF_AREAS=0-14`. V sezóně 2026/27 vrací soutěže oblasti 0–8 a 12, oblasti 9–11 a 13–20 jsou prázdné (prázdná odpověď stojí 1 levný request). Oblasti se nepřekrývají.
- **Host:** `https://cbf.cz/xml/` (ověřený). Klienti používají i `www.cbf.cz`.
- **Pohlaví:** obě (`female=0|1`).
- **Hierarchie:** `v2.divs` (sezóna × oblast × pohlaví) → `v2.phases` → pro fázi `v2.games` + `v2.table` + `v1.sched` → pro zápas `v2.game`, pro tým v soutěži `v2.team` + `v2.teamplayers`. K tomu `v1.divs` jednou za sezónu (všechny soutěže sezóny v jednom dokumentu: 331 pro 2026/27, 190 pro 2012/13, s flagy) a `v2.seasonList`. `v1.table` se stahuje jen v discovery (duplicita `v2.table`).

### Šetrnost

- Mezi requesty je pauza **2000 ms + náhodně 0–1000 ms** (`CBF_MIN_INTERVAL_MS`, `CBF_JITTER_MS`), počítaná od konce předchozího requestu. V jednu chvíli běží **jen jeden request** (single active consumer, ADR 0003).
- Na **403 / 429 / 503** se všechny requesty zastaví na **15 min** (`CBF_BLOCK_PAUSE_MS`) a zpráva jde do retry. Pauza přežije restart workeru i převzetí fronty jiným procesem: obnoví se z posledního `error_blocked` v `fetch_log`.
- **Chybová stránka s HTTP 200** (ČBF při výpadku databáze vrací text „Nepovedlo se pripojit k databázi…“ místo dat) se neukládá jako obsah zdroje, jinak by zdroj platil za čerstvý a zpracování by selhalo natrvalo. Bere se jako chyba serveru, jde do retry, a když text zmiňuje databázi, všechny requesty se zastaví na `CBF_BLOCK_PAUSE_MS` (tato pauza restart workeru nepřežije). Zdroje, u kterých se taková stránka uložila dřív, znovu stáhne `cbf.js refetch-failed` (doplněno 2026-10-16).
- Retry 30 s / 5 min / 30 min, maximálně 6 pokusů, pak dead queue. 404 se uloží jako prázdný zdroj a neopakuje se.
- User-Agent identifikuje projekt (`cbf-ref-interface/0.1 (+https://github.com/sberan1/cbf-ref-interface)`) a jde změnit přes `CBF_USER_AGENT`.
- Uživatelské requesty aplikace nikdy nevolají ČBF (ADR 0001 bod 4).

### Obnova dat (`ingest.fetch_state.next_due_at`)

| Zdroj (aktuální sezóna) | Výchozí obnova |
| --- | --- |
| divs, phases, games, table, v1 sched | 20 h (běh `daily`) |
| seasonList, v1 divs | 24 h |
| game detail | podle stavu zápasu (přepočítá se po každém stažení, i když se obsah nezměnil): odehraný a starší než 3 dny → nikdy (nepotvrzený výsledek se kontroluje ještě 14 dní); v okně −3/+14 dní → 12 h (stejně jako `window`); dál v budoucnu → 7 dní; bez data nebo neodehraný → 3 dny |
| team | 7 dní |
| teamplayers | 3 dny |

- **Historické sezóny** (rok < aktuální) se považují za neměnné: stáhnou se jednou (`next_due_at = NULL`). Znovu jen přes `--force`, který se u rekurzivní úlohy propaguje na celý podstrom.
- **Plány:** `bootstrap` jednorázově, `daily` 1× denně (cron), `window` po několika hodinách (cron, detail zápasu se znovu stáhne, pokud je starší než `CBF_WINDOW_MAX_AGE_HOURS=12`).
- **Priority:** window > daily > bootstrap. Bootstrap jde od nejnovější sezóny, takže nedávná historie je použitelná nejdřív.

### Odhad objemu (neověřeno)

Řádově 10–25 tis. requestů na sezónu, většinou detaily zápasů. Za 21 sezón tedy 200–500 tis. requestů. Při průměru 2,5 s to je zhruba **6–15 dní nepřetržitého běhu**.

Z discovery víme počty soutěží (331 v 2026/27, 190 v 2012/13). Počet zápasů na fázi se pohybuje kolem stovky (NBL 132 a 156), mládež a oblasti mají méně. Přesnější číslo dá první sezóna bootstrapu (`npm run cbf -- stats`).

## Zvažované alternativy

- **1 request / 1 s:** bootstrap 2–3× rychlejší, ale vyšší riziko blokace. Uživatel zvolil pomalejší variantu.
- **Jen v2:** méně requestů, ale bez kontroly delegací proti dokumentovanému v1 a bez flagů soutěží.
- **Window přes detail zápasu každé 3 h:** pro tisíce zápasů v okně příliš mnoho requestů.
- **Window přes `games.php` (1 request na fázi):** discovery ukázala, že seznam zápasů nese jen komisaře, ne rozhodčí. Rozhodčí jsou jen v detailu, takže window zůstává na `game.php` s `CBF_WINDOW_MAX_AGE_HOURS=12`. Rozhodčí se navíc před zveřejněním delegace skrývají („Delegováno“).

## Důsledky

- \+ Zátěž ČBF odpovídá jednomu pomalému klientovi a každý request je auditovaný v `fetch_log`.
- − Bootstrap trvá dny až týdny. Je přerušitelný a navazuje (freshness v `fetch_state`).
- − Data aktuální sezóny jsou zpožděná podle frekvence `daily` / `window`.

## Otevřené otázky

- ~~Význam parametru `season`?~~ Rok začátku sezóny (ověřeno). v2 i v1 mají data aspoň od 2012/13. Sezóny 2006–2011 ověří bootstrap.
- ~~Má `games.php` delegace?~~ Jen komisaře (viz alternativy).
- Kdy ČBF zveřejňuje jména rozhodčích (místo „Delegováno“)? Podle toho naladit okno a TTL.
- Vyřadit z `daily` ukončené soutěže (`competition.flags.finished`).
- Oficiální žádost ČBF o přístup / domluva na tempu (ADR 0001).
