---
date: 2026-06-08T10:45:00+05:30
researcher: Anjay Sahoo
project: SoWhat News App
topic: "User clustering strategy + per-user notification send-time, constrained by newsdata.io capabilities"
tags:
  [
    system-design,
    clustering,
    segmentation,
    persona,
    bucket-hash,
    k-prototypes,
    hdbscan,
    notification-timing,
    send-time-optimization,
    timezone,
    newsdata.io,
    fcm,
    dynamic-clustering,
  ]
status: complete
git_commit: d972536720dd7da6d09cbb18d3d6bb3b9103342f
branch: master
repository: so-what-news-app
last_updated: 2026-06-08
last_updated_by: Anjay Sahoo
---

# Research: User Clustering & Per-User Notification Timing for SoWhat News

**Date**: 2026-06-08 10:45 IST
**Researcher**: Anjay Sahoo
**Git Commit**: `d972536720dd7da6d09cbb18d3d6bb3b9103342f`
**Branch**: `master`
**Repository**: so-what-news-app
**Builds on**: [`2026-05-26-sowhat-news-mvp-tech-stack-architecture.md`](./2026-05-26-sowhat-news-mvp-tech-stack-architecture.md)

---

## 1. Research Question

Two design questions, plus a constraint:

1. **Notification timing** — *what time should we send the daily push to each user?* Region (timezone) and age (different cohorts engage at different times) clearly matter, and there may be other factors.
2. **Clustering** — *how do we cluster users from the onboarding persona, and should we use a clustering algorithm?* And critically: **everything must be dynamic** — as new users onboard and new attribute combinations appear, clusters should form automatically without a manual rebuild.
3. **Constraint** — *the clustering must respect what `newsdata.io` can actually do.* There's no point clustering along a dimension the news API can't filter on.

