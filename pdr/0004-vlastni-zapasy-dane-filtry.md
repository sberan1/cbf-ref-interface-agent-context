# 0004 – Vlastní zápasy, odměny po kalendářních letech, filtry kolegů

- **Status:** Accepted
- **Datum:** 2026-10-20
- **Související:** [PDR 0001](0001-prihlaseni-a-dashboard-rozhodciho.md), [PDR 0002](0002-odmeny-rozhodcich.md), [PDR 0003](0003-cestovne-stravne-vsedni-den.md)

## Problém / příležitost

Požadavky uživatele z 2026-10-20:

- **Kolegové:** žebříček zápasů s kolegou filtrovat podle sezóny a pod tím podle oblasti. Celá historie musí zůstat jako možnost.
- **Daně:** vyjet si, kolik vydělal za kalendářní rok. Sezóny přecházejí přes Nový rok, přiznání se dělá za rok.
- **Soutěže:** u nejčastějších soutěží vždy ukázat kategorii, aby bylo jasné, o jaký přebor šlo.
- **Vlastní zápasy:** přáteláky a místní turnaje mimo systém ČBF zapsat do vlastní evidence.

## Rozhodnutí

1. **Filtry kolegů** jsou parametry URL `ks` (začátek sezóny) a `ko` (oblast, 0 = celostátní).
   - Bez parametrů se ukazuje celá historie.
   - Nabízejí se jen sezóny a oblasti, kde má osoba zápasy. Oblasti jsou ty z vybrané sezóny a neplatná kombinace se ignoruje.
   - Žebříček ukazuje 10 kolegů.
   - Přepnutí grafu filtr zachová.
2. **Kalendářní rok:** odřízené zápasy se počítají podle data zápasu, náhrady podle data herního dne. Započítají se jen dny do dneška.
   - Na přehledu je tabulka po letech: zápasy, odměny, z toho odhad, náhrady.
   - Ke každému roku je CSV `/dashboard/export/<rok>` se středníky, desetinnou čárkou a BOM, aby ho český Excel otevřel napřímo. Po řádcích obsahuje odměny ČBF (s kategorií, rolí, sazbou a příznakem odhadu), vlastní zápasy a náhrady za dny (cestovné, stravné, všední den, km).
   - Na stránce je upozornění, že jde o výpočet ze sazebníků, ne o výplaty.
3. **Kategorie soutěže** se bere z katalogu (`cbf.category`) a jinak z payloadu (`catname`). Ukazuje se pod názvem, pokud ji název už neobsahuje.
4. **Vlastní zápasy** jsou v tabulce `app.own_game` a na stránce `/own-games`, kde se dají přidat, upravit a smazat.
   - **Údaje:** datum, popis akce, role (rozhodčí, komisař, stolek, jiné), počet zápasů (turnaj = jeden záznam), odměna celkem, místo a poznámka. Prázdná odměna znamená nezaplaceno nebo zatím neznámo.
   - **Kde se počítají:** odměna je skutečná, nikdy odhad. Započítá se do odměn na přehledu (vyděláno, sezóna, očekáváno u budoucích), do grafu odměn, do tabulky po letech a do CSV.
   - **Kde se nepočítají:** do statistik zápasů ČBF ani do cest a náhrad.

## Zvažované alternativy

- **Rok podle data výplaty:** ten neznáme. Datum zápasu je nejbližší dostupná náhrada a uživatel výsledek porovná s výplatami.
- **PDF výkaz místo CSV:** CSV se dá dál zpracovat v Excelu a k přiznání stačí.
- **Vlastní zápasy jako umělé zápasy v `cbf.game`:** míchalo by to data ČBF s ručními. Oddělená tabulka v `app` je čistší.

## Důsledky

- \+ Podklad k přiznání na jedno kliknutí, včetně akcí mimo ČBF.
- − Náhrady k vlastním zápasům se zatím nepočítají. Cesta na turnaj se dá zapsat jen jako poznámka.

## Otevřené otázky

- Cesty a náhrady i k vlastním zápasům (průvodce cestou pro den bez zápasu ČBF).
