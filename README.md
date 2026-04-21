# MyBodyAI

**AI-powered health analytics platform** — turns wearable data into actionable daily guidance: illness prediction, biological age, training readiness, seasonal pattern detection.

🌐 **Live:** https://aiclysm.com/mybodyai/info/
📧 **Contact:** vaclav.frcek@aiclysm.com

<a href="https://aiclysm.com/">
  <img src="https://aiclysm.com/img/dashboard-preview.jpg" alt="MyBodyAI Dashboard — 13 health indices, Body Status, biological age" width="820">
</a>

> 🇨🇿 Česká verze níže / [Czech version below](#czech-version)

---

## What it does

- **Body Status** — single-verdict daily state from 13 health indices (Alert → Recovery → Steady → Strong → Peak)
- **Illness Risk** — AI prediction based on HRV drift, RHR rise, SpO₂ and respiratory-rate deviations — often days before symptoms appear
- **Biological Age** — 8-domain composite with age + gender adjustment, capped ±12 years with dampening
- **Training Readiness** — industry-aligned (Garmin TR / WHOOP Recovery / Oura Readiness)
- **Pattern detection** — seasonal and cross-metric correlations across 30+ days
- **Bilingual** (English, Czech; all Czech copy uses formal address)

**Four wearable providers live** — Polar, Fitbit, Withings, Strava. **Garmin** is in API onboarding (fifth total).

---

## Architecture at a glance

```text
┌──────────────────────────────────────────────────────────────────┐
│                         Caddy (TLS 1.3)                           │
│   aiclysm.com (website) │ /mybodyai/* (SPA) │ /api/* (Flask)      │
└──────────────────────────────────────────────────────────────────┘
          │                      │                      │
          ▼                      ▼                      ▼
   Static HTML            React 19 + TypeScript   Flask + Gunicorn
   (65 pages)             Vite / Tailwind 4       Python 3.12
                          Framer Motion           APScheduler
                                                          │
            ┌─────────────────────────────────────────────┤
            ▼                                             ▼
    PostgreSQL 18 (shared)              SQLite (per-user, 17 tables)
    users, auth, settings,              daily_summary (43 cols),
    pending registrations,              hrv_daily, activities,
    app config, webhook audit           health_scores, baseline,
            │                           personal distributions,
            ▼                           achievements, self_reports
    Open Wearables platform
    (ow-backend + Celery workers + Redis)
    OAuth + provider normalization
```

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript 5.9, Vite 7, Tailwind 4, Three.js, Recharts, Framer Motion |
| Backend | Python 3.12, Flask, Gunicorn, APScheduler |
| Data | PostgreSQL 18 (shared), SQLite (per-user), Redis (cache + Celery broker) |
| Integrations | Open Wearables platform, OAuth 2.0 for 5 providers |
| Payments | Stripe (webhook-driven) |
| Email | SMTP + Jinja2 templates |
| Analytics | Umami (self-hosted) |
| Proxy / TLS | Caddy with automatic HTTPS |
| Infrastructure | Docker Compose, 8 orchestrated containers |
| Testing | pytest (1200+ tests), Vitest (170+ tests) |
| Monitoring | Telegram error alerts, structured logging |

---

## Key engineering decisions

### Hybrid data model
- **PostgreSQL** for shared identity, auth, app configuration, audit trails
- **Per-user SQLite files** for health time-series (43-column daily summaries, HRV history, scores, achievements)
- Rationale: health data is user-bound, mostly read-heavy, benefits from per-user isolation — trivial backup/export, no cross-tenant contention, atomic per-user rollback

### 6-layer metric contract system
Every metric passes through explicit layers:

```
L0 storage   → L1 signal concepts → L2 provider contracts
L3 metric contracts → L4 availability snapshot → L5 API response
```

Adding a new provider is a **1-file change** — `contracts/providers.py` registers a `ProviderContract`. Every existing metric automatically gains availability inference, coverage suggestions and status badges. Zero modifications elsewhere.

### Event-driven sync
- Webhook-triggered rebuilds on `POST /api/internal/webhook-notify` (HMAC-signed from Open Wearables)
- Per-user 60-second debounce prevents redundant scoring runs
- Background schedulers run digests, backups, stuck-sync GC, freshness audits

### Zero-downtime deployment
- **Pre-push hook** runs `tsc --noEmit`, `npm run build`, full test suite — push is blocked on failure
- Frontend `dist/` is bind-mounted into Caddy — UI deploys are static file swaps, no backend restart
- Backend deploys copy specific files only (never `/app/` wholesale) — preserves user data volumes

### Inverted display convention
Risk/load metrics (`illness_risk`, `stress_load`, `overtraining`) show **raw** values with inverted color scale. `14/100 illness_risk` displays as "14" with a short green bar — no cognitive math on "is 94 good or bad?".

---

## Infrastructure

8 containers (all health-checked):

| Container | Role |
|---|---|
| `mybodyai` | Flask API + SPA shell |
| `ow-backend` | Open Wearables backend (OAuth + data ingestion) |
| `ow-celery-worker` | Async wearable sync jobs |
| `ow-celery-beat` | Periodic sync scheduler |
| `ow-db` | PostgreSQL 18 |
| `ow-redis` | Redis 8 (cache + broker) |
| `caddy` | Reverse proxy + automatic TLS |
| `umami` | Self-hosted analytics |

**APScheduler jobs (`mybodyai`):**
- Weekly digest emails
- 6-hour PostgreSQL backups (30-day retention)
- Stuck-sync GC
- Premium data freshness audit (every 6 hours)
- System health check (SQLite, disk, pool)
- Reconcile & reminder queues

---

## Security posture

- TLS 1.3, HTTP/2, HSTS preload
- Scrypt-hashed OTPs (no plain-text storage at any stage)
- CSRF token (JSON body + custom header)
- Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`
- Anti-enumeration on auth endpoints (no `exists` field on login errors)
- 128-character `SECRET_KEY`, non-root Docker user
- 100% admin audit logging (24/24 POST actions logged with user, timestamp, payload hash)
- Jinja2 autoescape, rate limiting per IP (real client IP via `X-Forwarded-For`)
- UFW host firewall, PostgreSQL not exposed to host network
- GDPR-compliant data export (`/api/user/data/export`) and user-initiated account deletion

---

## Engineering practices

- **Contract-driven architecture** — metric contracts, provider contracts, signal availability; every layer has an explicit schema
- **i18n everywhere** — 575+ translation keys (EN + CS), `data-en` / `data-cs` system on static pages; Czech uses formal address (vykání) consistently
- **Accessibility** — WCAG 2.1 AA; keyboard navigation, focus traps, ARIA labels throughout; `prefers-reduced-motion` respected
- **SEO** — 65 pages with JSON-LD structured data, `hreflang`, canonical URLs, sitemap auto-maintained (append-only, never delete URLs)
- **Observability** — Sentry for unhandled exceptions, Telegram error alerts (filtered to real errors, not info noise), structured `metric_contract` INFO log per metric evaluation (PII-free)
- **CI / Deploy discipline** — GitHub Actions workflow for automated tests; pre-push hook enforces `tsc --noEmit`, `npm run build` and the full test suite — push is blocked on failure
- **PWA** — installable, service worker, offline fallback
- **Prerendering** — 5 high-SEO-value SPA routes prerendered at build time for Googlebot + LCP

---

## Numbers

- **17,000** LOC Python backend (77 files)
- **15,000** LOC React frontend
- **65** static HTML pages (website)
- **81** API routes across 7 Flask blueprints
- **13** health indices + composite Body Status + Biological Age (8 domains)
- **26** achievements, **10** user ranks, **5** Body Status states
- **1,200+** unit & integration tests (backend), **170+** (frontend)

---

## Status

Currently in **beta (v0.98)** with **35+** registered users across Polar, Fitbit, Withings and Strava. Rapid 2026 iteration cycle — commits pushed daily, every push runs full tests then deploys via pre-push hook. Solo-built, production-grade.

Live demo (read-only landing): https://aiclysm.com/mybodyai/info/
Changelog: https://aiclysm.com/changelog/

---

📧 vaclav.frcek@aiclysm.com

<a name="czech-version"></a>

---

# 🇨🇿 Česky

**AI platforma pro zdravotní analytiku** — převádí data z chytrých zařízení do každodenních doporučení: predikce nemoci, biologický věk, tréninková připravenost, detekce vzorců.

🌐 **Živé:** https://aiclysm.com/mybodyai/info/
📧 **Kontakt:** vaclav.frcek@aiclysm.com

## Co to dělá

- **Body Status** — jeden denní verdikt ze 13 zdravotních indexů (Varování → Regenerace → Stabilní → Silný den → Špičkový den)
- **Riziko nemoci** — AI predikce na základě poklesu HRV, růstu klidového tepu, SpO₂ a dechové frekvence — často několik dní před příznaky
- **Biologický věk** — kompozit z 8 domén s věkovou a genderovou korekcí, omezen ±12 let s tlumením
- **Tréninková připravenost** — v souladu s průmyslovým standardem (Garmin TR / WHOOP Recovery / Oura Readiness)
- **Detekce vzorců** — sezónní a multi-metrické korelace za 30+ dní dat
- **Dvoujazyčné** (angličtina, čeština; čeština důsledně vyká)

**Čtyři poskytovatelé aktivní** — Polar, Fitbit, Withings, Strava. **Garmin** je v procesu napojení (pátý celkem).

## Architektura

```text
┌──────────────────────────────────────────────────────────────────┐
│                         Caddy (TLS 1.3)                           │
│   aiclysm.com (web) │ /mybodyai/* (SPA) │ /api/* (Flask)          │
└──────────────────────────────────────────────────────────────────┘
          │                      │                      │
          ▼                      ▼                      ▼
   Statické HTML           React 19 + TypeScript   Flask + Gunicorn
   (65 stránek)            Vite / Tailwind 4       Python 3.12
                           Framer Motion            APScheduler
                                                          │
            ┌─────────────────────────────────────────────┤
            ▼                                             ▼
    PostgreSQL 18 (sdílená)            SQLite (per-user, 17 tabulek)
    users, auth, nastavení,            daily_summary (43 sloupců),
    čekající registrace,               hrv_daily, activities,
    app config, webhook audit          health_scores, baseline,
            │                          personální distribuce,
            ▼                          achievements, self_reports
    Open Wearables platforma
    (ow-backend + Celery workers + Redis)
    OAuth + normalizace dat z poskytovatelů
```

## Tech stack

| Vrstva | Technologie |
|---|---|
| Frontend | React 19, TypeScript 5.9, Vite 7, Tailwind 4, Three.js, Recharts, Framer Motion |
| Backend | Python 3.12, Flask, Gunicorn, APScheduler |
| Data | PostgreSQL 18 (sdílená), SQLite (per-user), Redis (cache + Celery broker) |
| Integrace | Open Wearables platforma, OAuth 2.0 pro 5 poskytovatelů |
| Platby | Stripe (webhook-driven) |
| Email | SMTP + Jinja2 šablony |
| Analytika | Umami (self-hosted) |
| Proxy / TLS | Caddy s automatickým HTTPS |
| Infrastruktura | Docker Compose, 8 orchestrovaných kontejnerů |
| Testy | pytest (1200+ testů), Vitest (170+ testů) |
| Monitoring | Telegram error alerty, strukturované logy |

## Klíčová inženýrská rozhodnutí

### Hybridní datový model
- **PostgreSQL** pro sdílenou identitu, autentizaci, app config, audit trails
- **Per-user SQLite soubory** pro zdravotní časové řady (43 sloupců v daily summary, historie HRV, skóre, achievementy)
- Důvod: zdravotní data patří uživateli, čtení dominuje, izolace per-user zjednodušuje zálohování/export, žádný cross-tenant lock contention, atomický rollback per uživatel

### 6-vrstvý kontraktní systém metrik
Každá metrika prochází explicitními vrstvami:

```
L0 úložiště  → L1 signal concepts → L2 provider contracts
L3 metric contracts → L4 availability snapshot → L5 API response
```

Přidání nového poskytovatele je **změna 1 souboru** — `contracts/providers.py` registruje `ProviderContract`. Každá existující metrika automaticky získá inference dostupnosti, coverage suggestions a status badges. Žádné úpravy jinde.

### Event-driven sync
- Webhook-triggered rebuildy na `POST /api/internal/webhook-notify` (HMAC-podepsané z Open Wearables)
- Per-user 60sekundový debounce brání redundantním scoring runs
- Plánovače na pozadí spouští digest, zálohy, stuck-sync GC, freshness audity

### Zero-downtime deployment
- **Pre-push hook** spouští `tsc --noEmit`, `npm run build`, celou test suite — push je blokován při selhání
- Frontend `dist/` je bind-mount do Caddy — UI deploys jsou výměna statických souborů, žádný backend restart
- Backend deploys kopírují jen konkrétní soubory (nikdy celé `/app/`) — chrání user data volumes

### Invertovaná konvence zobrazení
Metriky typu riziko/zátěž (`illness_risk`, `stress_load`, `overtraining`) zobrazují **syrové** hodnoty s invertovanou barevnou škálou. `14/100 illness_risk` se zobrazí jako „14" s krátkým zeleným barem — žádný kognitivní výpočet „je 94 dobré nebo špatné?".

## Infrastruktura

8 kontejnerů (všechny health-checked):

| Kontejner | Role |
|---|---|
| `mybodyai` | Flask API + SPA shell |
| `ow-backend` | Open Wearables backend (OAuth + data ingestion) |
| `ow-celery-worker` | Asynchronní jobs pro sync |
| `ow-celery-beat` | Periodic sync scheduler |
| `ow-db` | PostgreSQL 18 |
| `ow-redis` | Redis 8 (cache + broker) |
| `caddy` | Reverse proxy + automatický TLS |
| `umami` | Self-hosted analytika |

**APScheduler úlohy (`mybodyai`):**
- Týdenní digest emaily
- 6hodinové PostgreSQL zálohy (30denní retence)
- Stuck-sync GC
- Audit čerstvosti dat u premium uživatelů (každých 6 h)
- System health check (SQLite, disk, pool)
- Reconcile a reminder fronty

## Bezpečnost

- TLS 1.3, HTTP/2, HSTS preload
- OTP hashované scrypt (žádné plain-text ukládání v žádné fázi)
- CSRF token (JSON body + custom header)
- Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`
- Anti-enumeration na auth endpointech (žádné `exists` pole v chybách loginu)
- 128znakový `SECRET_KEY`, non-root Docker uživatel
- 100% admin audit logování (24/24 POST akcí s uživatelem, timestamp, hash payloadu)
- Jinja2 autoescape, rate limiting per IP (real client IP přes `X-Forwarded-For`)
- UFW host firewall, PostgreSQL mimo host network
- GDPR-compliant export dat (`/api/user/data/export`) a user-initiated mazání účtu

## Inženýrské praktiky

- **Kontraktně orientovaná architektura** — metric contracts, provider contracts, signal availability; každá vrstva má explicitní schema
- **i18n všude** — 575+ překladových klíčů (EN + CS), `data-en` / `data-cs` systém na statických stránkách; čeština konzistentně vyká
- **Přístupnost** — WCAG 2.1 AA; klávesová navigace, focus traps, ARIA labels; respekt pro `prefers-reduced-motion`
- **SEO** — 65 stránek s JSON-LD structured data, `hreflang`, kanonické URL, sitemap auto-udržovaný (append-only, URL se nikdy nemažou)
- **Observabilita** — Sentry pro neošetřené výjimky, Telegram error alerty (filtrované na skutečné chyby), strukturované `metric_contract` INFO logy per vyhodnocení (bez PII)
- **CI / deploy disciplína** — GitHub Actions workflow pro automatické testy; pre-push hook vynucuje `tsc --noEmit`, `npm run build` a celou test suite — push je blokován při selhání
- **PWA** — instalovatelná, service worker, offline fallback
- **Prerendering** — 5 SPA cest s vysokou SEO hodnotou prerendered při buildu pro Googlebota + LCP

## Čísla

- **17 000** LOC Python backend (77 souborů)
- **15 000** LOC React frontend
- **65** statických HTML stránek (web)
- **81** API routes napříč 7 Flask blueprinty
- **13** zdravotních indexů + kompozitní Body Status + Biologický věk (8 domén)
- **26** achievementů, **10** uživatelských hodností, **5** stavů Body Status
- **1 200+** unit a integration testů (backend), **170+** (frontend)

## Stav

Aktuálně v **beta verzi (v0.98)** s **35+** registrovanými uživateli napříč Polar, Fitbit, Withings a Strava. Rychlý iterační cyklus 2026 — commity denně, každý push spouští celou test suite a pak deploy přes pre-push hook. Solo-built, production-grade.

Živé demo (read-only úvod): https://aiclysm.com/mybodyai/info/
Changelog: https://aiclysm.com/changelog/

---

📧 vaclav.frcek@aiclysm.com
