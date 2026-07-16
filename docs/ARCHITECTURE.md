# ARCHITECTURE — Architecture-is-Cool

## 0. Architecture style: Modular Monolith

**Explicit choice, not a default.** Single deployment (one FastAPI container,
`docker compose up`), but with module boundaries **enforced in code**, not just folders:
`api/routes` never touches DB/LLM directly — only through `services`; `agents` never touch
the DB directly — only through `services`/`vector`. Each module (`agents`, `vector`,
`services`) could theoretically be extracted into a separate microservice later without a
logic rewrite, just a change in the transport layer.

**Why not microservices**: would add infrastructure (service discovery, network calls,
distributed tracing) that costs more development time than it's worth for a side project
built in hours-per-week increments.
**Why not a "plain" (non-modular) monolith**: risks free coupling between components —
hard to demonstrate "this is a real multi-agent system" if the boundaries between agents
and the rest of the system aren't clear and tested in code.

**Modular Monolith** gives both: simple to deploy, but the internal boundaries are real —
and this is also the substrate needed for the LangGraph graph (multiple nodes with shared
state) to be a genuine module, not "one big function called an agent."

## 1. Guiding Principles

1. **Agent-driven from day one** — shared state flows between agents, orchestration is
   explicit in a graph (not scattered if/else), and a Critic loop can send work back for
   another pass. This is what distinguishes the system from "an endpoint that calls an
   LLM."
2. **12-factor / cloud-agnostic** — all configuration via env vars. No `if GCP` /
   provider-specific API calls in application logic.
3. **Thin interfaces around every external dependency** (LLM, embeddings, storage) — so
   swapping providers/environments is a config change, not a rewrite.
4. **Short, readable code** — no layering "because we can"; every layer exists because
   it's tested/replaced independently.

## 2. Tech Stack

| Component | Technology | Note |
|---|---|---|
| Language | Python 3.12 | |
| API framework | FastAPI + Uvicorn | async-first |
| Validation | Pydantic v2 | also for agent structured output |
| ORM | SQLAlchemy 2.x (async) | |
| Migrations | Alembic | |
| DB | PostgreSQL + pgvector | dev and prod identical (cloud-parity) |
| Agent orchestration | **LangGraph 1.x (GA)** | not the pre-1.0 API |
| LLM | Gemini (behind `LLMProvider` interface) | |
| Embeddings | `gemini-embedding-001` (text) | via `EmbeddingProvider` interface |
| Frontend | Next.js (React) | |
| Containers | Docker + docker-compose | |

## 3. Folder Structure

```
app/
├── api/routes/        # topics.py, chat.py, quiz.py, progress.py — HTTP only, no business logic
├── core/              # config.py (Pydantic Settings), security.py
├── models/            # SQLAlchemy ORM models
├── schemas/            # Pydantic request/response schemas
├── services/          # topic_service, embedding_service, rag_service, llm_service, recommendation_service
├── agents/            # graph.py + retriever_agent.py, mode_agent.py, quiz_agent.py, quiz_critic_agent.py
├── vector/            # pgvector access layer
├── prompts/           # prompt templates (kept out of code)
├── utils/
└── main.py
```

**Key rule**: `api/routes` never call the DB/LLM directly — only `services`. `agents` call
`services` (e.g. `rag_service` for retrieval), not the DB directly. Dependency chain is
one-directional: `routes → agents/services → vector/DB`.

## 4. Core Interfaces

```python
class LLMProvider(Protocol):
    async def generate(self, prompt: str, *, schema: type[BaseModel] | None = None) -> str | BaseModel: ...

class EmbeddingProvider(Protocol):
    async def embed(self, texts: list[str]) -> list[list[float]]: ...

class StorageProvider(Protocol):
    async def save(self, content: bytes, uri_hint: str) -> str: ...   # returns URI/path
    async def read(self, uri: str) -> bytes: ...
```

First implementation for all: Gemini (LLM + embeddings) and local filesystem (storage,
used for fetched HTML source snapshots). Extending to Claude/OpenAI/S3/GCS is a new
implementation behind the interface, not a change to calling code.

## 5. Multi-Agent Graph (LangGraph)

