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

## Login a dashboard rozhodčího (třetí část konverzace)

Zadání: „hodnocení neřeš, osobní informace ukládej, nebudu to exposovat. Udělej základní login, kde si rozhodčí vybere sebe (svoje ID) a spáruje se k danému rozhodčímu. Po přihlášení jednoduchý dashboard s pár statistikami (graf s počtem zápasů za sezónu / týden / měsíc v čase). Pak přidám sazebníky soutěží.“

- **Rozhodnutí:** [PDR 0001](../pdr/0001-prihlaseni-a-dashboard-rozhodciho.md) (přihlášení výběrem bez hesla, obsah dashboardu v1) a [ADR 0006](../adr/0006-session-a-parovani-uctu.md) (schéma `app`, session v DB, hash tokenu, klouzavých 30 dní).
- **ADR 0004:** hodnocení rozhodčích se nenormalizuje. Osobní údaje se ukládají celé, kontakty z `team.php` → `cbf.team_contact`. GDPR otázka je uzavřená rozhodnutím uživatele.
- **SvelteKit 3:** `$lib` alias už není, používá se `#lib/...` (package.json `imports`) s explicitní příponou `.ts`. `Handle` je v `@sveltejs/kit/hooks`. Odkazy jdou přes `resolve()` z `$app/paths` (vyžaduje lint).
- **Graf:** vlastní SVG komponenta (skládané sloupce, legenda, tooltip na hover i focus, tabulková verze). Paleta: referenční paleta dataviz skillu (slot 1 modrá = rozhodčí, slot 2 oranžová = komisař), ověřená validátorem ve světlém i tmavém režimu.

## CI/CD a nasazení (čtvrtá část konverzace)

Zadání: „Připrav CI/CD skript a řekni mi, co mám nastavit. Poběžím to na svém VPS v Dockeru. Před tím mergni momentální PR a začni na novém.“ Odpovědi uživatele: vlastní nginx/Traefik, basic auth si udělá sám v nginx, deploy automaticky po pushi do `master`.

- PR #1 (scaffold, login, dashboard) je mergnutý do `master`. CI/CD je v novém PR.
- **Rozhodnutí:** [ADR 0007](../adr/0007-nasazeni-docker-vps.md): `adapter-node`, jeden image pro všechny role, `deploy/compose.yaml`, služby `migrate` a `scheduler`, GitHub Actions (check → image + smoke test → GHCR → deploy přes SSH).
- **adapter-node 6 nemá `ORIGIN`:** origin odvozuje z `Host` a `X-Forwarded-Proto`. Bez těchto hlaviček z nginx login padá na 403 (CSRF). Ověřeno curlem proti `node build` i smoke testem image.
- **Nalezeno při testu:** `docker stop` aplikaci zabil (exit 137), protože ji naživu držel idle pool Postgresu. Pool se teď zavře na `sveltekit:shutdown`, ukončení trvá asi 1 s. RabbitMQ potřebuje pevný `hostname`, jinak by přetvoření kontejneru ztratilo fronty.
- Ověřeno lokálně:
  - Docker build.
  - Smoke test.
  - Celý `deploy/compose.yaml` stack (migrate → app healthy, worker a scheduler se připojí, `stats` přes `docker compose run`).
  - `deploy.sh` s podvrženým `docker login`.
  - Čisté ukončení workeru a scheduleru na SIGTERM.
  - Drift check migrací.

## První nasazení na VPS (pátá část konverzace)

- **SSH:** deploy job nejdřív selhal na `Permission denied (publickey)`. Na VPS byl `/home/deploy/.ssh/authorized_keys` adresář místo souboru (sshd: „is not a regular file“). Po opravě deploy prošel. Všechny služby jsou healthy a worker stahuje sezónu 2026 tempem ~1 request / 2,7 s. `bootstrap` zařadil 652 kořenových úloh (2006–2026).
- **Drobnosti z provozu:**
  - Při studeném startu se worker jednou restartuje (`ECONNREFUSED` na 5672). Healthcheck RabbitMQ (`ping`) projde dřív, než listener přijímá spojení.
  - Několik `network fetch failed` během prvních minut. Retry je vyřešil, dead-letter fronty zůstaly prázdné.
- **Reverse proxy:** uživatel používá Nginx Proxy Manager v Dockeru (síť `nginx_default`). `app` se do ní připojuje přímo z `compose.yaml` ([ADR 0007](../adr/0007-nasazeni-docker-vps.md)), NPM míří na `cbf-ref-app:3000`. Odkud přišla 403 při prvním nastavení NPM, zatím ověřené není (access list NPM, nebo kontrola originu v aplikaci). Možné příčiny popisuje `deploy/README.md`.

## Odměny rozhodčích (šestá část konverzace)

