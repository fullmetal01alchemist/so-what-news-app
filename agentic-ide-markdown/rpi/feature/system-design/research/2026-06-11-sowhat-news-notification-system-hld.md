---
date: 2026-06-11T11:30:00+05:30
researcher: Anjay Sahoo
project: SoWhat News App
topic: "HLD — Backend: daily push notification system (regional fixed-time MVP; data capture for recommended notifications)"
tags:
  [
    system-design,
    hld,
    notifications,
    push,
    fcm,
    regional-fan-out,
    timezone,
    iana-tz,
    dst,
    scheduler,
    cron,
    idempotency,
    notification-log,
    send-time-optimization,
    daily-top5,
    backend,
  ]
status: complete
git_commit: d972536720dd7da6d09cbb18d3d6bb3b9103342f
branch: master
repository: so-what-news-app
last_updated: 2026-06-11
last_updated_by: Anjay Sahoo
---

# Research / HLD: SoWhat News — Daily Push Notification System (Backend)

**Date**: 2026-06-11 11:30 IST
**Researcher**: Anjay Sahoo
**Git Commit**: `d972536720dd7da6d09cbb18d3d6bb3b9103342f`
**Branch**: `master`
**Repository**: so-what-news-app
**Scope**: **Backend only.** The Compose Multiplatform client is described only where it drives a backend contract (FCM token registration, open-acknowledgement, notification rendering).
**Builds on**:
- [`2026-05-26-sowhat-news-mvp-tech-stack-architecture.md`](./2026-05-26-sowhat-news-mvp-tech-stack-architecture.md) — the stack (Hono on Vercel, Supabase Postgres + Auth, Upstash Redis, GitHub Actions cron, **FCM**), the `profiles` / `push_tokens` / `daily_top5` tables, the `POST /v1/notifications/token` + `PUT /v1/notifications/preferences` endpoints, and the original **8 AM IST daily push** cron (§6.7, §5.3, Decision 3).
- [`2026-06-08-sowhat-news-user-clustering-and-notification-timing.md`](./2026-06-08-sowhat-news-user-clustering-and-notification-timing.md) — **Layer 3 notification timing**: the 4-tier model (Tier 0 timezone default → Tier 1 segment prior → Tier 2 population mode → Tier 3 per-user learned), timezone-bucket fan-out, `notification_events` open-histogram schema (§6.5), and the FCM "own the scheduler, FCM is dispatch only" caveat (§6.4).
- [`2026-06-10-sowhat-news-login-and-onboarding-flow-hld.md`](./2026-06-10-sowhat-news-login-and-onboarding-flow-hld.md) — produces `profiles.timezone`, `region_country`, `notifications_enabled` at onboarding.
- [`2026-06-10-sowhat-news-relevant-news-for-a-persona-hld.md`](./2026-06-10-sowhat-news-relevant-news-for-a-persona-hld.md) — materialises `daily_top5`; its **lead story `daily_top5[0]`** is exactly what this push delivers (its §13 step 9 explicitly defers the wiring to this doc).

> This doc designs the **notification dispatch path**: how the backend sends one daily push per user at a fixed **evening local time**, grouped into **3–4 regions** instead of per-user timezone fan-out. It deliberately **collapses the clustering doc's 4-tier timing model down to Tier-0-only (regional fixed time)** for the MVP — and specifies the **data we capture from day 1** (every send + open) so the *recommended / send-time-optimised* notifications (Tiers 1–3, and content recommendation) can be built in the **next iteration** with no schema scramble.

---

## 1. Research Question

> Design the **backend HLD for notifications**. Keep it **very simple for the MVP**: divide the user base into **3–4 regions**; each region gets the daily news push at **8 PM local time** for that region (e.g. India → 8 PM IST; likewise 8 PM for the other regions). **All recommended / personalised-timing notifications are deferred to the next iteration** — but whatever **data** that future iteration needs must start being **persisted in the DB now**. Define the regions, the scheduler, the data model, the API surface, idempotency/abuse handling, and the data-capture for the next iteration.

---

## 2. TL;DR — The Six Decisions

1. **MVP = Tier 0 only, at *region* granularity (not per-IANA-zone, not per-user).** The clustering doc (§6.2) lays out a 4-tier ladder ending in per-user learned send-times. The user's explicit instruction is to ship **only the coarsest rung**: a handful of regions, each firing at a fixed local evening time. We map the 6 MVP countries (`in, us, gb, ae, sg, ca`) onto **4 regions**, each anchored to one IANA timezone, and send at **20:00 in that anchor zone**. Tiers 1–3 are **next iteration** (§9).

