# 0002 – Odměny rozhodčích na dashboardu

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [PDR 0001](0001-prihlaseni-a-dashboard-rozhodciho.md), [ADR 0004](../adr/0004-db-schema-ingestu.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Problém / příležitost

Rozhodčí dostává za každý odřízený zápas odměnu podle soutěže. Chce vidět, kolik vydělal celkem, za sezónu a v čase, a kolik ho čeká z nadcházejících delegací. Uživatel dodal přehledy odměn ČABR za sezóny 2023/24, 2025/26 a 2026/27 (verze 2 a 3) a zápis VV ČABR z 22. 9. 2026 se zvýšením ligových odměn od 1. 1. 2027. Pro Prahu 2017–2020 doplnil údaje sám: minižactvo 300 Kč, 3. třída 400 Kč, každá vyšší soutěž o 100 Kč víc, stejný vzor jako v posledních letech.

## Rozhodnutí

1. **Počítá se jen odměna za řízení utkání.** Cestovné, stravné a příplatky za všední den závisí na bydlišti rozhodčího, které aplikace nezná. Na dashboardu je to uvedené.
2. **Komisaři odměnu nemají.** Přehledy odměny komisařů neobsahují.
3. **Sazebníky jsou data v kódu** (`src/lib/server/fees/schedules.ts`), ne tabulky v DB. Každý má rozsah (ČBF, Český pohár, Praha), platnost `[od, do)` podle data zápasu a původ:
   - **`document`:** z přehledu odměn nebo zápisu.
   - **`reported`:** od rozhodčího (Praha 2017–2020).
   - **`assumed`:** odvozeno.

   Sazba se vybírá podle data zápasu, ne podle sezóny, protože zvýšení od 1. 1. 2027 platí uprostřed sezóny. Změna sazebníku tedy znamená commit a deploy, který je automatický (ADR 0007).
4. **Zařazení soutěže** (`classify.ts`) se řídí:
   - oblastí (0 = celostátní, 1 = Praha),
   - pohlavím,
   - věkovou kategorií (bere se z názvu kategorie, protože starší sezóny mají soutěže pojmenované jen „extraliga“ / „liga“),
   - názvem soutěže a ligy,
   - fází (U15/U14: extraliga vs. ostatní fáze),
   - kolem (Český pohár, Pražský pohár).

   Nejisté případy jsou označené jako odhad:
   - kvalifikace o extraligu U19/U17 = celostátní liga,
   - kvalifikace o ligu U19/U17 = NRL dorost,
   - kvalifikace U15/U14 = ostatní fáze,
   - neznámé kolo poháru = 1. kolo,
   - „Přebor“ bez skupiny = Přebor A.
5. **Bez sazby** (zápas se nezapočte a je vidět v seznamu „Soutěže bez sazby“):
   - NBL a ŽBL,
   - reprezentace, 3x3 a MČR mládeže,
   - kvalifikace a baráže dospělých,
   - ostatní oblasti (sazebníky v přehledech jsou, ale mapování názvů zatím není),
   - Český pohár před 2025/26,
   - cokoli před 2017/18.
6. **Jeden rozhodčí = 1,5násobek.** Platí jen u odehraných zápasů s jediným delegovaným rozhodčím a jen kde to přehled stanoví:
   - Praha kromě minižactva,
   - ligy ČBF od 2025/26.

   U budoucích zápasů může být druhý rozhodčí skrytý („Delegováno“), proto se tam 1,5násobek nepočítá.
7. **Přiřazení sazeb k obdobím:**

   | Rozsah | Období | Zdroj |
   | --- | --- | --- |
   | Praha | 2017/18–2019/20 | `reported` |
   | Praha | 2020/21–2022/23 | `assumed` (stejné hodnoty jako před i po) |
   | Praha | 2023/24 | `document` |
   | Praha | 2024/25 | `assumed` = 2023/24 |
   | Praha | 2025/26 | `document` |
   | Praha | 2026/27 | `document`, verze 3 z 30. 9. 2026 (+100 Kč), platná celou sezónu |
   | ČBF ligy a NRL | 2017/18–2022/23 | `assumed` = 2023/24 |
   | ČBF ligy a NRL | 2023/24–31. 12. 2026 | `document` (2024/25 odvozeno, okolní sezóny jsou shodné) |
   | ČBF ligy a NRL | od 1. 1. 2027 | `document` podle zápisu; NRL beze změny |
   | Český pohár mužů | od 2025/26 | `document` |
   | Český pohár žen | od 2026/27 | `document` |

8. **UI:**
   - Blok „Odměny za řízení“: vyděláno celkem, aktuální sezóna, očekáváno z nadcházejících delegací, počet zápasů bez sazby.
   - Tabulka po sezónách (vyděláno / z toho odhad / očekáváno / bez sazby) a rozbalovací seznam soutěží bez sazby s důvodem.
   - Graf má přepínač „Zápasy / Odměny“. U odměn je očekávaná část aktuálního období tlumená nad vydělanou.
   - V seznamech zápasů je u rozhodčího částka, s „≈“, když jde o odhad.

## Zvažované alternativy

- **Sazebníky v DB s editací v UI:** pohodlnější změny, ale sazebníky se mění jednou za sezónu a mapování soutěží je stejně kód. Lze doplnit později.
- **Počítat i příplatky za všední den:** vyžaduje bydliště rozhodčího a pravidla per oblast. Odloženo.

## Otevřené otázky

- Praha 2024/25: platily sazby 2023/24 (předpoklad), nebo už sazby 2025/26?
- Platí pražská verze 3 (+100 Kč) od začátku sezóny 2026/27, nebo až od jejího vydání 30. 9. 2026?
- Sazebníky ČBF a Českého poháru před 2023/24 a mapování ostatních oblastí (názvy soutěží se mezi sezónami liší).
- Ověřit na produkčních datech, jak se jmenují fáze žákovské ligy (extraliga) a kola pohárů v `game.round`.
