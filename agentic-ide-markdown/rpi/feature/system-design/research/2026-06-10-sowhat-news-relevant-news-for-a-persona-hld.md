---
date: 2026-06-10T20:30:00+05:30
researcher: Anjay Sahoo
project: SoWhat News App
topic: "HLD — Backend: serving relevant news for a persona (matching, ranking, impact-rewrite, daily top-5)"
tags:
  [
    system-design,
    hld,
    personalization,
    relevance-ranking,
    candidate-generation,
    two-stage-recommender,
    bucket-hash,
    impact-rewrite,
    llm,
    freshness-decay,
    mmr,
    diversity,
    dedup,
    event-clustering,
    importance-score,
    daily-top5,
    fan-out,
    feed-cache,
    cold-start,
    backend,
  ]
status: complete
git_commit: d972536720dd7da6d09cbb18d3d6bb3b9103342f
branch: master
repository: so-what-news-app
last_updated: 2026-06-10
last_updated_by: Anjay Sahoo
---

# Research / HLD: SoWhat News — Relevant News for a Persona (Backend)

**Date**: 2026-06-10 20:30 IST
**Researcher**: Anjay Sahoo
**Git Commit**: `d972536720dd7da6d09cbb18d3d6bb3b9103342f`
**Branch**: `master`
**Repository**: so-what-news-app
**Scope**: **Backend only.** The Compose Multiplatform client is described only where it drives a backend contract.
**Builds on**:
- [`2026-05-26-sowhat-news-mvp-tech-stack-architecture.md`](./2026-05-26-sowhat-news-mvp-tech-stack-architecture.md) — the stack (Hono on Vercel, Supabase Postgres + Auth, Upstash Redis, Gemini + Groq, GitHub Actions cron, FCM), the `articles` / `personas` / `impact_rewrites` / `daily_top5` tables, the ingestion pipeline (§6.4), the request path `GET /v1/feed/top5` (§6.5), and the LLM budget math (§5.2).
- [`2026-06-08-sowhat-news-user-clustering-and-notification-timing.md`](./2026-06-08-sowhat-news-user-clustering-and-notification-timing.md) — the **3-layer model**: L1 fetch (`{country}×{category}`), L2 `bucket_hash = H(country, age_range, life_stage, employment_type, home_ownership, income_bracket)` as the *rewrite-cache key*, L3 notification timing. Critically: **topics of interest are a *filter/ranking* signal, NOT part of `bucket_hash`** (§5.4).
- [`2026-06-10-sowhat-news-login-and-onboarding-flow-hld.md`](./2026-06-10-sowhat-news-login-and-onboarding-flow-hld.md) — produces the `personas` row this HLD consumes; its §10 ("How This Feeds the Next Session") is the exact input contract for this document.

> This doc designs the **read / ranking / rewrite path**: how the backend turns *ingested articles + a persona* into a **ranked, reframed daily top-5**. The onboarding HLD ends where this one begins — its only durable output that personalisation consumes is the `personas` row (`bucket_hash`, derived `interests`→tags, `region_country`, `timezone`/`age_range`/`employment_type`, and `state`/`city`/`religion` as LLM context). This HLD owns everything downstream of that row.

---

## 1. Research Question

> Design the **backend HLD for serving relevant news to a persona**. Given the persona produced at onboarding and the articles ingested every 6h, define: how candidate articles are **matched** to a persona, how they are **ranked** for relevance, how the per-persona **"so what" impact rewrite** is generated and cached without blowing the free LLM budget, how the **daily top-5** is assembled and served on a fast serverless read path, and how the system **degrades gracefully** for brand-new / cold buckets. Stay inside the free-tier constraints from the prior docs.

---

## 2. TL;DR — The Architecture in One Idea

**This is a two-tier fan-out system.** The two tiers exist because the two expensive-or-personal things in the pipeline have *different cardinalities*:

| | What varies it | Cardinality | Cost shape | Fan-out tier |
|---|---|---|---|---|
| **Impact rewrite** (the "so what" reframing) | only `bucket_hash` | ~150–250 live buckets | **expensive** (LLM) | **fan-out-on-write, per *bucket*** (run on the 6h cron, shared by every user in the bucket) |
| **Article selection + ranking** (which 5, in what order) | the user's **topic interests** (not in `bucket_hash`) + freshness + importance | per *user* (~5K) | **cheap** (no LLM, pure scoring) | **per-user materialise** (precomputed `daily_top5`, served read-through) |

The single most important consequence, and the thing this HLD nails down that the prior docs left loose:

> **Two users in the same `bucket_hash` share the *rewrite* of any article they both see, but they may see *different articles* (and a different order) because ranking keys on per-user topic interests, which are deliberately *not* in `bucket_hash`.** Rewrite is bucket-shared; selection is user-specific. ([clustering doc §5.4](./2026-06-08-sowhat-news-user-clustering-and-notification-timing.md))

