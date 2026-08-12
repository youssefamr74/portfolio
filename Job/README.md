# Jōb — AI-Assisted Job Application Tracker

> Portfolio notes for `job-backend` + `job-mobile`. Source of truth is the actual code in
> `Documents/Job/job-backend` and `Documents/Job/job-mobile` — this README describes what's
> there today, not an aspirational feature list.

## What it does

Jōb tracks a job search automatically instead of making you maintain a spreadsheet. The core
loop:

1. The user connects their Gmail account (Google OAuth).
2. The backend periodically pulls recent messages via the Gmail API (`gmail/gmail_client.py`,
   `gmail/service.py`) and runs them through a two-stage classifier:
   - **Stage 1 — cheap heuristic filter** (`classifier/heuristics.py`, `quick_filter`) throws
     out obvious non-job emails without ever calling an LLM.
   - **Stage 2 — Claude Haiku** (`classifier/service.py`) classifies anything that survives
     stage 1: is this job-related, what's the company/role, has the status changed (Applied →
     In Review → Interview → Offer/Rejected)?
3. `applications/sync_service.py` takes the classified emails, de-duplicates/matches them
   against existing applications (by thread ID, job ID, or a fingerprint of company+role), applies
   status-transition rules, and persists everything.
4. The mobile app (React Native/Expo) surfaces the results: an application list with live
   status, an analytics/insights screen, weekly-review summaries, a goals/streak tracker, and a
   "companion" feature that generates short AI-written reactions to status changes ("Offer from
   {company} 🎉") with a scripted fallback if Claude fails or times out.
5. CV upload/parsing (`cv/`) and a support/contact flow (`support/`) round out the app. Payments
   (Pro tier) go through RevenueCat, with a webhook endpoint on the backend.

This is not a toy CRUD app — it has background jobs (`APScheduler` for daily notification
checks), two-stage AI classification with real failure handling (rate limits, timeouts, bad
JSON from the model all degrade gracefully instead of crashing the sync), and an actual
`tests/` suite (`pytest`) plus Alembic migrations.

## Tech stack

**Backend** (`job-backend/`)
- FastAPI + Uvicorn, SQLAlchemy 2.0 (async) + `asyncpg`, Alembic migrations
- Postgres via Supabase (also used for auth — Supabase JWT verification)
- Anthropic Claude (Haiku) for email classification and companion-message generation
- Google OAuth + Gmail API for the mailbox sync
- APScheduler for the daily notification job
- pytest for the test suite

**Mobile** (`job-mobile/`)
- React Native + Expo (SDK ~54), React Navigation (stack + bottom tabs)
- `axios` API client with auth-token interceptor and automatic 401 refresh-and-retry
- `react-native-purchases` (RevenueCat) for subscriptions
- `expo-notifications` for push, `expo-document-picker` for CV upload
- Google Sign-In (`@react-native-google-signin/google-signin`)

## Architecture

```
                        ┌───────────────────────────┐
                        │   Gmail API (user's inbox) │
                        └─────────────┬─────────────┘
                                      │ OAuth token
                                      ▼
┌────────────────┐   HTTPS/JSON  ┌──────────────────────────────┐
│   job-mobile     │◄────────────►│         job-backend           │
│  (Expo / RN)     │   REST API   │        (FastAPI, async)       │
│                  │              │                                │
│ - Applications   │              │  auth · applications · gmail   │
│ - Insights       │              │  classifier · companion        │
│ - Goals          │              │  cv · goals · notifications    │
│ - Companion/AI   │              │  analytics · support · webhooks│
│ - CV upload      │              │                                │
└───────┬──────────┘              └───────┬───────────┬───────────┘
        │                                 │           │
        │ push notifications              │           │
        ▼                                 ▼           ▼
┌────────────────┐            ┌────────────────┐  ┌───────────────────┐
│ Expo Push /     │            │ Postgres        │  │ Anthropic Claude   │
│ RevenueCat      │            │ (via Supabase)  │  │ (Haiku) — email    │
│ (subscriptions) │            │ - users         │  │ classification +   │
└────────────────┘            │ - applications  │  │ companion messages │
                                │ - goals/streaks │  └───────────────────┘
                                └────────────────┘
```

The backend is a single FastAPI service organized by domain module (not microservices) —
`auth/`, `applications/`, `gmail/`, `classifier/`, `companion/`, `cv/`, `goals/`,
`notifications/`, `analytics/`, `support/`, `webhooks/`, each with its own `routes.py` /
`service.py` / `models.py`. Deployed on Railway.

## Interesting technical decisions worth calling out

- **Two-stage classification pipeline.** Every synced email goes through a free/local
  heuristic filter *before* it's allowed to hit the Claude API. This keeps classification
  cost roughly proportional to "emails that actually look job-related," not "every email in
  the inbox."
- **Graceful AI degradation.** `classifier/service.py` explicitly catches
  `RateLimitError`, `APITimeoutError`, `APIConnectionError`, and generic `APIError` from the
  Anthropic SDK and just skips that email rather than failing the whole sync. `companion/service.py`
  ships hand-written `FALLBACK_MOMENTS` copy for when Claude fails to produce usable output —
  the feature degrades to static (but still on-brand) text instead of breaking.
- **Idempotent Gmail sync with multi-key matching.** `sync_service.py` matches incoming
  emails against existing application records by thread ID, job ID, *and* a
  company+role fingerprint (`classifier/normalization.make_fingerprint`) so the same
  application doesn't get duplicated when Gmail threads split or a recruiter emails from a new
  thread.
- **401-aware API client.** The mobile `services/api.js` axios instance queues concurrent
  requests during a token refresh and replays them afterward, instead of firing N parallel
  refresh calls when several screens hit the API at once.

## Running it locally

**Backend**
```bash
cd job-backend
python -m venv venv && source venv/bin/activate   # or venv\Scripts\activate on Windows
pip install -r requirements.txt
cp .env.example .env   # fill in real values — see this folder's job-backend.env.example
alembic upgrade head
uvicorn main:app --reload
```

**Mobile**
```bash
cd job-mobile
npm install
cp .env.example .env   # fill in real values — see this folder's job-mobile.env.example
npx expo start
```

The mobile app's `services/api.js` currently points `baseURL` at the deployed Railway
instance — for fully local development, point it at your local backend
(`http://localhost:8000`) instead.

## Configuration

See `job-backend.env.example` and `job-mobile.env.example` in this folder for every
environment variable the two apps actually read (names taken straight from
`job-backend/config.py` and the real `.env` files), with placeholder values only. The real
`.env` files (with live Supabase/Anthropic/Google/RevenueCat credentials) were **not** copied
into this portfolio folder.
