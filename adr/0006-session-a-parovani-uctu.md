# 0006 – Session a párování účtu s rozhodčím

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** [PDR 0001](../pdr/0001-prihlaseni-a-dashboard-rozhodciho.md), [ADR 0002](0002-stack-sveltekit-drizzle-postgres.md)

## Kontext

PDR 0001 chce jednoduché přihlášení výběrem rozhodčího bez hesla. Stack je SvelteKit 3 + Drizzle + PostgreSQL (ADR 0002). Session musí přežít restart serveru a jít zneplatnit.

## Rozhodnutí

- Nové Postgres schéma **`app`**:
  - `app.user` (`id`, `official_id` unique → `cbf.official`, `created_at`, `last_login_at`): jeden účet na záznam rozhodčího.
  - `app.session` (`id` = SHA-256 tokenu, `user_id`, `expires_at`).
- **Token:** 32 náhodných bajtů (base64url) v cookie `cbf_session`. Cookie je httpOnly, `SameSite=Lax`, `Secure` mimo dev (výchozí chování SvelteKitu). V DB je jen hash, takže únik DB nedává platné tokeny.
- **Platnost** 30 dní, klouzavá. Session použitá v posledních 15 dnech platnosti se prodlouží. Odhlášení session smaže.
- Validace probíhá v `src/hooks.server.ts` a výsledek ukládá do `event.locals.user` / `locals.session`. Stránky si přihlášení kontrolují v `load` (`/dashboard` přesměruje na `/login`).
- Vlastní implementace bez knihovny (~100 řádků, `src/lib/server/auth/session.ts`). Lucia je deprecated a Auth.js je pro tenhle případ zbytečně velký.
- Formuláře používají SvelteKit form actions (POST), takže CSRF ochranu řeší kontrola originu ve SvelteKitu.

## Zvažované alternativy

- **Bezstavový JWT v cookie:** nejde zneplatnit bez blacklistu. Pro jednoho uživatele na rozhodčího zbytečné.
- **Auth.js / Better Auth:** vhodné, až přibude heslo nebo OAuth. Teď by přidalo závislosti a vlastní schéma.

## Důsledky

- \+ Jednoduché, auditovatelné, bez dalších služeb.
- − Bez hesla (vědomě, PDR 0001). Přidání hesla znamená sloupec v `app.user` a krok v login action, model session zůstává.
- − Expirované session je potřeba občas mazat (`deleteExpiredSessions`), zatím se mažou jen při pokusu o použití.

## Otevřené otázky

- Silnější ověření, až bude aplikace přístupná dalším lidem (PDR 0001).
