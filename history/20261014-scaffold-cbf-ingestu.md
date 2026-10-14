# 2026-10-14 – Scaffold ingestu ČBF exportu

## Co se řešilo

Zadání: „udělej základní scaffold pro integraci s ČBF XML exporty, integrovat cca 20 sezón zpátky, ukládej všechno do databáze, udělej schéma a základní strukturu pro stahování, rozhodnutí ukládej do agent contextu (git submodul)“.

Výchozí stav: repo `sberan1/cbf-ref-interface` bylo čerstvě založené přes `sv create` (SvelteKit 3 + Drizzle + PostgreSQL, demo tabulka `task`). Submodul `docs/agent-context` obsahoval README, šablony, ADR 0001 a history 20260923. Zápis do knowledge-base repa (problém s 403 z handoffu) tentokrát fungoval přes git push z cloudové session.

## Otázky a odpovědi uživatele

| Otázka | Odpověď |
| --- | --- |
| Stack (repo je SvelteKit, handoff zmiňoval NestJS + React)? | SvelteKit fullstack → [ADR 0002](../adr/0002-stack-sveltekit-drizzle-postgres.md) |
| Jak řadit sync joby? | „klidně naimplementuj RMQ nebo Kafku“ → zvolen RabbitMQ, [ADR 0003](../adr/0003-sync-fronta-rabbitmq.md) |
| Tempo requestů? | 1 request / 2–3 s → [ADR 0005](../adr/0005-rozsah-a-setrnost-stahovani.md) |
| Verze exportu? | v2 JSON primárně, v1 XML doplňkově → ADR 0005 |

## Co vzniklo (repo cbf-ref-interface)

- DB schéma `ingest` (raw_payload, fetch_state, fetch_log) a `cbf` (18 normalizovaných tabulek) + migrace → [ADR 0004](../adr/0004-db-schema-ingestu.md).
- `src/lib/server/cbf/`: registr endpointů, HTTP klient s rate limiterem, parsery v1/v2, normalizery a writery (včetně historie delegací), fetch/normalize pipeline, RabbitMQ broker, scheduler (bootstrap / daily / window), discovery.
- CLI `npm run cbf -- …` (worker, bootstrap, daily, window, fetch, renormalize, requeue-dead, stats, discover), `compose.yaml` s RabbitMQ, `.env.example`.
- Unit testy (vitest, 33 testů): coercion, převod Europe/Prague → UTC přes DST, parsery na syntetických fixturách, diff delegací, rate limiter, plánování.

## Ověření

- **Ověřeno:** typecheck (svelte-check), ESLint, Prettier, unit testy. End-to-end běh proti **falešnému** ČBF serveru (syntetická data) s reálným PostgreSQL 17 a RabbitMQ 4 v Dockeru:
  - bootstrap 2 sezón: 53 requestů, z toho 2 simulované chyby HTTP 500, které se po 30 s zopakovaly přes retry frontu,
  - opakovaný bootstrap: 0 HTTP requestů (freshness),
  - `window` zachytil výměnu rozhodčího (`replaced` v `game_official_history`),
  - `renormalize --force`: 0 HTTP requestů a žádné duplicitní změny v historii.
- **Neověřeno:** živé API ČBF. Proxy sandboxu znovu odmítla `www.cbf.cz` i `cbf.cz.basketball` (403 na CONNECT). Mapování polí vychází z dokumentace a z kódu klientů `jknetl/CzechBasketballSchedule` (cbf-rest-api.adoc, model.py) a `ivosu/cbf_ical` (Cbf.py). Z nich je nové hlavně:
  - `game.php` vrací `{ gameInfo: [ {...} ], … }`,
  - `team.php` vrací `{ info: [...], match: [...] }`,
  - `seasonList.php` bez `json=1` vrací XML `<season><IDseason/><name/><short_name/><current/></season>`,
  - `table.php` (v2) obsahuje `tid`, `cid`,
  - v1 `sched.php`: `<team guest="0|1">`, `<result><pts|score><a/><b/></…><partials><partial ord>`, `<urllive>`,
  - ČBF posílá skóre 0:0 u neodehraných zápasů,
  - názvy oblastí 0–3, 5–8, 12–14 (z `ivosu/cbf_ical`).

## Další kroky

1. Lokálně spustit `npm run cbf -- discover` (cca 50 requestů, pár minut) a podle `discovery/out/*/summary.json` opravit místa označená PROVISIONAL v `parse/v2.ts`. Ověřit otevřené otázky ADR 0004 a 0005.
2. Po discovery spustit `bootstrap` a průběžně sledovat `stats` a management UI RabbitMQ.
3. Nastavit cron pro `daily` a `window`, rozhodnout o nasazení (ADR 0002, otevřená otázka).
4. Doplnit mapování statistik hráčů (`player_game_stats`).
5. Zvážit oficiální žádost ČBF o přístup. Vyjasnit GDPR u osobních údajů, hlavně nezletilých hráčů (ADR 0004).
