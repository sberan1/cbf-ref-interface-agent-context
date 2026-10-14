# 0004 – DB schéma ingestu (raw + normalizovaná vrstva)

- **Status:** Proposed
- **Datum:** 2026-10-14 (upraveno týž den po discovery)
- **Související:** [ADR 0001](0001-cbf-data-ingestion-strategy.md) (zpřesňuje bod 2 a 3), [ADR 0002](0002-stack-sveltekit-drizzle-postgres.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Kontext

ADR 0001 určilo dvouvrstvé úložiště a historii delegací. Uživatel chce „ukládat všechno do databáze“. Mapování polí původně vycházelo z dokumentace a z kódu klientů `jknetl/CzechBasketballSchedule` a `ivosu/cbf_ical`. Od 2026-10-14 je **ověřené na živých vzorcích** z discovery (sezóny 2026/27 a 2012/13, viz [history](../history/20261014-scaffold-cbf-ingestu.md)) a schéma je podle toho upravené. Payloady chodí v libovolném pořadí (fronta, retry, re-normalizace).

## Rozhodnutí

Implementace je v `src/lib/server/db/schema/{ingest,cbf}.ts`, migrace `drizzle/0000_init_cbf_ingest.sql`.

### Postgres schéma `ingest` (raw vrstva)

- `raw_payload`: tělo každé odpovědi beze změny (text, dekódovaný podle charsetu). Jeden řádek je jedna **verze obsahu** jednoho zdroje, unikátní je dvojice (`request_key`, `content_hash` sha256). Opakované stažení stejného obsahu jen zvýší `fetch_count` a posune `last_fetched_at`, takže historie změn zůstane zachovaná.
- `fetch_state`: jeden řádek na zdroj (`request_key`, např. `v2:api/game.php?game=273093`). Drží poslední payload a hash, `next_due_at` (NULL = automaticky už neobnovovat), počet chyb, kontext úlohy (sezóna, soutěž, …) a stav normalizace (`normalized_raw_payload_id`, `normalizer_version`, `last_normalize_error`).
- `fetch_log`: append-only audit každého HTTP requestu (čas, trvání, status, výsledek). Slouží jako doklad šetrného chování vůči ČBF.

### Postgres schéma `cbf` (normalizovaná vrstva)

`season`, `area`, `category`, `competition`, `phase`, `club`, `team`, `competition_team`, `venue`, `game`, `game_score_quarter`, `official`, `game_official`, `game_official_history`, `standing`, `player`, `team_player`, `player_game_stats`, `game_team_stats`.

Pravidla:

1. **PK jsou v2 ID ČBF.** Ověřeno discovery:
   - v1 a v2 sdílejí ID soutěží, fází a zápasů.
   - **Osoby a týmy mají ve v1 jiná ID.** v1 `sched.php` používá „group“ ID: rozhodčí `<ref><id>` = v2 `uNgroup`, tým `<team><id>` = v2 `tagroupby` / `groupby`. Group ID je kanonická identita osoby nebo týmu napříč sezónami. Ukládá se do `official.group_id`, `player.group_id` a `team.group_id`.
   - v2 ID osob (`uNid`, `commisarid`, `IDuser`) jsou stabilní napříč sezónami. Ověřeno na rozhodčím i komisaři, kteří mají stejné v2 ID v sezóně 2012/13 i 2026/27.
   - Tým má v2 ID na sezónu (`IDteam`), napříč sezónami ho spojuje `group_id`.
   - Venue je klíčované normalizovaným `name|city` (jediné, co je ve všech zdrojích). `IDhall` z detailu zápasu se ukládá do `venue.hall_id`.
2. **FK jen uvnitř agregátu** zapisovaného v jedné transakci (`game` → `game_score_quarter`, `game_official`, `player_game_stats`; `team_player` → `player`; `game_official` → `official`). Vazby mezi agregáty (game → phase → competition → season, game → team) jsou indexované sloupce bez FK, protože payloady přicházejí v libovolném pořadí. Chybějící „rodiče“ se zakládají jako stuby (team, season, area, category).
3. **Nic se neztrácí:** `extra` (jsonb) drží celý zdrojový záznam z v2, `extra_v1` záznam z v1. Typované sloupce jde rozšiřovat bez opětovného stahování.
4. **Priorita zdrojů:** v2 (`games.php`, `game.php`) je autoritativní, novější pozorování vyhrává (`game.observed_at`, starší payload nikdy nepřepíše novější). v1 (`sched.php`, `divs.php`) jen doplňuje chybějící hodnoty (`coalesce`). U soutěží zapisuje flagy (`competition.flags`), kategorii a ligu. v1 nikdy nezapisuje ID týmů ani delegace (jiný ID prostor). Jeho delegace slouží jen k doplnění `official.group_id` přes stejný zápas a roli, což potřebují hlavně komisaři, protože v2 pro ně group ID neposílá.
5. **Delegace:**
   - `game_official`: aktuální stav, PK (`game_id`, `role`), role `referee1..3` a `commissioner`.
   - Rozhodčí jsou **jen v `game.php`**. `games.php` nese jen komisaře.
   - Role chybí v payloadu (klíč `uNid` neexistuje) → stav neznámý, nic se nemění.
   - ID 0 **se jménem** (ČBF posílá „Delegováno“ před zveřejněním delegace) → také neznámý stav.
   - ID 0 bez jména → role je neobsazená. U autoritativního zdroje se tím delegace odebere.
   - `game_official.ratings` (jsonb): hodnocení rozhodčího domácími, hosty a komisařem (`rating_uN_home|guest|commisar`), ve vzorcích zatím vždy prázdné.
   - `game_official_history`: změny `assigned` / `replaced` / `removed` se zdrojem, časem pozorování a `raw_payload_id`. První obsazení dané role v zápase má `is_initial = true` (nejde o skutečnou změnu, typicky zveřejnění delegace).
6. **Re-normalizace:** každý normalizer má `version`. Když se mapování změní, zvýší se verze, nebo se spustí `npm run cbf -- renormalize --endpoint … --force`. Data se přepočítají z `raw_payload` bez HTTP requestů.
7. **Statistiky:**
   - `player_game_stats` a `game_team_stats` z `game.php` (`playerStats`, `teamStats`).
   - `team_player` ze `teamplayers.php` (`playerstatcount`, včetně sezónních statistik).
   - Statistiky zůstávají jako jsonb s kódy ČBF (`ZS2PP` = dvojky proměněné, `ZS2PM` = minuté, `ZS1P` / `ZS1M` trestné, `ZS3P` / `ZS3M` trojky, `ZRD` / `ZRO` doskoky, `XSP` = odehrané sekundy, `PTS`, `VAL`, `PLUSMINUS`…). Význam kódů je odvozený ze součtů bodů, není oficiálně zdokumentovaný.
8. **Skóre po čtvrtinách:** ČBF posílá v `score_quarter` i ve v1 `<partials>` **průběžný stav** po Q1–Q3 (při prodloužení i po Q4), konečné skóre zvlášť. `game_score_quarter` drží body za období. Všechna prodloužení jsou společně v období 5, protože ČBF je nerozlišuje.
9. Kontakty funkcionářů klubu z `team.php` (`contact`: data narození, adresy, telefony) se **nenormalizují**. Zůstávají jen v raw vrstvě.

## Zvažované alternativy

- **Vlastní surrogate PK + mapovací tabulky na ID ČBF:** po discovery zbytečné. v2 ID jsou stabilní a rozdílný ID prostor v1 pokrývá sloupec `group_id`.
- **Kanonické group ID jako PK osob:** v2 ho pro komisaře neposílá (zná ho jen v1 přes stejný zápas), takže by komisaře nešlo zapsat hned z v2. Zůstává jako sloupec.
- **Striktní FK všude:** vyžadovalo by pevné pořadí zpracování a selhávalo by při retry a re-normalizaci.
- **Raw payloady na disku / v S3:** zatím zbytečné. TOAST komprese Postgresu stačí, odhad je jednotky GB za 20 sezón.
- **Jen poslední verze payloadu:** ztratili bychom možnost zpětně dohledat změny (delegace, termíny).

## Důsledky

- \+ Úplná historie obsahu a změn delegací, re-normalizace bez zátěže ČBF.
- \+ Odolné vůči pořadí zpráv a duplicitám (idempotentní upserty).
- − Referenční integrita mezi agregáty není vynucená DB. Kontrolovat ji reportovacími dotazy.
- − Jsonb sloupce `extra` zdvojují část dat (raw + extra). Při problému s místem jde `extra` vypustit.

## Otevřené otázky

- ~~Sdílejí v1 a v2 ID prostor?~~ Jen soutěže, fáze a zápasy, osoby a týmy ne (viz bod 1).
- ~~Je ID rozhodčího stabilní napříč sezónami?~~ Ano, v2 ID i group ID (ověřeno na vzorcích). Po bootstrapu potvrdit dotazem nad celými daty: jedno `official.id` s víc `group_id` nebo naopak.
- ~~Vrací `divs.php` pro `area=0` i regionální soutěže?~~ Ne, oblasti se nepřekrývají a záznam nese `area`.
- ~~Struktura `teamplayers.php` a `game.php`?~~ Namapováno (bod 7). Statistiky zatím jen jako jsonb, typované sloupce podle potřeb UI.
- Formát hodnocení rozhodčích (`rating_*`), ve vzorcích vždy NULL.
- **Osobní údaje:** export obsahuje jména rozhodčích a hráčů (v mládeži i nezletilých, s výškou), u týmů i kontakty funkcionářů s datem narození a adresou. Ty jsou v raw vrstvě. Před zpřístupněním aplikace dalším lidem vyjasnit právní titul a rozsah (GDPR), případně raw payloady `team.php` čistit nebo hráče mládeže pseudonymizovat.
