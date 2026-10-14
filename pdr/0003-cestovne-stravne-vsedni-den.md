# 0003 – Cestovné, stravné a náhrada za všední den

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [PDR 0002](0002-odmeny-rozhodcich.md), [ADR 0008](../adr/0008-ucty-google-oauth-schvalovani.md), [history/20261014-scaffold-cbf-ingestu.md](../history/20261014-scaffold-cbf-ingestu.md)

## Problém / příležitost

K odměnám za řízení patří náhrady: cestovné, stravné a příplatek za všední den. Zadání uživatele:

- **Průvodce cestou pro každý den se zápasy:** kde vyzvedne kolegu a kde ho vysadí, a to se často mění.
- **Vzdálenosti:** adresa haly ze systému, kilometry spočítat.
- **Cestovné:** sám 4,50 Kč/km, s kolegou 5 Kč/km.
- **Stravné:** 155 Kč nad 5 h. Délka cesty = cesta + cca 3 h 30 min na zápas.
- **Všední den:** za zápasy soutěží ČBF mimo víkendy a svátky +300 Kč.

## Rozhodnutí

1. **Jednotkou je herní den.** Všechny delegace osoby v jeden den (rozhodčí i komisař) tvoří jednu cestu, uloženou v `app.trip` (unikátní uživatel + datum). Cesta se skládá ze zastávek:
   - **domov** (z profilu),
   - **vyzvednutí** a **vysazení** kolegy (adresa, jméno kolegy),
   - **haly** zápasů v pořadí, po sobě jdoucí zápasy v jedné hale se slučují.

   Haly a domov doplňuje server, klient posílá jen vyzvednutí a vysazení.
2. **Způsob cesty:**
   - **autem (řidič):** km × sazba,
   - **spolujezdec:** cestovné bere řidič, stravné se počítá z délky trasy,
   - **veřejná doprava:** jízdenky podle dokladů,
   - **bez cesty**.

   K tomu parkovné a jiné doklady.
3. **Kilometry:**
   - Každý úsek se routuje zvlášť přes Mapy.com (`car_fast`). Podle rozpisu rozhoduje spory nejrychlejší trasa na mapy.cz.
   - Úseky, kdy je v autě kolega (mezi vyzvednutím a vysazením), jdou za sazbu „s kolegou“, ostatní za sazbu „sám“.
   - Trasy a geokódování se cachují v `app.route_cache` a `app.geo_cache` pro všechny uživatele.
   - Bez API klíče nebo při výpadku se počítá vzdušná čára × 1,3 při 60 km/h. Výsledek je označený jako odhad a při další příležitosti se zkusí znovu.
4. **Sazby** (`src/lib/server/travel/rates.ts`, s platností a původem jako v PDR 0002):
   - **Cestovné** podle sezóny: 4,50 Kč/km sám, 5,00 Kč/km s kolegou a 6,30 Kč/km s kolegou v NBL, ŽBL, 1. lize mužů a Českém poháru (poznámka v přehledech). Přehledy 2023/24–2026/27, před 2023/24 předpoklad.
   - **Stravné** podle kalendářního roku (vyhláška MPSV, přehledy uvádějí stav „v době vydání“): ≥ 5 h / ≥ 12 h / ≥ 18 h.

     | Rok | Sazby (Kč) | Zdroj |
     | --- | --- | --- |
     | 2023 | 129 / 196 / 307 | přehled |
     | 2024 | 140 / 212 / 333 | vyhláška, v přehledech není, označeno jako odhad |
     | 2025 | 148 / 225 / 353 | přehled |
     | 2026 | 155 / 236 / 370 | přehled |
     | před 2023 | = 2023 | odhad |

   - **Délka cesty** = doba jízdy podle tras + 3 h 30 min na každý zápas dne.
5. **Náhrada za všední den** (300 Kč za den) platí pro celostátní soutěže ČBF (oblast 0), Český pohár má 650 Kč od 2025/26. Nevyplácí se:
   - o víkendu,
   - o státním svátku (český kalendář včetně Velkého pátku a Velikonočního pondělí),
   - když je obec bydliště stejná jako obec hal těchto zápasů (pravidlo „s bydlištěm v místě konání“).

   Pražské soutěže příplatek nemají, ostatní oblasti zatím nejsou pokryté.
6. **Výchozí cesta** (`auto`) = sám autem z domova přes haly domů. Tlačítko „Spočítat nevyplněné dny“ ji doplní pro sezónu, nejvýš 40 dnů na jeden request. Díky cache se API volá zhruba jednou na halu. Uložení v průvodci cestu potvrdí (`confirmed`). „Stejně jako jindy“ převezme vyzvednutí a vysazení z jiného potvrzeného dne a adresy kolegů se ukládají jako místa (`app.place`) pro příště.
7. **Částky se počítají při čtení** z uložených zastávek a úseků, takže změna sazeb platí zpětně bez přepočtu tras. Dashboard ukazuje součty náhrad celkem, za aktuální sezónu a ve sloupci tabulky sezón.

## Zvažované alternativy

- **Jedna trasa s waypointy místo úseků:** méně volání, ale nedá se rozdělit na část sám / s kolegou.
- **Počítat náhrady ve workeru:** zbytečné. Request s cache trvá sekundy a uživatel hned vidí výsledek.
- **Stravné podle sezóny:** přehledy uvádějí jen sazby platné v době vydání. Vyhláška se mění k 1. 1., proto se stravné řídí kalendářním rokem.

## Důsledky

- \+ Náhrady po dnech s rozpisem (km × sazba, doba cesty, důvod chybějícího příplatku), dohledatelné při sporu.
- − Bez `MAPY_API_KEY` jen odhady. Klíč je potřeba nastavit na serveru.
- − Bydliště je osobní údaj v `app.user`. V souladu s rozhodnutím uživatele ukládat osobní údaje.

## Otevřené otázky

- Ověřit Mapy.com API na produkci (odpovědi jsou implementované podle dokumentace a otestované proti mocku).
- Sazby stravného před rokem 2023 a NBL/ŽBL: platí pro ně příplatek za všední den stejně?
- Náhrady pro ostatní oblasti (příplatky 150–250 Kč, jiné podmínky).
- Export pro vyúčtování (CSV/PDF po měsících).
