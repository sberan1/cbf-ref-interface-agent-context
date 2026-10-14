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

## Přehled záznamů

| Záznam | Status | Téma |
| --- | --- | --- |
| [ADR 0001](adr/0001-cbf-data-ingestion-strategy.md) | Proposed | Strategie ingestu dat z ČBF exportu |
| [ADR 0002](adr/0002-stack-sveltekit-drizzle-postgres.md) | Accepted | Stack: SvelteKit fullstack + Drizzle + PostgreSQL |
| [ADR 0003](adr/0003-sync-fronta-rabbitmq.md) | Accepted | Fronta sync jobů: RabbitMQ |
| [ADR 0004](adr/0004-db-schema-ingestu.md) | Proposed | DB schéma ingestu (raw + normalizovaná vrstva) |
| [ADR 0005](adr/0005-rozsah-a-setrnost-stahovani.md) | Accepted | Rozsah stahování a šetrnost vůči ČBF |

History: [20260923 – analýza exportu](history/20260923-analyza-cbf-exportu.md), [20261014 – scaffold ingestu](history/20261014-scaffold-cbf-ingestu.md).

Kód projektu: [sberan1/cbf-ref-interface](https://github.com/sberan1/cbf-ref-interface) (tato báze je v něm submodulem `docs/agent-context`).

## Pravidla pro agenty

1. Před každým rozhodnutím načti tento README, relevantní soubory v `history/` a platné (`Accepted`, případně `Proposed`) ADR/PDR.
2. Nové rozhodnutí nesmí odporovat platnému záznamu bez toho, aby ho explicitně označilo jako `Superseded`.
3. Tvrzení o externích datech (API ČBF apod.) označuj jako ověřené / neověřené a uváděj zdroj.
4. Nové ADR/PDR a history přidej do přehledu záznamů výše.
