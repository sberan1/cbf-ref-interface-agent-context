# 2026-09-23 – Analýza CBF exportu

## Co se řešilo

Jaká data ČBF zveřejňuje v XML/JSON exportu a jak je ingestovat pro rozhraní pro rozhodčí. Návrh struktury této znalostní báze.

## Zdroje

- https://cbf.cz.basketball/xml-exporty/p58 (dokumentace exportu)
- kód klientů: `jknetl/CzechBasketballSchedule`, `ivosu/cbf_ical`
- **Živé API NEOVĚŘENO** (robots.txt + sandbox allowlist) – vše níže je z dokumentace a kódu klientů.

## Export v1 (dokumentovaný XML) – `https://www.cbf.cz/xml/`

- `divs.php?s=ROK` – soutěže + fáze
- `sched.php?p=FÁZE` – rozpis, výsledky, čtvrtiny, delegace: `<ref>` (id, firstname, lastname), `<sup>` komisař
- `table.php?p=FÁZE` – tabulka

## Export v2 (nedokumentovaný) – `https://www.cbf.cz/xml/api/`, parametr `json=1`

- `seasonList.php`
- `divs.php?season=&female=0|1&area=`
- `phases.php?competition=`
- `games.php?phase=`
- `game.php?game=`
- `table.php?phase=`
- `team.php?id=&competition=`
- `teamplayers.php?id=&competition=`

`game.php` obsahuje: `u1id/u1n1/u1n2` … `u3*` (rozhodčí 1–3), `commisarid/commisarn1/commisarn2`, `lat/lon`, `place`, `city`, `url_live`, `score_quarter` jako string `"20:18 15:22 …"`.

Flagy soutěže: `refeval`, `sups`, `clubeval` … (hodnocení rozhodčích v exportu – neověřeno).

`area`: 0 = celostátní, 1–14 = regiony (klientům chybí 4, 9–11).

## Zvláštnosti

- `gdate` = `0000-00-00` u nenaplánovaných zápasů.
- Časy v Europe/Prague bez offsetu.
- `pt` v tabulce NBL = % výher (float).
- ČBF žádá šetrné používání, jinak blokace.

## Závěry / rozhodnutí

- Strategie ingestu → [ADR 0001](../adr/0001-cbf-data-ingestion-strategy.md) (Proposed).
- Plánovaný stack NestJS API + React (zatím bez formálního ADR).
- Struktura znalostní báze: README + history/ + adr/ + pdr/ (PDR – předpoklad, nepotvrzeno).
- Repo je podle API public (původně mělo být private).

## Další kroky

1. Discovery skript (spouštět lokálně): vzorky všech v1/v2 endpointů, aktuální + starší sezóna, area 0–20 → skutečné schéma. Jazyk (Node/TS vs Python) nepotvrzen.
2. Ověřit stabilitu ID rozhodčích napříč sezónami.
3. Zvážit oficiální žádost ČBF o přístup.
