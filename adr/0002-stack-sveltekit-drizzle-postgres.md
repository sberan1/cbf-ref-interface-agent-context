# 0002 – Stack: SvelteKit fullstack + Drizzle + PostgreSQL

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [history/20260923-analyza-cbf-exportu.md](../history/20260923-analyza-cbf-exportu.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md), [ADR 0003](0003-sync-fronta-rabbitmq.md)

## Kontext

Handoff z 2026-09-23 zmiňoval „plánovaný stack NestJS API + React“, formální ADR k tomu ale nevzniklo. Repo `sberan1/cbf-ref-interface` mezitím uživatel založil přes `sv create` (SvelteKit 3, Svelte 5, TypeScript, Tailwind 4, Drizzle ORM + drizzle-kit, PostgreSQL přes `postgres.js`, ESLint, Prettier, Playwright). Uživatel 2026-10-14 potvrdil variantu SvelteKit fullstack.

## Rozhodnutí

1. **SvelteKit** dělá UI i API (server routes / form actions). Samostatná API služba nebude.
2. **Drizzle ORM + PostgreSQL**. Schéma je v `src/lib/server/db/schema/`, migrace generuje `drizzle-kit generate` do `drizzle/` a commitují se do repa (`db:migrate`, ne `db:push`).
3. **Sync worker ČBF** běží jako samostatný Node proces ve stejném repu (`scripts/cbf.ts`, spouští se přes `tsx`). S aplikací sdílí kód v `src/lib/server/` a DB schéma. Na SvelteKitu nezávisí: `$app/env` používá jen aplikace, worker čte `process.env` (případně `.env`).
4. Jazyk je **TypeScript** všude, tedy i discovery skript. Tím se uzavírá otevřená otázka Node/TS vs Python z history 20260923.

## Zvažované alternativy

- **NestJS API + React SPA:** dvě aplikace a dvě build pipeline. Pro jednoho vývojáře a interní nástroj zbytečná režie. Uživatel zvolil SvelteKit.
- **SvelteKit jen jako UI + samostatné API:** zatím neodůvodněné. Pokud bude API potřeba pro třetí strany, jde přidat později.

## Důsledky

- \+ Jeden repozitář, jeden jazyk, sdílené typy mezi DB, workerem a UI.
- \+ Worker jde nasadit a škálovat nezávisle na webu (jiný proces, stejný kód).
- − Worker lokálně běží přes `tsx` (transpilace za běhu). V produkci je zbundlovaný esbuildem (`dist/cbf.js`, [ADR 0007](0007-nasazeni-docker-vps.md)).
- − ~~`adapter-auto` zůstává, cílová platforma nasazení zatím není určená.~~ Nahrazeno `adapter-node`, viz [ADR 0007](0007-nasazeni-docker-vps.md).

## Otevřené otázky

- ~~Cílové nasazení (VPS + docker compose? jiná platforma?) a podle toho volba adapteru.~~ Uzavřeno [ADR 0007](0007-nasazeni-docker-vps.md): Docker na VPS, `adapter-node`.
