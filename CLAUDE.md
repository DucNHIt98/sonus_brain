<!-- GSD:project-start source:PROJECT.md -->
## Project

**Sonus API**

Sonus API là backend Django REST Framework (DRF) phục vụ toàn bộ tính năng của ứng dụng nghe nhạc Sonus (Flutter). Backend này thay thế hoàn toàn Supabase SDK phía client, đóng vai trò proxy + cache cho các nguồn nhạc bên ngoài (YouTube, Deezer, NCT, Jamendo), quản lý dữ liệu người dùng, và xử lý thanh toán subscription Premium qua Stripe.

**Core Value:** Flutter app chỉ cần gọi một backend duy nhất — mọi logic về audio, user data, recommendations, và payment đều do DRF xử lý server-side.

### Constraints

- **Tech stack:** Python/Django + DRF, không thay đổi
- **Database:** Kết nối vào PostgreSQL của Supabase (giữ nguyên schema hiện có, mở rộng nếu cần)
- **Auth hybrid:** Google OAuth flow vẫn qua Supabase, DRF verify JWT; email/password hoàn toàn qua DRF (djangorestframework-simplejwt)
- **Third-party:** YouTube/NCT/Deezer/Jamendo cần được proxy qua backend (tránh expose API keys ở client)
- **Payment:** Stripe (không dùng các gateway khác)
- **Compatibility:** API response phải tương thích với các model hiện có trong Flutter app (MusicModel, Home, PlaylistModel, etc.)
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Framework
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Python | 3.12.x | Runtime | LTS support through 2028; significant perf gains over 3.10; best async support; Django 5.x requires 3.10+ |
| Django | 5.1.x | Web framework | Current stable LTS-adjacent; Django 5.0 dropped Python 3.9; Django 5.1 adds async ORM improvements relevant to DB-heavy workloads |
| djangorestframework | 3.15.x | REST API layer | Latest stable as of mid-2025; adds Django 5.x support, improved schema generation |
### Authentication
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| djangorestframework-simplejwt | 5.3.x | JWT access + refresh tokens | De-facto standard for DRF JWT; actively maintained; supports token blacklisting for logout; token rotation for refresh; sliding tokens optional |
| PyJWT | 2.8.x | JWT decode (Supabase token verification) | simplejwt uses PyJWT internally; also used directly to verify Supabase-issued JWTs for Google OAuth path |
- Email/password: simplejwt issues tokens. `TokenObtainPairView`, `TokenRefreshView`, `TokenBlacklistView`.
- Google OAuth: Supabase handles the OAuth redirect/callback. Flutter sends the Supabase JWT to DRF. DRF verifies it with PyJWT using Supabase's JWT secret (from env: `SUPABASE_JWT_SECRET`). On first verified request, DRF upserts the user row and issues its own simplejwt pair.
- Do NOT let Supabase JWTs be used directly for DRF authentication permanently — they expire on Supabase's schedule and carry Supabase-specific claims. Always exchange for a DRF JWT immediately.
### Database
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| psycopg2-binary | 2.9.x | PostgreSQL adapter | `psycopg2-binary` for dev/CI simplicity; use `psycopg2` (without binary) in production Docker for portability. psycopg3 (`psycopg`) is stable but requires more migration effort — not worth it yet |
| dj-database-url | 2.1.x | Parse `DATABASE_URL` env var | Single env var `postgres://user:pass@host:5432/dbname` covers Supabase's connection string format cleanly |
### Async Task Queue
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Celery | 5.3.x | Async/background tasks | yt-dlp URL resolution, Gemini API calls, NCT scraping — all slow (1-10s); must not block request/response cycle |
| redis | 5.x (py library) | Celery broker + result backend | `redis` Python client v5 is async-capable; use as Celery broker AND Django cache backend to minimize infra |
| django-celery-results | 2.5.x | Store Celery results in Django DB | Optional — allows querying task status from DRF endpoint; useful for polling audio URL resolution |
| celery[redis] | — | Install alias | `pip install celery[redis]` installs redis transport |
# Two queues — fast vs slow
### Caching
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| django-redis | 5.4.x | Redis as Django cache backend | Single Redis instance serves both Celery broker and Django cache; django-redis is the standard adapter |
| Redis | 7.x (server) | Cache store | Redis 7 adds ACLs, better memory management; use Redis Cloud free tier or Railway Redis for dev |
| Data | TTL | Rationale |
|------|-----|-----------|
| YouTube resolved stream URL | 4 hours | yt-dlp URLs expire; 4h is safe window |
| Deezer track metadata | 24 hours | Rarely changes |
| Gemini recommendations | 1 hour | Semi-personalized; short TTL acceptable |
| NCT scrape results | 6 hours | NCT HTML rarely changes intra-day |
| Jamendo search results | 12 hours | Static catalog |
| Home feed (charts) | 30 minutes | Freshness matters for "trending" |
### Audio Source Integration
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| yt-dlp | latest (pin weekly) | YouTube + fallback audio URL extraction | youtube-dl is dead; yt-dlp is the maintained fork; YouTube changes formats frequently — pin to latest, not a specific version; update weekly via CI |
| httpx | 0.27.x | Async HTTP client for Deezer + Jamendo APIs | Preferred over `requests` for async-capable code; works in both sync and async contexts; connection pooling |
| beautifulsoup4 | 4.12.x | NCT web scraping | Standard choice; pair with `lxml` parser for speed |
| lxml | 5.x | HTML parser for bs4 | Faster than html.parser; required for complex NCT page structure |
# music/tasks.py
### Stripe Integration
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| stripe | 10.x | Stripe Python SDK | Stripe SDK v10 uses typed responses; supports async; webhook signature verification built-in |
# payments/views.py
### AI Integration
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| google-generativeai | 0.8.x | Gemini API client | Official Google SDK; Gemini 1.5 Flash is cost-effective for recommendations; responses cacheable |
### HTTP + CORS
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| django-cors-headers | 4.3.x | CORS for Flutter client | Standard DRF companion; supports `CORS_ALLOWED_ORIGINS` list or `CORS_ALLOW_ALL_ORIGINS=True` for dev |
# settings.py
# For mobile Flutter (iOS/Android), CORS does not apply — native HTTP clients
# don't enforce CORS. Only needed for Flutter Web.
# Still set it correctly — defense in depth.
### Rate Limiting
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| django-ratelimit | 4.1.x | Per-view rate limiting | Lightweight; Redis-backed or cache-backed; ideal for protecting third-party proxy endpoints |
| Endpoint | Limit | Window | Rationale |
|----------|-------|--------|-----------|
| `/music/resolve/youtube/` | 10 req | per user per minute | yt-dlp is CPU-heavy; prevent abuse |
| `/music/search/` | 30 req | per user per minute | Multi-source search; expensive |
| `/ai/recommendations/` | 5 req | per user per minute | Gemini API cost control |
| `/auth/token/` (login) | 5 req | per IP per minute | Brute-force protection |
### Dev + Operations
| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| gunicorn | 22.x | WSGI server | Standard for Django in production; pair with gevent worker class for better concurrency during Celery dispatch waits |
| python-decouple | 3.8.x | Env var management | Cleaner than `os.environ.get`; reads from `.env` in dev, real env in prod |
| django-extensions | 3.2.x | Dev utilities | `shell_plus`, `runscript`, graph models — essential for DRF development |
| pytest-django | 4.8.x | Testing | pytest ecosystem is preferred over unittest for DRF; fixtures, parametrize, better output |
| factory-boy | 3.3.x | Test fixtures | Model factories for test data generation; integrates with pytest-django |
| Flower | 2.x | Celery monitoring | Web UI for Celery task queue — critical for debugging yt-dlp task failures |
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Auth JWT | simplejwt | dj-rest-auth | dj-rest-auth wraps simplejwt + adds social auth but adds complexity; hybrid Supabase auth is custom anyway |
| Auth JWT | simplejwt | jose / python-jose | Less Django integration; simplejwt is purpose-built for DRF |
| HTTP client | httpx | requests | `requests` has no native async support; httpx is a drop-in with async capability needed for potential ASGI migration |
| Async tasks | Celery | Django Q2 | Celery has better ecosystem, Flower monitoring, mature retry semantics; Django Q2 is lighter but less proven at scale |
| Async tasks | Celery | Huey | Huey is simpler but lacks advanced routing needed for slow/fast queue split |
| Cache | Redis | Database cache | DB cache (Django's built-in `DatabaseCache`) works but adds query load to Supabase remote DB; Redis is faster and keeps cache traffic local |
| Cache | Redis | Memcached | Memcached can't serve as Celery broker; Redis does both with one infra component |
| Scraping | beautifulsoup4 + lxml | Playwright | NCT structure is server-rendered HTML; bs4 + lxml is 10x simpler and faster for this use case; Playwright only if NCT uses heavy JS rendering |
| WSGI server | gunicorn | uvicorn | uvicorn is ASGI; Django 5 supports ASGI but DRF is synchronous by default; ASGI migration would require async views throughout — not justified yet |
## Settings Architecture
## Installation
# Core framework
# Auth
# Database
# Async tasks + cache
# Audio + scraping
# Payments + AI
# HTTP + CORS + rate limiting
# Operations
# Dev + testing
# requirements.txt
## Sources
- Training knowledge through August 2025 — no live documentation access during this session
- **MEDIUM confidence on exact versions** — run `pip index versions <package>` to verify current patch versions before committing requirements.txt
- Django release schedule: https://www.djangoproject.com/download/ (verify Django 5.1 vs 5.2 status)
- simplejwt changelog: https://django-rest-framework-simplejwt.readthedocs.io/en/latest/changelog.html
- Supabase direct connection docs: https://supabase.com/docs/guides/database/connecting-to-postgres
- yt-dlp: https://github.com/yt-dlp/yt-dlp (check for breaking changes before updating)
- Stripe webhook guide: https://stripe.com/docs/webhooks
- Celery 5 docs: https://docs.celeryq.dev/en/stable/
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
