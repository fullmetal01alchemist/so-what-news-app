---
date: 2026-05-26T20:43:00+05:30
researcher: Anjay Sahoo
project: SoWhat News App
topic: "MVP Tech Stack, Hosting, and System Architecture (free tier up to 5K users)"
tags:
  [
    system-design,
    mvp,
    tech-stack,
    architecture,
    newsdata.io,
    llm,
    vercel,
    cloudflare,
    railway,
    supabase,
    supabase-auth,
    google-oauth,
    fcm,
    revenuecat,
    gemini,
    hono,
  ]
status: complete
last_updated: 2026-05-26
last_updated_by: Anjay Sahoo
last_updated_note: "Added Vercel vs Cloudflare vs Railway comparison, switched to Supabase Auth (Google + email) with RevenueCat subscriptions, FCM daily push, multi-country MVP, English-only LLM prompt with Inshort-style 60-word summaries, and persona-edit cache invalidation. Resolved the Section 14 open questions."
---

# Research: SoWhat News — MVP Tech Stack, Hosting & System Architecture

**Date**: 2026-05-26 21:15 IST (revised)
**Researcher**: Anjay Sahoo
**Project**: SoWhat — Contextual Utility News Engine (Android, Compose Multiplatform)
**Goal**: Decide a backend tech stack and a hosting platform that stays inside free tiers for up to ~5,000 MVP users, can scale beyond that, and uses `newsdata.io` efficiently.

> **Revision 2 (2026-05-26 21:15 IST):** Adds (1) a deep Vercel vs Cloudflare Workers vs Railway comparison, (2) full authentication via **Supabase Auth** (Google OAuth + email magic link) so we can drive subscriptions, (3) **RevenueCat + Google Play Billing** as the path to "ask user to subscribe to our plan", (4) **multi-country MVP** with daily **FCM push at 8 AM IST** and English-only LLM output styled like Inshorts (10-word headline + 60-word summary), and (5) cache-invalidation rules for mid-day persona edits. The original anonymous-JWT design from Revision 1 is now an *optional* fallback, not the primary auth path.

---

## 1. Research Question

> Pick the best backend tech stack + hosting platform that is **free up to 5,000 MVP users**, supports our three core features (persona onboarding, top‑5 personalised news/day, bookmarks), optimises `newsdata.io` API usage (free tier only), and **scales gracefully** beyond MVP. The user's initial guess is Node.js + "OpenAPI" + Vercel AI SDK; challenge it where needed.

---

## 2. TL;DR — Final Recommended Stack

| Layer                       | Recommendation                                                                                                                                               | Why                                                                                                                                                                                            |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mobile                      | Compose Multiplatform (Android first)                                                                                                                        | Given                                                                                                                                                                                          |
| API runtime                 | **Node.js 22 LTS + Hono**                                                                                                                                    | Edge/Node portable, ~12 kB, ~128k RPS on Node 24, smallest cold starts; future-proofs us for Cloudflare Workers/Vercel Edge                                                                    |
| API contract                | **OpenAPI 3.1** spec → generate Kotlin client for Compose app via `openapi-generator` + `@hono/zod-openapi` on the server                                    | Single source of truth; type-safe mobile client                                                                                                                                                |
| **Auth (primary)**          | **Supabase Auth — Google OAuth + Email Magic Link**, with optional **anonymous sign-in** for first-open                                                      | Free for 50K MAU, ships Google/Apple/Email out of the box, gives us a stable `user_id` we can attach a paid subscription to. Anonymous sign-in lets users skip the wall and convert later.     |
| **Subscriptions (Android)** | **RevenueCat + Google Play Billing v8** (free up to **$2,500/mo MTR**, then 1% of MTR)                                                                       | Required: Google forces Play Billing for digital subscriptions on Android. RevenueCat handles entitlements, webhooks, receipt validation, and gives us a server-verified `is_pro` flag.        |
| **Push notifications**      | **Firebase Cloud Messaging (FCM)** + nightly cron at 02:30 UTC (= 08:00 IST)                                                                                 | FCM is free with no daily message cap (project rate-limit is 600K msgs/min — irrelevant at MVP scale). Native Compose support.                                                                 |
| Database                    | **Supabase (Postgres) free tier**                                                                                                                            | 500 MB DB + 50 K MAU + 1 GB file storage; covers DB, **Auth**, and storage in one                                                                                                              |
| Cache / rate-limit          | **Upstash Redis free tier** (10 K commands/day, 256 MB, REST API)                                                                                            | Serverless-safe Redis for cached personalised feeds, dedupe keys, rate limits, **persona-edit cache invalidation**                                                                             |
| Background / cron           | **GitHub Actions cron** (free) for ingestion + rewrites + 8 AM IST push fan-out                                                                              | Vercel Hobby cron is capped at **1 fire/day** — not enough for 4-6h news refresh. Cloudflare Workers cron is free (5 triggers, 1-min granularity) but constrained by 10 ms CPU on free tier.   |
| LLM (impact rewriter)       | **Google Gemini 2.5 Flash** primary, **Groq Llama 3.1 8B Instant** fallback (English-only output, Inshort-style: ≤10-word headline + ~60-word summary)       | Both free; Gemini 1500 RPD + 1M TPM, Groq 14,400 RPD; via Vercel AI SDK. Multi-country fan-out fits if we batch + load-balance across the two.                                                 |
| Hosting (API)               | **Vercel Hobby** (Node functions) for API; **GitHub Actions** for cron jobs                                                                                  | Vercel wins on DX, cold-start, and Node compat. Cloudflare Workers free is cheaper at the limit but the **10 ms CPU/request** ceiling is too tight for our LLM-orchestrating endpoints. Railway has no real free tier in 2026 ($5/mo Hobby minimum). See §9. |
| Observability               | Vercel logs + Supabase logs + Better Stack free tier (or Axiom free)                                                                                         | Enough for MVP; swap to OTel + Grafana Cloud later                                                                                                                                             |

> **Scaling path:** the same code runs unchanged on Cloud Run / Fly.io / Railway Hobby when we outgrow Vercel Hobby. The only swaps you'll likely make in year 2: newsdata.io Basic ($199.99/mo), Supabase Pro ($25/mo), Vercel Pro ($20/mo) — total ≤ $250/mo to support tens of thousands of DAU. Subscription revenue at 1-2% conversion of 5K MAU at ₹99/mo easily covers this from month 3.

---

## 3. Critique of Your Initial Stack Guess

You proposed: **Node.js + OpenAPI + Vercel AI SDK**.

- **Node.js**: Agreed. Mature ecosystem, strong AI SDK support, free hosts everywhere.
- **"OpenAPI"**: I'm reading this two ways:
  - If you meant **OpenAPI spec (API description)** → keep it; it gives you typed clients for the Compose app.
  - If you meant **OpenAI (LLM)** → swap it. OpenAI does not offer a free production tier in 2026. Vercel AI SDK is provider-agnostic, so use **Gemini 2.5 Flash** (free, 1500 req/day, 1M TPM) as default and **Groq** (free, ~14.4K req/day on Llama 3.1 8B Instant) as fallback.
- **Vercel AI SDK**: Agreed. Best abstraction for streaming + tool calls + provider swap.

What's missing from your guess (and added in this revision):

- A **database** (Supabase Postgres).
- An **identity layer** with Google + Email login (Supabase Auth) — needed because we want to drive subscriptions, and the Play Store will refuse "subscribe" prompts to anonymous users.
- A **subscription pipeline** for Android (RevenueCat + Play Billing).
- A **push channel** for the daily 8 AM IST notification (Firebase Cloud Messaging).
- A **cache** for personalised feeds with explicit **invalidation hooks** when a persona is edited mid-day (Upstash Redis).
- A **scheduler** that's actually capable of running multiple times per day on Vercel Hobby (GitHub Actions; Vercel cron caps at 1/day on Hobby).
- A **framework** on top of Node (Hono).

---

## 4. The Hard Constraints (Free Tier Math)

These are the numbers your design has to fit inside.

### 4.1 newsdata.io Free Plan
Source: newsdata.io pricing & credit-consumption docs (verified May 2026).

| Limit | Value |
|---|---|
| Credits per day | **200** |
| Articles per request | **10** (free) |
| Total articles/day theoretical max | **2,000** |
| Data delay | **12 hours** |
| Keyword search limit | 100 chars |
| Countries/categories/languages per query | up to **5** each |
| Commercial use | **Allowed** on free plan |
| Full article body | **Not** available on free plan (titles + summaries only) |

**Implication:** never call the news API from a user request path. We must do **centralised ingestion** and serve from our DB.

### 4.2 Vercel Hobby (verified May 2026)
| Resource | Limit |
|---|---|
| Function invocations | 1,000,000 / month |
| Active CPU | 4 CPU-hours / month |
| Provisioned memory | 360 GB-hours / month |
| Function max duration | 300 s (with Fluid Compute) |
| Function memory | 2 GB / 1 vCPU fixed |
| **Cron jobs frequency** | **1 fire / day max** ← this is the killer |
| Cron jobs per project | 100 |

### 4.3 Supabase Free
- 500 MB Postgres, 1 GB file storage
- 50 K monthly active auth users (perfect for 5K MVP)
- 500 K edge-function invocations / month
- **Paused after 7 days inactivity** — irrelevant once we have any traffic; a `/health` ping from any cron prevents pause anyway.