Zadání: „Za zápasy jsou peníze, pošlu ti pár recentních sazebníků, udělej tam výpočet získaných výplat podle soutěže. Co tam chybí: mezi lety 2017 a 2020 bylo za pražské soutěže mini 300 a 3. třída 400 a každá vyšší soutěž byla vždy o 100 Kč dražší, pattern je stejný jako poslední roky.“ Podklady:
- přehledy odměn ČABR 2023/24, 2025/26, 2026/27 (verze 2 a 3),
- zápis VV ČABR z 22. 9. 2026 (zvýšení ligových odměn od 1. 1. 2027).

- **Rozhodnutí:** [PDR 0002](../pdr/0002-odmeny-rozhodcich.md). Sazebníky jsou v kódu s platností podle data zápasu a s původem (document / reported / assumed). Počítá se jen odměna za řízení, ne cestovné, stravné ani příplatky. Pokryté jsou celostátní soutěže, NRL, Český pohár a Praha.
- **Z přehledů:**
  - Pražské sazby rostly: 2023/24 → 2025/26 o +100 Kč (kromě minižactva), verze 3 pro 2026/27 o dalších +100 Kč.
  - Ligy ČBF a NRL jsou 2023/24–2026 beze změny. Ligy se zvedají od 1. 1. 2027, NRL se nemění.
  - Přehledy neuvádějí odměny pro NBL, ŽBL ani komisaře.
- **Ověřeno:**
  - Unit testy pokrývají sazby z PDF, zařazení soutěží podle reálných názvů z discovery, 1,5× pro jednoho rozhodčího a přechod 1. 1. 2027.
  - UI jsem prošel v Playwrightu nad syntetickými daty (vymyšlený rozhodčí, 77 zápasů 2016–2027), ve světlém i tmavém režimu a na mobilu.
- **Doplněno uživatelem:** Praha 2024/25 = sazby 2023/24. Pražské zvýšení z verze 3 platí od začátku sezóny 2026/27.

## Účty a náhrady (sedmá část konverzace)

Zadání:
- „Pořádný login: každý si založí účet a spáruje ho se sebou, robustně přes OAuth, pro můj administrátorský účet možnost schvalovat účty.“
- „K výplatám patří stravné a cestovné: průvodce pro každý den (kde vyzvednu kolegu, kde ho vysadím), adresa haly ze systému, výpočet km, sám 4,50, s kolegou 5 Kč/km, stravné 155 Kč nad 5 h (3 h 30 min na zápas + cesta), za zápasy ČBF ve všední den +300 Kč.“
- Nové PR postavené nad #3.
- Volby uživatele: jen Google, vzdálenosti z Mapy.com.

- **Rozhodnutí:**
  - [ADR 0008](../adr/0008-ucty-google-oauth-schvalovani.md): `arctic`, Google OIDC s PKCE, `ADMIN_EMAILS`, schvalování spárování na `/admin`.
  - [PDR 0003](../pdr/0003-cestovne-stravne-vsedni-den.md): herní den = cesta, zastávky a úseky přes Mapy.com, sazby s platností, stravné podle kalendářního roku, státní svátky.
- **SvelteKit 3:** `redirect()` na cizí doménu vyžaduje `{ external: [...] }`. Formuláře s `use:enhance` po odeslání resetují inputy, proto `update({ reset: false })`.
- **Ověřeno:**
  - Unit testy: claims ID tokenu, cesta účtu podle stavu, svátky a Velikonoce, sazby, úseky s kolegou, stravné, všední den, parsování odpovědí Mapy.com.
  - Účty proti testovací DB (nový účet, žádost, auto-schválení správce, obsazený rozhodčí, schválení, zrušení, zamítnutí).
  - Přesměrování na Google (redirect URI z `Host` a protokolu).
  - Playwright nad syntetickými daty: výchozí cesty sezóny, průvodce s vyzvednutím kolegy, spolujezdec, správa účtů, profil, mobil bez vodorovného posunu.
  - Bez API klíče Mapy.com se jede přes odhad vzdušnou čarou.

## Další kroky

1. ~~Sazebníky soutěží a výdělek na dashboardu.~~ Hotovo pro celostátní soutěže a Prahu ([PDR 0002](../pdr/0002-odmeny-rozhodcich.md)). Zbývá případně doplnit ostatní oblasti a starší sazebníky ČBF.
2. Spustit `bootstrap` (lokálně nebo na serveru) a průběžně sledovat `stats` a management UI RabbitMQ. Ověřit sezóny 2006–2011.
3. ~~Nastavit cron pro `daily` a `window`, rozhodnout o nasazení.~~ Hotovo: služba `scheduler`, [ADR 0007](../adr/0007-nasazeni-docker-vps.md). Na VPS zbývá jednou spustit `bootstrap` (`deploy/README.md`).
4. Stáhnout DTD specifikace ČBF a porovnat je s mapováním.
5. Typované sloupce statistik podle potřeb UI, výklad kódů (`ZS2DP` / `ZS2KP` …).
6. Zvážit oficiální žádost ČBF o přístup.
7. Nastavit na VPS `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `ADMIN_EMAILS` / `MAPY_API_KEY` a ověřit Mapy.com API na produkci. Export náhrad pro vyúčtování.