2. **Send time is 20:00 (8 PM) local — an explicit change from the foundational doc's 08:00 IST.** The MVP tech-stack doc (Decision 3, §6.7) hard-coded a *morning* 8 AM IST push. The product call here is an **evening 8 PM digest** ("here's what mattered today, and so what it means for you") — which fits the consequence-oriented, non-breaking-news product better than a morning alert. The send-local-time is **config-driven** (`notification_regions.send_local_time`), so morning-vs-evening (or per-region variation) is a one-row change, not a redeploy.

3. **A region maps to exactly one anchor IANA timezone; we fire at the anchor's 20:00.** Some countries within a region won't be at *exactly* 8 PM, but the drift is bounded and accepted as the cost of "keep it very simple" (§3.2). The scheduler computes the fire instant **from the IANA tz**, so it is **DST-correct automatically** for the regions that observe DST (UK, US, Canada) without ever editing cron YAML.

4. **Own the scheduler; FCM is the dispatch layer only.** Per the clustering doc §6.4, FCM does not reliably do per-recipient-local-time scheduling. We run a **frequent UTC tick (every 30 min) on GitHub Actions** that asks *"which regions just reached 20:00 local?"* and fans those regions out via FCM. This replaces the single `02:30 UTC` push cron from the foundational doc (§6.7).

5. **The push payload is the already-materialised `daily_top5[0]`.** No LLM, no news API, no ranking in the dispatch path — the relevant-news HLD already computed and cached the user's ranked, reframed lead story on the 6 h cron. We send a **`data` payload** (not `notification`) so the Compose app renders the Inshorts-style headline + summary from local cache, identical to opening the app (foundational doc §5.3, §6.7).

6. **Capture send + open telemetry from day 1 — this is the "keep data in DB for next iteration" requirement.** A `notification_log` table records every send, its lead `article_id`, delivery status, and (when the client acks a tap) `opened_at` / `local_hour` / `dow`. This **is** the clustering doc's `notification_events` table (§6.5), seeded now so that Tier 2 (population best-hour) and Tier 3 (per-user learned histogram) have weeks of training data the moment we start the next iteration.

```
                         daily_top5 materialised by 6h build-feeds cron
                         (relevant-news HLD §7.1) — lead story = daily_top5[0]
                                            │
 GitHub Actions cron  ── every 30 min ──►  POST /v1/_internal/cron/push-regional
 (UTC, DST-agnostic)                            │
                                                ▼
                         for each ENABLED region R:
                           now_local = now() in R.anchor_tz       ← DST-correct
                           if now_local ∈ [20:00, 20:30) AND not already sent today:
                              claim region_send_log(R, today)     ← atomic, idempotent
                              users = profiles WHERE notif_region=R AND notifications_enabled
                              for each user: lead = daily_top5[user, today][0]
                                 FCM data-push to user's push_tokens
                                 INSERT notification_log(user, R, article_id, sent_at, status)
                                            │
                                            ▼
                         Compose app renders from `data` payload → user taps
                                            │
                         POST /v1/notifications/ack {log_id}  ─► opened_at, local_hour, dow
                                            │
                         (next iteration) histogram of opens → Tier 2/3 send-time optimisation
```

---

## 3. The Regional Model (the core of the MVP)

### 3.1 Why "region = one anchor timezone", not per-user fan-out

The clustering doc (§6.4) recommended **timezone-bucket fan-out** — a scheduler that fires per *IANA zone* (~25–38 zones globally). That is already a simplification of per-user scheduling. The user wants to go **one rung simpler still**: group the launch countries into **3–4 regions**, each firing once at a fixed local evening time. The trade-off is explicit and accepted:

- **Win:** trivially simple. 4 fire instants/day, 4 config rows, no per-user `next_send_at` arithmetic, no learned histograms, no quiet-hours solver. A new region is one row in `notification_regions`.
- **Cost:** users in a country whose local time differs from its region's anchor get the push at *anchor*-20:00, not their own 20:00. We keep this drift bounded (§3.2) and **log the real per-user local hour anyway** (§6), so the next iteration can tighten it without losing history.

This is still **dynamic by construction** (clustering doc §7): a region "exists" the moment its config row is enabled and a user's `region_country` maps into it; no job runs to create one.

### 3.2 Recommended 4-region layout for the 6-country MVP

The 6 MVP countries span **UTC+8 (SG) down to UTC−8 (US Pacific) ≈ 16 hours** — so 3–4 regions *cannot* give everyone exactly 8 PM. The recommended cut keeps every country inside an **evening window** by anchoring each region to a representative IANA zone:

| Region | Countries | Anchor IANA tz | Local send | UTC fire (standard / DST) | Within-region drift |
|---|---|---|---|---|---|
| **R1 — South Asia + Gulf** | `in`, `ae` | `Asia/Kolkata` (IST, UTC+5:30, no DST) | 20:00 IST | **14:30 UTC** (year-round) | AE (UTC+4) receives at ~18:30 local — still evening |
| **R2 — APAC** | `sg` | `Asia/Singapore` (SGT, UTC+8, no DST) | 20:00 SGT | **12:00 UTC** (year-round) | none (single zone) |
| **R3 — UK / Europe** | `gb` | `Europe/London` (GMT/BST) | 20:00 local | **20:00 UTC** (GMT winter) / **19:00 UTC** (BST summer) | none (single zone), DST-shifting |
| **R4 — Americas** | `us`, `ca` | `America/New_York` (ET) | 20:00 ET | **01:00 UTC** (EST winter) / **00:00 UTC** (EDT summer) | US/CA West gets ~17:00 local — documented simplification |

**Notes on the cut:**
- **India is effectively exact** — IST is the R1 anchor, so India (the primary market) gets 8 PM to the minute. Gulf rides along ~1.5 h early, which is acceptable for an evening digest.
- **Americas is the one real compromise**: anchored to Eastern, a Pacific user gets the push at ~5 PM. That is the deliberate "keep it simple" cost. If product wants tighter, split R4 into `AMERICAS_EAST` / `AMERICAS_WEST` (→ 5 regions) — a **one-row config change**, no code.
- **If India volume dominates** and you want zero Gulf drift, split R1 into standalone `IN` and `GULF` (→ 5 regions). Also one row.
- The scheme is intentionally **config-driven** (`notification_regions`), so 4 ⇄ 5 region cuts and the 20:00 time are tunable without an app or backend release — same philosophy as the onboarding/feed config in the sibling HLDs.

### 3.3 DST is handled by the scheduler, not by editing cron

Three of the four anchors (IST, SGT, GST) **don't** observe DST, so their UTC fire instant is fixed. London and New York **do**, so their UTC instant shifts by an hour twice a year. Hard-coding UTC cron times would silently send an hour off after each DST change. We avoid this entirely (§5): the tick **computes `now()` in the anchor IANA tz** and checks whether the local clock is in the send window. The IANA database encodes DST rules, so the fire instant tracks DST automatically and the cron YAML never changes.

---

## 4. Data Model (deltas to prior docs)

```sql
-- 4.1 profiles: add the resolved region. timezone / notifications_enabled /
--     notification_time_local already exist (foundational doc §6.3).
alter table profiles
  add column notif_region text;          -- 'R1_SOUTH_ASIA_GULF' | 'R2_APAC' | 'R3_UK' | 'R4_AMERICAS'
-- notification_time_local default flips to '20:00' for MVP (was '08:00').
-- notif_region is derived from region_country at persona write (§7.3) via the
-- country→region map in notification_regions.countries; stored so the regional
-- send query is a single indexed lookup.
create index profiles_notif_region_idx on profiles (notif_region) where notifications_enabled;

-- 4.2 notification_regions: the tunable region config (the heart of §3).
create table notification_regions (
  region_code     text primary key,         -- 'R1_SOUTH_ASIA_GULF'
  display_name    text not null,
  anchor_tz       text not null,            -- IANA, e.g. 'Asia/Kolkata' — DST source of truth
  send_local_time time not null default '20:00',
  countries       text[] not null,          -- ISO-2 list that maps into this region
  enabled         boolean not null default true
);
-- Seeded with the 4 rows from §3.2. Editable without a deploy.

-- 4.3 region_send_log: idempotency guard — one logical regional send per region per day.
create table region_send_log (
  region_code  text not null,
  send_date    date not null,               -- date in the region's anchor tz
  claimed_at   timestamptz not null default now(),
  finished_at  timestamptz,
  recipients   int default 0,
  primary key (region_code, send_date)
);

-- 4.4 notification_log: per-user/device send + open telemetry.
--     THIS IS the clustering doc's `notification_events` (§6.5) — the data we
--     persist now so the NEXT iteration (Tier 1/2/3 send-time optimisation)
--     has history from day one. Prune > 90 days (clustering doc §6.5).
create table notification_log (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid references profiles(id) on delete cascade,
  region_code     text,
  feed_date       date not null,
  article_id      uuid,                      -- lead story sent (daily_top5[0])
  sent_at         timestamptz not null default now(),
  fcm_message_id  text,
  delivery_status text not null default 'sent', -- sent | failed | token_invalid
  -- open telemetry (filled by POST /v1/notifications/ack) — feeds Tier 2/3 later:
  opened_at       timestamptz,
  local_hour      smallint,                  -- opened_at expressed in the user's profiles.timezone, 0-23
  dow             smallint                   -- day-of-week 0-6
);
create index notif_log_user_idx on notification_log (user_id, sent_at desc);
create index notif_log_open_idx on notification_log (user_id, opened_at desc);

-- 4.5 push_tokens: UNCHANGED (foundational doc §6.3). Multiple devices per user.
--     Stale tokens are pruned reactively on FCM 'UNREGISTERED'/'INVALID_ARGUMENT' (§5.3).

-- 4.6 daily_top5: UNCHANGED. push_sent_at (foundational doc §6.3) stays as a
--     coarse per-user-per-day "did the daily push fire" marker; notification_log
--     is the granular, analytics-grade record.
```

