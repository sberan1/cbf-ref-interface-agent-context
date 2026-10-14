# 2026-10-14 – Scaffold ingestu ČBF exportu

## Co se řešilo

Zadání: „udělej základní scaffold pro integraci s ČBF XML exporty, integrovat cca 20 sezón zpátky, ukládej všechno do databáze, udělej schéma a základní strukturu pro stahování, rozhodnutí ukládej do agent contextu (git submodul)“.

Výchozí stav: repo `sberan1/cbf-ref-interface` bylo čerstvě založené přes `sv create` (SvelteKit 3 + Drizzle + PostgreSQL, demo tabulka `task`). Submodul `docs/agent-context` obsahoval README, šablony, ADR 0001 a history 20260923. Zápis do knowledge-base repa (problém s 403 z handoffu) tentokrát fungoval přes git push z cloudové session.

V druhé části konverzace uživatel povolil síťový přístup na `cbf.cz` a proběhlo discovery živého API (viz níže).

## Otázky a odpovědi uživatele

| Otázka | Odpověď |
| --- | --- |
| Stack (repo je SvelteKit, handoff zmiňoval NestJS + React)? | SvelteKit fullstack → [ADR 0002](../adr/0002-stack-sveltekit-drizzle-postgres.md) |
| Jak řadit sync joby? | „klidně naimplementuj RMQ nebo Kafku“ → zvolen RabbitMQ, [ADR 0003](../adr/0003-sync-fronta-rabbitmq.md) |
| Tempo requestů? | 1 request / 2–3 s → [ADR 0005](../adr/0005-rozsah-a-setrnost-stahovani.md) |
| Verze exportu? | v2 JSON primárně, v1 XML doplňkově → ADR 0005 |

## Co vzniklo (repo cbf-ref-interface)

- DB schéma `ingest` (raw_payload, fetch_state, fetch_log) a `cbf` (po discovery 19 normalizovaných tabulek) + migrace → [ADR 0004](../adr/0004-db-schema-ingestu.md).
- `src/lib/server/cbf/`: registr endpointů, HTTP klient s rate limiterem, parsery v1/v2, normalizery a writery (včetně historie delegací), fetch/normalize pipeline, RabbitMQ broker, scheduler (bootstrap / daily / window), discovery.
- CLI `npm run cbf -- …` (worker, bootstrap, daily, window, fetch, renormalize, requeue-dead, stats, discover), `compose.yaml` s RabbitMQ, `.env.example`.
- Unit testy (vitest, po discovery 38 testů): coercion, převod Europe/Prague → UTC přes DST, převod průběžného skóre, parsery (fixtury podle skutečné struktury, smyšlené osoby), diff delegací, rate limiter, plánování.

## Ověření (scaffold)

- Typecheck (svelte-check), ESLint, Prettier, unit testy.
- End-to-end běh proti **falešnému** ČBF serveru (syntetická data) s PostgreSQL 17 a RabbitMQ 4 v Dockeru:
  - retry po HTTP 500,
  - opakovaný bootstrap bez requestů,
  - zachycení výměny rozhodčího,
  - `renormalize` bez HTTP.

## Discovery živého API (týž den, druhá část konverzace)

Uživatel povolil v síťové politice prostředí `cbf.cz` (`www.cbf.cz` zůstal blokovaný). Node `fetch` v sandboxu potřebuje `NODE_USE_ENV_PROXY=1`.

`npm run cbf -- discover` udělal 45 requestů (všechny HTTP 200, sezóny 2026/27 a 2012/13, oblasti 0–20). Pak jeden ruční request na `game.php?game=544556` kvůli ověření stability ID. Vzorky jsou lokálně v `discovery/out/20261014/` (gitignored, obsahují osobní údaje).

**Ověřená fakta** (zdroj: živé odpovědi ze `https://cbf.cz/xml/`):

- `seasonList.php` vrací 35 sezón od 1992/93 (XML, s `json=1` i JSON), s `short_name` = rok začátku a `date_of_begin` / `date_of_end`. `IDseason` 2026/27 = 36 a odpovídá `sid` v `divs.php`.
- `season=` / `s=` je rok začátku sezóny.
- `divs.php` (v2) nese `area`, oblasti se nepřekrývají. Data v 2026/27 mají oblasti 0–8 a 12. `cgid` / `cgname` je liga (NBL, Oblasti), ne skupina kategorie.
- `divs.php?s=` (v1) obsahuje všechny soutěže sezóny v jednom dokumentu (331 / 190) s `<flags><flag name/>`, `<cat>`, `<league>`, fázemi s `<type>`.
- **ID prostory:**
  - v1 a v2 sdílejí ID soutěží, fází a zápasů.
  - Osoby a týmy jsou ve v1 sched.php identifikované „group“ ID (= v2 `uNgroup`, `tagroupby`). v1 `table.php` ale používá v2 `IDteam` + `<group>`.
  - v2 ID rozhodčích a komisařů jsou stejná v 2012/13 i 2026/27 (ověřeno na jednom rozhodčím a jednom komisaři) a group ID taky (15 osob v obou v1 vzorcích, všechny se stejným jménem).
