# 0008 – Účty přes Google OAuth se schvalováním správcem

- **Status:** Accepted
- **Datum:** 2026-10-14
- **Související:** nahrazuje přihlašovací část [ADR 0006](0006-session-a-parovani-uctu.md) a bod 1 [PDR 0001](../pdr/0001-prihlaseni-a-dashboard-rozhodciho.md); [ADR 0007](0007-nasazeni-docker-vps.md), [PDR 0003](../pdr/0003-cestovne-stravne-vsedni-den.md)

## Kontext

Přihlášení výběrem rozhodčího bez hesla (PDR 0001) stačilo, dokud aplikaci používal jen uživatel za basic auth. Uživatel chce „pořádný login“: každý si založí účet a spáruje ho se sebou. Správce (uživatel) má účty schvalovat, protože spárování otevírá osobní údaje a výdělky konkrétního rozhodčího. Jako poskytovatele OAuth zvolil jen Google.

## Rozhodnutí

1. **Google OpenID Connect** přes knihovnu `arctic` (authorization code flow s PKCE).
   - `state` a `code_verifier` jsou v httpOnly cookies na 10 minut.
   - ID token přichází přímo z token endpointu Googlu přes TLS. Kontrolují se claims (`iss`, `aud`, `exp`, `email_verified`), podpis ne (OIDC Core 3.1.3.7).
   - Redirect URI je `${origin}/login/google/callback`, origin se odvozuje z `Host` a `X-Forwarded-Proto` (ADR 0007).
   - SvelteKit 3 povoluje přesměrování na cizí doménu jen výslovně: `redirect(302, url, { external: ['https://accounts.google.com'] })`.
2. **Model** (schéma `app`):
   - `user`: e-mail (unique), jméno, avatar, `role` (`user` / `admin`), `status` (`pending` / `approved` / `rejected`), `requested_official_id`, `official_id` (unique, nastaví ho až schválení), kdo a kdy rozhodl, bydliště (PDR 0003).
   - `oauth_account`: (`provider`, `provider_user_id`) → `user`. Další poskytovatel jde přidat bez změny schématu.
   - Session zůstávají podle ADR 0006: hash tokenu v DB, klouzavých 30 dní.
3. **Role podle `ADMIN_EMAILS`** (env, čárkami oddělené e-maily). Role se nastavuje při každém přihlášení, env je zdroj pravdy.
4. **Tok:**
   - Google → nový účet `pending` → `/pair` (vyhledání se a „To jsem já“, uloží `requested_official_id`) → `/pending` → správce na `/admin` schválí → `/dashboard`.
   - Správce se při výběru svého záznamu schválí sám.
   - Správce může schválit i jiné ID, zamítnout nebo zrušit spárování.
   - Rozhodčího spárovaného s jiným účtem nejde vybrat.
5. **Ochrana stránek:** `requireApproved` / `requireAdmin` v `load` a v actions. `landingPath` posílá každý stav účtu na jeho stránku. Layout posílá do prohlížeče jen to, co potřebuje hlavička.
6. **Migrace `0002`** smaže staré účty bez e-mailu a jejich session. Neobsahovaly nic než spárování.

## Zvažované alternativy

- **Auth.js / Better Auth:** víc poskytovatelů hned, ale vlastní schéma a sessions vedle již existujících, a nejasná podpora SvelteKit 3. `arctic` + vlastní session je ~200 řádků a plně pod kontrolou.
- **Magic link e-mailem:** potřebuje SMTP a deliverability. Google má prakticky každý.
- **Schvalovat jen účet a spárování nechat volné:** kdokoli by si mohl zobrazit výdělky cizího rozhodčího. Schvaluje se proto spárování.

## Důsledky

- \+ Bez hesel, ověřený e-mail, správce má přehled, kdo k čemu má přístup.
- − Závislost na Google účtu. Další poskytovatel (Microsoft, Apple) = další provider v `arctic` a řádky v `oauth_account`.
- − Správce musí reagovat na žádosti. Notifikace (e-mail) zatím nejsou, jen počet čekajících na `/admin`.

## Otevřené otázky

- Notifikace správci o nové žádosti.
- Mazání účtu a dat na žádost uživatele (GDPR).
