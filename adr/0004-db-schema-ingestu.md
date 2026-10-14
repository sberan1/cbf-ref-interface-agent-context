# 0004 – DB schéma ingestu (raw + normalizovaná vrstva)

- **Status:** Proposed
- **Datum:** 2026-10-14
- **Související:** [ADR 0001](0001-cbf-data-ingestion-strategy.md) (zpřesňuje bod 2 a 3), [ADR 0002](0002-stack-sveltekit-drizzle-postgres.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Kontext

ADR 0001 určilo dvouvrstvé úložiště a historii delegací. Uživatel chce „ukládat všechno do databáze“. Skutečné schéma exportu není ověřené (sandbox nemá přístup na `www.cbf.cz`), mapování polí proto vychází z dokumentace a z kódu klientů `jknetl/CzechBasketballSchedule` a `ivosu/cbf_ical`. Payloady chodí v libovolném pořadí (fronta, retry, re-normalizace).

## Rozhodnutí

Implementace je v `src/lib/server/db/schema/{ingest,cbf}.ts`, migrace `drizzle/0000_init_cbf_ingest.sql`.

### Postgres schéma `ingest` (raw vrstva)

- `raw_payload`: tělo každé odpovědi beze změny (text, dekódovaný podle charsetu). Jeden řádek je jedna **verze obsahu** jednoho zdroje, unikátní je dvojice (`request_key`, `content_hash` sha256). Opakované stažení stejného obsahu jen zvýší `fetch_count` a posune `last_fetched_at`, takže historie změn zůstane zachovaná.
- `fetch_state`: jeden řádek na zdroj (`request_key`, např. `v2:api/game.php?game=273093`). Drží poslední payload a hash, `next_due_at` (NULL = automaticky už neobnovovat), počet chyb, kontext úlohy (sezóna, soutěž, …) a stav normalizace (`normalized_raw_payload_id`, `normalizer_version`, `last_normalize_error`).
- `fetch_log`: append-only audit každého HTTP requestu (čas, trvání, status, výsledek). Slouží jako doklad šetrného chování vůči ČBF.

### Postgres schéma `cbf` (normalizovaná vrstva)

`season`, `area`, `category`, `competition`, `phase`, `club`, `team`, `competition_team`, `venue`, `game`, `game_score_quarter`, `official`, `game_official`, `game_official_history`, `standing`, `player`, `team_player`, `player_game_stats`.

Pravidla:

1. **PK jsou ID ČBF** všude, kde ČBF nějaké ID má. Předpoklad (neověřeno): v1 a v2 sdílejí ID prostor, protože fáze z v2 se používají jako `p=` ve v1. Venue nemá ID, klíčem je normalizované `name|street|city`.
2. **FK jen uvnitř agregátu** zapisovaného v jedné transakci (`game` → `game_score_quarter`, `game_official`, `player_game_stats`; `team_player` → `player`; `game_official` → `official`). Vazby mezi agregáty (game → phase → competition → season, game → team) jsou indexované sloupce bez FK, protože payloady přicházejí v libovolném pořadí. Chybějící „rodiče“ se zakládají jako stuby (team, season, area, category).
3. **Nic se neztrácí:** `extra` (jsonb) drží celý zdrojový záznam z v2, `extra_v1` záznam z v1. Typované sloupce jde rozšiřovat bez opětovného stahování.
4. **Priorita zdrojů:** v2 (`games.php`, `game.php`) je autoritativní, novější pozorování vyhrává (`game.observed_at`, starší payload nikdy nepřepíše novější). v1 (`sched.php`, `divs.php`) jen doplňuje chybějící hodnoty (`coalesce`), u soutěží zapisuje flagy (`competition.flags`) do `extra_v1`.
5. **Delegace:**
   - `game_official`: aktuální stav, PK (`game_id`, `role`), role `referee1..3` a `commissioner`.
   - Role chybí v payloadu (klíč `uNid` neexistuje) → stav neznámý, nic se nemění. Klíč existuje s ID 0 nebo prázdný → role je neobsazená. U autoritativního zdroje se tím delegace odebere.
   - `game_official_history`: změny `assigned` / `replaced` / `removed` se zdrojem, časem pozorování a `raw_payload_id`. První pozorování zápasu má `is_initial = true` (nejde o skutečnou změnu).
6. **Re-normalizace:** každý normalizer má `version`. Když se mapování změní, zvýší se verze, nebo se spustí `npm run cbf -- renormalize --endpoint … --force`. Data se přepočítají z `raw_payload` bez HTTP requestů.
7. `teamplayers.php` a statistiky z `game.php` mají **provizorní** mapování: `team_player.stats` je celý záznam, `player_game_stats` se zatím neplní. Typované sloupce přibudou po discovery.

## Zvažované alternativy

- **Vlastní surrogate PK + mapovací tabulky na ID ČBF:** robustnější, kdyby se ukázalo, že v1 a v2 ID nesdílejí. Za cenu složitějších upsertů a joinů. Vrátit se k tomu, pokud to discovery vyvrátí.
- **Striktní FK všude:** vyžadovalo by pevné pořadí zpracování a selhávalo by při retry a re-normalizaci.
- **Raw payloady na disku / v S3:** zatím zbytečné. TOAST komprese Postgresu stačí, odhad je jednotky GB za 20 sezón.
- **Jen poslední verze payloadu:** ztratili bychom možnost zpětně dohledat změny (delegace, termíny).

## Důsledky

- \+ Úplná historie obsahu a změn delegací, re-normalizace bez zátěže ČBF.
- \+ Odolné vůči pořadí zpráv a duplicitám (idempotentní upserty).
- − Referenční integrita mezi agregáty není vynucená DB. Kontrolovat ji reportovacími dotazy.
- − Jsonb sloupce `extra` zdvojují část dat (raw + extra). Při problému s místem jde `extra` vypustit.

## Otevřené otázky

- Sdílejí v1 a v2 ID prostor (fáze, zápasy, osoby)? → discovery.
- Je ID rozhodčího stabilní napříč sezónami (`official.id`)? → dotaz nad daty po bootstrapu, např. stejné ID s různými jmény / stejné jméno s různými ID.
- Vrací `divs.php` pro `area=0` i regionální soutěže? Pak by se `competition.area_id` přepisovalo podle pořadí zpracování.
- Struktura `teamplayers.php` a statistické části `game.php`.
- **Osobní údaje:** export obsahuje jména rozhodčích a hráčů, v mládežnických soutěžích i nezletilých. Před zpřístupněním aplikace dalším lidem vyjasnit právní titul a rozsah (GDPR), případně hráče mládeže neukládat nebo pseudonymizovat.