- `games.php`: komisař ano, **rozhodčí ne**.
- `game.php`:
  - `gameInfo` obsahuje `u1..u3` (`id`, `n1` = jméno, `n2` = příjmení, `group`), hodnocení `rating_uN_home|guest|commisar` (ve vzorcích NULL), `IDhall`, `checked`, `num_visitors`.
  - `playerStats` a `teamStats` jsou box score s kódy ČBF.
  - Neveřejná delegace: `uNid=0`, `uNn2="Delegováno"`.
- `score_quarter` a v1 `<partials>` jsou **průběžný stav** (po Q1–Q3, s prodloužením i po Q4), ne body za čtvrtinu.
- `table.php` (v2): `countmatch`, `countwin`, `countloose`, `sumscore1` / `2`, `sumpoints` (u NBL procento výher), `position`, `gr` / `group`.
- `teamplayers.php`: `playerstatcount` (hráč + sezónní statistiky, `IDplayer` = `IDuser` z game.php).
- `team.php`: `info`, `contact` (osobní údaje funkcionářů včetně data narození a adresy), `match`, `matchaggr`.
- Odkazy na DTD specifikace: `http://www.cbf.cz/xml/spec/{divs,sched,table}.dtd`, `http://www.cbf.cz/xml/api/spec/seasonlist.dtd` (zatím nestažené).

**Úpravy podle discovery** (PR sberan1/cbf-ref-interface#1):

- Parsery v1/v2 přepsané na skutečná pole. v1 sched nezapisuje ID týmů ani delegace, jen doplní `official.group_id`.
- Převod průběžného skóre na body za období.
- „Delegováno“ se bere jako neznámý stav.
- Mapování statistik (`player_game_stats`, nová tabulka `game_team_stats`).
- Sloupce `group_id` (official, player, team), `venue.hall_id`, `game.result_checked` / `attendance`, `game_official.ratings`, `season.begins_on` / `ends_on`, liga u `competition`.
- `is_initial` se počítá pro každou roli.
- Výchozí host je `cbf.cz`.
- Replay všech vzorků přes celou pipeline (Postgres + RabbitMQ): 0 chyb normalizace. Součty bodů hráčů sedí na konečné skóre. Všech 33 rozhodčích a komisařů má `group_id`, komisaři přes v1 `<sup>`.
- Aktualizované [ADR 0004](../adr/0004-db-schema-ingestu.md) a [ADR 0005](../adr/0005-rozsah-a-setrnost-stahovani.md).

## Review PR (review agent se skillem `code-review`)

Na žádost uživatele prošel PR review agent. Nálezy a opravy:

- **Blokující:** refetch se stejným obsahem přepsal content-aware `next_due_at` výchozím TTL, takže odehrané zápasy by se stahovaly denně. Normalizer teď má hook `nextDueAt`, který se vyhodnotí po každém fetchi. Ověřeno replayem.
- **Window:** obnova v okně sjednocená na 12 h (dřív politika 6 h vs. `window` 12 h).
- **Delegace:** odebrání delegace už nejde přepsat starším snapshotem (strop z historie), paralelní zápisy jednoho zápasu serializuje advisory lock.
- **HTTP:** čtení těla odpovědi je pod stejným timeoutem i klasifikací chyb (audit v `fetch_log`). Pauza se počítá od konce requestu.
- **Broker:** listenery chyb na kanálech, restart při zavření kanálu nebo zrušení consumera. Ověřeno smazáním fronty pod běžícím workerem.
- **Blokační pauza:** přežije reconnect i restart (obnova z `fetch_log`). Ověřeno.
- **Drobnosti:**
  - prázdné env hodnoty nejsou 0,
  - propojení v1 group ID vyžaduje shodu příjmení,
  - `LOG_LEVEL` se čte až po načtení `.env`,
  - opravené texty.
- Nepřevzato: čistě kosmetické cleanupy (duplicitní bloky consumerů, SQL po jednom řádku u venue).

## Další kroky

1. Spustit `bootstrap` (lokálně nebo na serveru) a průběžně sledovat `stats` a management UI RabbitMQ. Ověřit sezóny 2006–2011.
2. Nastavit cron pro `daily` a `window`, rozhodnout o nasazení (ADR 0002, otevřená otázka).
3. Stáhnout DTD specifikace ČBF a porovnat je s mapováním.
4. Typované sloupce statistik podle potřeb UI, výklad kódů (`ZS2DP` / `ZS2KP` …).
5. Zvážit oficiální žádost ČBF o přístup. Vyjasnit GDPR (hráči mládeže, kontakty z `team.php`), viz ADR 0004.