This collapses what would be **N per-user LLM calls into B per-bucket calls** (B ≪ N) — the reason the free Gemini tier survives multi-country scale — while keeping each user's feed genuinely theirs. It is the textbook *fan-out-on-write for the broadcast/expensive part, fan-out-on-read for the personal part* hybrid, applied at *bucket* granularity so we never suffer the per-user write-amplification ("celebrity") problem. (Sources: fan-out trade-off [ByteByteGo](https://bytebytego.com/courses/system-design-interview/design-a-news-feed-system); segment caching cuts LLM cost 35–80% [callsphere](https://callsphere.ai/blog/llm-caching-strategies-cost-optimization-2026), [AWS](https://aws.amazon.com/blogs/database/optimize-llm-response-costs-and-latency-with-effective-caching/).)

**The five decisions:**

1. **Two-stage retrieval → ranking, deterministic and LLM-free for *selection*.** Stage 1 retrieves persona-eligible candidates (country + freshness window + topic-tag overlap). Stage 2 scores them with a tunable **weighted linear blend** of *relevance × freshness × importance*. The LLM is **never** in the selection/ranking path — only in the rewrite. (Retrieve-then-rank: [aman.ai](https://aman.ai/recsys/candidate-gen/); LLM-as-ranker is ~9–35× cost/latency and unstable, so it stays out of selection — [zeroentropy](https://zeroentropy.dev/articles/llm-as-reranker-guide/).)

2. **Relevance without click data = tag overlap now, embeddings later.** MVP scores relevance by **Jaccard overlap** between the persona's topic-derived tags and the article's `impact_tags` (clustering doc §5.4). Freshness is an **exponential half-life decay** `2^(−age_h / h)`; importance is `(1 + log(cluster_size)) · source_authority`. Phase 2 swaps Jaccard for sentence-embedding cosine if free-text topics dominate. ([Jaccard/BM25/embedding trade-offs](https://www.ai-bites.net/tf-idf-and-bm25-for-rag-a-complete-guide/); [HN gravity / half-life decay](https://medium.com/hacking-and-gonzo/how-hacker-news-ranking-algorithm-works-1d9b0cf2c08d); [Google importance patent — `1+log(cluster_size)`](https://patents.google.com/patent/WO2005029368A1/en).)

3. **Diversity & same-event dedup are a re-rank step, not an afterthought.** Before picking the final 5 we collapse near-duplicate articles reporting the *same event* into one (so the feed isn't three versions of one story), then apply **MMR (λ ≈ 0.7)** plus hard caps (≤1 per source, category spread) so the top-5 isn't five finance stories. ([MMR](https://aayushmnit.com/posts/2025-12-25-DiversityMMRPart1/DiversityMMRPart1.html); [same-event dedup thresholds](https://github.com/dukeblue1994-glitch/chronicle).)

4. **Rewrite only what will surface; cache forever on `(article_id, bucket_hash)`.** We generate the LLM impact-rewrite **only for the union of articles that actually rank into some user's candidate set within a bucket** — not the full cross-product — keyed `(article_id, bucket_hash)` so it is computed once and reused by every user in the bucket and on every later read. This is the prior doc's `impact_rewrites` cache (§6.3/§6.4), with the fan-out scoped precisely to "what ranks", resolving the prior doc's loose `bucketMatchesTags`.

5. **The read path never touches the LLM or the news API.** `GET /v1/feed/top5` is a Redis read-through over a **materialised `daily_top5`** + a join to cached `articles` + `impact_rewrites`. Cold/new buckets degrade gracefully: a brand-new user gets a **relevance-ranked feed immediately** (ranking is LLM-free) with `partial=true` (article text, no reframing) and the "so what" backfills on the next cron tick. (Read-through + materialised view: [leapcell](https://leapcell.io/blog/choosing-between-postgres-materialized-views-and-redis-application-caching); cold-start graceful degradation: [shaped.ai](https://www.shaped.ai/blog/mastering-cold-start-challenges).)

```
                        INGESTED ARTICLES (per country, every 6h)
                                    │  enrich on ingest:
                                    │  impact_tags · event_cluster_id · importance_score
                                    ▼
   PERSONA (user) ──┐        ┌──────────────────────────────────────────────────┐
   bucket_hash      │        │  PER-USER (cheap, no LLM)                          │
   interests→tags   ├───────►│  Stage 1  Retrieve candidates                     │
   region_country   │        │           country + freshness window + tag overlap│
   timezone/age/emp │        │  Stage 2  Score = w_rel·Jaccard                    │
                    │        │                 + w_fresh·2^(−age/h)               │
                    │        │                 + w_imp·(1+log(clusterSize))·auth  │
                    │        │  Stage 3  Dedup same-event + MMR + per-source caps │
                    │        │           → ranked candidate list (top ~15)        │
                    │        └───────────────┬──────────────────────────────────┘
                    │                        │ union of (article, bucket_hash) that rank
                    │                        ▼
                    │        ┌──────────────────────────────────────────────────┐
                    │        │  PER-BUCKET (expensive, LLM, shared)               │
                    │        │  Stage 4  Impact-rewrite missing (article,bucket)  │
                    │        │           Inshorts style → cache impact_rewrites   │
                    │        └───────────────┬──────────────────────────────────┘
                    │                        ▼
                    └───────►  Stage 5  Materialise daily_top5(user)  (free=5, pro=15)
                                             │
   GET /v1/feed/top5 ◄── Redis read-through ─┘   (request path: 1 DB read, no LLM/news API)
        cold bucket → partial=true (ranked articles, rewrites backfill next tick)
```

---

## 3. Input Contract — Exactly What a Persona Gives Us

Reproduced from the onboarding HLD §10, this is the *only* state the personalisation engine reads. Each field is tagged with the stage it drives:

| Persona field | Drives | Stage |
|---|---|---|
| `bucket_hash` = `H(region_country, age_range, life_stage, employment_type, home_ownership, income_bracket)` | the **rewrite cache key** | Stage 4 |
| `interests[]` → derived `topic_tags[]` (mapped to `impact_tags` taxonomy) | **candidate match + relevance score** | Stages 1–2 |
| `region_country` | L1 fetch scope **+** candidate country filter **+** part of `bucket_hash` | Stages 1, 4 |
| `region_state`, `region_city` | LLM prompt **context only** ("…for someone in Bengaluru") | Stage 4 |
| `religion` (Phase 2, consented) | LLM prompt **context only**, festival/policy stories | Stage 4 |
| `timezone`, `age_range`, `employment_type` | notification send-time (Layer 3 — separate doc) | n/a here |

**Key invariants carried in from the prior docs:**
- `bucket_hash` **excludes** topics, state, city, religion (clustering doc §5.3). → Selection (which articles) must be computed **per user**, not per bucket.
- The LLM prompt receives **only** the anonymous bucket descriptor + public article text (prior doc §9A.6, §13.1). No email/user_id/IP ever enters a prompt.
- newsdata.io free tier is **demographically blind** — it filtered only `{country}×{category}` at ingest (clustering doc §3). *All* persona richness is applied here, in our own ranking + rewrite, never at the news API.

---

## 4. Stage 0 — Enrichment at Ingest (extends prior doc §6.4)

The 6h ingestion cron already pulls newsdata.io, dedups on `source_url`, and tags `impact_tags` via batched Gemini calls (prior doc §6.4). This HLD adds three **user-independent** enrichments computed *once per article* at ingest, because they are inputs to ranking and must not be recomputed per user:

### 4.1 Same-event clustering → `event_cluster_id`
Multiple sources report the same event; the feed must show it **once**. On ingest, group new articles (within a country, within a 48h window) into event clusters:
- **MVP (cheap, no GPU):** TF-IDF / title-cosine similarity with a threshold, or MinHash+LSH over title+description shingles (4-grams, ~128 perms, Jaccard ≥ 0.85) for verbatim/wire-copy dedup. ([chronicle](https://github.com/dukeblue1994-glitch/chronicle), [NAACL 2025 near-dup](https://aclanthology.org/2025.naacl-industry.73.pdf))
- **Phase 2 (semantic same-event):** sentence-embedding cosine (e.g. MiniLM, 384-dim) + agglomerative/HDBSCAN (cosine sim ≥ ~0.4). Embeddings beat MinHash for *different-wording, same-event* grouping (F1 ~71% vs ~22% in the cited benchmark — [ACM benchmark](https://dl.acm.org/doi/pdf/10.1145/3701716.3715303)).
- Store `event_cluster_id` on each article; one article per cluster is flagged the **representative** (highest `importance_score`).

### 4.2 Importance score → `importance_score` (0–1)
A cheap, user-independent newsworthiness signal, computed from signals we already have:
```
importance = norm( (1 + log(cluster_size)) · source_authority · recency_decay )
```
- `cluster_size` = number of distinct sources in the article's `event_cluster_id` — **free** from §4.1 and the single strongest cheap importance signal. ([Google patent WO2005029368A1](https://patents.google.com/patent/WO2005029368A1/en), [arXiv 2402.10302](https://arxiv.org/pdf/2402.10302))
- `source_authority` = a precomputed static weight per `source_name` (small seed table; default 0.5 for unknown sources). ([source-rank signals — same patent](https://www.seobythesea.com/2019/10/news-ranking-algorithm/))
- `recency_decay` = the same half-life term used in §5.

### 4.3 Article embedding → `embedding` (Phase 2 only)
A `vector(384)` column (pgvector) for Phase-2 semantic relevance + MMR similarity. **Not built for MVP** (Jaccard + tag overlap suffice while the corpus is small and tag-driven).

These three enrichments are **per article, not per user/bucket**, so they cost O(articles/day) ≈ 300/day — trivial, and they fold into the existing batched-Gemini tagging pass (4.1/4.3) or are pure arithmetic (4.2).

---

## 5. Stages 1–3 — Per-User Candidate Generation, Ranking & Diversity (cheap, no LLM)

This is run per active user on each cron tick (and on-the-fly for a cold read). It is **deterministic, LLM-free, and fast**, which is why a brand-new user can get a *relevant* feed instantly even before any rewrite exists.

### 5.1 Stage 1 — Candidate retrieval (eligibility)
Narrow the corpus to a shortlist before the (slightly) costlier scoring:
1. `articles.country = persona.region_country` (or in the persona's country set).
2. `published_at` within the **freshness window** (MVP: last **48h**; tunable).
3. **Topic-tag overlap**: `article.impact_tags ∩ persona.topic_tags ≠ ∅` (the `articles_tags_idx` GIN index, prior doc §6.3, makes this an indexed array-overlap). Articles with zero tag overlap are dropped *unless* the candidate set is too small, in which case high-`importance` country-wide stories backfill (so the feed is never empty).
4. **Same-event collapse**: keep only the **representative** article per `event_cluster_id` (§4.1).

Retrieve-then-rank keeps the expensive scoring on a shortlist of ~tens, not the whole table. ([two-stage recsys](https://github.com/mahnoorsheikh16/Two-Stage-News-Recommendation-System))

### 5.2 Stage 2 — Relevance ranking (weighted linear blend)
Score every surviving candidate for *this user*:
```
score(article, user) =
      w_rel   · relevance(article, user)          // Jaccard(topic_tags, impact_tags)  ∈ [0,1]
    + w_fresh · 2^(−age_hours / H)                  // exponential half-life, H ≈ 24h    ∈ [0,1]
    + w_imp   · importance_score(article)           // §4.2                                ∈ [0,1]
```
- **Relevance** = `|topic_tags ∩ impact_tags| / |topic_tags ∪ impact_tags|` (clustering doc §5.4). Cheapest possible, interpretable, needs no training, no click data. ([Jaccard](https://www.ibm.com/think/topics/jaccard-similarity))
- **Freshness** = half-life decay, `H` (half-life) ≈ 24h for daily news; a clean 0–1 multiplier you can tune. The HN "gravity" `(P−1)/(T+2)^1.8` and Reddit `log10(s)+t/45000` are the canonical references; we use the additive-blend half-life form for inspectability. ([HN](https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/), [Reddit](https://medium.com/@niruthiha2000/reddits-ranking-algorithm-for-content-curation-systems-2daa3f33a14f))
- **Importance** = §4.2.
- **Weights** (`w_rel`, `w_fresh`, `w_imp`, `Σ=1`) are **server-driven config**, not hardcoded — same philosophy as the onboarding `GET /v1/onboarding/config` (onboarding doc §4). MVP start: `w_rel=0.55, w_fresh=0.30, w_imp=0.15`, tuned by editorial spot-check (the field-standard "hand-tune over weeks against qualitative review" approach — [signal-scoring](https://blakecrosley.com/blog/signal-scoring-pipeline)). Normalise each signal to 0–1 first so no term dominates by scale. ([weighted linear combination](http://klab.lt/weighted-retrieval-signals-recency-and-authority))

This linear-fusion-of-signals shape is exactly Google News' original design (MinHash + content signals fused by a **linear model** — [Das et al. 2007](http://glinden.blogspot.com/2007/05/google-news-personalization-paper.html)).

### 5.3 Stage 3 — Diversity & dedup re-rank
A high-relevance list can be five near-identical finance stories. Re-rank the scored shortlist into the final slots:
1. **MMR** greedy fill, picking each next slot by
   `MMR = λ·score(article) − (1−λ)·max_{j∈selected} Sim(article, j)`, with **λ ≈ 0.7** (relevance-leaning) and `Sim` = category/source overlap (MVP) or embedding cosine (Phase 2). ([MMR](https://docs.opensearch.org/latest/vector-search/specialized-operations/vector-search-mmr/))
2. **Hard caps** as guardrails: **≤1 article per source**, and spread across **≥3 distinct topic categories** in the top-5 where the candidate pool allows; relax the cap one notch at a time if the pool is too thin (graceful relaxation — [D-RDW](https://arxiv.org/pdf/2508.13035)).
3. Output the **ranked candidate list** — keep the **top ~15** (headroom for Pro = 15 and for swap-on-dedup), not just 5.

> **Why ranking is per-user but rewrite is per-bucket (restated, because it's the crux):** Stages 1–3 key on `persona.topic_tags`, which are **not** in `bucket_hash`. So two users in the same bucket can produce different ranked lists. But the *reframing* of any given article depends only on `bucket_hash`, so when both users surface the same article they share one cached rewrite. Selection diverges; reframing converges.

---

## 6. Stage 4 — Impact Rewrite (per-bucket, LLM, the expensive shared tier)

### 6.1 What gets rewritten, and how we bound the cost
The prior doc's rewrite loop (§6.4) fanned out per `(article, bucket)` "where bucket country matches article country and `bucketMatchesTags(...)`" — but `bucket_hash` carries no tags, so that predicate was underspecified. **This HLD resolves it:**

> The set of `(article_id, bucket_hash)` pairs that need a rewrite on a given tick = the **union, over all active users, of `{(article_id, user.bucket_hash) : article_id ∈ user's Stage-3 top-N}`**.

i.e. we rewrite **exactly what ranks into someone's feed**, deduplicated by bucket. Properties:
- Two users in the same bucket whose feeds both contain article X create **one** pair `(X, bucket)` → one LLM call, shared.
- An article that never ranks for anyone is **never rewritten** — no wasted LLM spend (the prior doc's biggest implicit risk).
- The pair set is **idempotent**: skip any `(article, bucket)` already present in `impact_rewrites` (prior doc §6.4 `exists()` check). Re-ingestion and re-ticks cost nothing.

This keeps us inside the Gemini budget. Prior doc §5.2 estimated ~540 batched calls/day for the naive full fan-out; scoping to "what ranks" can only *reduce* that, and the Gemini→Groq fallback (prior doc §10) absorbs any 429.

### 6.2 The rewrite call (unchanged from prior doc §6.4, restated for completeness)
Per `(article, bucket)` pair, generate the Inshorts-style output, validated by the Zod schema (prior doc §6.4):
- `impact_headline` — ≤10 words, present tense, names the concrete consequence for this persona.
- `why_summary` — 50–70 words: what happened · why it matters to *this* persona · one action/watch-out.
- Prompt input = the **anonymous bucket descriptor** + article title/description/country + (Phase 2, consented) `region_city`/`religion` as context. Never any PII.
- Cache row: `impact_rewrites(article_id, persona_bucket_hash, impact_headline, why_summary, llm_model, generated_at)` (prior doc §6.3).

**LLM-as-ranker is explicitly *not* used** for selection — pointwise LLM scoring is ~10× cost for *lower* ranking accuracy and the scores are unstable run-to-run even at temperature 0, breaking any threshold logic ([zeroentropy](https://zeroentropy.dev/articles/llm-as-reranker-guide/)). The LLM does the one thing it's uniquely good at here: *reframing* an article into a persona's "so what". A Phase-2 **listwise LLM rerank of only the top-10** is noted as an optional quality lever (§9), gated on cost.

---

## 7. Stage 5 — Materialise `daily_top5` & the Read Path

### 7.1 Materialisation (per user, cheap, on the cron)
After Stages 1–4, write the user's final ordered IDs into the existing `daily_top5` table:
- `article_ids uuid[]` = top **5** (free) / top **15** (Pro — prior doc §9A.4 "more than 5 stories/day"). One materialised row per `(user_id, feed_date)`.
- This step is a **pure join + array write — no LLM, no news API**. It's the per-user tier of the two-tier fan-out; cheap enough to precompute for every active user every tick.
- Incremental: only re-materialise users whose bucket got new rewrites this tick, or who are in `persona:dirty:set` (prior doc §6.6).

### 7.2 Read path — `GET /v1/feed/top5` (extends prior doc §6.5)
1. Verify Supabase JWT (JWKS, cached) → `user_id` (+ `is_anonymous`, onboarding doc §3.1).
2. **Redis read-through** `feed:{user_id}:{today}` → return on hit. ([read-through + materialised](https://leapcell.io/blog/choosing-between-postgres-materialized-views-and-redis-application-caching))
3. Miss → read `daily_top5(user_id, today)`, join `articles` + `impact_rewrites` on `(article_id, persona.bucket_hash)`, assemble `ImpactStory[]`.
4. Cache 6h in Redis; return.
5. **The request path never calls Gemini, Groq, or newsdata.io.** All heavy work happened on the cron. Hot path stays ~30ms active CPU → inside the Vercel Hobby budget (prior doc §5.4).

Pro pagination: `GET /v1/feed/top5?limit=15` (clamped to the user's entitlement from `/v1/me`); free users are capped at 5 server-side regardless of `limit`.

### 7.3 Cold-start / partial feed (graceful degradation)
A brand-new user (just onboarded mid-day) or a never-seen bucket has **no rewrites yet**. We do **not** block on the LLM:
- Stages 1–3 are **LLM-free**, so we can still return a **relevance-ranked list of articles immediately** — original titles + summaries, `partial: true`, no `impact_headline`/`why_summary` yet.
- The user (or their bucket) is added to `persona:dirty:set`; the next persona-drain tick (≤5 min, prior doc §6.6) generates the missing `(article, bucket)` rewrites and re-materialises `daily_top5`.
- Subsequent reads return `partial: false` with the reframed cards swapped in place. The client polls every 5s for up to 30s (prior doc §11, onboarding doc §7.3).
- **Optional fallback:** serve a *neighbouring* already-populated bucket's rewrites, or a country-wide popularity default, to fill the gap before backfill — a Phase-2 nicety. ([cold-start layers](https://www.shaped.ai/blog/mastering-cold-start-challenges))

This is the key payoff of keeping selection LLM-free: **relevance is instant; only the reframing is eventually-consistent.**

---

## 8. Data Model (deltas to prior docs)

```sql
-- 8.1 articles: add enrichment columns (Stage 0). Base table is prior doc §6.3.
alter table articles
  add column event_cluster_id uuid,                 -- same-event grouping (§4.1)
  add column is_cluster_lead  boolean default true, -- representative of its cluster
  add column importance_score real not null default 0.0, -- §4.2, 0..1
  add column embedding vector(384);                 -- Phase 2 only (pgvector), else null
create index articles_cluster_idx     on articles (event_cluster_id);
create index articles_importance_idx  on articles (importance_score desc);
-- existing & load-bearing: source_url(unique), country, published_at, impact_tags(GIN),
--                          articles_tags_idx, articles_country_idx, articles_published_idx.

-- 8.2 sources: precomputed authority weights (small seed table; §4.2)
create table sources (
  source_name text primary key,
  authority   real not null default 0.5,            -- 0..1, hand-seeded then tuned
  country     text
);

-- 8.3 daily_top5: extend for Pro size, ranking provenance, partial state.
alter table daily_top5
  add column is_partial boolean not null default false, -- true until rewrites backfilled (§7.3)
  add column ranked_at  timestamptz,                    -- when Stages 1-3 last ran
  add column feed_size  smallint not null default 5;    -- 5 (free) | 15 (pro)
-- existing: user_id, feed_date, article_ids[], push_sent_at, computed_at  (prior doc §6.3)

-- 8.4 impact_rewrites: UNCHANGED (prior doc §6.3). Key = (article_id, persona_bucket_hash).
--     This HLD only changes WHICH pairs we populate (§6.1), not the schema.

-- 8.5 personas.topic_tags: derived tag set for ranking (clustering doc §5.4).
--     Either a generated column from interests[] or written by PUT /v1/personas.
alter table personas
  add column topic_tags text[] not null default '{}';   -- mapped impact_tags for Jaccard
create index personas_topic_tags_idx on personas using gin (topic_tags);
```
RLS unchanged: `auth.uid() = user_id` on `personas`, `daily_top5` (prior doc §13.2). `articles`, `sources`, `impact_rewrites` are global/read-shared (no per-user rows). Ranking weights `{w_rel, w_fresh, w_imp, H, freshness_window_h, mmr_lambda, per_source_cap}` live in **config** (env / a small `feed_config` row / Redis) so they're tunable without a deploy — mirroring the onboarding-config philosophy.

---

## 9. API Surface (additions to prior docs §7)

> Auth model unchanged: the client uses Supabase Auth directly; our Hono API only **verifies** the JWT (JWKS, cached 1h) and reads `sub` + `is_anonymous`. We mint no tokens.

### 9.1 `GET /v1/feed/top5` — extended
```yaml
get:
  summary: Today's ranked, persona-reframed stories (Inshorts-style)
  security: [{ bearerAuth: [] }]
  parameters:
    - { in: query, name: limit, schema: { type: integer, default: 5, maximum: 15 } } # clamped to entitlement
  responses:
    '200':
      content:
        application/json:
          schema:
            type: object
            properties:
              partial: { type: boolean, description: 'true if rewrites still backfilling (§7.3)' }
              feed_date: { type: string, format: date }
              stories:
                type: array
                items: { $ref: '#/components/schemas/ImpactStory' }  # prior doc §7
```
- Free users: `limit` capped at 5 server-side. Pro: up to 15 (`/v1/me.is_pro`, prior doc §9A.4).
- `ImpactStory` carries `impact_headline`/`why_summary` (null when `partial`), `original_title`, `source_url`, `published_at`, `impact_tags` — schema from prior doc §7.

### 9.2 Internal cron endpoints (extend prior doc §9.4 / §6.4)
`POST /v1/_internal/cron/news-ingest` (CRON_SECRET-protected) now also runs **Stage 0** (cluster + importance). A new logical pass — runnable inside the same ingest job or its own workflow — runs **Stages 1–5** for active users:
```
POST /v1/_internal/cron/build-feeds   # rank (1-3) → rewrite missing pairs (4) → materialise (5)
POST /v1/_internal/cron/persona-drain # prior doc §6.6: same pipeline, scoped to persona:dirty:set
```
Both reuse the exact pipeline; `persona-drain` just scopes the user set. No new request-path endpoint is introduced for personalisation — by design, all of it is precomputed.

### 9.3 (Phase 2) `GET /v1/feed/topic/{tag}` — Pro "topic deep-dive"
Prior doc §9A.4 lists "show me everything about real-estate this week" as a Pro unlock. Phase 2: a ranked, rewrite-on-demand feed scoped to one `impact_tag`. Out of MVP scope; noted for continuity.

---

## 10. End-to-End Sequence Flows

### 10.1 Steady-state user, cron-built feed (the common case)
```
[cron 6h] news-ingest ─► pull newsdata.io ─► dedup(source_url) ─► tag impact_tags
                        └► Stage0: event_cluster_id + importance_score
[cron]    build-feeds  ─► for each active user U:
              S1 retrieve: country + last48h + tag-overlap (drop non-lead cluster members)
              S2 score:   w_rel·Jaccard + w_fresh·2^(−age/24h) + w_imp·importance
              S3 rerank:  MMR(λ=0.7) + ≤1/source + category spread → top15(U)
          ─► pairs = ⋃_U {(a, U.bucket_hash) : a ∈ top15(U)}  \  already-cached
          ─► S4 rewrite missing pairs (Gemini→Groq fallback) ─► impact_rewrites
          ─► S5 materialise daily_top5(U) = top5/top15, is_partial=false
App ─ GET /v1/feed/top5 ─► Redis hit / daily_top5 join ─► 200 {partial:false, stories:[…]}
```

### 10.2 Brand-new user onboards mid-day (cold bucket)
```
PUT /v1/personas {full persona}  (onboarding doc §7.3)
   └► compute bucket_hash, topic_tags; sadd persona:dirty:set; del daily_top5(today)
   ◄─ 202 {partial:true}
App ─ GET /v1/feed/top5
   └► daily_top5 empty → run S1–S3 on the fly (LLM-FREE) ─► ranked articles
   ◄─ 200 {partial:true, stories:[original titles, no impact_headline]}
[cron persona-drain ≤5min] ─► S4 rewrite (article, bucket) for U's top15 ─► S5 materialise
App ─ GET /v1/feed/top5 (poll) ─► 200 {partial:false, reframed cards swap in}
```

### 10.3 Persona edit (re-bucket + re-rank) — reuses prior doc §6.6
```
PUT /v1/personas {edited}
   └► recompute bucket_hash (maybe new) + topic_tags (changes ranking!)
      del feed:{U}:{today}; del daily_top5(U,today); sadd persona:dirty:set
   ◄─ 202
[cron persona-drain] ─► S1–S3 (new topic_tags → new selection)
                       ─► S4 rewrite any (article, NEW bucket) not yet cached
                       ─► S5 re-materialise
```
Note both levers move on an edit: changed **topics** change *selection* (Stages 1–3); a changed **bucket attribute** (e.g. income) changes *reframing* (Stage 4). The dirty-set drain handles both in one pass.

---

## 11. How This Sits Inside the Free-Tier Budget (ties to prior doc §4–§5)

- **newsdata.io:** untouched by this HLD — selection/ranking/rewrite all run on *our* stored articles; the read path never calls it (prior doc §4.1 implication). Still ~40 credits/day of 200.
- **LLM (Gemini 1500 RPD):** Stage 4 rewrites **only what ranks**, deduped by bucket and idempotent → ≤ the prior doc's ~540 batched calls/day estimate (§5.2), with Groq (14,400 RPD) as overflow. Stages 0 tagging/clustering fold into the same batched pass.
- **Vercel Hobby CPU:** the read path is one DB read + Redis (≈30ms active CPU, prior doc §5.4). All ranking/rewrite runs on **GitHub Actions cron** (prior doc §9.4), not the request path.
- **Postgres 500MB:** `+ importance_score/event_cluster_id` are tiny scalars; `impact_rewrites` text already budgeted at ~63MB (prior doc Decision 4). `embedding vector(384)` is Phase-2 and ~1.5KB/article × ~80K articles ≈ 120MB → only enable with Supabase Pro or aggressive retention.
- **Redis (10K cmds/day):** `feed:{user}:{day}` read-through + `persona:dirty:set` (prior doc §5.7). Unchanged.

No new vendor, no new paid tier for the MVP path.

---

## 12. Resolved Decisions & Open Questions

> Decisions taken 2026-06-10. **⟳** = needs a runtime/product confirmation before hardening.

1. **Ranking = deterministic weighted blend; LLM stays out of selection.** **DECIDED** — Jaccard + half-life + importance, weights in config. LLM is reframing-only (cost/instability of LLM-as-ranker — [zeroentropy](https://zeroentropy.dev/articles/llm-as-reranker-guide/)). Phase-2 listwise LLM rerank of top-10 is an optional quality lever, gated on cost.
2. **Selection is per-user; rewrite is per-bucket.** **DECIDED** — forced by `bucket_hash` excluding topics (clustering doc §5.4). Resolves the prior doc's loose `bucketMatchesTags`: rewrite the *union of what ranks* (§6.1).
3. **Relevance signal = tag-overlap Jaccard for MVP; embeddings Phase 2.** **DECIDED** — embeddings (pgvector `vector(384)`) deferred until free-text topics dominate or buckets feel generic; gated on Postgres storage (Supabase Pro).
4. **Freshness half-life `H` and weights** ⟳ — **TENTATIVE:** `H=24h`, window 48h, `w_rel/w_fresh/w_imp = 0.55/0.30/0.15`. Tune by editorial spot-check post-launch; keep server-driven so no app/deploy needed.
5. **Same-event dedup method** ⟳ — **DECIDED MVP:** title/TF-IDF or MinHash-LSH (Jaccard ≥ 0.85). Phase-2 embedding clustering only if lexical dedup misses too many same-event-different-wording stories.
6. **Pro feed size** — **DECIDED:** free=5, Pro=15 (prior doc §9A.4). Enforced server-side off `/v1/me.is_pro`.
7. **Diversity caps** ⟳ — **TENTATIVE:** MMR λ=0.7, ≤1/source, ≥3 categories in top-5, relax-by-one when pool thin. Validate the caps don't starve thin country pools (e.g. AE/SG, prior doc Decision 1 risk).
8. **Cold-bucket fallback depth** ⟳ — MVP serves `partial=true` ranked articles + async backfill. Whether to also serve a *neighbouring bucket's* rewrites or a popularity default before backfill is a Phase-2 product call.
9. **`source_authority` seeding** ⟳ — needs an initial hand-seeded table per country; default 0.5 for unknowns. Refine from observed coverage/quality over time.
10. **Empty-feed guard** — **DECIDED:** if tag-overlap yields too few candidates, backfill with high-`importance` country-wide stories so the feed is never empty (§5.1).

---

## 13. Build Order (backend)

1. **Stage 0 migrations** (§8.1–8.2): `articles` enrichment columns + `sources` table + indexes. Seed `sources.authority`.
2. **Stage 0 logic** in `news-ingest`: same-event clustering (§4.1, MVP lexical) + `importance_score` (§4.2), folded into the existing batched-tag pass.
3. **`personas.topic_tags`** derivation (§8.5) in `PUT /v1/personas` (onboarding doc §7.3 already normalises interests → tags; persist the tag set).
4. **Stages 1–3** ranker (`build-feeds` cron): retrieval → weighted score → MMR/dedup/caps → top-15. **LLM-free.** Weights in config.
5. **Stage 4** rewrite fan-out scoped to the *union of what ranks* (§6.1), idempotent on `impact_rewrites`, Gemini→Groq fallback.
6. **Stage 5** materialise `daily_top5` (free=5 / Pro=15, `is_partial`).
7. **Read path** `GET /v1/feed/top5` extensions (§9.1): `partial`, `limit`/entitlement clamp, Redis read-through.
8. **Cold-start path** (§7.3): on-the-fly Stages 1–3 when `daily_top5` empty; `persona:dirty:set` backfill (reuses prior doc §6.6 drain).
9. **Wire to notification job** — `daily_top5[0]` (the lead story) is what the 8 AM-local push sends (prior doc §6.7 / clustering doc §6). No change needed here beyond materialisation landing before the push tick.
10. **(Phase 2)** embeddings + semantic dedup + MMR-by-embedding; optional listwise LLM rerank of top-10; `GET /v1/feed/topic/{tag}` deep-dive.

---

## 14. Sources (verified June 2026)

### Two-stage recommenders / candidate generation
- Candidate generation overview: <https://aman.ai/recsys/candidate-gen/>
- Open-source two-stage news recommender (retrieve → rerank): <https://github.com/mahnoorsheikh16/Two-Stage-News-Recommendation-System>
- News candidate-generation survey (2025): <https://arxiv.org/html/2508.13978>
- Two-stage recommender theory: <https://arxiv.org/pdf/2403.00802>

### Content-based relevance without click data
- TF-IDF vs BM25 vs embeddings: <https://www.ai-bites.net/tf-idf-and-bm25-for-rag-a-complete-guide/>, <https://medium.com/@dineshkarthik_kandregula/hybrid-retrieval-bm25-vs-dense-embeddings-smarter-search-using-elasticsearch-huggingface-3dd6a6351b05>
- BM25 (Okapi) formula & defaults `k₁=1.2, b=0.75`: <https://en.wikipedia.org/wiki/Okapi_BM25>
- Jaccard similarity: <https://www.ibm.com/think/topics/jaccard-similarity>

### Freshness / recency decay
- Hacker News gravity `(P−1)/(T+2)^1.8`: <https://medium.com/hacking-and-gonzo/how-hacker-news-ranking-algorithm-works-1d9b0cf2c08d>, <https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/>
- Reddit "hot" ranking (`log10(s)+t/45000`): <https://medium.com/@niruthiha2000/reddits-ranking-algorithm-for-content-curation-systems-2daa3f33a14f>

### Signal blending / weighting (no ML)
- Hand-tuned weighted signal pipeline: <https://blakecrosley.com/blog/signal-scoring-pipeline>
- Recency/authority weighting & time buckets: <http://klab.lt/weighted-retrieval-signals-recency-and-authority>
- Weighted linear combination: <https://www.sciencedirect.com/topics/engineering/weighted-linear-combination>
- Google News personalization (linear fusion, MinHash): <http://glinden.blogspot.com/2007/05/google-news-personalization-paper.html>

### Diversity (MMR) & same-event dedup
- MMR formula & λ guidance: <https://aayushmnit.com/posts/2025-12-25-DiversityMMRPart1/DiversityMMRPart1.html>, <https://docs.opensearch.org/latest/vector-search/specialized-operations/vector-search-mmr/>
- Near-dup / same-event detection (MinHash 0.85, embedding clustering): <https://github.com/dukeblue1994-glitch/chronicle>, <https://aclanthology.org/2025.naacl-industry.73.pdf>, <https://dl.acm.org/doi/pdf/10.1145/3701716.3715303>
- Newsfeed diversity rules / per-source caps: <https://shree6791.medium.com/designing-a-newsfeed-ranking-system-how-social-media-decides-what-you-see-d9edee45c087>, <https://arxiv.org/pdf/2508.13035>

### Importance / newsworthiness scoring
- Google news-ranking patent (`1+log(cluster_size)` + 13 source-rank signals): <https://patents.google.com/patent/WO2005029368A1/en>, <https://www.seobythesea.com/2019/10/news-ranking-algorithm/>
- Cluster property ↔ importance/urgency: <https://arxiv.org/pdf/2402.10302>

### LLM as ranker / judge (and why not for selection)
- LLM-as-reranker cost/latency/instability: <https://zeroentropy.dev/articles/llm-as-reranker-guide/>
- Re-rankers as relevance judges, self-preference bias: <https://arxiv.org/pdf/2601.04455>

### Feed architecture (fan-out, caching, cold-start)
- Fan-out-on-write vs on-read: <https://bytebytego.com/courses/system-design-interview/design-a-news-feed-system>, <https://www.techinterview.org/post/3233474168/system-design-twitter-news-feed-timeline-fanout-on-write-fanout-on-read-celebrity-problem-ranking-caching/>
- Segment / prompt caching to cut LLM cost: <https://callsphere.ai/blog/llm-caching-strategies-cost-optimization-2026>, <https://aws.amazon.com/blogs/database/optimize-llm-response-costs-and-latency-with-effective-caching/>
- Cache invalidation (TTL + event-driven): <https://leapcell.io/blog/cache-invalidation-strategies-time-based-vs-event-driven>
- Materialised views vs Redis for serving: <https://leapcell.io/blog/choosing-between-postgres-materialized-views-and-redis-application-caching>
- Cold-start graceful degradation: <https://www.shaped.ai/blog/mastering-cold-start-challenges>, <https://en.wikipedia.org/wiki/Cold_start_(recommender_systems)>

---
