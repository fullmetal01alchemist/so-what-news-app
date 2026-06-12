---
date: 2026-06-10T10:45:00+05:30
researcher: Anjay Sahoo
project: SoWhat News App
topic: "HLD — Backend Login & Onboarding (persona-capture) flow"
tags:
  [
    system-design,
    hld,
    onboarding,
    persona,
    authentication,
    supabase-auth,
    anonymous-sign-in,
    identity-linking,
    google-oauth,
    email-magic-link,
    bucket-hash,
    timezone-resolution,
    progressive-profiling,
    income-bracket,
    topics-of-interest,
    privacy,
    dpdp,
    gdpr,
  ]
status: complete
git_commit: d972536720dd7da6d09cbb18d3d6bb3b9103342f
branch: master
repository: so-what-news-app
last_updated: 2026-06-10
last_updated_by: Anjay Sahoo
---

# Research / HLD: SoWhat News — Backend Login & Onboarding Flow

**Date**: 2026-06-10 10:45 IST
**Researcher**: Anjay Sahoo
**Git Commit**: `d972536720dd7da6d09cbb18d3d6bb3b9103342f`
**Branch**: `master`
**Repository**: so-what-news-app
**Scope**: **Backend only.** Client (Compose Multiplatform) is described only where it drives a backend contract.
**Builds on**:
- [`2026-05-26-sowhat-news-mvp-tech-stack-architecture.md`](./2026-05-26-sowhat-news-mvp-tech-stack-architecture.md) — the stack: Supabase Auth (Google + Email magic link + **Anonymous**), Postgres, Hono on Vercel, `personas`/`bucket_hash`, RLS, RevenueCat.
- [`2026-06-08-sowhat-news-user-clustering-and-notification-timing.md`](./2026-06-08-sowhat-news-user-clustering-and-notification-timing.md) — the 3-layer clustering model; `bucket_hash = H(country, age_range, life_stage, employment_type, home_owner, income_bracket)`; Country+State+City → IANA timezone at onboarding; topics are a *filter* signal, not a bucket key.

> This doc designs **how a user goes from first-open to a fully-personalised, identity-bearing account**, and exactly what the backend stores and computes along the way. The very next session ("getting relevant news for a persona") consumes the `personas` row + `bucket_hash` this flow produces.

---

## 1. Research Question

> Design the **backend HLD for login + onboarding**. On first open we prompt the user to fill a persona form (age; country/state/city; income bracket [recommend a range]; married/unmarried; children/none; employment type [salaried/businessman/… recommend others]; religion [optional]; home owner/rental; up to 5–6 topics of interest [predefined or free-text, recommend topics]). **After** filling, the user chooses **Continue as Guest (anonymous)** or **sign in with Email / Google / etc.** Define the data model, API surface, the auth/identity mechanics, and the recommended/suggested values.

---

## 2. TL;DR — The Five Decisions

1. **Anonymous-session-first.** On first app open the client silently calls Supabase `signInAnonymously()` (gated by an invisible CAPTCHA / Cloudflare Turnstile). The user gets a real JWT and a **stable `user_id`** *before* filling anything. The persona form is therefore filled **against an authenticated (anonymous) session** — the backend never has to invent a "pre-auth" identity. This is Supabase's officially-documented pattern for "collect data before the user signs in." ([Supabase Anonymous Sign-Ins guide](https://supabase.com/docs/guides/auth/auth-anonymous))