RLS: `auth.uid() = user_id` on `notification_log` (read-own only), consistent with all per-user tables (foundational doc §13.2). `notification_regions` and `region_send_log` are global/server-only (no client read path).

> **Why a `notification_log` *and* `daily_top5.push_sent_at`:** `push_sent_at` answers "did we already push to this user today?" cheaply inside the existing row. `notification_log` is the durable event stream — multiple devices, delivery status, and the **open histogram inputs** the next iteration's send-time optimiser trains on. We could rename it `notification_events` to match the clustering doc verbatim; the schema is the same.

---

## 5. The Scheduler & Dispatch (engineering)

### 5.1 The tick — `POST /v1/_internal/cron/push-regional`

A single GitHub Actions cron fires **every 30 minutes** (`*/30 * * * *`), `CRON_SECRET`-protected exactly like the existing cron endpoints (foundational doc §9.4). 30-min granularity is required because IST's 20:00 lands at `14:30 UTC` (a half-hour boundary). 48 ticks/day is trivially cheap and short-circuits instantly when no region is due.

```ts
// POST /v1/_internal/cron/push-regional  (CRON_SECRET-protected)
const regions = await db.notification_regions.where({ enabled: true });

for (const r of regions) {
  const nowLocal = nowInTz(r.anchor_tz);                 // DST-correct via IANA db
  const sendDate = nowLocal.toDateString();              // date in the region's tz
  const target   = r.send_local_time;                    // 20:00

  // Fire only inside the [20:00, 20:30) window for the region's local clock.
  if (!withinWindow(nowLocal.time, target, '30m')) continue;

  // IDEMPOTENT CLAIM: only the first tick in the window wins; later ticks no-op.
  const claimed = await db.query(
    `insert into region_send_log (region_code, send_date)
     values ($1, $2) on conflict do nothing returning region_code`,
    [r.region_code, sendDate]
  );
  if (claimed.length === 0) continue;                    // already sent today

  await fanOutRegion(r, sendDate);
}
```

`fanOutRegion` streams the region's recipients and dispatches:

```ts
async function fanOutRegion(r, sendDate) {
  let recipients = 0;
  // Stream in pages to stay within Vercel function memory/time (foundational §4.2).
  for await (const users of db.profiles.pageBy(500, {
    notif_region: r.region_code, notifications_enabled: true,
  })) {
    for (const u of users) {
      const top = await db.daily_top5.first(u.id, today());   // already materialised
      if (!top || top.article_ids.length === 0) continue;     // no feed → skip (logged)
      const lead = await db.firstStoryFor(u.id, top.article_ids[0]); // join impact_rewrites
      const tokens = await db.push_tokens.forUser(u.id);
      if (tokens.length === 0) continue;

      const resp = await fcm.sendEachForMulticast({
        tokens,
        data: {                                                // DATA payload, not notification
          type: 'daily_top',
          log_id: /* pre-generated notification_log id */,
          article_id: top.article_ids[0],
          headline: lead.impact_headline,                      // null-safe if partial
          summary:  lead.why_summary,
        },
        android: { priority: 'high', notification: { channelId: 'daily_top' } },
      });

      await db.notification_log.insert({
        user_id: u.id, region_code: r.region_code, feed_date: sendDate,
        article_id: top.article_ids[0], delivery_status: resp.failureCount ? 'failed' : 'sent',
        fcm_message_id: resp.responses[0]?.messageId,
      });
      await pruneInvalidTokens(resp, tokens);                  // §5.3
      recipients++;
    }
  }
  await db.region_send_log.update({ region_code: r.region_code, send_date: sendDate },
                                  { finished_at: now(), recipients });
}
```

