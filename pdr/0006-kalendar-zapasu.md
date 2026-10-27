# 0006 – Kalendář zápasů (iCal)

- **Status:** Accepted
- **Datum:** 2026-10-27
- **Související:** [PDR 0002](0002-odmeny-rozhodcich.md), [PDR 0004](0004-vlastni-zapasy-dane-filtry.md), [PDR 0005](0005-vyuctovani-po-dnech.md), [ADR 0010](../adr/0010-kalendar-tajna-adresa.md)

## Problém / příležitost

Uživatel má starou službu (`CABR-web-scraper`), která se přihlašuje do systému ČABR, stahuje stránku delegací a dává zápasy do iCal kalendáře. Chce ji vypnout a nahradit kalendářem z této aplikace. Podmínka: kalendář musí obsahovat všechno o zápase.

Stará služba dávala:

- název „týmy - kategorie“;
- halu jako místo;
- popis s rozhodčími, číslem utkání a soutěží;
- délku 1,5 h;
- jen aktuální delegace ze stránky ČABR.

## Rozhodnutí

1. **Kalendář k odběru** na tajné adrese `/calendar/<token>.ics`. Zapíná se v Profilu, kde jsou:
   - adresa s tlačítkem Kopírovat;
   - odkazy „Přidat do Google Kalendáře“ (`calendar.google.com/calendar/r?cid=webcal://…`) a „Otevřít v Apple Kalendáři / Outlooku“ (`webcal://…`);
   - tlačítko „Vygenerovat novou adresu“ (stará přestane fungovat);
   - tlačítko „Vypnout kalendář“.
2. **Rozsah:** delegace ČBF za posledních 365 dní a všechny budoucí. Vlastní zápasy (PDR 0004) ve stejném rozsahu.
3. **Událost zápasu ČBF:**
   - **Název:** „Domácí – Hosté · soutěž“, u komisaře s předponou „Komisař:“.
   - **Čas:** začátek zápasu, délka 2 h (zápas s přestávkami a papíry po něm). Zápas bez času je celodenní, zápas bez data se vynechá.
   - **Místo:** hala s ulicí, PSČ a městem a GEO souřadnice, takže navigace funguje přímo z kalendáře.
   - **Popis:**
     - soutěž a kategorie; fáze, kolo a číslo utkání; moje role;
     - celá delegace (rozhodčí 1–3, komisař);
     - hala s adresou;
     - odměna podle PDR 0002, ≈ u odhadu, jinak důvod, proč chybí;
     - výsledek s body po čtvrtinách a prodloužením, jakmile je znám;
     - televize, video a live statistiky;
     - odkaz na Vyúčtování daného měsíce (PDR 0005).
   - **URL události** vede na live statistiky, pokud existují.
4. **Vlastní zápas** je celodenní událost s názvem akce, místem, rolí, počtem zápasů, odměnou a poznámkou.
5. **Aktualizace:**
   - Kalendář se generuje při každém stažení z aktuálních dat.
   - Změna času, haly nebo delegace se projeví při příští synchronizaci klienta. Feed klientům doporučuje obnovu po hodině (`REFRESH-INTERVAL`, `X-PUBLISHED-TTL`).
   - Google Kalendář obnovuje odběry podle sebe (typicky po několika hodinách).
   - Zápas, ze kterého byl rozhodčí odvolán, z feedu zmizí, a tím i z kalendáře.

## Zvažované alternativy

- **Export jednorázového .ics souboru:** nesleduje změny delegací, což je hlavní důvod kalendáře.
- **Synchronizace přes Google Calendar API:** vyžaduje další OAuth scope a funguje jen pro Google.
- **Scrapování ČABR jako dřív:** potřebuje heslo uživatele a prohlížeč na serveru. Data ČBF, která už máme, pokrývají stejné zápasy a víc údajů.

## Dopady

- \+ Stará služba se dá vypnout. Kalendář má navíc adresu haly, GEO, celou delegaci, odměnu, výsledek a odkazy.
- \+ Funguje v každém kalendáři, který umí odběr iCal.
- − Zápas se v kalendáři objeví, až ČBF zveřejní jméno rozhodčího (do té doby posílá „Delegováno“, ADR 0004) a aplikace ho stáhne (ADR 0005). ČABR může delegaci ukázat dřív. Kdy ČBF jména zveřejňuje, je otevřená otázka ADR 0005.
- − Kdo zná adresu, vidí zápasy a odměny. Proto jde adresa vyměnit nebo vypnout (ADR 0010).

## Metriky / jak poznáme úspěch

- Uživatel vypne `CABR-web-scraper` a nic mu v kalendáři nechybí.

## Otevřené otázky

- O kolik dní dřív ukazuje delegaci ČABR než veřejný export ČBF? Ověřit na `cbf.game_official_history` (první `observed_at` delegace vs. začátek zápasu). Pokud je rozdíl velký, nechat starou službu běžet, dokud se nenajde jiný zdroj.