2. **"Continue as Guest" = do nothing; "Google/Email" = link in place.** Because the user is already an anonymous Supabase user:
   - **Guest** → keep the anonymous account as-is. No call needed.
   - **Google** → `linkIdentity({ provider: 'google' })` (requires *Manual Linking* enabled in the Supabase dashboard).
   - **Email** → `updateUser({ email })` then verify via OTP/magic-link.
   In **all three** cases the **`user_id` stays the same**, so the persona we already saved carries over with **zero data migration**. ([Identity Linking guide](https://supabase.com/docs/guides/auth/auth-identity-linking); confirmed: *"the user id remains the same … any data associated with the user's id would be carried over"* — [blog](https://supabase.com/blog/anonymous-sign-ins))

3. **Persona is the backend's source of truth; auth choice only changes the identity row.** The `PUT /v1/personas` contract from the prior docs is the spine. This HLD extends the `personas` schema to hold every onboarding field, computes `life_stage`, `age_range`, `bucket_hash`, and the IANA `timezone` server-side, and treats `religion`, `state`, `city` as **LLM-context-only / non-bucketed** per the clustering doc.

4. **Recommendations are server-driven, not hardcoded in the app.** A single `GET /v1/onboarding/config` endpoint returns the country-localised **income brackets**, the **employment-type** list, the **topic catalog**, and the **recommended topic set** — so we can tune suggestions without an app release. (Defaults/prefill measurably reduce form effort and abandonment — [Zuko](https://www.zuko.io/blog/how-to-use-defaults-to-optimize-your-form-ux), [Reform](https://www.reform.app/blog/how-smart-defaults-reduce-form-errors); delaying auth behind value lifts retention — [Rownd/Localytics](https://rownd.com/blog/why-you-should-delay-authentication-onboarding-in-your-app).)

5. **Privacy by construction.** `religion` and `income` are optional with explicit, separate, purpose-scoped consent (GDPR special-category for religion; India DPDP purpose-limited consent). Sensitive fields never enter the LLM prompt unless their consent flag is set, and **never** enter `bucket_hash`. ([GDPR Art. 9](https://gdpr-info.eu/art-9-gdpr/); [DPLA India DPDP](https://www.dlapiperdataprotection.com/?t=law&c=IN))

```
FIRST OPEN ──► signInAnonymously()+captcha ──► JWT (is_anonymous=true), stable user_id
   │
   ▼
MULTI-STEP PERSONA FORM  (autosaved as a draft against the anon user)
   step1 age │ step2 location │ step3 economic │ step4 household │ step5 topics
   │
   ▼
SUBMIT ──► PUT /v1/personas ──► server computes life_stage, age_range,
   │                              timezone (Country+State+City→IANA), bucket_hash
   │                              → persona_complete=true, enqueue first feed build
   ▼
IDENTITY CHOICE  ┌─ "Continue as Guest"  → keep anonymous user (no-op)
                 ├─ "Continue with Google" → linkIdentity(google)   ┐ same user_id,
                 └─ "Continue with Email"  → updateUser(email)+OTP   ┘ persona carries over
   │
   ▼
ONBOARDED ──► (next session) feed/top5 reads persona + bucket_hash
```

---

## 3. Why Anonymous-First (and why it fits the stated flow exactly)

The product flow is unusual but deliberate: **persona form first, login choice second.** A naïve reading ("collect data with no account, then create one") forces a throwaway server-side "pre-auth draft store" and a migration step at sign-up. We avoid all of that:

- Supabase **Anonymous Sign-In** issues a full, RLS-valid session immediately. The form writes to the real `personas` table under that `user_id`, protected by `auth.uid() = user_id` RLS.
- Converting later (`linkIdentity` for OAuth, `updateUser` for email) **preserves the `user_id`**, so the persona, draft, bookmarks, push token — everything keyed on `auth.users.id` — survives the upgrade untouched. This is the single most important mechanic in this HLD and it is officially guaranteed. ([Anonymous guide](https://supabase.com/docs/guides/auth/auth-anonymous), [blog](https://supabase.com/blog/anonymous-sign-ins))
- "Continue as Guest" is then **literally a no-op** on the backend: the user simply remains anonymous. They can use the app, get a feed, get the daily push. They are *prompted* to link an identity only when they hit a wall that needs one (subscribe to Pro — Play Billing needs a durable account; cross-device sync).

This also satisfies the clustering doc's "everything dynamic": the moment the persona is submitted, the user's `bucket_hash` exists and the next cron tick fills its LLM rewrites — no clustering job runs.

### 3.1 Distinguishing anon vs permanent server-side
The Supabase JWT carries an **`is_anonymous`** boolean claim. Our Hono auth middleware already verifies the JWT via JWKS (prior doc §6.5); it additionally surfaces `is_anonymous` so endpoints and RLS can gate "permanent-only" actions: `(auth.jwt()->>'is_anonymous')::boolean is false`. ([Anonymous guide](https://supabase.com/docs/guides/auth/auth-anonymous))

---

## 4. Onboarding State Machine

The backend tracks onboarding progress on the profile so we can resume an abandoned form, measure step drop-off, and decide what the app shows on relaunch.

```
            signInAnonymously()
ANONYMOUS ───────────────────────►  PROFILE row auto-created (trigger on auth.users INSERT)
   │  onboarding_status = 'started'
   │
   │  PATCH /v1/onboarding/draft (per step, optional)   ← autosave; status='draft'
   ▼
DRAFT ── PUT /v1/personas (all required fields valid) ─► status='persona_complete'
   │                                                      bucket_hash + timezone computed
   │
   │  identity choice:
   │    guest               → identity_type='anonymous'   (status='onboarded')
   │    linkIdentity(google)→ identity_type='google'      (status='onboarded')
   │    updateUser(email)+OTP→ identity_type='email'      (status='onboarded')
   ▼
ONBOARDED
```

- `onboarding_status ∈ {started, draft, persona_complete, onboarded}`.
- The **identity choice does not gate `persona_complete`** — a guest is fully onboarded. `onboarded` is reached as soon as the user finishes the form *and* taps any of the three identity buttons (guest included).
- Resume logic: on relaunch the client calls `GET /v1/me`; if `onboarding_status='draft'` it rehydrates the saved draft (§7.2) and resumes at the first incomplete step.

---

## 5. The Onboarding Fields — Schema, Validation & Recommended Values

Multi-step is the right shape (multi-step out-converts one long form; keep ≤~3 fields/step, show a labelled progress indicator — [Baymard on field count](https://baymard.com/blog/checkout-flow-average-form-fields), [multi-step guidance](https://voiceforms.anvevoice.app/blog/multi-step-form-best-practices/)). Proposed 5 steps:

| Step | Fields | In `bucket_hash`? | Notes |
|---|---|---|---|
| 1 — About you | age, employment_type | age→`age_range` ✅; employment ✅ | age also feeds notification timing (Layer 3) |
| 2 — Where you are | country, state, city | country ✅; state⇒coarsen; city ✗ | resolves IANA `timezone` (Layer 3) |
| 3 — Money & home | income_bracket *(opt)*, home_ownership | both ✅ | income optional + "Prefer not to say" |
| 4 — Household *(+ optional)* | marital_status, has_children, religion *(opt, consented)* | marital+children⇒`life_stage` ✅; religion ✗ | religion = explicit opt-in only |
| 5 — Interests | topics (3–6, predefined + free-text) | ✗ (filter signal) | maps to categories + `impact_tags` |

`bucket_hash = H(country, age_range, life_stage, employment_type, home_ownership, income_bracket)` (from clustering doc §5.3). `life_stage = f(marital_status, has_children)`.

### 5.1 Age — discrete number
- Input: integer. **Validate `13 ≤ age ≤ 120`** (13 = minimum for Play Store data collection / COPPA-style floor; reject below). Store the **raw integer** `age` *and* derive `age_range ∈ {18-24,25-34,35-44,45-54,55+}` (with a `13-17` minor band that, if present, restricts personalisation and disables religion/income prompts). Only `age_range` enters `bucket_hash`; raw `age` is kept for future re-bucketing and analytics.

### 5.2 Country / State / City
- `country`: ISO-3166-1 alpha-2, constrained to the live MVP set `{in, us, gb, ae, sg, ca}` (prior doc). Drives **Layer 1 news fetch** and is part of `bucket_hash`.
- `state`, `city`: free selection from a server-provided gazetteer (or free text fallback). **Neither is a news filter on the free tier** (newsdata.io `region` is paid — clustering doc §3). Their job:
  1. **Resolve `timezone`** — `city → IANA tz` lookup, fall back to `state → tz`, then `country` primary tz. Stored on `profiles.timezone` for the notification scheduler (Layer 3).
  2. **LLM context** — passed to the rewrite prompt ("…for someone in Bengaluru") but **not** bucketed (city is too high-cardinality; state is *coarsened* to a metro/region tier if used at all — clustering doc §5.3 / open Q1).

### 5.3 Income bracket — optional, **country-localised**, recommended band highlighted
Bracketed ranges (not exact figures), non-overlapping, always with **"Prefer not to say"** ([SurveyMonkey income best-practice](https://www.surveymonkey.com/learn/survey-best-practices/how-to-ask-income-survey-questions/)). The app shows the bracket set for the user's selected `country`, with the **modal/median band pre-highlighted as "recommended"** (editable). Served by `GET /v1/onboarding/config`. Suggested annual-income schemes (tune with PM; reference bands from national statistics — [India Statista](https://www.statista.com/statistics/482584/india-households-by-annual-income/), [US Census ASEC], [UK ONS ASHE], [Pollfish cross-country](https://resources.pollfish.com/pollfish-school/household-income-mapping/)):

| Country | Bands (annual, local currency) | Recommended (pre-highlight) |
|---|---|---|
| 🇮🇳 IN (₹) | `<3L` · `3–7L` · `7–15L` · `15–30L` · `30L+` | `7–15L` |
| 🇺🇸 US ($) | `<30k` · `30–60k` · `60–100k` · `100–200k` · `200k+` | `60–100k` |
| 🇬🇧 GB (£) | `<20k` · `20–35k` · `35–60k` · `60–100k` · `100k+` | `35–60k` |
| 🇦🇪 AE (AED) | `<120k` · `120–250k` · `250–500k` · `500k–1M` · `1M+` | `250–500k` |
| 🇸🇬 SG (S$) | `<40k` · `40–80k` · `80–150k` · `150–300k` · `300k+` | `80–150k` |
| 🇨🇦 CA (C$) | `<40k` · `40–70k` · `70–120k` · `120–200k` · `200k+` | `70–120k` |

Stored as a **stable enum token** decoupled from display (e.g. `IN_B3`), so currency formatting/labels live in config, and the value survives a country change prompt. Income changes **tax-slab / affordability framing** in the rewrite → it *is* in `bucket_hash` (clustering doc), but is droppable to prompt-only if live-bucket count gets too high.

### 5.4 Marital status & Children → `life_stage`
- `marital_status ∈ {single, married, prefer_not}`; `has_children ∈ {true, false, prefer_not}`.
- Server derives `life_stage`: `single` (unmarried, no kids) · `couple` (married, no kids) · `parent` (has kids, any marital) · `unknown` (prefer_not). Only `life_stage` enters `bucket_hash`; the raw two fields are retained for analytics/re-derivation. (Folding two booleans into one coarse dimension is the clustering doc's anti-bucket-explosion rule, §5.3.)

### 5.5 Employment type — recommended expanded set
The brief said "Salaried, Businessman etc. — suggest a recommended other relevant type." Recommended enum (served by config; single-select):

`salaried_private` · `salaried_government` · `business_owner` · `self_employed_professional` (doctor/lawyer/CA/consultant) · `freelancer_gig` · `student` · `homemaker` · `retired` · `unemployed` · `prefer_not`

Rationale: salaried-vs-business is the strongest "so what" splitter for tax/policy news (clustering doc §5.3) **and** a notification-timing prior (salaried → commute-window mornings; business → midday). Employment **is** in `bucket_hash`.

### 5.6 Religion — **optional, explicit opt-in, never bucketed**
- `religion ∈ {hindu, muslim, christian, sikh, buddhist, jain, jewish, other, prefer_not}` — **optional**, default unset.
- Requires a **separate, explicit consent toggle** (`consent_religion=true`) before the field is shown/stored, with a one-line purpose statement ("to surface festival/policy stories relevant to you"). Religion is **GDPR special-category data** (Art. 9 — explicit, unbundled consent required for EU/UK users); India's **DPDP Act has no separate sensitive category** but still mandates specific, purpose-limited consent + data minimisation. Keeping it optional + opt-in + purpose-scoped satisfies both. ([GDPR Art. 9](https://gdpr-info.eu/art-9-gdpr/), [DLA Piper India DPDP](https://www.dlapiperdataprotection.com/?t=law&c=IN))
- **Excluded from `bucket_hash`** (cardinality + bias). Used **only** as optional LLM prompt context for festival/policy stories, and **only when `consent_religion=true`**.

### 5.7 Home ownership
- `home_ownership ∈ {owner, renter, living_with_family, prefer_not}`. Sharp framing differences (interest-rate/property-tax vs rent stories). **In `bucket_hash`.**

### 5.8 Topics of interest — 3–6, predefined chips + free-text, recommended set
- Min 3, max 6 selections (product choice — note there is **no published benchmark** for an optimal news cold-start topic count; [ArXiv 2025 news-recsys survey](https://arxiv.org/pdf/2509.12361)). Predefined **chips** are standard cold-start practice (Inshorts/Flipboard/Google News — [Flipboard onboarding](https://medium.com/@kentran27/flipboards-on-boarding-b33f2d8e6831)).
- Recommended catalog (served by config), each mapped to **(a)** a newsdata.io category for Layer-1 fetch and **(b)** `impact_tags` for feed ranking (clustering doc §4, §5.4):

| Topic (display) | newsdata category | impact_tags |
|---|---|---|
| Personal Finance & Investing | business | finance, taxes |
| Jobs & Careers | business | jobs |
| Real Estate & Housing | business | real_estate |
| Startups & Business | business | business_owner |
| Taxes & Policy | politics | taxes, local_law |
| Politics & Government | politics | local_law |
| Technology & AI | technology | tech |
| Gadgets & Consumer Tech | technology | tech, consumer_prices |
| Science | science | tech |
| Health & Wellness | health | health |
| Education | education | education |
| Environment & Climate | environment | energy, environment |
| Crypto & Web3 | business | finance |
| Automobiles | top | consumer_prices |
| Sports | sports | — |
| Entertainment | entertainment | — |

- **Recommended-by-default set** (pre-selected, editable): `Personal Finance & Investing`, `Technology & AI`, `Jobs & Careers`, `Politics & Government` — the four with the highest "so-what" yield; user can deselect.
- **Free-text topics**: normalise to the nearest predefined topic at submit (keyword/embedding match), **store the raw string too**. Unmapped free-text → keyword (`q`) fetch on a low-frequency cron (clustering doc §4).
- Topics are a **filter/ranking signal, NOT a `bucket_hash` key** (they decide *which* stories, not *how* they're framed — clustering doc §5.4).

---

## 6. Data Model (deltas to prior doc §6.3)

### 6.1 `profiles` — identity, onboarding state, consent
```sql
alter table profiles
  add column onboarding_status text not null default 'started',  -- started|draft|persona_complete|onboarded
  add column identity_type      text not null default 'anonymous',-- anonymous|google|email
  add column consent_religion   boolean not null default false,
  add column consent_personalisation boolean not null default true, -- master toggle for sensitive context use
  add column persona_completed_at timestamptz;
-- is_anonymous already mirrored from auth.users; identity_type is the resolved post-link value.
-- timezone / notifications_* already exist (prior doc §6.3).
```

### 6.2 `personas` — extended to hold all onboarding fields
```sql
-- Reconciled with prior doc §6.3 + clustering doc §5.3.
alter table personas
  add column age            smallint,        -- raw discrete number (13..120)
  add column age_range      text,            -- derived: 13-17|18-24|25-34|35-44|45-54|55+
  add column region_state   text,            -- LLM context only (not a news filter on free tier)
  add column marital_status text,            -- single|married|prefer_not
  add column has_children   boolean,         -- null = prefer_not
  add column life_stage      text,           -- derived: single|couple|parent|unknown
  add column employment_type text,           -- enum from §5.5
  add column religion        text,           -- nullable; only if consent_religion
  add column home_ownership  text;           -- owner|renter|living_with_family|prefer_not
-- already present: income_bracket, region_country, region_city, interests[], bucket_hash,
--                  industry (legacy), investor_type[] (legacy/optional).
-- bucket_hash := H(region_country, age_range, life_stage, employment_type, home_ownership, income_bracket)
```
RLS unchanged: `auth.uid() = user_id` on `personas` and `profiles` (prior doc §13.2). Sensitive columns inherit the same row-level isolation.

### 6.3 `onboarding_drafts` — autosave for the multi-step form
```sql
create table onboarding_drafts (
  user_id    uuid primary key references profiles(id) on delete cascade,
  step       smallint not null default 1,        -- last completed step
  payload    jsonb    not null default '{}'::jsonb, -- partial, unvalidated field values
  updated_at timestamptz default now()
);
-- RLS: auth.uid() = user_id. Dropped (or ignored) once onboarding_status='persona_complete'.
```
Draft persistence prevents losing entered data across back-navigation/app-kill, a known abandonment driver ([multi-step best practices](https://voiceforms.anvevoice.app/blog/multi-step-form-best-practices/)). Drafts hold **unvalidated** values; the authoritative validated row only ever lands via `PUT /v1/personas`.

---

## 7. API Surface (additions to prior doc §7)

> **Auth model unchanged:** the client talks to **Supabase Auth directly** for `signInAnonymously` / `linkIdentity` / `updateUser`. Our Hono API only ever **verifies** the resulting JWT (JWKS, cached 1h) and reads `sub` + `is_anonymous`. We mint no tokens.

### 7.1 `GET /v1/onboarding/config`
Server-driven recommendations so we can tune without an app release.
```yaml
get:
  summary: Country-localised onboarding options + recommended defaults
  security: [{ bearerAuth: [] }]   # anon JWT is fine
  parameters: [{ in: query, name: country, schema: { type: string, example: in } }]
  responses:
    '200':
      content: { application/json: { schema: { $ref: '#/components/schemas/OnboardingConfig' } } }
# OnboardingConfig: { income_brackets[], recommended_income_bracket, employment_types[],
#                     topic_catalog[], recommended_topics[], religion_options[], min_topics, max_topics }
```

### 7.2 `PATCH /v1/onboarding/draft` — autosave a step (idempotent upsert)
```yaml
patch:
  summary: Save partial onboarding progress (per step)
  security: [{ bearerAuth: [] }]
  requestBody: { content: { application/json: { schema:
      { type: object, properties: { step: {type: integer}, payload: {type: object} } } } } }
  responses: { '204': { description: saved } }
```
Optional — the client may also hold the draft locally and only call `PUT /v1/personas` once. Server-side draft enables cross-device resume.

### 7.3 `PUT /v1/personas` — the spine (extends prior doc)
Accepts the full validated persona. Server-side, atomically:
1. Validate every field (enums, `age` range, `min_topics ≤ topics ≤ max_topics`, country in MVP set).
2. Derive `age_range`, `life_stage`, normalise free-text topics → tags.
3. Resolve `timezone` from `country/state/city`.
4. Gate sensitive fields: drop `religion` unless `consent_religion=true`; null `income_bracket` if "prefer_not".
5. Compute `bucket_hash`.
6. Upsert `personas`; set `profiles.onboarding_status='persona_complete'`, `persona_completed_at=now()`.
7. Invalidate/queue feed build: `redis.sadd('persona:dirty:set', userId)` + delete today's `daily_top5` (prior doc §6.6).
8. Delete the `onboarding_drafts` row.

Returns **200** (bucket already has cached rewrites → feed ready) or **202** (new bucket → regenerating; client polls `GET /v1/feed/top5` until `partial=false`). Identical to the prior doc's persona-edit semantics — onboarding is just the **first** persona write. Rate-limited 5/hr/user (anti-abuse, prior doc §14).

### 7.4 Identity finalisation — mostly client-SDK; one backend reconciliation hook
Linking happens **client-side** via Supabase SDK. The backend reacts via a tiny confirm endpoint (called by the client after a successful link, and/or by a Supabase Auth Hook) to record `identity_type` and flip `onboarding_status='onboarded'`:
```yaml
post: # /v1/me/identity
  summary: Record the resolved identity after a successful link/guest choice
  security: [{ bearerAuth: [] }]   # the NEW (possibly non-anon) JWT
  requestBody: { content: { application/json: { schema:
      { type: object, required: [choice],
        properties: { choice: { type: string, enum: [guest, google, email] } } } } } }
  responses: { '200': { description: profile updated },
               '409': { description: identity already belongs to another account (see §8.3) } }
```
The server re-reads `is_anonymous` from the (refreshed) JWT to confirm the link actually happened before setting `identity_type` to a permanent value.

### 7.5 `GET /v1/me` (extended)
Now also returns `onboarding_status`, `identity_type`, `consent_religion`, and (when `draft`) enough to resume. Drives the relaunch/resume decision.

---

## 8. Sequence Flows

### 8.1 Happy path — Guest
```
App(first open) ─ signInAnonymously(captcha) ─► Supabase ─► JWT(is_anonymous=true), user_id=U
App ─ GET /v1/onboarding/config?country=in ─► API ─► brackets/topics/recommendations
App ─ (PATCH /v1/onboarding/draft per step)* ─► API ─► 204
App ─ PUT /v1/personas {full persona} ─► API: validate→derive→bucket_hash→timezone
                                             upsert personas; status=persona_complete
                                             sadd persona:dirty:U
                                         ◄─ 202 {partial:true}
App (taps "Continue as Guest") ─ POST /v1/me/identity {choice:guest}
                                         ─► API: identity_type=anonymous; status=onboarded ◄─ 200
[next cron tick] persona-drain ─► LLM rewrites for bucket(U) ─► daily_top5(U) built
App ─ GET /v1/feed/top5 ─► 200 {partial:false, stories:[…]}
```

### 8.2 Happy path — link Google (or Email) after the form
```
…(persona submitted as above, user_id=U)…
App (taps "Continue with Google")
  ─ supabase.auth.linkIdentity(Google)   # requires Manual Linking enabled in dashboard
       └─► OAuth round-trip ─► identity linked to SAME user_id=U  (persona unchanged)
  ─ (session refreshes; JWT now is_anonymous=false, provider=google)
App ─ POST /v1/me/identity {choice:google}
       ─► API verifies is_anonymous=false ─► identity_type=google; status=onboarded ◄─ 200

# Email variant:
App ─ supabase.auth.updateUser({ email })  ─► OTP/magic-link sent
App ─ (user enters OTP / clicks link) ─► email identity confirmed, user_id=U unchanged
App ─ POST /v1/me/identity {choice:email} ─► identity_type=email; status=onboarded
```
Key property: **persona written under the anonymous `U` is the persona of the permanent account** — no copy, no merge. ([blog confirmation](https://supabase.com/blog/anonymous-sign-ins))

### 8.3 Conflict — the email/Google identity already belongs to an existing account
`linkIdentity` / `updateUser` **fails** if the candidate identity is already linked to another user (Supabase enforces unique emails). ([Identity Linking guide](https://supabase.com/docs/guides/auth/auth-identity-linking))
```
linkIdentity(Google) ─► ERROR (identity already exists)        # exact code: confirm at runtime
App shows: "This account already exists. Sign in to continue."
App ─ signOut(anon U) ─ signInWith(Google) ─► existing user_id=E (has its OWN older persona)
       │
       ▼  Resolution (product call):
   Option A (default): keep E's existing persona; discard the throwaway anon persona of U.
   Option B (merge): POST /v1/me/account/merge {from:U}  → API copies U's persona/bookmarks
                     into E IF E has none, else prompts which to keep. App-side orchestration;
                     the anon user U is then deleted (DELETE /v1/me on U's token).
```
For MVP, **Option A (discard anon draft)** is simplest and lowest-risk; Option B is a Phase-2 nicety. Either way the backend never silently overwrites an existing account's persona.

---

## 9. Security, Privacy & Abuse (additions to prior doc §13)

- **Anonymous endpoint abuse:** bad actors can spam `signInAnonymously` to bloat `auth.users`. Mitigate with **invisible CAPTCHA / Cloudflare Turnstile** (Supabase: Authentication → Bot and Abuse Protection) and the default **30 anon sign-ins/hour/IP** rate limit. ([Anonymous guide](https://supabase.com/docs/guides/auth/auth-anonymous), [rate limits](https://supabase.com/docs/guides/auth/rate-limits), [CAPTCHA](https://supabase.com/docs/guides/auth/auth-captcha))
- **MAU billing flag:** Supabase docs do **not** state that anonymous users are excluded from MAU; treat anon users as **potentially billable** and confirm with Supabase before relying on "guests are free." At MVP scale (≤5K) it's inside the 50K free MAU regardless. ([MAU usage](https://supabase.com/docs/guides/platform/manage-your-usage/monthly-active-users))
- **Sensitive-data minimisation:** `religion` stored only with `consent_religion=true`; `income` optional with "prefer not to say"; both **excluded from `bucket_hash`** (income is in the hash but droppable; religion is never in it). Neither enters the LLM prompt unless its consent flag is set. ([GDPR Art. 9](https://gdpr-info.eu/art-9-gdpr/))
- **Minors:** `age < 13` rejected; `13–17` band disables religion/income prompts and restricts personalisation depth (product/legal call).
- **PII boundary unchanged:** email lives in `auth.users` (Supabase-managed); `personas` holds no email/name/phone; LLM prompt receives only the anonymous bucket descriptor (prior doc §9A.6, §13.1).
- **Manual Linking** must be explicitly enabled in the Supabase dashboard for `linkIdentity` to work — a deploy-time checklist item, not a code change. ([Identity Linking guide](https://supabase.com/docs/guides/auth/auth-identity-linking))
- **`/v1/me/identity` & `/v1/me/account/merge`** require a **non-anonymous** JWT (verified via `is_anonymous=false`) to perform permanent-account actions; RLS enforces row isolation regardless.

---

## 10. How This Feeds the Next Session ("relevant news for a persona")

This flow's **only durable output that personalisation consumes** is the `personas` row, specifically:
- `bucket_hash` → the cache key for `impact_rewrites` (which articles get reframed for this user).
- `interests`/derived tags → **feed ranking** (Jaccard overlap with `article.impact_tags`, clustering doc §5.4).
- `region_country` → Layer-1 fetch scoping + part of `bucket_hash`.
- `timezone` + `age_range` + `employment_type` → Layer-3 notification timing.
- `region_state/region_city/religion` (consented) → LLM prompt **context** only.

The next session designs the read/ranking/rewrite path on top of exactly these fields — nothing else from onboarding is needed downstream.

---

## 11. Build Order (backend)

1. Schema migrations §6 (profiles/personas/onboarding_drafts deltas) + RLS on new tables/columns.
2. `GET /v1/onboarding/config` with the §5 income/employment/topic/religion catalogs (seed data, country-localised).
3. Enable in Supabase dashboard: **Anonymous sign-ins**, **Manual Linking**, **CAPTCHA/Turnstile**.
4. `PATCH /v1/onboarding/draft` (autosave) + `GET /v1/me` extensions.
5. `PUT /v1/personas` v2: validation → derive `age_range`/`life_stage` → topic normalisation → timezone resolution → `bucket_hash` → dirty-queue enqueue → draft cleanup.
6. `POST /v1/me/identity` reconciliation + `is_anonymous` re-check; conflict path §8.3 (Option A).
7. Analytics: emit step-completion + drop-off events from draft/persona endpoints to measure abandonment.
8. (Phase 2) `POST /v1/me/account/merge` for the conflict-merge path; state coarsening scheme (open Q).

---

## 12. Resolved Decisions & Open Questions

> Decisions taken 2026-06-10. Items marked **⟳** still require a runtime/external confirmation before they harden.

1. **State coarsening** (inherited from clustering doc Q1) — **DECIDED:** keep `state` out of `bucket_hash` entirely (timezone-only). State/City are **LLM context only, never a news filter**. No metro/region tiering for MVP. *(Updates §5.2, §6.2: `region_state` stays context-only; `bucket_hash` definition unchanged.)*
2. **Income in `bucket_hash`** — **DECIDED:** **keep it** (sharper tax/affordability framing). Remains droppable to prompt-only later if live-bucket count exceeds LLM budget (one-line tuning change + cache flush), but ships in the hash. *(Confirms §5.3, §6.2.)*
3. **Religion** — **DECIDED:** **defer to Phase 2.** Do not collect for MVP — the consent overhead (GDPR Art. 9 / DPDP) isn't worth it at launch. Drop the religion step/field and `consent_religion` plumbing from the MVP build; re-introduce in Phase 2. *(Affects §5, §5.6, §6.1, §6.2, §9, §11 build order — religion-gating becomes Phase-2 scope.)*
4. **Guest MAU billing** ⟳ — **ACTION:** confirm with Supabase whether anonymous users count toward MAU **before** scaling assumptions. At MVP scale (≤5K) it's inside the 50K free MAU regardless, so this gates scaling, not launch. *(See §9 MAU billing flag.)*
5. **Conflict resolution policy** — **DECIDED:** ship **Option A (discard anon draft)** for MVP. Keep the existing account's persona; discard the throwaway anon persona on duplicate-identity. Option B (merge) is Phase 2. *(Confirms §8.3, §11 step 6.)*
6. **Minimum topics** ⟳ — **TENTATIVE:** `min 3` (current default). No public benchmark exists for news cold-start; revisit via A/B post-launch. Keep `min_topics=3` server-driven in `GET /v1/onboarding/config` so it's tunable without an app release. *(See §5.8, §7.1.)*
7. **Exact Supabase error code** for the duplicate-identity case ⟳ — **ACTION:** verify at runtime whether it's `identity_already_exists` vs `email_exists` before hardcoding the client branch. *(See §8.3.)*

---

## 13. Sources (verified June 2026)

### Supabase anonymous → permanent linking
- Anonymous Sign-Ins guide (data-before-auth pattern, captcha, rate limit, `is_anonymous`): <https://supabase.com/docs/guides/auth/auth-anonymous>
- Anonymous sign-ins announcement (user_id preserved, data carries over): <https://supabase.com/blog/anonymous-sign-ins>
- Identity Linking (Manual Linking required; link fails on duplicate identity): <https://supabase.com/docs/guides/auth/auth-identity-linking>
- Kotlin SDK refs — signInAnonymously / linkIdentity / updateUser: <https://supabase.com/docs/reference/kotlin/auth-signinanonymously>, <https://supabase.com/docs/reference/kotlin/auth-linkidentity>, <https://supabase.com/docs/reference/kotlin/auth-updateuser>
- CAPTCHA / rate limits / MAU: <https://supabase.com/docs/guides/auth/auth-captcha>, <https://supabase.com/docs/guides/auth/rate-limits>, <https://supabase.com/docs/guides/platform/manage-your-usage/monthly-active-users>
- Auth error codes: <https://supabase.com/docs/guides/auth/debugging/error-codes>

### Onboarding / forms / cold-start
- Delay auth → retention (Localytics 28%): <https://rownd.com/blog/why-you-should-delay-authentication-onboarding-in-your-app>
- Progressive profiling: <https://auth0.com/blog/progressive-profiling/>, <https://www.descope.com/learn/post/progressive-profiling>
- Multi-step form best practices (autosave, steps, progress): <https://voiceforms.anvevoice.app/blog/multi-step-form-best-practices/>
- Field-count evidence (Baymard): <https://baymard.com/blog/checkout-flow-average-form-fields>, <https://baymard.com/blog/checkout-optimization-from-16-fields-to-8>
- Smart defaults / prefill: <https://www.zuko.io/blog/how-to-use-defaults-to-optimize-your-form-ux>, <https://www.reform.app/blog/how-smart-defaults-reduce-form-errors>
- Income question best-practice + brackets: <https://www.surveymonkey.com/learn/survey-best-practices/how-to-ask-income-survey-questions/>, <https://resources.pollfish.com/pollfish-school/household-income-mapping/>, <https://www.statista.com/statistics/482584/india-households-by-annual-income/>
- Cold-start topic elicitation / news-recsys gap: <https://www.emergentmind.com/topics/cold-start-personalization>, <https://arxiv.org/pdf/2509.12361>, <https://medium.com/@kentran27/flipboards-on-boarding-b33f2d8e6831>

### Privacy / sensitive attributes
- GDPR Art. 9 special-category (religion) + explicit consent: <https://gdpr-info.eu/art-9-gdpr/>, <https://www.reform.app/blog/how-to-collect-consent-for-special-category-data>
- India DPDP (no separate sensitive category; purpose-limited consent): <https://www.dlapiperdataprotection.com/?t=law&c=IN>, <https://www.cookieyes.com/blog/india-digital-personal-data-protection-act-dpdpa/>

---
