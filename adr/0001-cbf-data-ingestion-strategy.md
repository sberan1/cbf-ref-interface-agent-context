# 0001 – Strategie ingestu dat z CBF exportu

- **Status:** Proposed
- **Datum:** 2026-09-23
- **Související:** [history/20260923-analyza-cbf-exportu.md](../history/20260923-analyza-cbf-exportu.md)

## Kontext

Aplikace staví nad daty ČBF, primárně o rozhodčích (delegace, zápasy, komisaři), ale z exportu chceme uložit co nejvíc. XML/JSON export ČBF nemá endpoint „zápasy rozhodčího“ – delegace jsou jen v detailu zápasu / rozpisu fáze. ČBF žádá šetrné používání exportu, jinak hrozí blokace. Živé API zatím neověřeno (viz history).

## Rozhodnutí

1. **Stahujeme celou hierarchii:** sezóna → oblast → pohlaví → soutěž → fáze → zápas → detail zápasu + týmy, hráči, tabulky. Index podle rozhodčího stavíme lokálně.
2. **Dvouvrstvé úložiště:**
   - `raw_payload` – endpoint, params, fetched_at, hash, tělo odpovědi (beze změn).
   - normalizovaná vrstva – `season`, `area`, `competition`, `phase`, `team`, `venue`, `game`, `game_score_quarter`, `official`, `game_official`, `standing`, `player`, `player_stats`.
3. **`game_official_history`** – změny delegací odvozené z diffů po sobě jdoucích snapshotů.
4. **Žádné live dotazy na ČBF z uživatelských requestů** – vše přes sync joby, aplikace čte jen z vlastní DB.
5. **Sync plán:**
   - bootstrap všech sezón sekvenčně s pauzami mezi requesty,
   - denně: struktura (soutěže, fáze) + zápasy aktivních fází,
   - častěji: game detail v okně −3 / +14 dní od data zápasu,
   - po odehrání: plný detail + statistiky + tabulky,
   - detekce změn hashem payloadu (neměněné payloady se znovu nenormalizují).

## Zvažované alternativy

- **Live proxy na ČBF API** – jednoduché, ale riziko blokace, pomalé, nelze dotazovat podle rozhodčího.
- **Stahovat jen zápasy s delegacemi** – nelze zjistit bez stažení detailu, a ztratili bychom kontext (tabulky, týmy).

## Důsledky

- + Nezávislost na dostupnosti ČBF, možnost historie a diffů delegací, dotazy podle rozhodčího.
- + Raw vrstva umožňuje přenormalizovat data při změně schématu bez opětovného stahování.
- − Větší objem dat a nutnost udržovat sync joby.
- − Data jsou jen tak čerstvá, jak často běží sync.

## Otevřené otázky

- Skutečné schéma v1/v2 endpointů (discovery skript).
- Stabilita ID rozhodčích napříč sezónami.
- Konkrétní intervaly syncu a pauzy mezi requesty.
- Oficiální žádost ČBF o přístup.
