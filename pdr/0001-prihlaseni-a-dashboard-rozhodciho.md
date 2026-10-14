# 0001 – Přihlášení výběrem rozhodčího a dashboard v1

- **Status:** Accepted (bod 1, přihlášení bez hesla, nahrazuje [ADR 0008](../adr/0008-ucty-google-oauth-schvalovani.md))
- **Datum:** 2026-10-14
- **Související:** [ADR 0006](../adr/0006-session-a-parovani-uctu.md), [ADR 0004](../adr/0004-db-schema-ingestu.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Problém / příležitost

Rozhodčí (a komisaři) chtějí vidět své zápasy a statistiky napříč sezónami bez procházení webu ČBF zápas po zápasu. ČBF nemá endpoint „zápasy rozhodčího“, aplikace ho staví z vlastní DB (ADR 0001).

## Rozhodnutí

1. **Přihlášení = spárování se záznamem rozhodčího.**
   - Uživatel vyhledá sám sebe podle jména (bez ohledu na diakritiku a pořadí slov) nebo podle ID: v2 ID nebo kanonické/v1 ID.
   - U kandidáta vidí počet zápasů a datum posledního zápasu, aby rozlišil jmenovce.
   - Tlačítkem „To jsem já“ vznikne (nebo se použije) účet navázaný na `official.id` a otevře se session.
   - **Bez hesla.** Uživatel (2026-10-14): aplikace nebude veřejně vystavená. Přihlásit se tak může kdokoli za kohokoli, což je pro interní použití přijatelné. Silnější ověření (heslo, e-mail, kód) přijde, až bude aplikace přístupná dalším lidem.
2. **Osoba = všechny záznamy se stejným `group_id`.** Statistiky sčítají všechny official záznamy jedné kanonické osoby (ADR 0004).
3. **Dashboard v1:**
   - Celkový počet zápasů (hero číslo).
   - Dlaždice: aktuální sezóna, jako rozhodčí, jako komisař, nadcházející delegace, počet sezón a soutěží.
   - **Graf počtu zápasů v čase:** skládané sloupce rozhodčí / komisař, přepínač sezóny (vše), měsíce (posledních 24) a týdny (posledních 26). Nadcházející delegace jsou světlejší. Data jsou i v tabulkové podobě.
   - Nejbližší delegace a poslední zápasy (termín, soutěž, týmy, skóre, hala, role, kolegové).
   - Nejčastější soutěže (soutěž × sezóna) a nejčastější kolegové.
4. **Další krok (zadání uživatele):** sazebníky soutěží, tedy odměny za zápas podle soutěže a role, a z nich výdělek na dashboardu.

## Zvažované alternativy

- **Přihlášení e-mailem / heslem hned teď:** ČBF export e-maily rozhodčích neobsahuje, takže by chyběl způsob, jak účet ověřit. Pro interní použití zbytečné.
- **Napojení na přihlášení ČBF:** neexistuje veřejné API.

## Dopady

- Data jsou čitelná pro každého, kdo se k aplikaci dostane. Aplikace proto nesmí být veřejně dostupná, dokud nebude silnější přihlášení.
- Dashboard čte jen z normalizovaných tabulek. Rozhodčí jsou jen v `game.php`, takže úplnost závisí na stažení detailů zápasů (bootstrap).

## Metriky / jak poznáme úspěch

- Rozhodčí se najde a spáruje na první pokus.
- Čísla na dashboardu sedí s tím, co rozhodčí zná ze svého přehledu delegací.
