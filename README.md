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
| [ADR 0006](adr/0006-session-a-parovani-uctu.md) | Accepted | Session a párování účtu s rozhodčím |
| [ADR 0007](adr/0007-nasazeni-docker-vps.md) | Accepted | Nasazení: Docker na VPS, CI/CD přes GitHub Actions |
| [ADR 0008](adr/0008-ucty-google-oauth-schvalovani.md) | Accepted | Účty přes Google OAuth se schvalováním správcem |
| [ADR 0009](adr/0009-logovani-graylog.md) | Accepted | Logování do Graylogu |
| [PDR 0001](pdr/0001-prihlaseni-a-dashboard-rozhodciho.md) | Accepted | Přihlášení výběrem rozhodčího a dashboard v1 |
| [PDR 0002](pdr/0002-odmeny-rozhodcich.md) | Accepted | Odměny rozhodčích na dashboardu |
| [PDR 0003](pdr/0003-cestovne-stravne-vsedni-den.md) | Accepted | Cestovné, stravné a náhrada za všední den |

History: [20260923 – analýza exportu](history/20260923-analyza-cbf-exportu.md), [20261014 – scaffold ingestu](history/20261014-scaffold-cbf-ingestu.md).

Kód projektu: [sberan1/cbf-ref-interface](https://github.com/sberan1/cbf-ref-interface) (tato báze je v něm submodulem `docs/agent-context`).

## Pravidla pro agenty

1. Před každým rozhodnutím načti tento README, relevantní soubory v `history/` a platné (`Accepted`, případně `Proposed`) ADR/PDR.
2. Nové rozhodnutí nesmí odporovat platnému záznamu bez toho, aby ho explicitně označilo jako `Superseded`.
3. Tvrzení o externích datech (API ČBF apod.) označuj jako ověřené / neověřené a uváděj zdroj.
4. Nové ADR/PDR a history přidej do přehledu záznamů výše.
