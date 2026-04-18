# Peechz — Core Architecture & Engineering Showcase

[![Next.js](https://img.shields.io/badge/Next.js_15-black?style=flat-square&logo=next.js&logoColor=white)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![Supabase](https://img.shields.io/badge/Supabase-3FCF8E?style=flat-square&logo=supabase&logoColor=white)](https://supabase.com/)
[![Groq](https://img.shields.io/badge/Groq-FF4B00?style=flat-square&logo=databricks&logoColor=white)](https://groq.com/)
[![Llama 3.3](https://img.shields.io/badge/Llama_3.3-0668E1?style=flat-square&logo=meta&logoColor=white)](https://www.llama.com/)
[![GPT-OSS](https://img.shields.io/badge/GPT--OSS_20B-412991?style=flat-square&logo=openai&logoColor=white)](https://openai.com/open-models/)
[![Cloudflare R2](https://img.shields.io/badge/Cloudflare_R2-F38020?style=flat-square&logo=cloudflare&logoColor=white)](https://developers.cloudflare.com/r2/)
[![Vercel](https://img.shields.io/badge/Vercel-000000?style=flat-square&logo=vercel&logoColor=white)](https://vercel.com/)

> **Live product:** [peechz.com](https://peechz.com)
> **Author:** Mathew Sekanjako — [LinkedIn](https://linkedin.com/in/sekanjako-mathew) · [sekanjakomathew3.0@gmail.com](mailto:sekanjakomathew3.0@gmail.com)
> **Source code is private** to protect the product during beta. This repository is a technical case study documenting real architecture decisions, system design, and engineering tradeoffs — every claim below is grounded in the actual codebase.

---

## What Peechz Is

Peechz is a TikTok-style short-form video pitch platform. Builders, founders, and engineers upload 60-second proof-of-work clips. A 3-layer AI system routes those pitches to the recruiters, investors, and collaborators most likely to care.

The engineering problem isn't the UI. It's the distributed-systems work that makes the UI feel instant while keeping counts atomic, recommendations personalized, and the backend nearly stateless on the video hot path.

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────────┐
│                    CLIENT (Next.js 15 · TypeScript)               │
│                                                                   │
│   ┌─────────────────────┐  ┌─────────────────────┐                │
│   │ InteractionManager  │  │  CommentManager     │                │
│   │  · 30s batch sync   │  │  · Supabase-first   │                │
│   │  · 50-item flush    │  │  · Active-pitch     │                │
│   │  · localStorage     │  │    realtime only    │                │
│   │  · 500-entry LRU    │  │  · Optimistic add   │                │
│   └─────────────────────┘  └─────────────────────┘                │
│           │                           │                           │
│   Supabase JS SDK (Realtime WS)      │  REST to FastAPI          │
└───────────┼───────────────────────────┼───────────────────────────┘
            │                           │
            ▼                           ▼
┌─────────────────────────┐   ┌──────────────────────────────────┐
│  Supabase (source of    │   │  FastAPI (Python)                │
│  truth for reads/writes)│   │  · /api/interactions/pitched     │
│                         │   │  · /api/interactions/batch-track │
│  · Postgres + RLS       │   │  · /api/interactions/recommend   │
│  · Auth (email + OAuth) │   │  · /api/comments/sync            │
│  · Realtime publication │   │                                  │
│  · Storage (R2 signed)  │   │  ┌────────────────────────────┐  │
│  · DB triggers maintain │   │  │  AI Agents                 │  │
│    counts atomically    │   │  │  · Searcher (Groq)         │  │
│                         │   │  │  · Detector (behavior)     │  │
└─────────┬───────────────┘   │  │  · Recommender (scoring)   │  │
          │                   │  └────────────────────────────┘  │
          │                   └────────────────┬─────────────────┘
          │                                    │
          │                                    ▼
          │                    ┌──────────────────────────────┐
          │                    │   Groq API                   │
          │                    │   Multi-key rotation (3)     │
          │                    │   Model fallback chain (3)   │
          │                    └──────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────┐
│  Cloudflare R2 + CDN                    │
│  · Object storage for videos            │
│  · Direct public URLs (pub-*.r2.dev)    │
│  · Backend NEVER proxies video bytes    │
└─────────────────────────────────────────┘
```

**Data-flow summary:**

- All writes go to **Supabase** as the single source of truth. Realtime WebSockets broadcast row changes to connected clients.
- The **FastAPI backend** handles AI recommendations, behavioral analytics, and interaction batching. It intentionally never touches video bytes.
- **Video delivery** is 100% Cloudflare R2 + CDN, routed directly from the client. This keeps the Python backend stateless on the heaviest path.
- **Interaction analytics** are batched by the client and flushed to FastAPI every 30 seconds or when 50 events queue — whichever comes first.

---

## The Sekan Algorithm — 3-Layer AI Routing

Peechz's recommendation system rejects bloated "agent frameworks" in favor of three single-responsibility agents that compose deterministically.

### Layer 1 — Searcher Agent *(LLM · Groq)*

The Searcher handles all LLM-facing work: tag generation, summary generation, and search-query expansion. It does not make recommendation decisions directly — it only enriches pitch content and expands user queries into related keywords for broader matching.

All LLM calls go through a custom `AIProvider` wrapper that:

- Loads up to **3 Groq API keys** from env vars (`GROQ_API_KEY`, `GROQ_API_KEY_2`, `GROQ_API_KEY_3`)
- On `429 Rate Limit`, puts the current key on a **60-second cooldown** and rotates to the next available key
- On repeated failure, **falls back across a model chain**: `openai/gpt-oss-20b` → `llama-3.1-8b-instant` → `llama-3.3-70b-versatile`
- Loads prompt templates from `agents/instructions/*.md` with an inline fallback dict if files are missing

This means the AI layer degrades gracefully: no single rate limit, bad key, or model outage can take the platform down.

### Layer 2 — Detector Agent *(Behavior Vectorizer)*

The Detector converts raw interaction events into structured `InteractionVector` objects. It is deterministic, stateless, and does not call any LLM. It tracks:

- `like`, `swipe_left` (viewed profile — strong interest), `swipe_right` (skipped — disinterest)
- `watch_time`, `video_duration`, computed `watch_ratio`
- `watched_enough = 1` when `watch_ratio >= 0.6`
- Device type and time-of-day for contextual signals

Interactions are batched (`batch_size = 10`) before being flushed to the Recommender. Each insert also writes a row to the `user_actions` table, which the Recommender reads during profile construction.

### Layer 3 — Recommender Agent *(Weighted Scoring Engine)*

The Recommender scores candidate pitches against each user's behavior profile. The scoring function uses **six weighted signals** that sum to 1.0, plus a random variety boost:

| Signal | Weight | What it measures |
|---|---|---|
| Tag similarity | **25%** | Overlap between pitch tags and the user's top 10 preferred tags (weighted by interaction strength) |
| Creator preference | **20%** | Whether the pitch's creator is in the user's top 5 preferred creators |
| Popularity | **18%** | Normalized combination of `views_count` (40%) and `likes_count` (60%) |
| Recency | **15%** | Exponential decay: `1.0` for <24h, `0.8` for <72h, `0.6` for <1wk, `0.3` older |
| Audience targeting | **12%** | Boost when the pitch's `target_audience` array includes the viewer's role |
| Engagement quality | **10%** | `likes / views` ratio, capped at 1.0 |

Plus a **±10% randomness factor** on the final score to prevent the same pitches from always ranking highest — especially important for new users with sparse profiles.

```python
# From backend/agents/recommender_agent.py
score += tag_score * 0.25
score += creator_score * 0.20
score += recency_score * 0.15
score += popularity_score * 0.18
score += engagement_score * 0.10
if user_role in pitch_audience:
    score += 0.12
randomness = random.uniform(0.9, 1.1)
final_score = score * randomness
```

**Graceful degradation:** if too few candidates exist after filtering seen pitches, the engine auto-supplements with trending content (scored 0.5) and retries without exclusion before giving up.

**Cache policy:** user profiles and trending pitches are cached in-memory with a 5-minute TTL. Profiles auto-invalidate when a user takes a new action — so "online learning" happens within seconds of a like or skip.

---

## Engineering Highlights

### 1. The PTCHD Metric — Honest View Counting

Vanity view counts are dishonest. I reject them. A pitch only counts as "PTCHD" (pitched) when one of two conditions is met:

1. **The viewer watched ≥ 50% of the video**, measured client-side via `IntersectionObserver` (to confirm the video is actually in-viewport) and `onTimeUpdate` (to track watch ratio).
2. **OR the viewer took a high-intent action** — like, save, or comment.

**Deduplication happens at two layers:**

- **Client-side (`InteractionManager`):** a `Set<string>` called `reportedPitched` tracks which pitches have already been reported in the current session. Duplicate events within the session are dropped before they leave the browser.
- **Server-side (`pitch_views` table):** a UNIQUE INDEX on `(pitch_id, user_id)` and `(pitch_id, session_id)` catches any duplicates the client missed. FastAPI catches the `duplicate key` error and returns `{counted: false, reason: "already_counted"}` instead of propagating a 500.

The actual counter increment uses a Postgres RPC — `increment_pitch_pitched_count` — for atomicity. No read-modify-write from Python, no race condition.

```ts
// From frontend/src/managers/InteractionManager.ts
if (watchRatio >= 0.5 || hasHighIntent) {
  this.reportedPitched.add(pitchId);
  await fetch(`${API_BASE_URL}/api/interactions/pitched`, { ... });
}
```

### 2. Hybrid Manager Pattern — ~95% Reduction in API Calls

`InteractionManager` and `CommentManager` are client-side singletons that intercept every user action before it hits the network.

**InteractionManager behavior:**
- Holds a `Map<pitchId, InteractionData>` cache, max size 500 (LRU eviction of synced entries)
- Every interaction (like, save, watch progress) updates the cache **instantly** so the UI feels zero-latency
- A **30-second sync interval** (`SYNC_INTERVAL_MS = 30000`) flushes unsynced events to `POST /api/interactions/batch-track`
- Early flush fires the moment **50 unsynced items** accumulate (`BATCH_SIZE = 50`)
- The entire unsynced queue is persisted to `localStorage` under `pchz_interactions_queue` — a hard refresh or a flaky connection won't lose data
- A fresh session ID is generated per user and persisted to `localStorage` for cross-device analytics

This design:
- Avoids a round-trip on every click (previously: N clicks = N API calls; now: N clicks = `ceil(N/50)` calls within a 30s window)
- Keeps the UI authoritative for optimistic writes
- Makes the backend's analytics pipeline pull-friendly rather than push-spammed

**CommentManager applies the same pattern** — it reads comments directly from Supabase (bypassing the batch backend for reads), but queues `CREATE / LIKE / UNLIKE / DELETE` actions locally and reconciles optimistic entries with real rows as they arrive via Realtime.

### 3. Atomic Counts via Database Triggers + Realtime Publication

> **Why it's hard:** You can't maintain `likes_count`, `saves_count`, and `comments_count` on the `pitches` row from application code without creating race conditions under concurrent load. Two simultaneous likes read the same count, both increment from `N`, both write back `N+1` — you lose a count.

> **How I solved it:** Push the counter mutations into the database itself, behind `AFTER INSERT OR DELETE` triggers, so Postgres handles the concurrency.

```sql
-- From supabase/migrations/20251117_add_count_triggers.sql
CREATE OR REPLACE FUNCTION handle_user_action_count()
RETURNS TRIGGER AS $$
BEGIN
  IF (TG_OP = 'INSERT') THEN
    IF NEW.action_type = 'like' THEN
      UPDATE pitches SET likes_count = COALESCE(likes_count, 0) + 1
      WHERE id = NEW.pitch_id;
    ELSIF NEW.action_type = 'save' THEN
      UPDATE pitches SET saves_count = COALESCE(saves_count, 0) + 1
      WHERE id = NEW.pitch_id;
    END IF;
  ELSIF (TG_OP = 'DELETE') THEN
    IF OLD.action_type = 'like' THEN
      UPDATE pitches SET likes_count = GREATEST(COALESCE(likes_count, 0) - 1, 0)
      WHERE id = OLD.pitch_id;
    ELSIF OLD.action_type = 'save' THEN
      UPDATE pitches SET saves_count = GREATEST(COALESCE(saves_count, 0) - 1, 0)
      WHERE id = OLD.pitch_id;
    END IF;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_user_action_change
AFTER INSERT OR DELETE ON public.user_actions
FOR EACH ROW EXECUTE FUNCTION handle_user_action_count();
```

Why this works:
- `SECURITY DEFINER` lets the trigger bypass RLS to update the aggregate row while still keeping per-row RLS strict for direct reads.
- `GREATEST(..., 0)` prevents negative counts if a delete arrives before its matching insert (which is structurally impossible but defensively coded anyway).
- `AFTER` trigger with `RETURN NULL` keeps the trigger transparent to the inserting transaction.

**Realtime fan-out:** the relevant tables are added to the `supabase_realtime` publication, so every count change broadcasts as a WebSocket event:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE public.likes;
ALTER PUBLICATION supabase_realtime ADD TABLE public.comments;
ALTER PUBLICATION supabase_realtime ADD TABLE public.saved_pitches;
ALTER PUBLICATION supabase_realtime ADD TABLE public.matches;
ALTER PUBLICATION supabase_realtime ADD TABLE public.pitches;
```

The frontend home feed subscribes to the `pitches` table and **debounces** incoming updates so rapid events (many users liking at once) batch into a single UI re-render — preventing mid-scroll layout shifts that would otherwise cause the feed to jitter.

### 4. Offline Comment Sync Queue — Optimistic UI with Reconciliation

Comments can be created before the network confirms them. The flow:

1. **User submits a comment.** The `CommentManager` generates a temp ID (`temp-{timestamp}-{random}`) and adds the comment to the cache with `isOptimistic: true`. UI renders instantly.
2. **An action is queued** (`CREATE`) and `sync()` fires immediately (creates don't wait for the 10-second retry timer).
3. **The sync writes to Supabase** and gets back the real row with the real ID.
4. **Reconciliation** — `reconcileRealComment()` walks the cache and replaces the optimistic entry with the real one by matching either the temp ID or a fingerprint of (`userId`, `content`, `parentId`, `createdAt` within 10 seconds).
5. **If the network was down**, the action stays in `localStorage` under `pchz_comments_queue` and retries every 10 seconds.
6. **If the same comment arrives via Realtime** before the sync response (from another device), the reconciler dedupes by ID so the user never sees their comment twice.

Realtime is scoped to the **currently-open pitch only** — not a global listener — via `subscribeToPitchComments(pitchId)` which swaps channels when the user opens a different pitch. This prevents fan-out of unrelated comment events to idle clients.

### 5. Video Delivery Is Stateless on the Hot Path

The Python backend never touches video bytes. Videos live in **Cloudflare R2** and are served via public R2 URLs (`pub-*.r2.dev`) straight from the CDN edge to the client.

**Upload flow:**
1. Client requests a presigned PUT URL from `/api/r2/presign`
2. Client uploads directly to R2 with the presigned URL
3. Client writes the video URL to the `pitches` row in Supabase
4. The backend observes zero video traffic

**Why this matters:** Python + FastAPI is excellent for AI/analytics work but terrible at proxying 50-500MB video streams at scale. By keeping video routing out of Python entirely, the backend can run on minimal infra while the CDN handles the bandwidth problem.

---

## Tech Stack

### Frontend
- **Next.js 15** (App Router) + **TypeScript**
- **Tailwind CSS** for styling, **Framer Motion** for animations
- **Supabase JS SDK** for auth, data, storage, and Realtime WebSockets
- **Singleton manager pattern** (`InteractionManager`, `CommentManager`, `ChatManager`) for state + network batching

### Backend
- **FastAPI** (Python 3.9+) with Pydantic models
- **AI agents** as Python classes, exposed through REST routes
- **Supabase-py** client with service role key for server-side writes
- **python-dotenv** for config

### Database / Realtime
- **Supabase Postgres** with Row Level Security (RLS) on every table
- **Database triggers** for atomic count maintenance (`handle_user_action_count`, `handle_comment_count`, `handle_comment_like_count`, `increment_pitch_pitched_count`)
- **UNIQUE INDEX** constraints for server-side deduplication
- **`supabase_realtime` publication** for WebSocket fan-out on `likes`, `comments`, `saved_pitches`, `matches`, `comment_likes`, `pitches`
- **pg_cron + Edge Functions** for scheduled cleanup work (not yet hot, infra is in place)

### AI / Inference
- **Groq API** for LLM inference — multi-key rotation (up to 3 keys, 60s cooldown on 429)
- **3-model fallback chain**, in order of preference:
  1. `openai/gpt-oss-20b` — OpenAI's open-source 20B-param model (default)
  2. `llama-3.1-8b-instant` — Meta's fast 8B model (fallback on rate limit)
  3. `llama-3.3-70b-versatile` — Meta's 70B model (fallback on quality-critical tasks)
- **Markdown-based prompt templates** (`agents/instructions/*.md`) with inline Python fallbacks

### Infrastructure
- **Cloudflare R2** for object storage (video + thumbnails), fronted by Cloudflare's CDN
- **Vercel** for Next.js hosting
- **Supabase Cloud** for DB, auth, realtime, and storage
- **GitHub** for source, deploys via `git push origin main`

---

## Deliberate Non-Choices

A few things I intentionally did *not* use, and why:

- **No LangChain / CrewAI / LlamaIndex.** The agent framework ecosystem adds latency and opacity for a 3-agent system. Three Python classes are clearer and faster.
- **No Redis / separate cache tier.** User profile + trending caches live in the Python process memory with a 5-minute TTL. One FastAPI instance is enough for current load, and horizontal scaling isn't the current bottleneck. Moving to Redis is a one-hour change when it's needed.
- **No client-side vector DB for recommendations.** Tag overlap + weighted scoring delivers fast, explainable results. Semantic embeddings are the right next step when the corpus grows past ~10K pitches, not before.
- **No video transcoding in the backend.** Browsers handle MP4/WebM natively. Users get a clear error if they upload an unsupported format. Ingesting through a transcoder would burn CPU on a path that doesn't need it yet.

Each of these is a deliberate "not yet" — each has a clear trigger condition that would flip the decision.

---

## Status

- **Live beta:** [peechz.com](https://peechz.com)
- **Ohio Hackathon 2025** — Most Original & Innovative Product
- **Current focus:** hardening comment realtime, Safari video/audio playback, and the recommender's cold-start path for first-time users

---

## About the Author

Built and maintained by **Mathew Sekanjako** — founder, full-stack engineer, grad student. Previously shipped [P31 on the iOS App Store](https://apps.apple.com/us/app/p31/id6759008572), [AkademData](https://github.com/mathewisrich/akademdata-open-source) (deployed at Mount Vernon Nazarene University), and CILGD — a government-commissioned national platform for Ghana's Chartered Institute of Local Governance.

Open to **founding engineer roles, AI/ML contracts, and technical consulting.**

**Contact:** [sekanjakomathew3.0@gmail.com](mailto:sekanjakomathew3.0@gmail.com) · [LinkedIn](https://linkedin.com/in/sekanjako-mathew) · [GitHub](https://github.com/mathewisrich)
