# 0007 – Nasazení: Docker na vlastním VPS, CI/CD přes GitHub Actions

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [ADR 0002](0002-stack-sveltekit-drizzle-postgres.md) (uzavírá otevřenou otázku nasazení), [ADR 0003](0003-sync-fronta-rabbitmq.md), [ADR 0005](0005-rozsah-a-setrnost-stahovani.md), [ADR 0006](0006-session-a-parovani-uctu.md)

## Kontext

Uživatel chce aplikaci provozovat v Dockeru na vlastním VPS a nasazovat automaticky po pushi do `master`. Na VPS už má vlastní nginx/Traefik a před aplikaci si sám dá basic auth (aplikace zatím nemá hesla, PDR 0001). Kromě webu musí trvale běžet sync worker, plánovač `daily` / `window`, PostgreSQL a RabbitMQ.

## Rozhodnutí

1. **Adapter:** `@sveltejs/adapter-node` 6 místo `adapter-auto`. Origin se bere z hlavičky `Host` a schéma z `X-Forwarded-Proto` (`PROTOCOL_HEADER`, nastavené v image). Proměnnou `ORIGIN` adapter-node 6 nemá. Reverse proxy proto musí posílat `Host` a `X-Forwarded-Proto`, jinak CSRF kontrola SvelteKitu odmítne POST loginu (403).
2. **Jeden image pro všechny role** (`Dockerfile`, multi-stage, `node:22-bookworm-slim`, běží jako uživatel `node`). Obsahuje `build/` (web), `dist/cbf.js` a `dist/migrate.js` (worker CLI a migrace zbundlované esbuildem, npm balíčky zůstávají externí) a `drizzle/`. Worker v produkci už neběží přes `tsx`.
3. **Produkční stack** `deploy/compose.yaml` (projekt `cbf-ref`):
   - `db` (`postgres:17`) a `rabbitmq` (`rabbitmq:4-management`, pevný `hostname`, aby volume přežil přetvoření kontejneru).
   - `migrate`: jednorázová služba spouštěná při každém `up`. `app`, `worker` a `scheduler` na ní závisí přes `service_completed_successfully`.
   - `app` na `127.0.0.1:3000`, `worker`, `scheduler`.
   - Porty jen na loopback (web pro nginx na hostu, management UI RabbitMQ přes SSH tunel). Rotace logů, `init: true`, healthcheck aplikace na `/healthz` (dotaz do DB).
4. **Plánovač jako služba**, ne cron na hostu: příkaz `cbf scheduler` zařadí `daily` jednou za pražský den po `CBF_DAILY_AT` (04:30) a `window` při startu a pak každých `CBF_WINDOW_EVERY_HOURS` (6 h). Stav drží jen v paměti. Restart proto může úlohy zařadit znovu, ale čerstvé zdroje fetch krok přeskočí bez HTTP requestu (ADR 0005), takže ČBF nic navíc nedostane.
5. **DB připojení aplikace je líné** (`getDb()`), aby `vite build` nepotřeboval `DATABASE_URL`. Na `sveltekit:shutdown` se pool zavře, takže `docker stop` aplikaci ukončí čistě, ne SIGKILLem.
6. **CI/CD** (`.github/workflows/ci-cd.yml`):
   - **check:** lint, svelte-check, unit testy a kontrola driftu migrací (`drizzle-kit generate` nesmí nic vygenerovat).
   - **image:** build přes buildx s GHA cache a smoke test `scripts/docker-smoke-test.sh` (migrace dvakrát, `/healthz`, login za TLS proxy, CSRF 403 při cizím originu, CLI). Na `master` push do GHCR jako `:<sha>` a `:latest`.
   - **deploy** (jen `master`, environment `production`): SSH s pinovaným `known_hosts`, nahraje `compose.yaml` a `deploy.sh`. `deploy.sh` se přihlásí do GHCR tokenem jobu ze stdin, udělá `pull` a `up -d`, počká na healthy `app` a odhlásí se. Na VPS nezůstává žádný token. Deploy se přeskočí, dokud není nastavená proměnná `VPS_HOST`.
7. **Konfigurace na VPS** je jen `.env` vedle `compose.yaml` (hesla DB a RabbitMQ, tempo a rozsah syncu). Volitelný `compose.override.yaml` CI nepřepisuje (např. připojení `app` do sítě Traefiku).
8. **Bootstrap historie** se spouští ručně jednou (`docker compose exec worker node dist/cbf.js bootstrap`). Automaticky při deployi ne, aby se 1–2 týdny dlouhý crawl nezařazoval opakovaně.

## Zvažované alternativy

- **Build image přímo na VPS** (git pull + `docker compose build`): bez registru, ale build na produkčním stroji, pomalejší deploy a horší rollback. GHCR tagy podle commitu dávají rollback jedním příkazem.
- **Watchtower / pull z VPS:** bez SSH klíče v GitHubu, ale bez spárování s výsledkem CI a bez čekání na healthcheck.
- **Cron na hostu pro `daily` / `window`:** jedna služba méně, ale konfigurace mimo repo. Služba `scheduler` se nasazuje stejně jako zbytek.
- **Migrace při startu aplikace:** jednodušší, ale závod při víc replikách a míchání odpovědností. Samostatná služba `migrate` je explicitní a padne viditelně.
- **Traefik / nginx v compose:** uživatel už reverse proxy má.

## Důsledky

- \+ Push do `master` → během několika minut otestovaný image na VPS. Rollback přes `IMAGE_TAG=<sha> docker compose up -d`.
- \+ Smoke test v CI ověří produkční image včetně chování za proxy.
- − Migrace jsou jen dopředné. Rollback na starší image předpokládá, že starý kód zvládne novější schéma.
- − Deploy klíč má přístup do skupiny `docker`, tedy root-ekvivalent na VPS.
- − Postgres a RabbitMQ běží na stejném stroji bez replikace. Zálohy jsou `pg_dump` z cronu (popsané v `deploy/README.md`).

## Otevřené otázky

- Off-site zálohy a monitoring (uptime check na `/healthz`, alert na hloubku dead-letter front).
- Až bude aplikace přístupná dalším lidem: vlastní přihlášení místo basic auth (PDR 0001, ADR 0006).