```
                  ┌────────────┐
   State ───────► │  Retriever │──┐
                  └────────────┘  │
                                  ▼
                          ┌───────────────┐
                          │  Mode Router  │  (joke | analogy | code)
                          └───────┬───────┘
                                  │  (quiz mode selected)
                                  ▼
                          ┌───────────────┐     ┌──────────────┐
                          │ Quiz Generator│────►│ Quiz Critic  │
                          └───────────────┘     └──────┬───────┘
                                                        │ (wrong/unclear)
                                          ┌─────────────┘
                                          ▼
                                  (re-explain / retry, retry_count guard)
                                          │ (resolved)
                                          ▼
                                ┌────────────────────┐
                                │ Recommendation step │──► suggest_related_topic()
                                └────────────────────┘
```

**State** (shared, passed between nodes):
```python
class TopicState(TypedDict):
    topic_id: str
    retrieved_chunks: list[SourceChunk]
    mode: Literal["joke", "analogy", "code", "quiz"]
    level: Literal["junior", "senior", "architect"] | None   # code mode only
    quiz_question: str | None
    user_answer: str | None
    critic_verdict: str | None
    retry_count: int
    related_topics_suggested: list[str]
```

**Role of each node:**
- **Retriever** — fetches the grounding chunk(s) for the selected topic (deterministic,
  via the `topics` link table), or performs a similarity search for free-form questions
  scoped to the current topic.
- **Mode Router** — generates a joke, analogy, or leveled code sample from the same
  retrieved chunk(s), depending on the user's chosen mode. Consistency across modes comes
  from grounding all three in the same source content.
