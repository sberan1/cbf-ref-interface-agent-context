# cbf-ref-interface-agent-context

Znalostní báze pro projekt **Cz.basketball referee interface** – rozhraní nad daty ČBF pro rozhodčí (delegace, zápasy, komisaři). Slouží jako sdílený kontext pro AI agenty i lidi.

## Struktura

- `history/` – shrnutí konverzací, jeden soubor na konverzaci: `YYYYMMDD-tema.md`
- `adr/` – Architecture Decision Records (technická rozhodnutí)
- `pdr/` – Product Decision Records (produktová rozhodnutí)
- `adr/0000-template.md`, `pdr/0000-template.md` – šablony

## Konvence

- Číslování záznamů: `NNNN-nazev.md` (např. `0001-cbf-data-ingestion-strategy.md`).
- Statusy: `Proposed` → `Accepted` → `Deprecated` / `Superseded`.
- Superseded záznam odkazuje na nový záznam, který ho nahrazuje (a naopak).

## Pravidla pro agenty

1. Před každým rozhodnutím načti tento README, relevantní soubory v `history/` a platné (`Accepted`, případně `Proposed`) ADR/PDR.
2. Nové rozhodnutí nesmí odporovat platnému záznamu bez toho, aby ho explicitně označilo jako `Superseded`.
3. Tvrzení o externích datech (API ČBF apod.) označuj jako ověřené / neověřené a uváděj zdroj.