### 5.2 Freshness ordering — build the feed *before* the push

Each region's 20:00 fire must find a current `daily_top5`. The ingestion + `build-feeds` cron already runs every 6 h (foundational §6.4, relevant-news §10.1). Because the four regions fire at very different UTC instants (12:00, 14:30, 16:00–20:00, 00:00–01:00), the **6 h build cadence guarantees a fresh-enough `daily_top5` exists before every regional tick**. No new build cron is needed; we only ensure the build job's schedule (`0 */6 * * *`) is offset to complete before `12:00 UTC` (the earliest regional fire, R2/APAC). If a user onboarded mid-day with no `daily_top5` yet, the push simply skips them that day (they still get the in-app cold-start feed, relevant-news §7.3); they enter the next day's send.

### 5.3 FCM dispatch details

- **`data` payload, not `notification`** — so the backgrounded Compose app's `onMessageReceived` renders the Inshorts-style card from the payload (and the locally cached `daily_top5`), keeping the look identical to opening the app (foundational §5.3, §6.7).
- **Multicast** via `sendEachForMulticast` (a user may have several devices). One `notification_log` row per user (delivery summarised); per-device failures handled inline.
- **Stale-token pruning:** on FCM `messaging/registration-token-not-registered` or `invalid-argument`, delete that `push_tokens` row so the next send is clean. Reactive cleanup, no separate cron.
- **Volume:** ~1,500 active devices/day split across 4 regional sends, each completing in seconds — far under FCM's 600 K msgs/min project quota (foundational §4.10, §5.3).
- **Idempotency at three levels:** (1) `region_send_log` unique `(region, date)` claim stops duplicate *regional* fan-outs; (2) `daily_top5.push_sent_at` per-user guard; (3) `notification_log` is the audit trail. A retried/overlapping tick cannot double-send.

### 5.4 Opt-out & preferences (MVP-simple)

- `notifications_enabled` (existing) gates the whole thing — the regional query filters on it; a disabled user is never selected.
- `PUT /v1/notifications/preferences` (existing, foundational §7) stays, but for the **MVP only the on/off toggle is honoured**. The *time* is region-derived and fixed at 20:00; **custom send-time is a Pro / next-iteration feature** (foundational §9A.4 already lists "push at custom times" as a Pro unlock). The endpoint still accepts `timezone` so a user who travels updates `profiles.timezone` (used only for the open-histogram `local_hour` today; used for true per-user timing in the next iteration).

---

## 6. What We Capture Now for the Next Iteration (the explicit requirement)

The user's instruction: *"All the recommended notification we do in next iteration. For that whatever data we need to have, we keep things in DB."* Here is exactly what we persist from day 1 and why each field is needed by a specific future tier (clustering doc §6.2):

