# MyBodyAI

Health analytics from wearable data. MyBodyAI reads HRV, sleep and resting heart rate from your devices and flags the days your body breaks its own pattern. It compares you with your own baseline, never with population averages.

**Live:** https://aiclysm.com/mybodyai/info/ · **Contact:** vaclav.frcek@aiclysm.com

<a href="https://aiclysm.com/">
  <img src="https://aiclysm.com/img/app/overview-en-dark.webp" alt="MyBodyAI Overview screen" width="820">
</a>

The source code is private. This repository describes how the product is built.

[Česká verze níže](#cesky)

---

## What it does

- **Body Status:** one daily verdict built from 12 health indices, from immunity to training load, each measured against your own baseline.
- **Biological age** in 8 domains: heart, nervous system, sleep, body composition, activity, stress resilience, recovery and VO2max, each against published reference values.
- **Sources:** Polar, Fitbit, Withings and Oura sync on their own over OAuth. Garmin comes in through a data export, through the phone (Android app with Health Connect, on Google Play) and through a Connect IQ watch app.
- English and Czech.

## Architecture

```text
Caddy (automatic HTTPS)
 ├─ aiclysm.com ............ static site, English and Czech
 ├─ /mybodyai/ ............. React single-page app
 └─ API .................... Flask on Gunicorn, APScheduler jobs
        │
        ├─ PostgreSQL 18 ... accounts, sign-in, billing, audit, sync bookkeeping
        ├─ SQLite .......... one file per user: health time series, baselines, scores
        └─ Open Wearables .. FastAPI + Celery + Redis: provider OAuth and data normalisation
```

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript 5.9, Vite 7, Tailwind 4, Recharts, Framer Motion |
| Backend | Python 3.12, Flask 3.1, Gunicorn, APScheduler |
| Data | PostgreSQL 18, SQLite (per user), Redis 8 |
| Integrations | Open Wearables (FastAPI, Celery), OAuth 2.0 |
| Clients | Web app (PWA), Android app, Garmin Connect IQ watch app |
| Payments | Stripe |
| Analytics | Umami, self-hosted, cookie-free |
| Infrastructure | Docker Compose, 8 containers |

## Key decisions

**Hybrid data model.** PostgreSQL holds what is shared: accounts, sign-in, billing, audit trails. Each user's health data lives in a SQLite file of its own. Health data belongs to one person and is read far more than written, so a file per user gives isolation, simple export and deletion, and a restore of one account without touching the others. Jobs across all users go user by user.

**Metric contracts.** Every metric is described in layers, from storage and signal concepts through provider contracts and metric contracts to what the API returns. A provider is registered once as a `ProviderContract`, and each metric works out from the contracts what it can compute for a given account and what it is missing.

**Event-driven sync.** Open Wearables tells the app when new data arrives, and the app rebuilds that user's scores. Rebuilds for one user are merged so they do not run several times in a row. An hourly sweep catches anything a notification missed.

**One gate for every deploy.** A push to production runs through a pre-push gate on the server: the backend test suite, the website tests, frontend lint, type-check, build and Vitest, and configuration checks. A red gate stops the deploy. GitHub Actions workflows exist for runs on demand.

## Security

- HTTPS only, HSTS with preload, TLS 1.2 and 1.3.
- Passwords and one-time email codes hashed with scrypt. Tokens stored as HMAC-SHA256. TOTP secrets encrypted at rest.
- Two-factor sign-in with TOTP, passkeys (WebAuthn) and recovery codes. Admin access requires a second factor.
- Server-side sessions in Redis. Cookies are `HttpOnly`, `Secure` and `SameSite=Lax`, with a per-session CSRF token.
- Rate limits plus lockouts after failed attempts, counted per address, per account and per IP. fail2ban on the sign-in endpoints.
- Stripe webhooks verified by signature and processed once per event id.
- Admin actions are audit-logged.
- UFW firewall; only the reverse proxy is reachable from outside. The app container runs as a non-root user.
- Data export and account deletion from the app (GDPR).

## Backups and monitoring

- Every 6 hours an encrypted archive (GPG, AES-256) of every per-user SQLite file and the PostgreSQL databases, integrity-checked and kept for 30 days. One archive a day goes off-site. A restore drill runs automatically every month.
- Sentry on the backend and the frontend, Telegram alerts, health checks every 5 minutes and heartbeat monitoring of scheduled jobs.

## Testing

- 6,600+ backend tests (pytest), 2,500+ frontend tests (Vitest), 300 website tests.
- 1,700+ tests in the Open Wearables backend.

## Links

- Product: https://aiclysm.com/mybodyai/info/
- Wearable comparison (113 devices): https://aiclysm.com/compare/
- Changelog: https://aiclysm.com/changelog/

---

<a name="cesky"></a>

# Česky

Zdravotní analytika z dat chytrých zařízení. MyBodyAI čte HRV, spánek a klidový tep z vašich hodinek a upozorní vás, když se vaše tělo začne chovat jinak. Srovnává vás jen s vaším vlastním normálem, ne s populačním průměrem.

**Aplikace:** https://aiclysm.com/mybodyai/info/ · **Kontakt:** vaclav.frcek@aiclysm.com

Zdrojový kód je soukromý. Tento repozitář popisuje, jak je produkt postavený.

## Co umí

- **Body Status:** jeden denní verdikt z 12 zdravotních indexů, od imunity po tréninkovou zátěž, každý proti vlastnímu normálu.
- **Biologický věk** v 8 oblastech: srdce, nervový systém, spánek, složení těla, pohyb, odolnost vůči stresu, regenerace a VO2max, každá proti publikovaným referenčním hodnotám.
- **Zdroje dat:** Polar, Fitbit, Withings a Oura se synchronizují samy přes OAuth. Garmin přichází z exportu dat, přes telefon (Android aplikace s Health Connect na Google Play) a přes aplikaci do hodinek v Connect IQ.
- Čeština a angličtina.

## Architektura

```text
Caddy (automatické HTTPS)
 ├─ aiclysm.com ............ statický web, česky a anglicky
 ├─ /mybodyai/ ............. React aplikace
 └─ API .................... Flask na Gunicornu, úlohy v APScheduleru
        │
        ├─ PostgreSQL 18 ... účty, přihlášení, platby, audit, evidence synchronizací
        ├─ SQLite .......... jeden soubor na uživatele: časové řady, normály, skóre
        └─ Open Wearables .. FastAPI + Celery + Redis: OAuth k výrobcům a sjednocení dat
```

| Vrstva | Technologie |
|---|---|
| Frontend | React 19, TypeScript 5.9, Vite 7, Tailwind 4, Recharts, Framer Motion |
| Backend | Python 3.12, Flask 3.1, Gunicorn, APScheduler |
| Data | PostgreSQL 18, SQLite (na uživatele), Redis 8 |
| Integrace | Open Wearables (FastAPI, Celery), OAuth 2.0 |
| Klienti | Webová aplikace (PWA), Android aplikace, aplikace pro hodinky Garmin |
| Platby | Stripe |
| Analytika | Umami na vlastním serveru, bez cookies |
| Infrastruktura | Docker Compose, 8 kontejnerů |

## Klíčová rozhodnutí

**Hybridní datový model.** PostgreSQL drží to, co je společné: účty, přihlášení, platby a audit. Zdravotní data každého uživatele leží v jeho vlastním souboru SQLite. Patří jednomu člověku a čtou se mnohem častěji, než zapisují, takže soubor na uživatele dává oddělení, jednoduchý export i smazání a obnovu jednoho účtu bez zásahu do ostatních. Úlohy přes všechny uživatele jdou postupně po jednom.

**Kontrakty metrik.** Každá metrika je popsaná ve vrstvách, od úložiště a signálů přes kontrakty poskytovatelů a metrik až po odpověď API. Poskytovatel se zaregistruje jednou jako `ProviderContract` a každá metrika si z kontraktů odvodí, co pro daný účet spočítá a co jí chybí.

**Synchronizace podle událostí.** Open Wearables dá aplikaci vědět, že přišla nová data, a aplikace přepočítá skóre daného uživatele. Přepočty jednoho uživatele se slučují, aby neběžely zbytečně několikrát za sebou. Co oznámení mine, zachytí hodinový průchod.

**Jedna brána pro každé nasazení.** Push do produkce projde bránou na serveru: backendové testy, testy webu, lint, kontrola typů, build a Vitest pro frontend a kontrola konfigurace. Červená brána nasazení zastaví. Workflowy v GitHub Actions jsou k dispozici pro ruční spuštění.

## Bezpečnost

- Jen HTTPS, HSTS s preload, TLS 1.2 a 1.3.
- Hesla a jednorázové kódy z e-mailu hashované přes scrypt. Tokeny uložené jako HMAC-SHA256. Tajné klíče pro TOTP jsou uložené šifrovaně.
- Dvoufázové přihlášení přes TOTP, passkeys (WebAuthn) a záložní kódy. Přístup do administrace vyžaduje druhý faktor.
- Relace na serveru v Redisu. Cookies `HttpOnly`, `Secure` a `SameSite=Lax`, CSRF token pro každou relaci.
- Limity požadavků a zámky po neúspěšných pokusech, počítané na adresu, na účet a na IP. fail2ban na přihlašovacích cestách.
- Webhooky ze Stripu ověřené podpisem a zpracované jen jednou podle id události.
- Akce v administraci se zapisují do auditního logu.
- Firewall UFW, zvenku je dostupná jen reverzní proxy. Kontejner aplikace běží pod uživatelem bez práv roota.
- Export dat a smazání účtu přímo v aplikaci (GDPR).

## Zálohy a dohled

- Každých 6 hodin šifrovaný archiv (GPG, AES-256) všech souborů SQLite a databází PostgreSQL, s kontrolou integrity, uchovaný 30 dní. Jeden archiv denně jde mimo server. Jednou měsíčně proběhne automatická zkušební obnova.
- Sentry na backendu i frontendu, upozornění přes Telegram, kontrola dostupnosti každých 5 minut a hlídání pravidelných úloh.

## Testy

- Přes 6 600 backendových testů (pytest), přes 2 500 frontendových (Vitest), 300 testů webu.
- Přes 1 700 testů v backendu Open Wearables.

## Odkazy

- Produkt: https://aiclysm.com/mybodyai/info/
- Srovnání wearables (113 zařízení): https://aiclysm.com/compare/
- Changelog: https://aiclysm.com/changelog/
