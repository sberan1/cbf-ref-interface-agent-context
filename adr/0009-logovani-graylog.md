# 0009 – Logování do Graylogu

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [ADR 0007](0007-nasazeni-docker-vps.md), [ADR 0008](0008-ucty-google-oauth-schvalovani.md), [PDR 0003](../pdr/0003-cestovne-stravne-vsedni-den.md)

## Kontext

Uživatel chce logy všeho, „třeba do Graylogu“: v compose a s webovým UI ve stejné síti reverse proxy jako aplikace (`nginx_default`, Nginx Proxy Manager). Worker už loguje JSON řádky (`t`, `level`, `msg`, další pole), web dosud jen `console.error`.

## Rozhodnutí

1. **Jeden JSON logger pro všechno** (`src/lib/server/log.ts`, `cbf/log.ts` ho jen re-exportuje): web, worker, scheduler i `graylog-setup`.
2. **Web loguje:**
   - každý request kromě `/healthz` a `/_app/*` (metoda, cesta, route, status, ms, userId) v `handle`,
   - neošetřené chyby se stackem v `handleError` (`kind === 'unknown'`),
   - události: přihlášení, žádost o spárování, rozhodnutí správce, uložení cesty, výpadky Mapy.com.

   Query stringy ani osobní údaje se nelogují (jen ID účtu).
3. **Doprava logů:**
   - Docker logging driver `gelf` (UDP na `127.0.0.1:${GELF_PORT}`, tag = název kontejneru) pro app, worker, scheduler, migrate, Postgres a RabbitMQ.
   - Kontejnery samotného logovacího stacku logují lokálně (`json-file`).
   - Docker dual logging zachovává `docker compose logs`.
   - UDP neblokuje: výpadek Graylogu nezastaví aplikaci.
4. **Stack v compose:**
   - `mongodb` (`mongo:7.0`), `opensearch` (`opensearchproject/opensearch:2.19.1`, single node, bez security pluginu, heap `OPENSEARCH_HEAP`) a `graylog` (`graylog/graylog:6.1`).
   - Root heslo je v `.env` v čitelné podobě (`GRAYLOG_ROOT_PASSWORD`). SHA-256, které Graylog vyžaduje, spočítá entrypoint.
   - Graylog je v síti `proxy` s aliasem `cbf-ref-graylog` (UI na portu 9000) a na loopbacku hostu pro SSH tunel.
5. **Bez klikání v UI:** jednorázová služba `graylog-setup` (`dist/graylog-setup.js` z image aplikace) přes REST API založí:
   - globální vstup *GELF UDP (docker)*,
   - JSON extraktor na pole `message` s prefixem `app_`, aby `level` z JSONu nekolidoval s číselným GELF `level` v mapování OpenSearch.

   Je idempotentní a sama čeká na API.
6. **Logovací stack nesmí blokovat deploy:** závislosti na něm jsou `service_started`, ne `service_healthy`, a aplikace na něm nezávisí vůbec. `deploy.sh` nasadí nejdřív aplikaci (db, rabbitmq, migrate, app, worker, scheduler), logovací stack až potom a jeho selhání (např. limit Docker Hubu) je jen varování. Bez `GRAYLOG_*` v `.env` se přeskočí.
7. **Unikátní DNS aliasy:** aplikace a Graylog sdílejí síť proxy s cizími stacky, kde může existovat `db` nebo `mongodb`. Služby se proto navzájem oslovují aliasy `cbf-ref-db`, `cbf-ref-rabbitmq`, `cbf-ref-mongodb` a `cbf-ref-opensearch` (doplněno po review).

## Ověřeno lokálně

- Celý stack se spustí, `graylog-setup` založí vstup i extraktor a druhé spuštění nic nezdvojí.
- V Graylogu jsou requesty webu (`app_msg:request`, `app_status`, `app_ms`) i varování workeru (`app_err`).
- UI odpovídá ze sítě proxy na `cbf-ref-graylog:9000`.
- Zachycené chyby:
  - `plugins.security.disabled` spolu s `DISABLE_SECURITY_PLUGIN` OpenSearch odmítne (duplicitní nastavení),
  - extraktor API chce `cursor_strategy` a `converters` jako pole.

## Zvažované alternativy

- **Grafana Loki + Promtail/Alloy:** lehčí, ale uživatel chtěl Graylog. Fulltext a pole z JSONu v Graylogu fungují bez dalšího.
- **Graylog Data Node místo OpenSearch:** doporučený pro novější Graylog, ale vyžaduje certifikáty a preflight. Na jeden VPS je jednodušší OpenSearch.
- **GELF přímo z Node:** posílalo by jen logy aplikace. Docker driver pokryje všechny kontejnery včetně Postgresu a RabbitMQ.

## Důsledky

- \+ Jedno místo pro logy všech služeb, s poli pro filtrování (status, trvání, uživatel, chyba).
- − Zhruba 2 GB RAM (heapy 512 MB a tvrdé limity kontejnerů, doplněno 2026-10-14 kvůli malému VPS) a 2–3 GB disku: indexy po 512 MB, drží se 4 nejnovější, journal max. 512 MB. Hodnoty indexů se do Graylogu propíšou při prvním startu, potom se mění v UI.
- − OpenSearch doporučuje `vm.max_map_count=262144` na hostu.
- − Verze jsou připnuté (Graylog 6.1, OpenSearch 2.19, MongoDB 7.0). Upgrade = změna tagů podle kompatibility Graylogu.

## Otevřené otázky

- Alerty v Graylogu (např. `app_level:error`, mrtvé fronty, `error_blocked` od ČBF) a notifikace.
- Retence a velikost indexů podle reálného objemu.