The persona we collect at onboarding (this doc's source of truth, reconciled with the schema in the prior doc §6.3):

| Field | Type | Cardinality |
|---|---|---|
| Age | number → bucketed (`18-24`, `25-34`, `35-44`, `45-54`, `55+`) | low (5) |
| Country | ISO-2 | low (~6 at MVP) |
| State | string | medium |
| City | string | high |
| Income bracket | bucketed | low (~4) |
| Married / Unmarried | bool | 2 |
| Children / No children | bool | 2 |
| Employment (Salaried / Businessman / …) | categorical | low (~4-5) |
| Religion (optional) | categorical | low (~6) |
| Home owner / Rental | categorical | 2-3 |
| Topics of interest (up to 5-6) | multi-valued set; predefined **or** free-text | high / open |

---

## 2. TL;DR — The Answer in One Picture

**Do not build one clustering system. You have three different clustering problems, and each one may only use the dimensions its *downstream consumer* can actually act on.** Trying to feed the full persona into a single cluster is the mistake; the persona gets *split* across three layers:

```
                       PERSONA (11 fields)
                              │
        ┌─────────────────────┼──────────────────────────┐
        ▼                     ▼                          ▼
 ┌──────────────┐    ┌──────────────────────┐   ┌────────────────────┐
 │ LAYER 1      │    │ LAYER 2              │   │ LAYER 3            │
 │ NEWS-FETCH   │    │ IMPACT-REWRITE       │   │ NOTIFICATION-TIME  │
 │ cluster      │    │ bucket               │   │ cluster            │
 ├──────────────┤    ├──────────────────────┤   ├────────────────────┤
 │ Consumer:    │    │ Consumer: the LLM    │   │ Consumer: FCM cron │
 │ newsdata.io  │    │ rewrite cache        │   │ scheduler          │
 │              │    │                      │   │                    │
 │ Can ONLY use:│    │ Uses a COARSE subset │   │ Uses:              │
 │ • country    │    │ of persona as a      │   │ • timezone         │
 │ • category   │    │ deterministic key,   │   │   (Country+State+  │
 │ • language   │    │ rest as LLM context  │   │    City→IANA tz)   │
 │ • keyword    │    │                      │   │ • age cohort prior │
 │              │    │ {country, age_range, │   │ • employment prior │
 │ {country} ×  │    │  life_stage, employ, │   │ • learned per-user │
 │ {category}   │    │  home_owner, income} │   │   open histogram   │
 └──────────────┘    └──────────────────────┘   └────────────────────┘
   ~6×4 = 24            ~150-200 buckets            ~25-38 tz buckets
   fetch combos         (cached LLM rewrites)        → per-user later
```

**Headline recommendations:**

1. **Clustering algorithm?** — **No, not for MVP.** Use **deterministic composite-key bucketing** (the `bucket_hash` already in the prior doc). It is interpretable, instant to assign, needs no training pipeline, and — crucially — is **inherently dynamic**: a new bucket *exists the moment a user's attributes hash to a new combination*. No clustering job ever has to "run" to create it. ML clustering (k-prototypes / HDBSCAN) is a **Phase 2** upgrade once you have enough users that buckets are statistically meaningful and the rule-based ones are demonstrably too coarse.
2. **What newsdata.io can cluster on** — almost nothing demographic. On the free tier it segments only by **country, category, language, keyword**. State/city, AI-tags and sentiment filters are **paid**. Geo-radius and demographics are **impossible on any plan**. → All persona-driven personalization happens in **Layer 2 (LLM rewrite)**, never in the fetch.
3. **Notification time** — a **4-tier model**: start with `8 AM local` via timezone-bucketed fan-out (Tier 0), nudge by **(age cohort × employment)** prior (Tier 1), shift to your audience's empirical **best hour** once you have open data (Tier 2), then **per-user learned send-time histograms** (Tier 3). This is exactly what Braze/MoEngage/OneSignal/Airship all converge on.

---

## 3. The Hard Constraint: What newsdata.io Can Actually Segment On

This decides everything about Layer 1, so it comes first. Verified June 2026 (sources in §11).

| Dimension | Can newsdata.io filter on it? | Free or paid |
|---|---|---|
| **Country** | ✅ up to 5 codes/query (10 on paid) | **Free** |
| **Category** (17 topics: business, politics, technology, health, …) | ✅ up to 5/query | **Free** |
| **Language** (89) | ✅ up to 5/query | **Free** |
| **Keyword** (`q` / `qInTitle` / `qInMeta`) | ✅ 100-char limit on free | **Free** |
| **Domain / priority-domain** | ✅ | **Free** |
| **Recency** (`timeframe`, last 48h) | ✅ | **Free** |
| **Sub-country region / state** (`region` param) | ⚠️ yes but **Professional+** | **Paid** |
| **City / district** | ⚠️ only via paid AI `region` text label | **Paid (Corporate)** |
| **AI tags** (`ai_tag` micro-topics), **sentiment filter** | ⚠️ | **Paid (Professional+)** |
| **Lat/long / geo-radius** | ❌ does not exist | n/a |
| **Demographic (age/income/marital/homeowner/religion)** | ❌ does not exist | n/a |

**The consequence for clustering:** The news API is demographically blind. It cannot return "news for a 30-yr-old salaried homeowner." It can only return "business + politics news for `country=in`, in English." Therefore:

- **Layer 1 (fetch) clusters can only be `{country} × {category} × language=en`.** That's it. ~6 countries × 4 category buckets = **~24 fetch combinations**, which is exactly the `COUNTRY_BATCHES × CATEGORY_BUCKETS` design already in the prior doc §6.4. Nothing about age, income, marital status, or home ownership can influence the fetch.
- **All persona richness lives in Layer 2**, where the LLM "collides" a fetched article with the persona descriptor. State/City, marital status, children, homeowner, income, religion — none of these touch newsdata.io. They are **prompt context for the rewrite**, and (for the low-cardinality ones) **keys for the rewrite cache**.

> **Decision: stay on the free tier for MVP and treat State/City as LLM context only, not as a news filter.** Paying for `region` (Professional, ~$350/mo) to get state-level news is a Phase-2 question gated on whether users actually demand hyper-local stories. For now, India-wide / country-wide news reframed *for someone in Bengaluru* is the product.

---

## 4. Layer 1 — News-Fetch Clustering (already designed; confirmed)

No change from the prior doc §5.1 / §6.4. Restated here only to close the loop:

- Fetch key = `{country-batch} × {category-bucket}`, `language=en`, 4 runs/day.
- `COUNTRY_BATCHES = [[in,us,gb],[ae,sg,ca]]` (≤5 countries/query free limit).
- `CATEGORY_BUCKETS = ['business,politics','technology,science','health,environment','top']`.
- Budget: 2 batches × 4 buckets × 4 runs = **32 credits/day** of the 200 free.
- **Topics of interest → category mapping:** the user's free-text/predefined topics must be mapped onto newsdata.io's **17 fixed categories** to drive the fetch. Maintain a `topic → newsdata_category` lookup (e.g. "AI"/"Gadgets" → `technology`, "Investing"/"Stocks" → `business`). Free-text topics that don't map to a category fall back to keyword (`q`) search on a separate low-frequency cron, or are simply served via the closest category. **Topics are a *filter/ranking* signal, not a reframing signal** — they decide *which* articles a user sees, not *how* the headline is rewritten.

This layer is dynamic by construction: adding a 7th country is one entry in `COUNTRY_BATCHES`.

---

## 5. Layer 2 — Persona → Impact-Rewrite Bucketing (the core of your question)

### 5.1 Should we use a clustering algorithm? — No, use deterministic bucketing (for now)

Your persona is **overwhelmingly categorical** (country, employment, marital, children, homeowner, religion) with a few bucketed ordinals (age, income). Classic k-means is the wrong tool (it assumes continuous Euclidean space and means as centroids). The real choice is **deterministic rule-based bucketing vs. categorical ML clustering (k-modes / k-prototypes / HDBSCAN / LCA)**.

The practitioner consensus for a small, MVP-scale user base is clear: **deterministic bucketing wins.**

| | Deterministic `bucket_hash` (recommended MVP) | ML clustering (k-prototypes / HDBSCAN) |
|---|---|---|
| Interpretability | Total — bucket *is* its descriptor | Opaque; needs post-hoc labelling |
| Cold-start a new user | Trivial — attributes → hash, instant | Assign to nearest prototype (needs a trained model) |
| **Dynamic / auto-create** | **Free** — new combo = new bucket, no job runs | Needs assign-online + periodic batch recluster |
| Training pipeline | None | Required (fit, choose *k*, drift-monitor, redeploy) |
| Stability | Deterministic, reproducible | Sensitive to seed/init/`gamma` |
| Statistical validity at small N | N/A (rules encode domain knowledge) | Poor — clusters not meaningful below ~thousands |
| Risk | **Bucket explosion** if too many fields (see §5.3) | Over-engineering for MVP |

**The prior doc already chose this** (`bucket_hash` in personas table, §6.3). This research *confirms* that choice and adds the discipline to make it work: **field selection** (§5.3) and a **graduation path** (§5.5).

### 5.2 Why deterministic bucketing satisfies "everything must be dynamic"

This is the part worth being explicit about, because it directly answers *"as new users onboard, clusters should automatically be created."*

- A bucket is **not a stored, pre-enumerated entity**. It is the *output of a pure function* `bucket_hash = H(coarsen(persona))`.
- When a user onboards, the server computes their hash on the spot. If `impact_rewrites` already has rows for that `(article, bucket_hash)`, they're reused (cache hit → free). If the combination has **never been seen**, the next ingestion/persona-drain cron tick generates the LLM rewrites for it and caches them. **The "cluster" came into existence automatically, with zero clustering computation.**
- This is the cheapest possible "dynamic clustering": there is no model to retrain, no *k* to re-choose, no drift to monitor. The set of live buckets is simply `SELECT DISTINCT bucket_hash FROM personas` at any moment — see `personas.distinctBuckets()` already used in the prior doc's §6.4 rewrite loop.

ML clustering, by contrast, is dynamic only through extra machinery (assign-online + nightly batch recluster + drift triggers). You take on that machinery **only** when the payoff (tighter, behavior-aware segments) justifies it.

### 5.3 The real engineering problem: bucket explosion → field selection

If you naïvely hash **all** persona fields, every user becomes their own bucket and the rewrite cache stops sharing — you'd run the LLM once per user, blowing the Gemini 1500 RPD budget. The full cross-product is astronomical:

```
age(5) × country(6) × state(~30) × city(100s) × income(4) × married(2)
      × children(2) × employment(5) × religion(6) × homeowner(3) × topics(C(20,5)…)
```

So the `bucket_hash` must be a **deliberately coarse subset** — only the fields that change *how a news event is framed* (its "so what"), not the fields that merely filter *which* news, and not high-cardinality fields.

**Field-by-field allocation:**

| Persona field | Layer 1 (fetch) | In `bucket_hash`? | Layer 3 (timing) | Notes |
|---|---|---|---|---|
| **Age** (bucketed) | ✗ | ✅ **yes** | ✅ prior | Drives both framing (life-stage concerns) and send-time |
| **Country** | ✅ filter | ✅ **yes** | ✅ → timezone | Already in bucket_hash; gates news + tax/law context |
| **State** | ✗ (paid) | ⚠️ **coarsen** (metro tier / region group, not raw) | ✅ → timezone | Raw state explodes the hash; group into ~3-5 tiers or fold into timezone only |
| **City** | ✗ | ❌ **no** (LLM context only) | ✅ → timezone | Too high-cardinality for a key; pass as prompt text |
| **Income bracket** (bucketed) | ✗ | ✅ **yes** | ✗ | Changes tax-slab / affordability framing |
| **Married / Unmarried** | ✗ | ✅ **yes** (fold into `life_stage`) | ✗ | Combine with children → `life_stage` |
| **Children / No children** | ✗ | ✅ **yes** (fold into `life_stage`) | ✗ | e.g. `single`, `couple`, `parent` |
| **Employment** (salaried/business) | ✗ | ✅ **yes** | ✅ prior | Salaried vs business → very different tax/policy "so what" AND different commute/wake times |
| **Religion** (optional) | ✗ | ❌ **no** (LLM context only) | ✗ | Marginal for most news; including it explodes the hash and risks bias. Pass as optional prompt context only for festival/policy stories |
| **Home owner / Rental** | ✗ | ✅ **yes** | ✗ | Interest-rate / rent / property-tax framing differs sharply |
| **Topics of interest** | ✅ → category map | ❌ **no** (it's a *filter*, not a frame) | ✗ | Decides *which* articles match this user, via `impact_tags` overlap; not part of the rewrite cache key |

**Recommended `bucket_hash` =** `H(country, age_range, life_stage, employment_type, home_owner, income_bracket)`
where `life_stage = f(married, children)`.

Rough live-bucket count (after coarsening, only realised combinations exist):
`6 countries × 5 ages × 3 life_stages × 4 employment × 3 homeowner × 4 income` = 4,320 *possible*, but **realised** buckets for 5K users cluster heavily → expect **~150-250 live buckets**, consistent with the prior doc's ~180 estimate. Cache sharing stays healthy (≈20-30 users/bucket).

> **Tuning knob:** if live buckets climb too high (LLM budget pressure), drop `income_bracket` or `home_owner` from the hash and move them to prompt-only context. If buckets are too coarse (users complain rewrites feel generic), add a field back. Because it's a pure function, re-tuning is a one-line change + a cache flush — no model retrain.

### 5.4 Topics of interest: encoding the multi-valued set

Topics are a **set** (5-6 values, predefined or free-text) and are handled as a **match/ranking** signal, not a cluster key:

- **Predefined topics** → map to `newsdata.io` categories (Layer 1) **and** to the article `impact_tags` taxonomy (`finance, taxes, jobs, real_estate, tech, health, …` from prior doc §6.4). A user sees an article in their `daily_top5` when the article's `impact_tags` intersect the user's topic-derived tags.
- **Free-text topics** → normalise to the nearest predefined topic at onboarding (simple embedding/keyword match), store the raw string too. Unmapped free-text falls back to a keyword (`q`) fetch.
- **For ranking within the feed**, score articles per user by **Jaccard overlap** between the user's topic-tag set and the article's tag set. (Phase 2: replace with topic **embeddings** + cosine if free-text dominates.)

This keeps topics out of `bucket_hash` (they'd explode it) while still personalising *which* stories surface.

### 5.5 Graduation path to ML clustering (Phase 2, optional)

Move to real clustering **only when** (a) user base is in the tens of thousands, (b) you have *behavioral* signal (opens, dwell, bookmarks) worth clustering on, and (c) deterministic buckets are demonstrably too coarse. Then:

- **Algorithm:** **k-prototypes** (Python `kmodes` lib) — purpose-built for mixed numeric+categorical; or **HDBSCAN on a precomputed Gower distance matrix** if you want auto-*k* and outlier handling (no need to pick *k*). **LCA** is an option if you want soft/probabilistic membership.
- **Choosing k:** silhouette on a Gower/categorical distance matrix, seed-averaged k-modes cost-elbow; or sidestep entirely with HDBSCAN/LCA.
- **Dynamic strategy:** **assign-online + periodic batch recluster** — assign each new user to the nearest existing prototype (O(k), instant) for cold-start, then nightly/weekly recompute clusters over a sliding window to correct drift and discover/retire segments.
- **Re-cluster triggers:** volume threshold (N new users / X% growth) **and** distribution drift via **PSI / Chi-square** on incoming attribute distributions vs the reference batch.
- **Where it plugs in:** the cluster label simply *replaces* `bucket_hash` as the rewrite-cache key. The rest of the pipeline (Layer 1, Layer 3, the LLM prompt) is unchanged. This is why deterministic-now / ML-later is low-regret.

---

## 6. Layer 3 — Per-User Notification Timing

### 6.1 The factors, ranked by signal strength

1. **Region → timezone (strongest, mandatory).** "8 AM" only means anything in local time. Derive an IANA timezone from `Country + State + City` at onboarding (a city→tz lookup; fall back to country's primary tz). This is the dimension that *must* be right — sending IST-timed pushes to a US user at 3 AM kills retention. Store on `profiles.timezone` (already exists).
2. **Age cohort (good cold-start prior).** Empirically: younger users skew **late-night / later-morning**; older users skew **early-morning**. Use as a *prior*, overridden later by real data:
   - `18-24` → ~9 AM local (or an evening 8-10 PM option)
   - `25-44` → ~7-8 AM local (commute window)
   - `45+` → ~6:30-7:30 AM local
3. **Employment type (secondary prior).** Salaried → morning commute window (7-9 AM) is prime. Business owners → slightly later / midday. Folds in naturally with age.
4. **Per-user observed behavior (strongest *once available*).** Every major platform (Braze, MoEngage, OneSignal, Airship, CleverTap) converges on the same model: a **recency-weighted histogram of each user's open times (day-of-week × hour, in local tz), over a ~60-day window, blended with a population model, with a daytime-bias guard, recomputed weekly.** This beats send-now by ~23% and timezone-only by ~10% (OneSignal, 42B-notification study).

Other useful factors documented but lower priority for MVP: **OS** (Android opens ~10.7% vs iOS ~4.9% — compute best-hour per OS later; you're Android-first so moot now), **day-of-week** (Tue/Wed slightly higher).

### 6.2 The tiered model (build in this order)

| Tier | Send time =  | Needs | When |
|---|---|---|---|
| **0 — Global default** | `08:00` **local**, via timezone-bucket fan-out; quiet-hours guard 11 PM–6 AM | nothing | MVP day 1 |
| **1 — Segment prior** | Tier 0 nudged by **(age cohort × employment)** | persona only | MVP day 1 |
| **2 — Population mode** | your audience's empirical **best hour** (per region/OS) | aggregate open data | once you have ~weeks of opens |
| **3 — Per-user learned** | recency-weighted open histogram, blended w/ Tier-2, daytime bias | ≥~2-4 weeks of per-user opens | Phase 2 |

This mirrors the industry-standard cold-start → personalised progression and degrades gracefully: a user with no history sits at Tier 1; a heavy user graduates to Tier 3.

### 6.3 Notification timing IS also a clustering problem — but a different cluster

Note the symmetry with §2: the **timing cluster** is `{timezone} × {age_cohort} × {employment}` — a *different coarse subset* of the persona than the rewrite `bucket_hash`. It is also **dynamic by construction**: a new timezone bucket appears the moment the first user from a new region onboards. ~25-38 IANA timezone buckets cover the planet (incl. half-hour zones like IST).

### 6.4 Engineering: timezone-bucket fan-out → per-user rows

- **Tiers 0-2 (static/segment time): timezone-bucketed fan-out.** A scheduler runs **every 15-30 min in UTC**; on each tick it computes "which timezone buckets just reached their target local hour?" and fans those out via FCM. A global daily send completes over a rolling 24h. This is cheap and is the recommended MVP pattern (matches OneSignal "Optimize by User Timezone"). It **replaces** the prior doc's single fixed `02:30 UTC = 08:00 IST` cron (§6.7) with a timezone-aware loop — the prior doc already flagged this as the multi-timezone evolution ("schedule one cron tick per IANA-zone bucket").
- **Tier 3 (per-user learned minute): per-user scheduled rows.** Store an absolute UTC `next_send_at` per user; a 1-minute cron scans due rows and enqueues to FCM. Use **idempotent atomic update** (`UPDATE … WHERE id=:id AND sent=false`) so only one worker fires each row — no distributed lock needed.
- **FCM caveat:** FCM does **not** reliably do per-recipient-local-time scheduling itself (historically buggy on iOS). **Own the scheduler; use FCM purely as the dispatch layer.** Capture each user's IANA tz client-side at registration and refresh on app open.

### 6.5 Schema deltas for timing

The prior doc's `profiles` already has `notifications_enabled`, `notification_time_local`, `timezone`. Add:

```sql
-- Resolved send strategy + learned time
alter table profiles
  add column send_time_tier  smallint not null default 1,   -- 0=global,1=segment,2=pop,3=user
  add column resolved_send_local time,                       -- computed by the tiering job
  add column next_send_at timestamptz;                       -- Tier 3 absolute UTC (null until learned)

-- Per-user open events feeding Tier 2/3 (lightweight; prune > 90 days)
create table notification_events (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  sent_at timestamptz not null,
  opened_at timestamptz,                                     -- null if not opened
  local_hour smallint,                                       -- opened_at in user tz, 0-23
  dow smallint                                               -- day-of-week 0-6
);
create index notif_events_user_idx on notification_events (user_id, opened_at desc);
```

A weekly cron computes each active user's recency-weighted open histogram → `resolved_send_local` / `next_send_at`, blending with the population mode for sparse users, applying the 11 PM–6 AM daytime guard.

---

## 7. How It All Stays Dynamic (the unifying answer)

Every "cluster" in this design is the **output of a pure function over the current persona/behavior table**, not a manually maintained list:

| Cluster | Definition (dynamic) | Auto-creates when… |
|---|---|---|
| Fetch combo (L1) | `{country} × {category}` | a country/category appears in any persona's mapped topics |
| Rewrite bucket (L2) | `H(coarsened persona subset)` | a new attribute combination onboards → next cron caches its rewrites |
| Timezone bucket (L3) | `{IANA tz}` from Country+State+City | first user from a new region onboards |

No clustering job has to "run" to create a segment in any of the three layers at MVP — they materialise on demand. The only *scheduled* clustering work appears in **Phase 2**, if/when you graduate L2 to ML clustering (assign-online + nightly batch recluster, §5.5).

---

## 8. Proposed Build Order

**MVP (free tier, deterministic everything):**
1. Keep Layer 1 fetch exactly as prior doc.
2. Implement `bucket_hash = H(country, age_range, life_stage, employment_type, home_owner, income_bracket)`; `life_stage = f(married, children)`. State/City/Religion → LLM prompt context only. (Update `personas` schema from prior doc §6.3 to add `home_owner`, `income_bracket`, `married`, `children`, or derive `life_stage`.)
3. Topic→category map (L1) + topic→tag map for feed ranking (Jaccard).
4. Replace the single 8 AM IST push cron with **timezone-bucketed fan-out** (Tier 0) + **(age × employment) segment prior** (Tier 1). Add `notification_events` logging from day 1 so Tier 2/3 have data later.

**Phase 2 (data-driven, paid where justified):**
5. Tier 2/3 send-time optimization (population mode → per-user histograms).
6. Evaluate `newsdata.io` Professional for `region` (state-level news) **only if** users demand hyper-local stories.
7. Evaluate ML clustering (k-prototypes / HDBSCAN) for L2 **only if** buckets are too coarse and N is large enough.

---

## 9. Architecture Notes (from prior doc, unchanged)

The mechanisms these three layers ride on already exist in [`2026-05-26-sowhat-news-mvp-tech-stack-architecture.md`](./2026-05-26-sowhat-news-mvp-tech-stack-architecture.md):

- `personas.bucket_hash` + `personas_bucket_idx` (§6.3) — the L2 key and its index.
- `impact_rewrites (article_id, persona_bucket_hash)` (§6.3) — the per-bucket LLM cache that makes deterministic bucketing pay off.
- `personas.distinctBuckets()` + the per-(article,bucket) rewrite fan-out (§6.4) — already iterates live buckets dynamically.
- GitHub Actions cron (§9.4) — where the timezone-bucket fan-out and (Phase 2) batch-recluster jobs live; Vercel Hobby's 1-cron/day limit is already bypassed.
- `profiles.timezone` / `notification_time_local` / `notifications_enabled` (§6.3) + `PUT /v1/notifications/preferences` (§7) — the L3 inputs.
- Redis `persona:dirty:set` (§6.6) — on a persona edit the user's bucket_hash is recomputed and rewrites regenerated; the same hook lets a changed timezone/age re-resolve their send time.

---

## 10. Open Questions

1. **State coarsening scheme** — into how many tiers, and on what basis (metro vs non-metro? GDP tier? Or fold state entirely into timezone and keep only country in the hash)? Needs a product call on how "local" the impact framing must feel.
2. **Religion** — confirmed *excluded* from `bucket_hash` here (bias + cardinality). Is it ever needed even as LLM context, or dropped entirely for MVP? PM/legal input.
3. **Topic taxonomy** — final predefined topic list and its mapping to (a) newsdata.io's 17 categories and (b) the `impact_tags` taxonomy. Free-text handling policy.
4. **Tier-3 trigger threshold** — how many opens before trusting a per-user learned time (vendors use 60-day activity windows; pick a min event count).
5. **Quiet-hours definition per culture/region** — is 11 PM–6 AM the right universal guard, or region-specific?
6. **When (if ever) to pay for `newsdata.io` `region`** — depends on whether state/city-level news is a real user need or just nice-to-have.

---

## 11. Sources (verified June 2026)

### newsdata.io capabilities
- Documentation: <https://newsdata.io/documentation>
- "Latest" endpoint detail: <https://newsdata.io/blog/latest-news-endpoint/>
- Response objects (fields, AI fields by plan): <https://newsdata.io/blog/news-api-response-object/>
- Pricing plans: <https://newsdata.io/blog/pricing-plan-in-newsdata-io/>, <https://newsdata.io/pricing>
- q / qInTitle / qInMeta: <https://newsdata.io/blog/how-do-q-qintitle-qinmeta-works/>
- Categories: <https://newsdata.io/blog/news-categories-by-newsdata-io/>
- Regional/localized news (region param, paid): <https://newsdata.io/blog/regional-news-api/>
- Credit consumption: <https://newsdata.io/blog/newsdata-credit-consumption/>
- Rate limit: <https://newsdata.io/blog/newsdata-rate-limit/>

### Clustering for categorical/mixed data
- k-modes / k-prototypes lib: <https://github.com/nicodv/kmodes/blob/master/README.rst>, <https://pypi.org/project/kmodes/>
- KModes guide: <https://www.analyticsvidhya.com/blog/2021/06/kmodes-clustering-algorithm-for-categorical-data/>
- k-prototype for mixed data: <https://towardsdatascience.com/the-k-prototype-as-clustering-algorithm-for-mixed-data-type-categorical-and-numerical-fe7c50538ebb/>
- Gower distance / mixed-type clustering: <https://medium.com/data-science/clustering-on-mixed-data-types-5fe226f9d9ca>, <https://dpmartin42.github.io/posts/r/cluster-mixed-types>
- HDBSCAN: <https://pberba.github.io/stats/2020/01/17/hdbscan/>
- Latent Class Analysis: <https://medium.com/@ramdhanhdy/understanding-latent-class-analysis-lca-14ffccc885e7>
- Rule-based vs clustering (segmentation): <https://www.optimove.com/resources/learning-center/customer-segmentation>, <https://dfrieds.com/machine-learning/segmentation-vs-clustering.html>
- Online/streaming clustering: <https://www.numberanalytics.com/blog/mastering-streaming-k-means-clustering>, <https://arxiv.org/pdf/2102.09101>
- Drift detection (PSI/KS/JS): <https://www.evidentlyai.com/ml-in-production/concept-drift>
- Choosing k: <https://www.datanovia.com/en/lessons/determining-the-optimal-number-of-clusters-3-must-know-methods/>
- Cold-start via demographics: <https://aman.ai/recsys/cold-start/>
- Jaccard similarity: <https://www.ibm.com/think/topics/jaccard-similarity>

### Notification send-time optimization
- Braze Intelligent Timing: <https://www.braze.com/docs/user_guide/brazeai/intelligence/intelligent_timing>
- MoEngage Best Time to Send: <https://help.moengage.com/hc/en-us/articles/4410529561108-Best-time-to-send>
- OneSignal Intelligent Delivery (+23% / 42B sends): <https://onesignal.com/blog/increase-open-rates-by-up-to-23-percent-with-intelligent-delivery/>
- OneSignal timezone delivery: <https://onesignal.com/blog/deliver-by-timezone-push-notification/>
- Airship Optimal Send Time: <https://www.airship.com/docs/guides/features/intelligence-ai/predictive/optimal-send-time/>
- Airship ML model engineering: <https://www.airship.com/blog/our-machine-learning-model-for-predictive-send-time-optimization/>
- CleverTap Best Time / news timing: <https://docs.clevertap.com/docs/best-time>, <https://clevertap.com/blog/best-time-to-send-push-notifications/>
- Airship 2025 push benchmarks (Android 10.7% vs iOS 4.9%): <https://www.airship.com/resources/benchmark-report/mobile-app-push-notification-benchmarks-for-2025/>
- Age-cohort usage (younger = night, older = morning): <https://www.fastcompany.com/91407887/night-owl-genz-study-ties-staying-up-late-smartphone-social-media-addiction>, <https://www.statista.com/statistics/1310218/time-spent-using-smartphone-age-us/>
- FCM scheduler pattern (cron + idempotent atomic update): <https://dev.to/sangwoo_rhie/building-a-production-ready-scheduled-push-notification-system-with-nestjs-cron-and-firebase-4fai>
- FCM recipient-tz scheduling bug: <https://github.com/firebase/firebase-ios-sdk/issues/8172>

---