| Captured now | Where | Powers (next iteration) |
|---|---|---|
| `notification_log.sent_at` | every send | baseline send/open denominator |
| `notification_log.opened_at` | `POST /v1/notifications/ack` on tap | **open-rate**, the core optimisation target |
| `notification_log.local_hour` (open time in user's tz) | derived from `opened_at` + `profiles.timezone` | **Tier 2 population best-hour** + **Tier 3 per-user open histogram** (clustering §6.1.4) |
| `notification_log.dow` | derived | day-of-week effect (clustering §6.1) |
| `notification_log.article_id` + `delivery_status` | every send | **content** recommendation (which lead stories get opened) + deliverability monitoring |
| `profiles.timezone`, `age_range`, `employment_type` | already at onboarding | **Tier 1 segment prior** `(age × employment)` (clustering §6.2) |
| `notif_region` | this doc | region-level open-rate comparison, the unit Tier 2 first optimises within |

This is precisely the clustering doc's `notification_events` table (§6.5) plus the persona fields already collected — so when we start the next iteration there is **no schema migration on historical data and weeks of opens to train on immediately**.

The **next iteration** (out of scope here, fully specified in clustering doc §6.2–§6.5) then layers on:
- **Tier 1** — nudge the regional 20:00 by an `(age_cohort × employment)` prior (persona-only, no data needed).
- **Tier 2** — shift each region/segment to its **empirical best hour** from the `local_hour` histogram.
- **Tier 3** — per-user `next_send_at` from a recency-weighted open histogram, with a quiet-hours guard, dispatched by a 1-min due-row scanner (clustering §6.4). Requires adding `profiles.send_time_tier`, `resolved_send_local`, `next_send_at` (clustering §6.5) — **not added now** to keep the MVP schema minimal, but the *event data* they consume **is** being collected.
- **Recommended content** — beyond always sending `daily_top5[0]`, choose the most-likely-to-open story per user from observed `article_id` opens.

---

## 7. API Surface (additions to prior docs §7)

> Auth model unchanged: the client uses Supabase Auth directly; our Hono API only **verifies** the JWT (JWKS, cached 1 h) and reads `sub` + `is_anonymous`. We mint no tokens.

### 7.1 `POST /v1/notifications/token` — UNCHANGED (foundational §7)
Register/refresh an FCM device token. Already specified; no change.

### 7.2 `PUT /v1/notifications/preferences` — narrowed for MVP
```yaml
put:
  summary: Toggle the daily push (MVP); update timezone
  security: [{ bearerAuth: [] }]
  requestBody:
    content: { application/json: { schema:
      { type: object, properties: {
          notifications_enabled: { type: boolean },
          timezone: { type: string, example: 'Asia/Kolkata' } } } } }
  responses: { '204': { description: updated } }
```
- `notification_time_local` is **accepted but ignored** for free users in the MVP (region-derived 20:00 is authoritative); honoured later for Pro/Tier-3. Document this so the client doesn't expect a custom time to take effect yet.

### 7.3 `POST /v1/notifications/ack` — NEW (the data hook)
Client calls this when the user **taps** the daily push, so we capture the open. This single endpoint is what makes the next iteration's send-time optimisation possible.
```yaml
post:
  summary: Acknowledge a notification open/tap (captures open telemetry)
  security: [{ bearerAuth: [] }]
  requestBody:
    required: true
    content: { application/json: { schema:
      { type: object, required: [log_id],
        properties: { log_id: { type: string, format: uuid } } } } }
  responses:
    '204': { description: open recorded }
# Server: set opened_at=now(); derive local_hour = now() in profiles.timezone; dow.
# Idempotent: first ack wins; later acks for the same log_id no-op.
```
- The `log_id` is shipped in the FCM `data` payload (§5.1), so the client echoes it back on tap.
- RLS / ownership: the `notification_log` row must belong to `auth.uid()`.

### 7.4 Internal cron — `POST /v1/_internal/cron/push-regional` (replaces `push-daily`)
`CRON_SECRET`-protected (foundational §9.4). Replaces the old single `02:30 UTC` → `push-daily` endpoint and its cron YAML with the every-30-min regional tick (§5.1). The old `push-daily` workflow file is retired.

```yaml
# .github/workflows/cron-push-regional.yml  (replaces cron-push-daily.yml)
on:
  schedule: [{ cron: '*/30 * * * *' }]   # every 30 min UTC; handler decides which regions fire
  workflow_dispatch: {}
jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - run: |
          curl -fsS -X POST https://api.sowhat.app/v1/_internal/cron/push-regional \
               -H "Authorization: Bearer ${{ secrets.CRON_SECRET }}" --max-time 280
```

---

## 8. Sequence Flows

### 8.1 Steady-state — India user, 8 PM IST
```
[6h cron] build-feeds ─► daily_top5(U, today) materialised (lead = daily_top5[0])   (relevant-news §7.1)
[cron tick 14:30 UTC] push-regional
   R1 anchor=Asia/Kolkata → now_local=20:0x → in window
   claim region_send_log('R1', 2026-06-11)  ← wins
   users WHERE notif_region='R1' AND notifications_enabled
   for U: lead = daily_top5[U][0] → FCM data-push to U's tokens
          INSERT notification_log(U,'R1',article_id,sent_at,'sent', message_id)
App (background) onMessageReceived ─► renders Inshorts card from data payload
User taps ─► POST /v1/notifications/ack {log_id} ─► opened_at, local_hour(=20 in IST), dow
[14:30–15:00 UTC] later ticks for R1 → region_send_log conflict → no-op (idempotent)
```

### 8.2 DST boundary — UK user (R3), summer vs winter
```
Winter (GMT): anchor Europe/London 20:00 == 20:00 UTC → tick at 20:00 UTC fires R3
Summer (BST): anchor Europe/London 20:00 == 19:00 UTC → tick at 19:00 UTC fires R3
   (cron YAML unchanged; nowInTz('Europe/London') tracks DST automatically)
```

### 8.3 User disables notifications
```
PUT /v1/notifications/preferences { notifications_enabled: false } ─► 204
   profiles.notifications_enabled=false
[next regional tick] partial index profiles_notif_region_idx excludes U → never selected
```

### 8.4 Stale device token
```
fanOutRegion → FCM returns 'registration-token-not-registered' for token T
   pruneInvalidTokens ─► DELETE push_tokens WHERE fcm_token=T
   notification_log row recorded with delivery_status reflecting per-device outcome
```

---

## 9. Scope Boundary — MVP vs Next Iteration

| Capability | MVP (this doc) | Next iteration (clustering doc §6.2–6.5) |
|---|---|---|
| Send time | Fixed **20:00 anchor-local**, 4 regions (Tier 0) | Tier 1 segment prior → Tier 2 population best-hour → Tier 3 per-user learned |
| Granularity | Region (4) | IANA zone → per-user `next_send_at` |
| Scheduler | 30-min tick, region window check | + 1-min due-row scanner for Tier-3 `next_send_at` |
| Content | Always `daily_top5[0]` (lead) | Per-user most-open-likely story |
| Custom time | No (Pro/next iter) | Yes |
| Quiet hours | N/A (single evening send) | 11 PM–6 AM guard (clustering §6.2) |
| Data captured | **`notification_log` sends + opens, from day 1** | Consumes that history; adds `send_time_tier`/`resolved_send_local`/`next_send_at` columns |

The MVP ships rungs needed for a working daily push and **nothing more**, while the day-1 telemetry guarantees the next iteration is a pure additive build, not a data backfill.

---

## 10. How This Sits Inside the Free-Tier Budget (ties to foundational §4–5)

- **FCM:** free, no per-message cost, no daily cap; 4 regional sends × ~hundreds of devices each ≪ 600 K/min project quota (foundational §4.10, §5.3). Unchanged.
- **GitHub Actions cron:** every-30-min tick = 48 runs/day, each a sub-second `curl`; unlimited on public repos, well inside 2,000 min/mo on private (foundational §9.4).
- **Vercel Hobby CPU:** the dispatch handler is I/O-bound (DB read + FCM HTTP); paged at 500 users to bound memory; runs as a 300 s-capable Vercel function invoked by cron, not on the user request path (foundational §4.2, §5.4).
- **Postgres 500 MB:** `notification_log` at ~1,500 rows/day × ~150 bytes × 90-day retention ≈ **~20 MB** — comfortable; pruned > 90 days (clustering §6.5). `notification_regions`/`region_send_log` are negligible.
- **Redis:** unchanged — the dispatch path does not need it (it reads materialised `daily_top5` directly).

No new vendor, no new paid tier.

---

## 11. Security, Privacy & Abuse (additions to foundational §13)

- **Cron auth:** `/v1/_internal/cron/push-regional` validates `Authorization: Bearer ${CRON_SECRET}` with `crypto.timingSafeEqual` (foundational §13.2).
- **`POST /v1/notifications/ack` ownership:** the `notification_log` row must belong to `auth.uid()`; RLS enforces it. Rate-limited (Upstash sliding window) to prevent open-event spoofing that would poison the next iteration's histogram.
- **No PII in the payload:** the FCM `data` payload carries `article_id`, `log_id`, and the already-anonymous Inshorts headline/summary — no email, no `user_id` beyond the opaque `log_id` (which is a random UUID, not a user identifier). Consistent with the foundational doc's PII boundary (§13.1).
- **Token hygiene:** stale FCM tokens pruned reactively (§5.3); `push_tokens` never logged in cleartext (foundational §13.3 redaction rule).
- **Quiet-by-design:** exactly one push/user/day; no retry storms (idempotent claim). No abuse surface for over-notification.

---

## 12. Build Order (backend)

1. **Migrations §4:** `profiles.notif_region` (+ partial index), `notification_regions` (seed the 4 rows from §3.2), `region_send_log`, `notification_log` (+ indexes). RLS on `notification_log`.
2. **Region derivation** in `PUT /v1/personas` (onboarding §7.3): map `region_country` → `notif_region` via `notification_regions.countries`; default `notification_time_local='20:00'`.
3. **`POST /v1/_internal/cron/push-regional`** handler: region-window check (`nowInTz`), idempotent `region_send_log` claim, paged `fanOutRegion`, FCM `data` multicast, `notification_log` insert, stale-token prune.
4. **GitHub Actions** `cron-push-regional.yml` (`*/30 * * * *`); retire `cron-push-daily.yml`. Ensure `build-feeds` completes before `12:00 UTC` (§5.2).
5. **`POST /v1/notifications/ack`**: set `opened_at`, derive `local_hour`/`dow` from `profiles.timezone`; idempotent; rate-limited.
6. **`PUT /v1/notifications/preferences`**: honour `notifications_enabled` + `timezone`; document `notification_time_local` as accepted-but-ignored for MVP.
7. **Client contract** (drives backend only): FCM token registration (existing), `data`-payload rendering, tap → `ack` with `log_id`.
8. **Retention job:** prune `notification_log` > 90 days (fold into an existing nightly cleanup cron).
9. **(Next iteration)** Tier 1/2/3 per clustering doc §6.2–§6.5 on top of the captured `notification_log` history.

---

## 13. Resolved Decisions & Open Questions

> Decisions taken 2026-06-11. **⟳** = needs a runtime/product confirmation before hardening.

1. **Tier 0 only for MVP.** **DECIDED** — regional fixed-time send; Tiers 1–3 deferred (clustering §6.2). *(§2.1, §9.)*
2. **Send at 20:00 local, not 08:00.** **DECIDED** — evening digest supersedes the foundational doc's morning 8 AM IST (Decision 3). Config-driven, so reversible per region. *(§2.2.)*
3. **4 regions for the 6-country MVP** with the §3.2 cut (IN+AE / SG / GB / US+CA). **DECIDED**, config-driven; splitting Americas E/W or IN↔Gulf is a one-row change. *(§3.2.)*
4. **Americas anchored to Eastern** ⟳ — **TENTATIVE:** West-coast users get the push ~3 h early (≈5 PM). Confirm acceptable, or split into `AMERICAS_EAST`/`AMERICAS_WEST` (→5 regions) before US/CA volume grows. *(§3.2.)*
5. **DST handled in-scheduler** via IANA `nowInTz`, not UTC cron edits. **DECIDED.** *(§3.3, §5.1.)*
6. **`notification_log` = the clustering doc's `notification_events`.** **DECIDED** — captured from day 1; the explicit "keep data in DB for next iteration" requirement. *(§6.)*
7. **Custom send-time is Pro/next-iteration.** **DECIDED** — MVP honours only on/off. *(§5.4, §7.2.)*
8. **Open capture relies on the client calling `/ack` on tap** ⟳ — **ACTION:** confirm the Compose `onMessageReceived` / notification-tap intent reliably fires the ack even when the app was killed; otherwise open-rate data (and thus Tier 2/3) will be undercounted. *(§7.3.)*
9. **Build-feeds-before-push ordering** ⟳ — **ACTION:** verify the 6 h ingestion/build cron offset so `daily_top5` is fresh before the earliest regional fire (12:00 UTC / R2). *(§5.2.)*
10. **Should `notification_time_local` be silently ignored or rejected for free MVP users** ⟳ — product/UX call on whether the client even exposes a time picker pre-Pro. *(§7.2.)*

---

## 14. Sources (verified June 2026)

### Push delivery & timezone scheduling (reused from clustering doc §11 / foundational §16)
- FCM quotas (free, unlimited messages, 600 K/min project quota): <https://firebase.google.com/docs/cloud-messaging/throttling-and-quotas>
- OneSignal "Deliver by User Timezone" (the regional/timezone fan-out pattern): <https://onesignal.com/blog/deliver-by-timezone-push-notification/>
- OneSignal Intelligent Delivery (+23% vs send-now; the Tier-2/3 payoff deferred to next iteration): <https://onesignal.com/blog/increase-open-rates-by-up-to-23-percent-with-intelligent-delivery/>
- FCM recipient-tz scheduling is unreliable → own the scheduler, FCM is dispatch only: <https://github.com/firebase/firebase-ios-sdk/issues/8172>
- Production scheduled-push pattern (cron + idempotent atomic claim): <https://dev.to/sangwoo_rhie/building-a-production-ready-scheduled-push-notification-system-with-nestjs-cron-and-firebase-4fai>

### Send-time optimisation (the next iteration — clustering doc §6)
- Braze Intelligent Timing: <https://www.braze.com/docs/user_guide/brazeai/intelligence/intelligent_timing>
- MoEngage Best Time to Send: <https://help.moengage.com/hc/en-us/articles/4410529561108-Best-time-to-send>
- Airship Optimal Send Time: <https://www.airship.com/docs/guides/features/intelligence-ai/predictive/optimal-send-time/>

### Timezone / DST correctness
- IANA Time Zone Database (DST rules; basis for `nowInTz` correctness): <https://www.iana.org/time-zones>

---
