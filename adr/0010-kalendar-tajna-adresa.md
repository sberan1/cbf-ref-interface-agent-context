# 0010 – Kalendářový feed: tajná adresa místo přihlášení

- **Status:** Accepted
- **Datum:** 2026-10-27
- **Související:** [PDR 0006](../pdr/0006-kalendar-zapasu.md), [ADR 0006](0006-session-a-parovani-uctu.md), [ADR 0009](0009-logovani-graylog.md)

## Kontext

Kalendářové aplikace (Google, Apple, Outlook) stahují odběr iCal samy, bez cookies a bez přihlášení. Feed (PDR 0006) proto potřebuje jiné ověření než session z ADR 0006.

## Rozhodnutí

- **Token v URL:** `/calendar/<token>.ics`, kde token je 32 náhodných bajtů v base64url (43 znaků).
  - Je ve sloupci `app.user.calendar_token` (unique, null = feed vypnutý), migrace `0004_calendar_feed`.
  - Neplatný, vypnutý nebo vyměněný token, nebo účet, který už není schválený a spárovaný, dostane prosté `404` bez rozlišení důvodu.
- **Token je v DB čitelně, ne jako hash** (na rozdíl od session):
  - Uživatel potřebuje adresu kdykoli znovu zkopírovat, třeba pro další zařízení, a hash by to neumožnil.
  - Token dává jen čtení dat, která jsou ve stejné DB. Únik DB tedy útočníkovi nic dalšího neotevře.
- **Logy:** request log v `hooks.server.ts` nahrazuje cestu `/calendar/…`, takže token se do Graylogu nedostane. Route ID (`/calendar/[token].ics`) zůstává.
- **Výměna a vypnutí:** „Vygenerovat novou adresu“ přepíše token (stará adresa okamžitě končí), „Vypnout“ ho smaže.
- **Formát:** vlastní zápis RFC 5545 (`src/lib/server/calendar/ics.ts`):
  - escapování textu, zalamování řádků na 75 oktetů bez dělení UTF-8 znaků, CRLF;
  - časy v UTC, takže feed nepotřebuje VTIMEZONE;
  - `METHOD:PUBLISH`, stabilní UID `game-<id>-<role>@cbf-ref-interface` a `own-<id>@cbf-ref-interface`.
  - Bez knihovny: zápis má ~100 řádků a testy a závislost by přinesla víc, než ušetří.
- **Cache:** odpověď má `Cache-Control: private, max-age=900`.

## Zvažované alternativy

- **HTTP Basic auth:** Google Kalendář u odběrů přihlášení nepodporuje.
- **Token jako HMAC(server secret, user, verze):** v DB by nebylo nic citlivého, ale přibyla by povinná proměnná prostředí a rotace serverového tajemství by zneplatnila všechny kalendáře.
- **Hash tokenu v DB:** adresa by šla zobrazit jen jednou. Nepohodlné pro uživatele a bez reálného přínosu (viz výše).

## Důsledky

- \+ Funguje se všemi běžnými kalendáři.
- − Adresa se může dostat do přístupových logů reverzní proxy nebo do historie prohlížeče. Riziko snižuje výměna adresy jedním kliknutím.
