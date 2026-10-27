# 0005 – Vyúčtování po dnech

- **Status:** Accepted
- **Datum:** 2026-10-27
- **Související:** [PDR 0002](0002-odmeny-rozhodcich.md), [PDR 0003](0003-cestovne-stravne-vsedni-den.md), [PDR 0004](0004-vlastni-zapasy-dane-filtry.md)

## Problém / příležitost

Požadavek uživatele z 2026-10-27: obrazovka, která pro každý den, kdy rozhodčí píská, ukáže, kolik peněz má vypsat a jak rozdělené. Má to být na jedno kliknutí a přehledné.

Odměny jsou na přehledu (PDR 0002) a náhrady v Cestách (PDR 0003). Při vyplňování výplatního formuláře je ale potřeba mít obojí pohromadě, po dnech a po položkách.

## Rozhodnutí

Uživatel vybral z navržených variant **karty po dnech** s výběrem měsíce. Vstup je položka „Vyúčtování“ v hlavním menu (`/payouts`).

1. **Měsíc** se vybírá parametrem `?m=YYYY-MM`. Nabízejí se jen měsíce, kdy má osoba zápas ČBF nebo vlastní zápas.
   - Výchozí je aktuální měsíc, jinak poslední měsíc, který už začal.
   - Nahoře je součet měsíce: celkem, odměny, náhrady.
   - Upozornění ukazuje, u kolika dnů chybí spočítaná cesta. Takové dny mají v součtu jen odměny.
2. **Karta dne** (od nejnovějšího):
   - **Zápasy:** týmy, role, soutěž s kategorií a hala. Vlastní zápasy jsou označené „vlastní“.
   - **Položky:**
     - odměna za každý zápas, při více zápasech popsaná soutěží;
     - cestovné jako „X km × sazba + Y km × sazba s kolegou“, u veřejné dopravy jízdenky;
     - parkovné a jiné výdaje;
     - stravné s délkou dne nebo s důvodem, proč nevzniká;
     - náhrada za všední den.
   - **Bez částky:** položka bez částky ukazuje důvod (chybí sazebník, zápas bez výsledku, komisař).
   - **Celkem:** součet dne, ≈ u odhadu sazeb.
   - **Budoucí dny** mají štítek „nadcházející“.
3. **Kopírování:**
   - u každé položky tlačítko „Kopírovat“, které zkopíruje holé číslo pro vložení do formuláře;
   - „Kopírovat vše“ zkopíruje textový souhrn dne.
4. **Den bez spočítané cesty** má místo náhrad odkaz „Spočítat cestu“ na průvodce `/trips/<datum>`. Den s cestou má tlačítko „Upravit cestu“.
5. **Výpočty se nezdvojují.** Obrazovka skládá existující výpočty: odměny z `personFees` (PDR 0002, včetně jednodenní tolerance před zrušením) a náhrady z `personDays` (PDR 0003).

## Zvažované alternativy

- **Tabulka měsíce** (řádek = den, sloupce = položky): kompaktní, ale na mobilu nečitelná a bez prostoru pro rozpad km a důvody.
- **Detail dne ze seznamu:** o kliknutí víc na každý den, uživatel chtěl vše na jedno kliknutí.
- **Vyúčtování přímo na přehledu:** přehled je už dlouhý. Samostatná stránka v menu je dostupná stejně rychle.

## Dopady

- \+ Při vyplňování výplaty stačí otevřít měsíc a postupně kopírovat čísla.
- \+ Žádná nová data ani migrace. Vše vychází z odměn a uložených cest.
- − U dnů bez cesty ukazuje jen odměny, dokud uživatel cestu nespočítá.
- − Vlastní zápasy nemají náhrady (stejné omezení jako v PDR 0004).

## Metriky / jak poznáme úspěch

- Uživatel vyplní výplatu za měsíc bez přepínání mezi přehledem a cestami.