### 4.4 Gemini 2.5 Flash Free
- **1,500 requests/day** per project, **1,000,000 tokens/min**, 15 RPM
- No credit card required, permanent
- Trade-off: Google may use free-tier prompts for training (do not send PII; we don't, since persona is anonymous)

### 4.5 Groq (Llama 3.1 8B Instant) Free
- **14,400 RPD**, 6,000 TPM, 30 RPM (per-org)
- OpenAI-compatible API → drop-in via Vercel AI SDK
- Use as overflow / fallback for Gemini

### 4.6 Upstash Redis Free
- 256 MB, 10,000 commands/day, REST API (works in any serverless runtime)
- 14-day idle archive (with full restore) — non-issue with live traffic

### 4.7 Google Cloud Run Free (alternative compute)
- 2,000,000 requests / month
- 360,000 GiB-s memory, 180,000 vCPU-s
- Only in `us-central1`, `us-east1`, `us-west1`
- No daily-cron restriction; use Cloud Scheduler (3 free jobs/month) to fire as often as needed.

### 4.8 Cloudflare Workers Free (alternative compute, evaluated in §9)
- **100,000 requests/day** (~3M/month, conditional on bursts ≤ 1000 req/min)
- **CPU time: 10 ms per invocation** ← the critical ceiling for us
- 50 sub-requests/request (limits how many DB / LLM / 3rd-party calls one Worker can chain)
- **5 cron triggers/account** (1-min granularity), 10 ms CPU per cron tick
- **D1 (SQLite)**: 5 M rows read/day, 100 K rows write/day, 5 GB total per account, 500 MB/database
- **Workers KV**: 100 K reads, 1 K writes/day, 1 GB stored
- 128 MB memory, 3 MB worker bundle size

### 4.9 Railway Free / Trial (May 2026)
- **Trial:** one-time $5 credit, 30 days, 2 vCPU + 1 GB RAM, 5 services
- **Free plan ($0/mo):** $1/mo non-rollover credit, **1 vCPU + 0.5 GB RAM**, 1 project, 3 services, **no cron jobs**, community support only
- **Hobby:** $5/mo minimum spend (you pay $5 even if you use nothing) — not a free tier
- Net effect: Railway is **not viable for a free MVP** in 2026. The Free plan can host only one tiny service and explicitly excludes cron, so we'd lose news ingestion.

### 4.10 Firebase Cloud Messaging (FCM) — for daily 8 AM IST push
- **Completely free**, no per-message cost, no daily message cap on free.
- Project quota default: 600,000 messages/min (we'll never hit this).
- Per-device cap: 240 messages/min, 5,000/hour to a single Android device (irrelevant — we send 1 push/day per device).
- Native Compose support via Google Services plugin.

### 4.11 RevenueCat (Android subscription billing)
- Free up to **$2,500/mo Monthly Tracked Revenue (MTR)**, then 1% of MTR.
- Wraps Google Play Billing v8; gives us server-verified entitlements, webhooks for renewals/cancellations, customer charts.
- We never see card data — Google Play Billing handles payment; RevenueCat just tracks the subscription state.

---

## 5. Capacity Plan for 5,000 MVP Users (Multi-Country, English-Only, Daily Push)

Assume DAU = 30% of MAU = **1,500 DAU**, 1 app-open/day avg, peak 3. **6 countries** active at MVP: `in, us, gb, ae, sg, ca` (we can add `au, de` later — newsdata.io allows up to 5 countries per query, so we'll fire two queries to cover 6).

### 5.1 newsdata.io Budget (multi-country)
Strategy: **single ingestion job, many users.**

- 4 ingestion fires/day × 4 category buckets × 2 country-batches (3+3 countries) × 1 call = **32 credits/day**
- Add 8 buffer credits for re-ingestion / errors → **~40 credits/day used**, leaving ≥ 160 credits free as headroom (free plan = 200 credits/day).
- 12-hour data delay is acceptable since "Impact News" is consequence-oriented, not breaking news.
- If we expand to 8+ countries later we still fit (≤ 60 credits/day).

### 5.2 LLM Budget (Impact Headline Generation, multi-country)
The headline rewrite is **NOT per user**. It is **per (article, persona-bucket)**.

- A **persona bucket** = canonical hash of `{country, ageRange, industry, region, lifeStage, employmentType, investorType}`. `country` is part of the bucket — articles for India naturally only get rewritten for India buckets, not US ones.
- For 5K users spread across 6 countries (~833 users/country), expect **~30 active buckets per country = ~180 distinct buckets total** after rounding.
- We keep ~50 articles/day per country after filtering = ~300 articles/day total. For each article we generate rewrites for the 8-10 buckets that match its impact tags within its country.
- **Naive daily LLM calls** ≈ 300 articles × 9 buckets = **2,700 calls/day** → exceeds Gemini's 1,500 RPD.
- **Optimised LLM calls** with two changes:
  1. **Batch 5 articles per call** (Gemini 2.5 Flash handles a 5-article structured output cleanly under 1M TPM): 2,700 / 5 = **540 calls/day** → fits comfortably in Gemini.
  2. **Spillover to Groq** if Gemini hits 429: Groq Llama 3.1 8B Instant has 14,400 RPD — effectively unlimited at this scale.
- Cache rewrites in Postgres keyed by `(article_id, persona_bucket_hash)` — reused for every user in that bucket.

### 5.3 Push Notification Budget (8 AM IST daily fan-out)
- One cron at 02:30 UTC = 08:00 IST. Job loads users with `notifications_enabled = true`, looks up each user's `daily_top5[0]` (the lead story), and sends one FCM message per device.
- **1,500 active devices × 1 message = 1,500 FCM sends/day** → ~25 sends/sec to fan out in 60 seconds, well under FCM's 600K msgs/min project cap.
- Personalisation: use `data` payload, not `notification` payload, so the Compose app renders the localised headline + summary from the **already-cached** `daily_top5` payload (no extra DB read on tap).

### 5.4 Vercel Function Budget
- 1,500 DAU × ~12 API calls/day (open feed, scroll, bookmark, persona update, FCM token refresh) ≈ **540K invocations/month** (well under 1M).
- **Active CPU** is only the time the CPU is *actually doing work*, not waiting on I/O. Most of our latency is DB + Supabase Auth verification + FCM (all I/O), so real active CPU is closer to 25-30 ms per request → ~3.5-4.5 CPU-hours/month. **Borderline but fits.** If we get warnings, move the heaviest endpoint (`POST /v1/personas` on edit, which triggers regeneration) to a queue worker on GitHub Actions.

### 5.5 Auth (Supabase) Budget
- 5,000 MAU is **10% of the 50K free quota**. Plenty of headroom.
- Even if every user logs in twice/day = 10K JWT verifications/day, all happen against Supabase JWKS (public-key verification, free, cached on our side for 1 hour).

### 5.6 Database Budget
- 500 MB Postgres holds ~80K articles (multi-country) + 5K personas + 100K bookmarks + 5K subscription rows + 5K push-token rows easily (estimate: ~70 MB used in 6 months).
- Nightly cleanup of articles older than 14 days; nightly cleanup of `impact_rewrites` whose `article_id` is gone.

### 5.7 Redis Budget
- 10K commands/day = ~7 commands/min.
- Used for: `feed:{user_id}:{day}` cache, rate-limit counters, dedup of news-source URLs during ingestion, and **persona-edit invalidation flags** (`persona:dirty:{user_id}` set when user updates persona, picked up by next cron tick).

### 5.8 RevenueCat Budget
- Below $2,500 MTR → free.
- At ₹99/mo (~$1.20 USD) per Pro subscriber, we can have **~2,000 Pro subscribers** before paying RevenueCat anything (1% of MTR = ~$25/mo at that point — trivial).

---

## 6. System Architecture

### 6.1 High-Level Diagram (text)

```
┌──────────────────────────────────────┐
│   Compose Multiplatform Android App  │
│   ─────────────────────────────────  │
│   • Sign-in: Google / Email magic    │
│     link (Supabase Auth SDK) /       │
│     "Continue without account"       │
│     (anonymous Supabase user)        │
│   • RevenueCat SDK (Play Billing)    │
│   • Firebase SDK (FCM token)         │
└──────────────────┬───────────────────┘
                   │ HTTPS + Supabase JWT (Bearer)
                   ▼
┌──────────────────────────────────────────────────────────────┐
│  Vercel Hobby — Node 22 + Hono + OpenAPI                     │
│  ──────────────────────────────────────────────              │
│  Auth middleware: verify Supabase JWT via JWKS               │
│                                                              │
│  Public routes:                                              │
│   GET  /v1/health                                            │
│  Authenticated routes:                                       │
│   GET  /v1/me                  → user + subscription state   │
│   PUT  /v1/personas            → create/update persona       │
│                                  (triggers cache invalidate) │
│   GET  /v1/feed/top5           → daily top 5 (cached)        │
│   POST /v1/bookmarks           → add bookmark                │
│   GET  /v1/bookmarks           → list bookmarks              │
│   DELETE /v1/bookmarks/:id     → remove bookmark             │
│   POST /v1/notifications/token → register FCM device token   │
│   POST /v1/billing/webhook     → RevenueCat webhook          │
│   POST /v1/_internal/cron/*    → cron-secret protected       │
└────┬──────────────┬───────────────────┬─────────────────┬────┘
     │              │                   │                 │
     │              │ read / write      │ cache           │ FCM
     ▼              ▼                   ▼                 ▼
┌──────────────┐ ┌──────────────────┐ ┌──────────────┐ ┌───────────┐
│ Supabase     │ │ Vercel AI SDK    │ │ Upstash      │ │ Firebase  │
│ ──────────── │ │ → Gemini 2.5     │ │ Redis        │ │ Cloud     │
│ • Postgres   │ │   Flash (primary)│ │ (free, REST) │ │ Messaging │
│   + RLS      │ │ → Groq Llama     │ │ feed:U:D     │ │ (HTTP v1) │
│ • Auth       │ │   (fallback)     │ │ persona:     │ │           │
│   - Google   │ └──────┬───────────┘ │   dirty:U    │ └───────────┘
│   - Email    │        ▲             │ rate:IP:*    │
│   - Anon     │        │             └──────────────┘
│ • Storage    │        │ batch rewrites
└──────┬───────┘        │
       │         ┌──────┴───────────────────────┐
       │         │ Cron Worker — GitHub Actions │
       │         │ ──────────────────────────── │
       │         │ Every 6h (00,06,12,18 UTC):  │
       │         │   1. Pull newsdata.io        │
       │         │   2. Tag w/ Gemini (batch 5) │
       │         │   3. Persist to DB           │
       │         │   4. Rewrite per bucket      │
       │         │   5. Materialise daily_top5  │
       │         │ At 02:30 UTC (08:00 IST):    │
       │         │   6. Send daily push via FCM │
       │         │ Every 5 min:                 │
       │         │   7. Drain persona:dirty:*   │
       │         │      → re-materialise        │
       │         └──────┬───────────────────────┘
       ▼                ▼                       ┌────────────────┐
┌──────────────────────────────────┐            │ RevenueCat     │
│        newsdata.io API (free)    │            │ ────────────── │
│  countries=in,us,gb,ae,sg,ca     │            │ Play Billing   │
│  category=politics,business,…    │◄──webhook──│ entitlements   │
│  language=en                     │            │ → /billing/    │
└──────────────────────────────────┘            │   webhook      │
                                                └────────────────┘
```

### 6.2 Component Responsibilities

| Component | Responsibility | Why this choice |
|---|---|---|
| **Compose MP app** | Onboarding flow with **Supabase Auth UI** (Google + Email magic link + Continue-anonymously), persona form, feed view, bookmarks, paywall via RevenueCat, FCM token registration | Cross-platform-ready; Supabase provides drop-in Kotlin Multiplatform auth client |
| **Supabase Auth** | Issues JWTs with `sub = user.id`, runs Google OAuth flow, sends magic-link emails, supports anonymous→permanent account upgrade | Free 50K MAU, all providers included, integrates with Supabase RLS automatically |
| **API (Vercel + Hono + OpenAPI)** | Stateless HTTP. Verifies Supabase JWT via JWKS, reads pre-computed feed from DB, **never calls news API or LLM in request path** | Stays inside Vercel free CPU; auth is just a public-key verify (~1 ms) |
| **Cron Worker (GitHub Actions)** | Pulls news, tags impact categories, rewrites top headlines per persona bucket, materialises `daily_top5`, **sends 8 AM IST FCM push**, drains `persona:dirty:*` queue. Idempotent. | Decouples slow work from request path; bypasses Vercel Hobby's 1-cron/day limit |
| **Postgres (Supabase)** | System of record for articles, personas, bookmarks, impact-rewrites cache, push-tokens, subscriptions, RLS-scoped to `auth.uid()` | One free service does DB + Auth + Storage |
| **Redis (Upstash)** | Hot read cache for `GET /v1/feed/top5`, sliding-window rate limit, dedup of news URLs during ingestion, `persona:dirty:*` set for cache invalidation on persona edit | Reduces DB load & keeps under Vercel CPU budget |
| **LLM (Gemini + Groq)** | Three jobs: (1) tag impact categories on articles, (2) generate per-bucket impact headline (≤10 words) + 60-word "why this matters to you" summary, (3) regenerate on persona edit | Free, low latency, English-only output |
| **FCM** | Single daily push at 8 AM IST per active user, content delivered as `data` payload referencing the locally-cached `daily_top5[0]` | Free, native Compose integration, no rate-limit risk at MVP scale |
| **RevenueCat + Play Billing** | Subscription state, entitlements (`pro`, `pro_yearly`), webhook into our API to flip `users.is_pro` flag | Required for Android digital subscriptions, free up to $2,500 MTR |

### 6.3 Data Model (Postgres)

```sql
-- Profiles mirror auth.users (Supabase). Created automatically by a trigger
-- on auth.users INSERT. is_anonymous comes from Supabase's anonymous sign-in.
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text,                              -- null for anonymous users
  is_anonymous boolean not null default false,
  -- Subscription state, kept in sync by RevenueCat webhook
  is_pro boolean not null default false,
  pro_entitlement text,                    -- 'pro_monthly' | 'pro_yearly' | null
  pro_expires_at timestamptz,
  rc_user_id text unique,                  -- RevenueCat app_user_id
  notifications_enabled boolean not null default true,
  notification_time_local time not null default '08:00',
  timezone text not null default 'Asia/Kolkata',
  created_at timestamptz default now()
);

-- Articles ingested from newsdata.io (deduped on source_url)
create table articles (
  id uuid primary key default gen_random_uuid(),
  source_url text unique not null,
  source_name text,
  title text not null,
  description text,
  language text default 'en',
  country text not null,                   -- 'in' | 'us' | 'gb' | …
  published_at timestamptz,
  ingested_at timestamptz default now(),
  impact_tags text[] default '{}',         -- finance, local_law, tech, …
  impact_anchors jsonb default '{}'::jsonb -- structured anchors found in article
);
create index articles_published_idx on articles (published_at desc);
create index articles_tags_idx on articles using gin (impact_tags);
create index articles_country_idx on articles (country);

-- Persona is 1-to-1 with profile. Store full persona, plus a derived
-- bucket_hash that drives all LLM caching. PII is *not* stored here —
-- email/name/phone live in profiles + Supabase auth, never on persona.
create table personas (
  user_id uuid primary key references profiles(id) on delete cascade,
  age_range text,                          -- '18-24','25-34', …
  industry text,
  income_bracket text,
  employment_type text,
  region_country text not null,            -- ISO-2, drives news country
  region_city text,
  commute_method text,
  life_stage text,                         -- 'renter' | 'owner' | 'parent' | 'student'
  investor_type text[],
  interests text[],
  bucket_hash text not null,               -- deterministic hash incl. region_country
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
create index personas_bucket_idx on personas (bucket_hash);
create index personas_country_idx on personas (region_country);

-- Per-bucket precomputed impact rewrites (the magic cache)
create table impact_rewrites (
  article_id uuid references articles(id) on delete cascade,
  persona_bucket_hash text not null,
  impact_headline text not null,           -- ≤10 words, Inshort-style
  why_summary text not null,               -- ~60 words
  llm_model text not null,
  generated_at timestamptz default now(),
  primary key (article_id, persona_bucket_hash)
);

create table bookmarks (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  article_id uuid references articles(id) on delete cascade,
  saved_at timestamptz default now(),
  unique (user_id, article_id)
);
create index bookmarks_user_idx on bookmarks (user_id, saved_at desc);

-- Materialised daily top-5 per user for fast reads + 8 AM IST push
create table daily_top5 (
  user_id uuid references profiles(id) on delete cascade,
  feed_date date not null,
  article_ids uuid[] not null,
  push_sent_at timestamptz,                -- null until FCM cron sends it
  computed_at timestamptz default now(),
  primary key (user_id, feed_date)
);

-- FCM tokens. A user may have multiple devices.
create table push_tokens (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  fcm_token text unique not null,
  platform text not null default 'android',
  last_seen_at timestamptz default now()
);
create index push_tokens_user_idx on push_tokens (user_id);

-- RLS examples (enforced server-side by Supabase even if API leaks):
-- alter table bookmarks enable row level security;
-- create policy "users see only their bookmarks"
--   on bookmarks for all using (auth.uid() = user_id);
-- alter table personas enable row level security;
-- create policy "users see only their persona"
--   on personas for all using (auth.uid() = user_id);
-- alter table profiles enable row level security;
-- create policy "users see their profile"
--   on profiles for select using (auth.uid() = id);
```

### 6.4 Ingestion Pipeline (every 6 hours, multi-country)

Pseudocode for the cron worker. **All output is English-only**; we trust the per-country mix to surface what each persona cares about.

```ts
// COUNTRIES: split into ≤5-country batches because newsdata.io free
// allows up to 5 country codes per query.
const COUNTRY_BATCHES: string[][] = [
  ['in', 'us', 'gb'],
  ['ae', 'sg', 'ca'],
];
const CATEGORY_BUCKETS = [
  'business,politics',
  'technology,science',
  'health,environment',
  'top',
];

// 1. PULL — 4 buckets × 2 country-batches = 8 credits/run × 4 runs/day = 32/day
for (const countries of COUNTRY_BATCHES) {
  for (const category of CATEGORY_BUCKETS) {
    const res = await fetch('https://newsdata.io/api/1/latest?' + qs({
      apikey: NEWSDATA_KEY,
      country: countries.join(','),
      category,
      size: 10,
      language: 'en',          // Decision: English-only output for MVP
    }));
    const { results } = await res.json();
    await dedupAndInsert(results); // skip rows whose source_url already exists
  }
}

// 2. TAG — batch 5 articles per LLM call. Single call returns a tag array.
const fresh = await db.articles.where('impact_tags', '=', '{}');
for (const chunk of chunks(fresh, 5)) {
  const tagged = await gemini.generate({
    schema: ImpactTagBatchSchema,   // zod schema validated by Vercel AI SDK
    prompt: `For each article, tag with one or more of:
      finance, taxes, local_law, jobs, commute, real_estate, tech, health,
      energy, food, education, security, consumer_prices, business_owner.`,
    input: chunk,
  });
  await db.articles.bulkUpdateTags(tagged);
}

// 3. REWRITE — fan out per (article, bucket) where bucket country matches
//    article country. Batch 5 articles per Gemini call to stay under 1500 RPD.
const allBuckets = await db.personas.distinctBuckets();   // {hash, descriptor}
for (const article of fresh) {
  const matching = allBuckets.filter(b =>
    b.descriptor.region_country === article.country &&
    bucketMatchesTags(b, article.impact_tags)
  );
  for (const bh of matching) {
    if (await db.impact_rewrites.exists(article.id, bh)) continue; // idempotent
    const rewrite = await runWithFallback({
      primary:  () => gemini.generate({ schema: ImpactRewriteSchema, prompt: IMPACT_PROMPT(bh, article) }),
      fallback: () => groq.generate  ({ schema: ImpactRewriteSchema, prompt: IMPACT_PROMPT(bh, article) }),
    });
    await db.impact_rewrites.insert({ article_id: article.id, persona_bucket_hash: bh.hash, ...rewrite });
  }
}

// 4. MATERIALISE daily_top5 for every active user (today's feed_date)
await refreshDailyTop5();

// 5. DRAIN persona:dirty:* — users who edited persona since last run
const dirty = await redis.smembers('persona:dirty:set');
for (const userId of dirty) {
  await regenerateForUser(userId);                 // re-bucket, fresh top5
  await redis.del(`feed:${userId}:${today()}`);    // bust feed cache
  await redis.srem('persona:dirty:set', userId);
}
```

**Inshort-style output** (Decision 4) is enforced both by the prompt and by the Zod schema:

```ts
const ImpactRewriteSchema = z.object({
  impact_headline: z.string().min(20).max(80)             // ≤10 words ≈ ≤80 chars
                        .refine(s => s.split(/\s+/).length <= 10,
                                'headline must be ≤10 words'),
  why_summary:     z.string().min(280).max(420)           // ~60 words ≈ 280-420 chars
                        .refine(s => {
                           const w = s.trim().split(/\s+/).length;
                           return w >= 50 && w <= 70;
                        }, 'summary must be 50-70 words'),
});

const IMPACT_PROMPT = (b: PersonaDescriptor, a: Article) => `
You are SoWhat News, a "so what does this mean for me?" rewriter.

User context (anonymous): ${JSON.stringify(b)}
Article title: ${a.title}
Article description: ${a.description}
Article country: ${a.country}

Rewrite for this user in ENGLISH ONLY:
1. impact_headline — ≤10 words, present tense, no quotes, no clickbait,
   names the concrete consequence for this user.
2. why_summary — 50-70 words (target 60), 1-2 short sentences, plain English,
   styled like Inshorts: factual, calm, no exclamation marks. State (a) what
   happened in one clause, (b) why it matters to this specific user, (c) one
   concrete action or watch-out if relevant.

Return only valid JSON matching the schema.
`;
```

Key optimisations baked in:
- Dedup on `source_url` — repeated ingestion does not spend new LLM calls.
- Tags are generated **once per article**, rewrites **once per (article, bucket)**.
- Rewrites are scoped to articles whose `country` matches the bucket's `region_country` — no wasted work, no privacy mixing.
- Gemini→Groq fallback on 429/5xx, transparent to caller.
- The request-path API call `GET /v1/feed/top5` only does a single DB read (and a Redis read if cached).

### 6.5 Request Path: `GET /v1/feed/top5`

1. Verify Supabase JWT (JWKS, cached) → load `user_id`.
2. Check Redis `feed:{user_id}:{today}` → return if hit.
3. Read `daily_top5` row for `(user_id, today)` → join `articles` + `impact_rewrites` (on `bucket_hash`).
4. If `daily_top5` is empty (new user, freshly onboarded mid-day, or persona just edited and not yet processed), materialise on the fly using the persona's current `bucket_hash` and the latest `articles` matching their tags + country. This is the **only** request-path branch that may invoke Gemini, gated to ≤2 LLM calls per request.
5. Cache for 6 hours in Redis. Return.

This keeps the hot path under **50 ms** and ~30 ms of "active CPU" → fits the Vercel CPU budget comfortably.

### 6.6 Persona Edit Path: `PUT /v1/personas` (Decision 5)

1. Verify JWT, validate body, recompute `bucket_hash`.
2. `UPDATE personas` row.
3. **Invalidate cache and queue regeneration:**
   ```ts
   await redis.del(`feed:${userId}:${today()}`);
   await redis.sadd('persona:dirty:set', userId);
   await db.daily_top5.deleteWhere({ user_id: userId, feed_date: today() });
   ```
4. Return 202 + a `regenerating: true` flag. The mobile app polls `GET /v1/feed/top5` every 5 s for up to 30 s; the response contains a `partial: true` flag while rewrites are still being filled in (we serve from `articles` only).
5. The next cron tick (≤5 min) drains `persona:dirty:set` and runs full rewrites; subsequent app opens get the personalised version.

### 6.7 8 AM IST Push Job (Decision 3)

Cron at **02:30 UTC** (= 08:00 IST) hits `POST /v1/_internal/cron/push-daily`:

```ts
// 1. Pick users whose local 08:00 falls in the next 30-min window
const users = await db.profiles.where({ notifications_enabled: true })
                               .timezoneIs('Asia/Kolkata');
// 2. Pull their daily_top5[0]
for (const u of users) {
  const top = await db.daily_top5.first(u.id, today());
  if (!top) continue;
  const { impact_headline, why_summary } = await db.firstStoryFor(u.id, top.article_ids[0]);
  const tokens = await db.push_tokens.forUser(u.id);
  // 3. Fire FCM data message — Compose app renders from local cache
  await fcm.sendEachForMulticast({
    tokens,
    data: {
      type: 'daily_top',
      headline: impact_headline,
      summary:  why_summary,
      article_id: top.article_ids[0],
    },
    android: { priority: 'high', notification: { channelId: 'daily_top' } },
  });
}
// 4. Mark daily_top5.push_sent_at = now() so we never double-send
```

Multi-timezone scaling later: schedule one cron tick per IANA-zone bucket (`Asia/Kolkata`, `America/New_York`, etc.) instead of just IST.

---

## 7. API Surface (OpenAPI 3.1, partial)

> Auth note: We **do not** mint our own JWTs. The Compose app calls Supabase Auth directly (Google OAuth or email magic link), receives a Supabase-signed JWT, and sends it as `Authorization: Bearer <token>` to our API. Our Hono middleware verifies signatures via Supabase's JWKS (cached for 1 h).

```yaml
openapi: 3.1.0
info: { title: SoWhat News API, version: 0.2.0 }
paths:
  /v1/me:
    get:
      summary: Get the current user's profile + subscription state
      security: [{ bearerAuth: [] }]
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Me' }

  /v1/personas:
    put:
      summary: Create or update the persona for the authed user
      description: |
        Mid-day edits are supported. The server returns 202 if the new
        bucket_hash has no precomputed rewrites yet. Mobile clients should
        poll `GET /v1/feed/top5` until `partial=false`.
      security: [{ bearerAuth: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/Persona' }
      responses:
        '200': { description: persona updated, feed already personalised }
        '202': { description: persona updated, feed regenerating }

  /v1/feed/top5:
    get:
      summary: Get today's 5 personalised impact headlines (Inshort-style)
      security: [{ bearerAuth: [] }]
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  partial: { type: boolean, description: 'true if rewrites still being generated' }
                  stories:
                    type: array
                    items: { $ref: '#/components/schemas/ImpactStory' }

  /v1/bookmarks:
    get: { security: [{ bearerAuth: [] }], summary: List bookmarks }
    post:
      security: [{ bearerAuth: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [article_id]
              properties:
                article_id: { type: string, format: uuid }

  /v1/bookmarks/{id}:
    delete:
      security: [{ bearerAuth: [] }]
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string, format: uuid }

  /v1/notifications/token:
    post:
      summary: Register or refresh the FCM device token for daily 8 AM IST push
      security: [{ bearerAuth: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [fcm_token]
              properties:
                fcm_token: { type: string, minLength: 32 }
                platform:  { type: string, enum: [android, ios, web], default: android }

  /v1/notifications/preferences:
    put:
      summary: Toggle daily push or change preferred local time
      security: [{ bearerAuth: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                notifications_enabled: { type: boolean }
                notification_time_local: { type: string, example: '08:00' }
                timezone: { type: string, example: 'Asia/Kolkata' }

  /v1/billing/webhook:
    post:
      summary: RevenueCat → backend webhook for subscription state changes
      description: |
        Verified by the `Authorization: Bearer <REVENUECAT_AUTH_HEADER>` header
        we configure in the RevenueCat dashboard, compared with
        `crypto.timingSafeEqual`. Updates `profiles.is_pro`, `pro_entitlement`,
        `pro_expires_at`. Idempotent on RevenueCat `event.id`.
      responses:
        '204': { description: accepted }

components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  schemas:
    Me:
      type: object
      properties:
        user_id:        { type: string, format: uuid }
        email:          { type: string, nullable: true }
        is_anonymous:   { type: boolean }
        is_pro:         { type: boolean }
        pro_entitlement:{ type: string, nullable: true }
        pro_expires_at: { type: string, format: date-time, nullable: true }
        notifications_enabled: { type: boolean }
        timezone:       { type: string }
    Persona:
      type: object
      required: [age_range, industry, region_country]
      properties:
        age_range:      { type: string, enum: ['18-24','25-34','35-44','45-54','55+'] }
        industry:       { type: string }
        income_bracket: { type: string }
        employment_type:{ type: string }
        region_country: { type: string }
        region_city:    { type: string }
        commute_method: { type: string }
        life_stage:     { type: string }
        investor_type:  { type: array, items: { type: string } }
        interests:      { type: array, items: { type: string } }
    ImpactStory:
      type: object
      properties:
        article_id:     { type: string, format: uuid }
        original_title: { type: string }
        impact_headline:{ type: string, description: '≤10 words, Inshort-style' }
        why_summary:    { type: string, description: '50-70 words' }
        source_url:     { type: string, format: uri }
        published_at:   { type: string, format: date-time }
        impact_tags:    { type: array, items: { type: string } }
```

Use `@hono/zod-openapi` on the server to derive validators and OpenAPI doc from the same Zod schemas. Use `openapi-generator-cli` to emit a typed Kotlin client (`kotlin-multiplatform` target) for the Compose app.

---

## 8. Framework Pick: Why Hono Over Fastify/NestJS

From the comparisons above:

| Aspect | NestJS 11 | Fastify 5.8 | Hono 4 |
|---|---|---|---|
| Cold start | ~510 ms | ~110 ms | **~90 ms (Node), 40 ms (Bun)** |
| Throughput (Node 24) | 17K RPS (Express) / 72K RPS (Fastify adapter) | 114K RPS | **128K RPS** |
| Bundle size | ~150 kB | ~43 kB | **~12 kB** |
| Edge runtime support | No | No | **Yes** (Cloudflare Workers, Vercel Edge, Bun, Deno, Node) |
| Best for | Large opinionated team | High-throughput Node | **Edge-native, multi-runtime, serverless-friendly** |

For a serverless deployment on Vercel where cold starts and bundle size matter, **Hono wins**. It also gives you a free escape hatch to Cloudflare Workers (which would be even cheaper) if Vercel limits ever bite.

If you ever want a heavier framework, NestJS with the Fastify adapter remains an option, but for this MVP it is over-engineered.

---

## 9. Hosting Pick — Deep Comparison: Vercel vs Cloudflare Workers vs Railway

This is a side-by-side of the three options the user asked about, scoped to **our actual workload**: a JWT-verifying Hono API + a cron worker that ingests news, calls Gemini in batches, writes to Postgres, and fans out FCM pushes.

### 9.1 Free-Tier Numbers (verified May 2026)

| Resource (free tier)                  | **Vercel Hobby**                           | **Cloudflare Workers Free**                                                | **Railway Free** (post-trial)                              |
| ------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Requests / invocations                | 1,000,000 / month                          | 100,000 / day (~3M / month)                                                | Effectively unlimited *if* under the $1/mo credit          |
| **CPU time per request**              | 4 CPU-hours / month total (no per-req cap) | **10 ms hard cap per invocation**                                          | $1 / month resource credit ≈ ~25 CPU-hours of cheap shared |
| Function memory                       | 2 GB / 1 vCPU                              | 128 MB                                                                     | 0.5 GB                                                     |
| Function max duration                 | 300 s (Fluid Compute)                      | No wall-clock cap, but 10 ms CPU ceiling                                   | Long-running, container always on                          |
| Cold start                            | ~50-80 ms (Fluid Compute keeps warm)       | <1 ms (V8 isolate)                                                         | ~1-2 s on cold (worse on free)                             |
| **Cron jobs (free)**                  | **1 / day max** ← blocker                  | **5 cron triggers**, 1-min granularity, 10 ms CPU each                     | **None** — cron requires Hobby ($5/mo)                     |
| Cron CPU cap (free)                   | Same 4 CPU-hr/month pool                   | 10 ms / cron tick (50 ms on paid)                                          | n/a                                                        |
| Outbound sub-requests / external HTTP | Effectively unlimited                      | **50 / request** (limits chained Gemini → DB → FCM in one Worker)          | Unlimited                                                  |
| Native database                       | None (BYO)                                 | D1 (SQLite, 5 M reads + 100 K writes / day, 500 MB / DB)                   | Built-in Postgres add-on (uses your $1/mo credit)          |
| Native KV / cache                     | None (BYO)                                 | Workers KV (100 K reads + 1 K writes / day, 1 GB)                          | None                                                       |
| Always-on workers / persistent procs  | No (FaaS)                                  | Durable Objects (paid only for SQLite-backed)                              | Yes (always on, eats credit)                               |
| Egress                                | 100 GB Fast Data Transfer                  | Effectively unmetered                                                      | Counts against $1 credit                                   |
| Projects                              | 200                                        | 100 Workers / account                                                      | **1 project**, 3 services, 1 vCPU + 0.5 GB total           |
| Sleeps after idle?                    | No (functions warm via Fluid Compute)      | No (V8 isolates always available)                                          | **Yes** — service throttles when credit hits 0             |
| DX                                    | Best-in-class git push → deploy            | Wrangler CLI; superb if you stay in CF stack                               | Smooth Heroku-clone DX                                     |
| Real cost to run our MVP at 5K users  | $0                                         | $0 (but only after major refactor)                                         | ~$1-2/mo eats free credit, **service halts mid-month**     |

### 9.2 Workload Fit — Why Vercel Wins For *This* App

We have three execution shapes:

1. **JWT-verifying read-mostly API** — `GET /v1/feed/top5`, bookmarks, `/v1/me`. ~30 ms active CPU, mostly Postgres I/O.
2. **LLM-orchestrating ingestion cron** — pulls newsdata.io, batches 5-article calls to Gemini, writes to Postgres, sends FCM. Each tick can chain **dozens of subrequests** and may take 30-120 s wall-clock.
3. **Persona-edit regeneration** — triggered on `PUT /v1/personas`, fires up to 10 LLM calls. ~5-15 s wall-clock.

How each platform handles each shape:

| Shape                                                        | Vercel Hobby                                                                                               | Cloudflare Workers Free                                                                              | Railway Free                                                                                               |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **(1) Read API**                                             | ✅ ~30 ms active CPU comfortably fits                                                                      | ✅ JWT verify + DB read fits in 10 ms (DB I/O doesn't count)                                         | ✅ Always-on, but you lose the service when $1 credit drains                                               |
| **(2) Ingestion cron, 30-120 s with 50+ subrequests**        | ✅ 300 s function duration; runs as one Vercel Function invoked by GitHub Actions cron                     | ❌ **50 subrequest cap per invocation** breaks chained LLM+DB+FCM; 10 ms CPU ends rewrites prematurely | ❌ No free cron; would need to upgrade to Hobby ($5/mo) and run a cron worker container                  |
| **(3) Persona-edit regeneration**                            | ✅ Runs as a normal function; user polls until done                                                        | ⚠️ Possible only by farming out work to a queue + Durable Objects (free tier limits this)            | ✅ But again, blocked by no-cron and credit-burn                                                          |
| **Day-2: add a "regenerate now" button**                     | ✅ Fluid Compute keeps the function warm                                                                   | ⚠️ Same subrequest/CPU constraints                                                                   | ✅ Functionally fine                                                                                       |

### 9.3 Decision

- **Pick Vercel Hobby for the API.** It's the only free tier that handles our LLM-chained ingestion cron in a single function call without contortion. Subrequests are unmetered, function duration is 300 s, Node compatibility is 100%. The one limitation — 1 cron/day — is bypassed by GitHub Actions cron.
- **Skip Cloudflare Workers for now**, but keep it as a **Year-2 migration target** for the read-API once we're profitable. Workers' <1 ms cold start, $5/mo paid tier, and global edge make it a clear win for the read-only `GET /v1/feed/top5` endpoint when we want lower latency for international users. The blockers (10 ms CPU, 50 subrequests) only apply to cron/LLM work, which would stay on Vercel or move to Cloud Run.
- **Skip Railway entirely for MVP.** Railway in 2026 is no longer a "free tier" platform — the Free plan is a token offering ($1 credit, 1 vCPU, no cron) that will throttle your service mid-month. The Hobby plan is $5/mo *minimum*, payable even at zero usage. We get more for free elsewhere.

### 9.4 Why Vercel Hobby Cron Fails Us
> *"Hobby accounts are limited to cron jobs that run once per day. Cron expressions that would run more frequently will fail during deployment."* — Vercel docs

We need to ingest news 4×/day so users get fresh impact headlines as the world changes, plus a daily 8 AM IST push, plus a 5-min drain of the persona-dirty queue. The fix is **external scheduling** that calls our protected Vercel endpoint:

**GitHub Actions cron (recommended for MVP)**
- 2,000 minutes/month free for private repos, **unlimited for public**.
- 5-minute scheduling precision; reliable enough for news.
- Yaml lives next to code:

```yaml
# .github/workflows/cron-news-ingest.yml
on:
  schedule:
    - cron: '0 */6 * * *'      # ingestion every 6 h
  workflow_dispatch: {}
jobs:
  ingest:
    runs-on: ubuntu-latest
    steps:
      - name: Hit cron endpoint
        env:
          CRON_SECRET: ${{ secrets.CRON_SECRET }}
        run: |
          curl -fsS -X POST https://api.sowhat.app/v1/_internal/cron/news-ingest \
               -H "Authorization: Bearer $CRON_SECRET" \
               --max-time 280

# .github/workflows/cron-push-daily.yml
on:
  schedule:
    - cron: '30 2 * * *'        # 02:30 UTC = 08:00 IST
jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -fsS -X POST https://api.sowhat.app/v1/_internal/cron/push-daily \
               -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}"

# .github/workflows/cron-persona-drain.yml
on:
  schedule:
    - cron: '*/5 * * * *'       # every 5 min
jobs:
  drain:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -fsS -X POST https://api.sowhat.app/v1/_internal/cron/persona-drain \
               -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}"
```

Auth the endpoint with a secret in `Authorization: Bearer ${CRON_SECRET}` (validated using `crypto.timingSafeEqual` for constant-time comparison).

**Backup:** Cloud Scheduler → Cloud Run job (3 free jobs/month, infinite frequency) if GitHub Actions ever flakes.

---

## 9A. Authentication & Subscriptions (Supabase Auth + RevenueCat + Play Billing)

The user requested **"login using Google, email, etc. so that we can ask user to subscribe to our plan"**. This sub-section is the design for that requirement.

### 9A.1 Provider Comparison

| Option            | Free Tier (May 2026)                                                                  | Google OAuth | Email login              | Anonymous → real | Server-side JWKS | Verdict                                                                                          |
| ----------------- | ------------------------------------------------------------------------------------- | ------------ | ------------------------ | ---------------- | ---------------- | ------------------------------------------------------------------------------------------------ |
| **Supabase Auth** | **50,000 MAU free**, every social provider included, anonymous sign-ins included     | ✅           | Magic link + password    | ✅ (built-in)    | ✅               | **Pick** — already in our stack (DB + Storage), zero added vendors, zero extra cost              |
| Clerk             | 10,000 MAU free                                                                       | ✅           | Magic link + OTP + pwd   | ✅               | ✅               | Best DX but smaller free quota and an extra vendor; revisit if Supabase Auth UX disappoints      |
| Firebase Auth     | Spark plan: 50K verifications/mo, 10 phone OTPs/day                                   | ✅           | Email/password           | ✅               | ✅               | Solid free tier but couples us deeper into Firebase; FCM-only is fine, full Auth is overlap      |
| Auth0             | 25,000 MAU free                                                                       | ✅           | Email + many             | Limited          | ✅               | Enterprise-grade but overkill; Supabase covers the same surface for our needs                    |

**Decision:** **Supabase Auth.** It is already paid-for-by-zero in our DB choice, gives us 50K MAU free (10× our MVP target), supports Google, email magic link, password, **and** anonymous sign-ins, and JWTs are RLS-aware out of the box.

### 9A.2 Sign-In Flow Options for the User

We expose **three** entry-points on the onboarding screen:

1. **"Continue with Google"** — most-used, lowest friction. Supabase handles the OAuth round-trip via the Android Custom Tabs flow; on success we get a JWT with `email`, `email_verified`, `provider = 'google'`, and a stable `sub` (`auth.users.id`).
2. **"Continue with Email"** — magic link sent via Supabase's free SMTP (rate-limited; we'll switch to Resend free tier — 3K emails/mo — when we hit the cap). Fallback to email + password if user prefers.
3. **"Continue without an account"** — Supabase **anonymous sign-in**. The user gets a JWT immediately, can use the app, build a persona, even bookmark. They are *prompted* to upgrade to Google/email when they tap "Subscribe to Pro" (which is required because the Play Store needs a stable account to attach the entitlement to).

This three-door pattern preserves the privacy-first onboarding from Revision 1, while giving us the identifier we need to monetize.

### 9A.3 Subscription Architecture (Play Billing v8 + RevenueCat)

Google Play **mandates** Play Billing for digital subscriptions on Android (no Stripe, no PayPal). RevenueCat wraps Play Billing and gives us:

- A single `Purchases.purchasePackage(...)` call from the Compose app.
- Server-side receipt validation (we never trust the client).
- An `entitlement` system: we define `pro` once in the RC dashboard and offer it as monthly + yearly SKUs.
- Webhooks for renewals, cancellations, billing-issue grace periods, refunds.
- Cross-platform-ready: when we ship iOS later, the same RC `app_user_id` works.

```
User taps "Subscribe to Pro"
  └─► Compose app: Purchases.logIn(supabaseUserId)
       └─► Purchases.purchasePackage(monthlyPackage)
            └─► Google Play Billing UI → user pays via Play
                 └─► RevenueCat validates receipt with Google
                      ├─► RC SDK returns CustomerInfo to app (instant unlock)
                      └─► RC Webhook → POST /v1/billing/webhook
                           └─► API: UPDATE profiles SET is_pro = true,
                                                      pro_entitlement = 'pro_monthly',
                                                      pro_expires_at = ...
                                                  WHERE rc_user_id = $supabaseUserId
```

**Why both client SDK *and* webhook:**
- Client SDK gives instant UX (no spinner waiting for backend).
- Webhook is the **source of truth** for `is_pro` — the Compose app cannot fake it because gated features check `GET /v1/me`.

### 9A.4 What Pro Unlocks (initial guess, validate with PM)

Free tier gets the daily top-5; Pro unlocks any of:

- **Unlimited bookmarks** (free is capped at 50).
- **More than 5 stories/day** — e.g. show 15 with persona impact.
- **Push at custom times** (free is fixed at 8 AM IST).
- **Topic deep-dives** — "show me everything about real-estate this week".

We model this as a single boolean `is_pro` for MVP and let the Compose app toggle features off the `/v1/me` response.

### 9A.5 What Changes on the Server

- Drop the bespoke `device_token` JWT minting. The API only ever **verifies** Supabase JWTs.
- All tables previously keyed on `persona_id` become keyed on `user_id` (= `auth.users.id`). RLS policies `auth.uid() = user_id` enforce per-user isolation at the DB level even if the API leaks.
- Add `POST /v1/billing/webhook` that takes the RC payload, validates the auth header with `crypto.timingSafeEqual`, and idempotently updates `profiles`.
- Email is now PII (see §13 for the privacy implications).

### 9A.6 What Changes in LLM Prompts

The persona descriptor sent to Gemini is **still anonymous**. We never include `email`, `user_id`, or any RC identifier in the prompt — only the bucket descriptor (age range, industry, region, etc.). This preserves the Revision 1 privacy posture even though we now collect email at sign-up.

---

## 10. LLM Choice: Gemini 2.5 Flash primary, Groq fallback

| Provider | Model | Free RPD | TPM | Cost beyond free | Why use it |
|---|---|---|---|---|---|
| Google | Gemini 2.5 Flash | 1,500 | 1,000,000 | ~$0.10/M input | Best free quota; strong JSON-mode; multilingual (we only use English) |
| Groq | Llama 3.1 8B Instant | 14,400 | 6,000 | $0.05/M input | 5× more RPD than Gemini; ultra-fast (~300 TPS) |
| OpenAI | gpt-4o-mini | 0 free | – | $0.15/M input | Skip on MVP — no free tier |

With Vercel AI SDK the swap is one line:

```ts
import { generateObject } from 'ai';
import { google } from '@ai-sdk/google';
import { groq } from '@ai-sdk/groq';

const primary  = google('gemini-2.5-flash');
const fallback = groq('llama-3.1-8b-instant');

async function runWithFallback({ schema, prompt }: Args) {
  try {
    return await generateObject({ model: primary, schema, prompt });
  } catch (e) {
    if (is429OrTransient(e)) {
      return await generateObject({ model: fallback, schema, prompt });
    }
    throw e;
  }
}
```

The circuit breaker auto-switches if Gemini returns 429 or 5xx. In §6.4 we also batch 5 articles per call to keep daily usage well under Gemini's 1,500 RPD even with multi-country fan-out.

> **Privacy note:** Gemini free tier may use prompts for model training. We only send the **anonymous persona descriptor** (age range, industry, country, etc.) plus the public article title/description. We never send the user's email, `user_id`, RevenueCat ID, or any other identifier. This preserves the Revision 1 privacy posture even though §9A now collects email.

> **Decision 4 enforcement:** Output is constrained by the Zod schema defined in §6.4 — `impact_headline` ≤10 words, `why_summary` 50-70 words. The LLM must return JSON; if it returns an out-of-range value the call is retried with a corrective system message.

---

## 11. Mobile Integration Notes (Compose Multiplatform)

- **Auth:** use the Supabase `gotrue-kt` Kotlin Multiplatform client. It exposes `signInWith(Google)`, `signInWith(Email)`, and `signInAnonymously()` and persists the session in encrypted DataStore. Refresh tokens are handled automatically.
- **Subscriptions:** add `purchases-android` (RevenueCat) at app init: `Purchases.configure(...)`, then `Purchases.logIn(supabaseSession.user.id)` whenever the user signs in / converts from anonymous to permanent.
- **Push:** use Firebase BoM + `firebase-messaging-ktx`; on app launch call `FirebaseMessaging.getInstance().getToken()` and POST it to `/v1/notifications/token`. Implement `FirebaseMessagingService.onMessageReceived` to render a notification using the `data` payload (we use `data` not `notification` so the local Compose app controls rendering even when backgrounded).
- **HTTP:** use **Ktor client** with the OpenAPI-generated Kotlin client. Send `Authorization: Bearer <SupabaseJWT>` automatically via a Ktor interceptor that pulls the token from `gotrue-kt`'s session.
- **Local cache:** **SQLDelight** for `daily_top5`, `articles`, `bookmarks` so offline opens work; **DataStore** for `notification_time_local`, `is_pro` flag.
- **Bookmarks:** persist both locally (optimistic UI) and via API (source of truth).
- **Persona edit UX (Decision 5):** after `PUT /v1/personas` returns 202, show a non-blocking toast "Refreshing your feed…" and poll `GET /v1/feed/top5?include_partial=true` every 5 s for up to 30 s. The first response is partial (article-only); subsequent ones replace the cards in place once `partial=false`.
- **Paywall:** RevenueCat ships drop-in paywall components for Compose. We can also build a custom paywall reading `Offerings` from RC.

---

## 12. Scaling Path (when you outgrow free tier)

The same architecture scales linearly. Order of moves as MAU grows:

| Trigger                                | Move                                                                                                            | Estimated cost     |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------ |
| ~10K MAU / news staleness complaints   | Upgrade newsdata.io to **Basic** ($199.99/mo) for real-time news + 20K credits/mo                               | +$200              |
| ~50K MAU / Supabase 500 MB cap         | Supabase **Pro** ($25/mo + usage), 100K MAU included                                                            | +$25–60            |
| 50K → 100K MAU                         | Supabase Auth charges $0.00325 / MAU above 100K — almost free                                                   | +$0–10             |
| Vercel CPU warnings                    | Vercel **Pro** ($20/mo) **OR** move API to Cloud Run pay-as-you-go (~$5–15/mo) **OR** read-API on Cloudflare    | +$15–20            |
| LLM throttling                         | Stay on Gemini paid (~$0.10/M input) — costs scale with usage but stay tiny                                     | <$20/mo at 50K MAU |
| Cron volume                            | Cloud Scheduler + Cloud Run jobs (a few dollars/month)                                                          | <$5                |
| Cache misses                           | Upstash Pay-as-you-go ($0.20/100K commands)                                                                     | <$5/mo             |
| Subscription revenue > $2,500 MTR      | RevenueCat 1% of MTR. At ₹99/mo × 2,000 subs ≈ $2,400/mo MTR — still free                                       | 1% above $2,500    |
| FCM volume                             | FCM has no metered free tier — stays free past 1M users                                                         | $0                 |
| Multi-region read latency (year-2)     | Migrate `GET /v1/feed/top5` to Cloudflare Workers, keep cron on Vercel/Cloud Run                                | +$5/mo             |

**Year-2 cost ceiling for 50K MAU: ≈ $250–300/month.** Subscription revenue at 1% conversion of 5K MAU at ₹99/mo (~₹5,000/mo ≈ $60/mo) covers Supabase Pro + Vercel Pro from month 1.

If we ever want to nuke vendor risk: the entire stack (Hono + Postgres + Redis + LLM via OpenAI-compatible API) is portable. We can self-host on a $20/mo Hetzner CCX13 if needed.

---

## 13. Security & Compliance Notes (aligned to org rules)

### 13.1 PII Handling (changed in Revision 2)

We now collect **email** at sign-up via Supabase Auth (Google or magic link). This makes us a personal-data controller. Mitigations:

- **Email is the only PII we store**, kept in `auth.users` (managed by Supabase, encrypted at rest, separate from app DB).
- **Persona table never contains email/name/phone.** Even Supabase RLS-restricted joins keep PII separate from persona attributes used by the LLM.
- **LLM prompts contain only the anonymous persona descriptor** (age range, industry, country, etc.) — no email, no `user_id`, no IP, no RC ID. This is enforced by the prompt-builder being a pure function that takes `PersonaDescriptor`, not `User`.
- **Account deletion**: a `DELETE /v1/me` endpoint cascades through `profiles → personas, bookmarks, daily_top5, push_tokens, impact_rewrites_for_user_buckets` and invokes Supabase Auth `admin.deleteUser`. We honour deletion within 7 days.
- **Privacy policy + ToS**: must be public before Play Store submission; covers email storage, Google sign-in, FCM token, RevenueCat receipts. (Outside the scope of this doc — engineering will provide a draft for legal review.)

### 13.2 Auth & Authorization

- **JWTs** are issued by Supabase Auth (RS256, asymmetric). Our API verifies via Supabase's JWKS endpoint, cached for 1 hour. We **do not mint JWTs ourselves**.
- **RLS** in Supabase enforces `auth.uid() = user_id` on every row in `personas`, `bookmarks`, `daily_top5`, `push_tokens`, `profiles`. Even if an API bug returns the wrong row, Postgres refuses.
- **Anonymous users** receive a Supabase JWT with `is_anonymous = true`. The paywall path requires upgrading to a real account (Google or email) before `purchasePackage` can fire.
- **Webhook auth**: `/v1/billing/webhook` validates a shared secret using `crypto.timingSafeEqual`; rejected webhooks are logged with rate-limited counters.
- **Cron auth**: `/v1/_internal/cron/*` validates `Authorization: Bearer ${CRON_SECRET}` with `crypto.timingSafeEqual`.

### 13.3 General Hardening

- **All DB access** via parameterised queries (Supabase JS or Drizzle ORM). No string interpolation.
- **Outbound calls** restricted to allow-list: `*.supabase.co`, `newsdata.io`, `generativelanguage.googleapis.com`, `api.groq.com`, `fcm.googleapis.com`, `api.revenuecat.com`.
- **Logs redacted** of any JWT, secret, FCM token, RC `app_user_id`, or full URL with query params.
- **Rate-limit** `PUT /v1/personas`, `POST /v1/bookmarks`, `POST /v1/notifications/token` with Upstash sliding-window (10 req / 10 s per IP) to defend against scraping and abuse.
- **CSRF** is N/A for our pure-bearer-token JSON API (no cookies).
- **TLS** enforced — Vercel does this automatically; Supabase, Upstash, FCM, and RevenueCat all require HTTPS.

---

## 14. Resolved Decisions (was: Open Questions)

The five questions raised in Revision 1 have been resolved. The architecture above already reflects these answers.

### Decision 1 — News localisation: **multi-country MVP**

> *Q: start with India only (`country=in`), or include more countries from day 1? More countries = more newsdata.io credit usage but better TAM.*
> **A: include more countries (better TAM).**

**Choice:** ship MVP with **6 countries** — `in, us, gb, ae, sg, ca` — covering India + the major Indian-diaspora English-speaking markets.

**Consequences (already designed in §5.1, §6.4):**
- Ingestion uses 2 country-batches × 4 category buckets × 4 runs/day = **32 newsdata.io credits/day** (well under the 200/day free cap, with 168 to spare for re-ingestion / dedup / experiments).
- The `articles.country` column drives all per-country filtering.
- The persona's `region_country` is part of `bucket_hash`, so a US persona never sees Indian rewrites and vice versa.
- We can extend to `au, de, fr` later — we still fit comfortably within 200 credits/day.

**Risk:** non-Indian news sources can be sparser on the free plan. Mitigation: monitor `articles_country_idx` row counts daily; if a country has < 10 articles/day after a week, drop it.

### Decision 2 — Language: **English only for MVP**

> *Q: support Hindi/regional languages in MVP? Gemini Flash handles multilingual cleanly but persona localisation needs design work.*
> **A: only English is fine for MVP.**

**Consequences (already designed in §6.4, §10):**
- All `newsdata.io` calls use `language=en`.
- The LLM prompt explicitly says "Rewrite for this user in ENGLISH ONLY". Both `impact_headline` and `why_summary` are validated as English-output JSON.
- Persona schema has no `preferred_language` field for MVP (we'll add it when we expand).
- Localised UI strings in the Compose app remain English-only for MVP.

**Future work** (post-MVP): add `personas.preferred_language` and a per-language `impact_rewrites` row keyed on `(article_id, bucket_hash, lang)`. The LLM call multiplies by the number of supported languages, so we'd need to batch more aggressively or move to Gemini paid.

### Decision 3 — Notification cadence: **daily 8 AM IST FCM push**

> *Q: is a daily push notification at 8 AM IST in scope for MVP? Adds FCM but increases engagement materially.*
> **A: Yes, daily notification is required.**

**Consequences (already designed in §6.7, §11):**
- New table `push_tokens` and new endpoint `POST /v1/notifications/token`.
- New endpoint `PUT /v1/notifications/preferences` so users can opt out or change time.
- New cron `02:30 UTC` (= 08:00 IST) → `POST /v1/_internal/cron/push-daily`.
- We send a **`data` payload** (not `notification`) so the Compose app renders the headline + summary from the locally cached `daily_top5` — this keeps the "feel" identical to opening the app.
- New column `daily_top5.push_sent_at` for double-send protection.
- FCM is free; rate limits (600K msgs/min project quota) are far above our needs.

**Edge case:** the daily push fires for the article rendered as `daily_top5[0]`. If a user edited their persona between the 6h cron and the 8 AM push, we want them to receive the *post-edit* top story. The persona-drain cron (every 5 min) ensures that by 02:30 UTC the daily_top5 is already current.

### Decision 4 — Inshort-style headline + 60-word summary

> *Q: product spec says 10 words for headline + 1-2 sentence summary. Confirm with design before finalising LLM prompt.*
> **A: Inshort-style — short headline, ~60-word summary.**

**Choice:** mirror Inshorts' format exactly:
- `impact_headline`: ≤10 words.
- `why_summary`: 50-70 words (target 60), 1-2 short sentences, factual tone, no exclamation.

**Consequences (already designed in §6.4):**
- The Zod schema validates word count (`refine`) and rejects out-of-range LLM output.
- The prompt tells the model the structure and the tone explicitly ("plain English, calm, no exclamation, …").
- Storage cost: ~80 chars headline + ~420 chars summary × ~180 buckets × 50 articles/day × 14-day retention = ~63 MB of `impact_rewrites` text — comfortable inside Supabase 500 MB.
- Network cost: a `daily_top5` payload is ≤ 5 stories × ~600 chars = ~3 KB per call — instant on mobile.

### Decision 5 — Persona editing mid-day allowed

> *Q: allowed mid-day? If yes, we need to invalidate the cached `daily_top5` and possibly regenerate.*
> **A: Yes, mid-day persona edits are supported.**

**Consequences (already designed in §6.6):**
- `PUT /v1/personas` recomputes `bucket_hash`, deletes today's `daily_top5` row, deletes Redis `feed:{user_id}:{today}`, and adds the user to the Redis `persona:dirty:set`.
- The endpoint returns 202; the Compose app shows "Refreshing your feed…" and polls `GET /v1/feed/top5` every 5 s.
- The 5-minute persona-drain cron picks up the dirty set and runs full LLM rewrites for any new bucket combinations not yet cached in `impact_rewrites`.
- Until rewrites land, the API serves a partial response (article-only, no impact rewrite) with `partial: true` so the app can render placeholders.
- This adds at most ~10 LLM calls per persona edit. Even if 5% of DAU (75 users) edit persona daily, that's ~750 calls/day — well within Gemini headroom.

**Anti-abuse:** rate-limit `PUT /v1/personas` to 5 edits / hour / user (Upstash sliding window) so a malicious client can't burn LLM credits.

---

## 15. Recommended Next Steps

1. **Repo scaffold:**
   - `pnpm create hono@latest sowhat-api` (Node adapter).
   - Add `@hono/zod-openapi`, `drizzle-orm`, `@supabase/supabase-js`, `@supabase/auth-helpers`, `@upstash/redis`, `ai`, `@ai-sdk/google`, `@ai-sdk/groq`, `firebase-admin`, `revenuecat-webhooks` (or hand-rolled validator).
2. **Supabase project:**
   - Enable Auth providers: Google OAuth (set up in Google Cloud Console), Email magic link, **Anonymous sign-ins**.
   - Apply DB migrations from §6.3 incl. RLS policies and the `auth.users` → `profiles` insert trigger.
   - Copy `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_JWT_JWKS_URL`.
3. **Upstash Redis** (free) — copy REST URL/token.
4. **Free API keys:** `newsdata.io`, Google AI Studio (Gemini), Groq.
5. **Firebase project:** enable FCM (only — not Auth/Firestore). Download `google-services.json` for the Compose app, generate FCM v1 service-account JSON for the server.
6. **RevenueCat:**
   - Create project, link to Google Play Console.
   - Define one `pro` entitlement, two products: `pro_monthly` (₹99) and `pro_yearly` (₹799).
   - Set webhook → `https://api.sowhat.app/v1/billing/webhook` with a strong shared secret.
7. **Vercel:** push to GitHub, connect, set all env vars (Supabase, Upstash, Gemini, Groq, FCM, RC webhook secret, CRON secret, newsdata.io key).
8. **GitHub Actions cron files:** the three workflow YAMLs from §9.4 (`news-ingest`, `push-daily`, `persona-drain`).
9. **Mobile:**
   - Generate Kotlin client for Compose app via `openapi-generator-cli` (`-g kotlin`, `--library multiplatform`).
   - Add Supabase `gotrue-kt`, RevenueCat `purchases-android`, Firebase Messaging.
   - Build the three-door onboarding (§9A.2).
10. **Ship MVP slice:** onboarding (Google + Email + Anonymous) → persona form → top-5 with Inshort-style cards → bookmarks → daily push. Paywall behind a single "Pro" toggle.
11. **Privacy & legal:** publish privacy policy + ToS before Play Store submission (covers email, Google sign-in, FCM token, RC receipts).

---

## 16. Sources (verified May 2026)

- newsdata.io free-plan pricing & credit-consumption: <https://newsdata.io/blog/pricing-plan-in-newsdata-io/>, <https://newsdata.io/blog/newsdata-credit-consumption/>, <https://newsdata.io/blog/latest-news-endpoint/>
- Vercel Hobby limits & cron: <https://vercel.com/docs/plans/hobby>, <https://vercel.com/docs/functions/limitations>, <https://vercel.com/docs/cron-jobs/usage-and-pricing>, <https://vercel.com/docs/limits>, <https://vercel.com/docs/pricing>
- Cloudflare Workers free-tier limits & pricing: <https://developers.cloudflare.com/workers/platform/limits/>, <https://developers.cloudflare.com/workers/platform/pricing/>, <https://developers.cloudflare.com/workers/configuration/cron-triggers/>, <https://www.digitalapplied.com/blog/edge-computing-cloudflare-workers-development-guide-2026>
- Cloudflare Workers cron triggers: <https://cronjobpro.com/blog/cloudflare-workers-cron>
- Railway pricing & free tier 2026: <https://docs.railway.com/pricing/plans>, <https://docs.railway.com/pricing/free-trial>, <https://docs.railway.com/pricing/faqs>, <https://kuberns.medium.com/railway-free-tier-in-2026-what-you-get-and-when-it-runs-out-2101fdca0998>
- Supabase Auth & MAU pricing: <https://supabase.com/pricing>, <https://supabase.com/docs/guides/platform/manage-your-usage/monthly-active-users>, <https://www.wearefounders.uk/supabase-pricing-2026-every-tier-explained-for-indie-hackers/>
- Supabase vs Neon free tiers: <https://agentdeals.dev/neon-vs-supabase>, <https://www.bytebase.com/blog/neon-vs-supabase/>
- Gemini API free tier: <https://ai.google.dev/gemini-api/docs/rate-limits>, <https://tokenmix.ai/blog/gemini-api-free-tier-limits>
- Groq free tier: <https://tokenmix.ai/blog/groq-free-tier-limits-2026>, <https://www.grizzlypeaksoftware.com/articles/p/groq-api-free-tier-limits-in-2026-what-you-actually-get-uwysd6mb>
- Hono vs Fastify vs NestJS: <https://encore.dev/articles/nestjs-vs-fastify-vs-hono>, <https://nodewire.net/best-nodejs-frameworks-2026/>
- Cloud Run free tier: <https://cloud.google.com/run/pricing>, <https://agentdeals.dev/gcp-free-tier-2026>
- Upstash Redis free tier: <https://upstash.com/docs/redis/help/faq>, <https://upstash.com/pricing>
- Firebase Cloud Messaging quotas (free, unlimited messages, 600K/min project quota): <https://firebase.google.com/docs/cloud-messaging/throttling-and-quotas>, <https://www.buildmvpfast.com/tools/api-pricing-estimator/firebase-fcm>
- RevenueCat pricing (free up to $2,500 MTR, 1% above): <https://www.revenuecat.com/pricing>, <https://lushbinary.com/blog/revenuecat-integration-guide-native-vs-revenuecat-2026/>, <https://github.com/RevenueCat/purchases-android>
- Render / Railway / Fly.io comparisons: <https://render.com/articles/platforms-with-a-real-free-tier-for-developers-in-2026>, <https://techsy.io/en/blog/railway-vs-render-vs-fly-io>