- **Quiz Generator** — produces a question grounded in the retrieved chunk(s).
- **Quiz Critic** — evaluates the user's free-text answer **against the retrieved source
  chunk** (not general LLM judgment), explains what was right/wrong, and can trigger a
  secondary retrieval pass to surface related material from elsewhere in the corpus.
  `retry_count` prevents an infinite explain-retry loop (same anti-hallucination pattern
  as the reference project's Critic). **The "correct-but-not-in-source" case is a first-
  class verdict, not a failure**: when the user's answer is plausibly right but the
  retrieved chunk doesn't cover it, the Critic must return `unclear` with an honest
  "the source material doesn't address this — here's what it does say" response, never a
  false `incorrect`. The critic prompt states this explicitly; experienced developers
  answering from experience beyond the article is an expected demo scenario, and
  penalizing it would destroy trust in the whole loop.
- **Recommendation step** — calls `suggest_related_topic()` (see §6) after quiz resolution
  or on entry points described in PRD.md F8.

This Quiz Critic is the **anti-hallucination loop** for this project — it demonstrates
"an agent that checks another agent's output against source," not just a linear pipeline.

**LLM failure behavior** (not deferred — needed for a live demo): same pattern as the
reference project — retry with exponential backoff on 429/5xx inside the `LLMProvider`
implementation (agents don't know about it), and a final failure after retries doesn't
crash the graph — it surfaces a clear "system busy, try again" state instead of a stack
trace, using LangGraph 1.x per-node timeout + error recovery.

## 6. RAG Pipeline

This project has **two distinct retrieval modes** — keeping them conceptually separate is
what makes the architecture legible:

**1. Sequential/deterministic retrieval** (topic-driven):
```
user selects topic → topics table lookup → chunk_range for that topic's source_file → chunks
```
No similarity search involved — this is a direct lookup via the `topics` link table.

**2. Free-text similarity retrieval** (question-driven):
```
free-form question / quiz answer → embed → pgvector similarity search (top-k)
    → filtered to current topic's source_file (default) OR whole corpus (explicit "expand" / related-reference lookup)
    → prompt construction (with source citation) → LLM → answer + sources
```

**Ingestion pipeline:**
```
fetch(url) → html_to_text() → chunk() (heading/section-aware, doesn't cut mid-concept)
    → embed() (EmbeddingProvider) → pgvector upsert
```

**Two entry points into the same pipeline** — the pipeline itself is identical, only the
trigger differs:
1. **Seed script** (`scripts/seed_embeddings.py`) — curator-run, offline, batch of curated
   URLs (§8 seed/dev split).
2. **User-triggered live ingestion** (F1.5) — a single URL submitted via
   `POST /topics/from-url`, run synchronously (~10-30s) while the frontend shows a loading
   state. Same `ingestion_service`, same cache check, no separate code path — see §6.6.

**On source format**: fetching HTML rather than PDF only affects the extraction step
(`html_to_text()` vs. `pypdf` extraction). Chunking, embedding, and retrieval logic are
identical regardless of source format. Adding a PDF source later means adding a new
extraction branch behind the same ingestion interface — not a redesign of retrieval.

## 6.5. DB Schema (main tables)

Same separation principle as the reference project: a **cache layer** (expensive
processing artifacts — extraction, chunking, embeddings — keyed by content) and a
**domain layer** (which topic maps to which source content). Mixing the two is what
creates inconsistency — hence separate tables.

```
source_files                -- CACHE layer: processing artifacts, keyed by content
├── id               UUID PK
├── content_hash     text            -- SHA-256 of fetched URL content
├── source_url       text
├── pipeline_version text            -- identifies embedding model + chunking version
├── storage_uri      text
├── extracted_text   text NULL
├── created_at       timestamptz
└── UNIQUE (content_hash, pipeline_version)

topics                      -- DOMAIN layer: thin link between a learnable topic and its source
├── id               UUID PK
├── title            text            -- curator-entered (seed) or auto-derived from content (F1.5, §6.6)
├── source_file_id   UUID FK → source_files.id
├── chunk_range      int[]           -- which chunk indices belong to this topic
├── topic_embedding  vector(768)     -- for similarity-based recommendation (§8)
├── status           text            -- processing | ready | failed (seed topics are always 'ready')
├── source           text            -- seed | user_submitted (UI badge + curation filtering)
└── created_at       timestamptz

chunks                      -- RAG unit: belongs to cache, not to a specific topic exclusively
├── id               UUID PK
├── source_file_id   UUID FK → source_files.id
├── chunk_index      int
├── text             text
├── embedding        vector(768)     -- pgvector
├── embedded_at      timestamptz NULL
└── UNIQUE (source_file_id, chunk_index)
-- NOTE: no topic_id FK here — topic↔chunk mapping lives ONLY in topics.chunk_range
-- (single source of truth). A chunk can belong to overlapping topics from the same
-- source file; a per-chunk FK can't represent that and would drift from chunk_range.

user_progress               -- PLAIN SQL STATE — explicitly NOT embeddings-based
├── id               UUID PK
├── user_id          UUID
├── topic_id         UUID FK → topics.id
├── status           text            -- not_started | completed
├── mastery_score     float NULL
├── last_seen_at     timestamptz
└── UNIQUE (user_id, topic_id)

quiz_attempts                -- audit: every quiz interaction, including Critic retries
├── id               UUID PK
├── user_id          UUID
├── topic_id         UUID FK → topics.id
├── question         text
├── user_answer       text
├── critic_verdict    text            -- correct | incorrect | unclear
├── retry_count       int
└── created_at       timestamptz

generation_cache            -- CACHE layer: lazily-populated LLM outputs, keyed by
                             -- (topic, mode, level, prompt_version); NOT pre-seeded —
                             -- rows appear on first live request, see §8
├── id               UUID PK
├── topic_id         UUID FK → topics.id
├── mode             text            -- joke | analogy | code | quiz_question
├── level            text NULL       -- junior | senior | architect; NULL unless mode='code'
├── prompt_version   text            -- prompt template version; bump invalidates old rows
├── variant_text     text            -- the generated content served to the user
├── created_at       timestamptz
└── INDEX (topic_id, mode, level, prompt_version)   -- lookup + count-per-key on cache read
```

**Important distinction to preserve in any future extension**: `user_progress` is plain
relational state (what's done, how well) — it answers "has this user completed this
topic," a lookup, not a similarity question. It is deliberately **not** embeddings-based.
The recommendation mechanism (§8) is the only component that uses embedding similarity for
suggestions — conflating the two makes the system harder to reason about and to explain in
an interview.

**Cache mechanism** — same content-addressing pattern as the reference project:
`content_hash` (of the fetched URL content) + `pipeline_version` as the composite key,
checked **before** processing. A cache hit skips extraction/chunking/embedding entirely.
Re-fetching a URL whose content hasn't changed is a no-op beyond the initial hash
computation; changing the embedding model or chunking logic naturally invalidates the
cache (new `pipeline_version`), avoiding silent hits against stale vector space.

## 6.6. User-Submitted Topic Ingestion (F1.5)

The runtime entry point into the ingestion pipeline (§6). Flow for `POST /topics/from-url`:

1. **Insert a `topics` row with `status='processing'`, `source='user_submitted'`** and
   return its id immediately — the frontend polls `GET /topics/{id}` and shows a loading
   state until `status` flips. (Polling a plain row beats a blocking 30s HTTP call; no
   queue/worker infrastructure needed — the ingestion runs as a FastAPI background task.)
2. **Content-hash cache check first** (same mechanism as §6.5): if this URL's content was
   already ingested — e.g. another user submitted it — skip extraction/chunking/embedding
   entirely and link the topic to the existing `source_file`/chunks. Duplicate URL
   submissions cost one fetch + hash, nothing more.
3. **On cache miss**: run the standard pipeline (fetch → `html_to_text()` → chunk → embed
   → upsert) via the same `ingestion_service` the seed script uses — no separate code path.
4. **Auto-derive the title**: heuristic first — page `<title>` or first `<h1>` from the
   fetched HTML; if neither yields usable text, fall back to a one-line LLM summary of the
   first chunk (`LLMProvider`, already available). No user-entered title.
5. **Finalize the topic row**: `chunk_range` = all chunk indices from this source file;
   `topic_embedding` = centroid (mean) of the chunk embeddings — same computation used for
   curated topic embeddings, so recommendation similarity (§7) works uniformly across seed
   and user-submitted topics; set `status='ready'`.
6. **On any failure** (fetch error, non-HTML content, empty extraction): set
   `status='failed'` with a short user-facing reason. No partial state — a topic is either
   `ready` with full chunks/embedding, or `failed`; never left `processing` after the task
   ends.

Once `status='ready'`, the topic is **indistinguishable from a curated topic** to the rest
of the system — same F2-F5 loop, same graph, same recommendation pool. The `source` column
exists only for UI treatment and future curation, not for logic branching.

**v1 guardrails** (full hardening deferred to v2, PRD.md §6): existing API-key auth +
per-key rate limiting (§9) are the only protections. Basic sanity checks that are free to
include: reject non-`http(s)` schemes, cap fetched content size (e.g. 2 MB), and a fetch
timeout — these are input validation, not a moderation system.

## 7. Recommendation Engine — `suggest_related_topic()`

The one genuinely new RAG-adjacent capability beyond a standard Q&A RAG loop: retrieval
used for **recommendation**, not just for answering a question.

```python
async def suggest_related_topic(user_id: UUID, anchor_topic_id: UUID) -> Topic | None:
    anchor_embedding = await db.get_topic_embedding(anchor_topic_id)
    candidates = await vector.similarity_search(
        anchor_embedding, exclude_topic_id=anchor_topic_id, top_k=5,
        status="ready",   # never recommend a processing/failed topic (F1.5)
    )
    completed = await db.get_completed_topic_ids(user_id)
    return next((t for t in candidates if t.id not in completed), None)
```

**Triggered on three occasions**, all calling the same function with a different anchor:
1. User re-selects an already-`completed` topic → anchor = that topic.
2. User finishes a topic (quiz resolved) → anchor = the just-completed topic.
3. User opens the chat with no topic selected → anchor = the user's most recently
   completed topic (or highest-mastery topic).

This keeps the recommendation logic in exactly one place, tested once, called from three
different UX entry points.

## 7.5. Interview-Prep Mode (v2 — not built in v1)

**Deferred to v2** (see PRD.md §6). Kept here as a documented extension point so the v1
graph/schema design doesn't accidentally preclude it later. Not a separate pipeline — a
parameterization of the existing Retriever → Quiz Generator →
Quiz Critic graph and the `suggest_related_topic()` recommendation mechanism (§7).

**Session profile** (transient, not persisted state — passed as request params, not a new
DB table): `domain` (e.g. "architecture", "design patterns"), `language` (e.g. "Python"),
`level` (junior | senior | architect).

**What changes vs. the default flow:**
1. **Topic ranking anchor** — instead of anchoring `suggest_related_topic()` on a single
   completed topic, the anchor is an embedding of the session profile
   (`domain + language` combination). Same function, same filtering-by-`user_progress`
   logic, different anchor source.
2. **Quiz Generator prompt variant** — a `style="interview_question"` parameter switches
   the system prompt from "check conceptual understanding" to "ask like a technical
   interviewer would, calibrated to `level`." Same node, same grounding-in-retrieved-chunk
   requirement — only the prompt template differs (`prompts/quiz_interview.txt` vs.
   `prompts/quiz_standard.txt`).
3. **Quiz Critic** is unchanged — it still evaluates the user's answer against the
   retrieved source chunk, regardless of question style.

This mode is intentionally kept this thin: it demonstrates the graph is generic enough to
support a second use case without hardcoding "daily learning" as the only shape the system
supports — without adding new retrieval logic, new tables, or new agents.

## 8. Deployment

Same zero-cost path as the reference project — unchanged, proven:
- **DB (Postgres+pgvector)**: Supabase free tier (not Render's free Postgres — it's
  deleted after 30+14 days of inactivity).
- **Backend (FastAPI)**: Render Web Service, Docker-native, free tier.
- **Frontend**: Vercel.
- **Gemini API free tier**: 1,500 requests/day, 10/minute — sufficient for development and
  live demo scenarios.
- Zero cost guaranteed as long as no billing card is registered anywhere. If a card is
  ever added (e.g. to try a Pro model), set a hard **Budget + Budget Alert with Cap** in
  Google Cloud Console before doing so.

**Demo-scale & cost strategy** (supports a ~100-user live demo on cents, not architecture
changes). Topic selection is button-driven, so the (topic × mode × level) input space is
small and finite — but content is **lazily cached on first request, not pre-generated**,
so every user still sees a genuinely live LLM response the first time a combination is
requested, not canned copy loaded at startup:
1. **Explanations & quiz questions** — cached in `generation_cache` (§6.5), keyed
   `(topic_id, mode, level, prompt_version)`. `mode ∈ {joke, analogy, code, quiz_question}`;
   `level` is set only when `mode='code'`, else NULL. First request for a given key calls
   the LLM and inserts a row; every later request for that same key reads from the table.
   To avoid the "same joke word-for-word every time" feel, cap rows per key at **N=3**: on
   read, `COUNT(*)` for the key — if `< N`, call the LLM and insert a new row; if `== N`,
   skip the LLM and `SELECT` one row at random. This bounds LLM calls to
   `(topics × modes × levels × N)` regardless of user count, while every user still
   experiences it as live generation. A `prompt_version` bump (template change)
   naturally starts a fresh set of rows, same invalidation pattern as `source_files`.
2. **Recommendation neighbors** — computed live as a pgvector similarity query (§7). This
   costs a DB query, not an API call — no caching needed, and it stays correct as
   user-submitted topics (F1.5) enter the pool at runtime. (An earlier draft precomputed
   top-5 neighbors at seed time; dropped — it would go stale with every user-submitted
   topic, to save a query that was already free.)
3. **Domain guard applies to free-text input only** — button clicks skip it (nothing to
   check), saving an embedding call per interaction.

The only call that's never cached is the **Quiz Critic** (free-text answers are unique per
user by definition). It stays fully live — it's the core product value and the moment that
most proves "real AI" to a demo audience. For a high-traffic demo day: enable the paid
tier with a hard Budget Cap (see below); actual cost for ~500 critic calls on a
Flash-class model is under $1. Backoff/queue (§5 failure behavior) absorbs momentary
bursts.

**Seed/dev split** (same rationale as reference project — avoids burning API quota on
every restart):
1. `scripts/seed_embeddings.py` — one-off/deliberate run, only when the seed article
   corpus changes. Computes embeddings for the curated corpus, saves a snapshot
   (`data/embeddings_seed.json`).
2. `load_seed_into_db()` — called on service startup; loads the snapshot into pgvector if
   the table is empty. Never recomputes embeddings. Regular `docker compose up` /
   debugging never touches the Gemini API for the seed corpus.

## 9. Security & Secrets

Same three-layer discipline as the reference project — no exceptions:

1. `.gitignore` from the first commit: `.env`, `.env.*` (except `.env.example`), `*.pem`,
   `*.key`, credential files, and any fetched raw source snapshots that shouldn't be
   committed.
2. `.env.example` **does** go into git (variable names only, no real values) — this is
   what lets someone clone the repo and know which env vars to set, without exposing keys.
   The curated seed article corpus (`data/seed/`) is an explicit, intentional exception —
   it's public content, safe to commit, and needed for a working clone.
3. Before any `git push` of a pre-existing repo, run `git status` and check no `.env` or
   key file appears — don't rely on `.gitignore` alone if a file was ever tracked before
   the ignore rule existed (`git rm --cached` is required in that case).

**API protection before any public deploy:**
- Single static API key in a header (`X-API-Key`), loaded from env, checked by one FastAPI
  dependency across all routes except `/health`.
- Rate limiting (slowapi) on LLM-calling endpoints, matched to the Gemini free-tier
  10/minute limit.
- CORS restricted to the frontend's origin (env var, never `*`).
- Secrets loaded only through `core/config.py` (Pydantic `Settings`) — no scattered
  `os.environ[...]`, no hardcoded keys anywhere, including in scripts/tests.
- Logs never print full prompt content or keys — metadata only (topic_id, agent name,
  latency).

**Domain-scoping guard (prevents the chat being used as a free general-purpose LLM proxy):**

A single shared API key and standard rate limiting stop an anonymous outsider from
hammering the backend, but they don't stop an authenticated user (or anyone who copied the
key out of browser DevTools — it's client-visible by construction) from sending unrelated
prompts ("write me an essay," "what's the weather") straight to the LLM through this
endpoint. That's both an abuse vector (this becomes a free proxy to Gemini) and a cost
leak (burns the free-tier quota on traffic the product wasn't built for).

Fix: before any user message reaches `LLMProvider.generate()`, run it through a **domain
relevance check** — embed the incoming message and run a similarity search against the
topic corpus; if the top match score is below a threshold, short-circuit with "this
assistant only answers questions about software design patterns and architecture" instead
of forwarding to the LLM. This reuses the existing `EmbeddingProvider` and `vector`
similarity-search machinery already built for retrieval (§6) — no new dependency, just
another call site for the same `EmbeddingProvider.embed()` + pgvector similarity query.

```python
async def is_in_domain(message: str, *, threshold: float = 0.55) -> bool:
    message_embedding = await embedding_provider.embed([message])
    best_match = await vector.max_similarity_to_corpus(message_embedding[0])
    return best_match >= threshold
```

Placed as a guard clause in the chat route/service, before the graph runs — an
out-of-domain message never reaches the LLM at all, so it costs one embedding call instead
of a full generation call.

The `0.55` threshold is a placeholder, **not** a decision: cosine-similarity distributions
vary by embedding model. Before locking it, calibrate empirically — a small script
(`scripts/calibrate_domain_threshold.py`) runs ~20-30 known in-domain and out-of-domain
sample questions against the real corpus and reports the score distributions; pick the
threshold from the gap. The value lives in env config, not code.

**Per-key rate limiting** — the rate limiter (slowapi) must be keyed **per API
key/session**, not only globally on the backend process, so one user's traffic can't
exhaust the whole app's daily Gemini quota for everyone else.

## 10. Testing & CI

Principle: **the interesting logic is tested without an LLM and without network calls.**
The Protocol interfaces (§4) enable this — `FakeLLMProvider`/`FakeEmbeddingProvider`
(deterministic responses) are injected instead of Gemini.

**What's tested (pytest):**
- Graph logic: Quiz Critic's conditional edge, `retry_count` stop condition, fallback to
  a "system busy" state on repeated LLM failure — all with fake providers, zero API calls.
- Domain-scoping guard (§9): in-domain message passes through, out-of-domain message is
  short-circuited before reaching `LLMProvider` — testable with a fake embedding provider
  by controlling the similarity score returned.
- Cache mechanism (§6.5): hit skips processing, miss doesn't, differing `pipeline_version`
  forces a miss.
- Chunking: doesn't cut mid-concept/section.
- `suggest_related_topic()`: filtering logic (excludes completed topics, excludes anchor
  itself) — testable entirely with fake embeddings, no real similarity computation needed
  for correctness of the filter.
- Routes: smoke tests with `TestClient` (schema validation, error codes).

**CI**: one GitHub Actions job — ruff + pytest on every push. Nothing in CI requires a real
API key. CD (automatic deploy) is out of scope for v1.
