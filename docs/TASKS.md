# TASKS — Architecture-is-Cool

Organized by requirement (see REQUIREMENTS.md for file ownership and rationale). Tasks
inside one requirement are sequential; requirements in the same **Phase** can be developed
in parallel by different people/branches without touching each other's files, as long as
the noted coordination points are respected.

---

## Phase 0 — Foundation (must land first, blocks everything)

### R0 — Foundation
- [ ] `app/core/config.py` — Pydantic `Settings`, all env vars from `.env.example`
- [ ] `.env.example` + `docker-compose.yml` + `Dockerfile`
- [ ] `app/models/__init__.py` — SQLAlchemy async engine/session setup
- [ ] `alembic/` — migrations scaffold (empty initial migration)
- [ ] `app/vector/connection.py` — pgvector session/connection helper (no queries yet)
- [ ] Core interfaces in `app/core/interfaces.py`: `LLMProvider`, `EmbeddingProvider`,
      `StorageProvider` (Protocols per ARCHITECTURE.md §4)
- [ ] Gemini implementations + local-filesystem `StorageProvider` implementation
- [ ] `app/main.py` — FastAPI app instance, health check route only
- [ ] Freeze `app/agents/state.py` `TopicState` shape (ARCHITECTURE.md §5) — this is the
      contract R3/R4/R5 build against; changing it later touches three requirements at once

**Exit criteria**: `docker compose up` boots an empty FastAPI app with a working `/health`
route and a DB connection. Nothing product-specific yet.

---

## Phase 1 — Data & Domain (parallel-safe once Phase 0 lands)

### R1 — Ingestion Pipeline
- [ ] `app/models/source_file.py`, `app/models/chunk.py` + migration
- [ ] `app/utils/html_to_text.py`
- [ ] `app/utils/chunking.py` (heading/section-aware, tested for no mid-concept cuts)
- [ ] `app/services/ingestion_service.py` — fetch → extract → chunk → embed → upsert,
      content-hash + `pipeline_version` cache check (ARCHITECTURE.md §6.5)
- [ ] `data/seed/` — curate 15-25 licensing-clean source URLs/files
- [ ] `scripts/seed_embeddings.py` — one-off run, produces `data/embeddings_seed.json`
- [ ] `load_seed_into_db()` on startup — loads snapshot if table empty, never recomputes

**Depends on**: R0. **No file overlap with R7.**

### R7 — Progress Tracking
- [ ] `app/models/user_progress.py` + migration
- [ ] `app/services/progress_service.py` — get/set status, mastery_score
- [ ] `app/schemas/progress.py`
- [ ] `app/api/routes/progress.py` — CRUD-ish endpoints

**Depends on**: R0, R2 (needs `topics` table to FK against — can stub against a not-yet-
merged `Topic` model type as long as the FK name is agreed, but final migration needs R2
merged first).

---

## Phase 2 — Topic Domain (sequential gate before Phase 3 fan-out)

### R2 — Topic Domain & F1
- [ ] `app/models/topic.py` + migration (`chunk_range`, `topic_embedding`, plus `status`
      and `source` columns per ARCHITECTURE.md §6.5 — included now so R1.5 needs no
      follow-up migration)
- [ ] `app/services/topic_service.py` — deterministic topic → chunk lookup
- [ ] `app/schemas/topic.py` — include `status`/`source` in the response schema
- [ ] `app/api/routes/topics.py` — list/get topic endpoints

**Depends on**: R1. **Blocks R3, R4, R5, R6, R8, R1.5** — this is the sequential gate.
Land this before starting Phase 3 in earnest (Phase 3 requirements can be
scaffolded/stubbed earlier against the frozen `TopicState`, but need real `topics` data to
integration-test).

### R1.5 — User-Submitted URL Ingestion (after R2 lands; parallel-safe vs. all of Phase 3)
- [ ] `app/utils/title_extraction.py` — heuristic (`<title>` / first `<h1>`) + one-line
      LLM-summary fallback when the heuristic yields nothing usable
- [ ] `topic_service.create_topic_from_url(url)` — insert `processing` topic row → content-
      hash cache check via `ingestion_service` (hit: link existing source_file/chunks) →
      on miss run the standard pipeline → derive title → `chunk_range` = all chunks,
      `topic_embedding` = centroid of chunk embeddings → `status='ready'`; any failure →
      `status='failed'` + short reason (never left `processing`)
- [ ] `POST /topics/from-url` — inserts the `processing` row, kicks the ingestion off as a
      FastAPI background task, returns topic id immediately
- [ ] `GET /topics/{id}` returns `status` (already part of R2's schema) — frontend polls
      this for the loading state
- [ ] Basic input sanity (v1-level only, per ARCHITECTURE.md §6.6): http(s) scheme check,
      ~2 MB content-size cap, fetch timeout
- [ ] Tests: duplicate-URL cache hit (second submit → `ready` without re-ingestion),
      bad-URL failure path (`status='failed'`, clear reason), title fallback path

**Depends on**: R1, R2. **Blocks**: none — does not touch R3/R4/R5/R6 files; a `ready`
user-submitted topic is indistinguishable from a curated one downstream.

---

## Phase 3 — Agent Graph & Core Loop (parallel with coordination points)

Before starting: confirm `TopicState` (frozen in Phase 0) and route names
(`POST /chat/mode`, `POST /chat/quiz`, `POST /chat/ask` for free-text) so R3/R4/R6 don't
collide inside `chat.py`.

### R5 — Retriever Agent & Graph Orchestration
- [ ] `app/agents/retriever_agent.py` — deterministic topic lookup + similarity-scoped
      free-text retrieval
- [ ] `app/services/rag_service.py` — shared retrieval helper used by Retriever + R6
- [ ] `app/agents/graph.py` — LangGraph wiring: Retriever → Mode Router → Quiz Generator →
      Quiz Critic → Recommendation step (stub node calls until R3/R4/R8 land, wire for
      real after)
- [ ] Per-node timeout + error recovery → "system busy" fallback state (no stack traces)

**Depends on**: R0, R1, R2. **Blocks R3, R4, R6** for full integration (they can write
their own agent logic against the frozen `TopicState` in parallel, then wire into
`graph.py` once R5's skeleton exists).

### R3 — Mode Generation & F2
- [ ] `app/agents/mode_agent.py` — joke/analogy/code generation, grounded in retrieved
      chunk(s)
- [ ] `app/prompts/joke.txt`, `analogy.txt`, `code_junior.txt`, `code_senior.txt`,
      `code_architect.txt`
- [ ] `app/api/routes/chat.py` — `POST /chat/mode` handler only (**do not touch other
      handlers in this file** — R6 and R11 also add handlers/guards here)
- [ ] Wire into `graph.py`'s Mode Router node (coordinate merge with R5)

**Depends on**: R0, R2, R5 (skeleton). **Shares `chat.py` with R6, R11** — add your handler,
don't restructure the file.

### R4 — Quiz Generation & Critic Loop
- [ ] `app/models/quiz_attempt.py` + migration
- [ ] `app/agents/quiz_agent.py` — question generation grounded in retrieved chunk(s)
- [ ] `app/agents/quiz_critic_agent.py` — evaluates answer against source chunk, triggers
      multi-hop retrieval (F5) on demand, `retry_count` guard
- [ ] `app/prompts/quiz_standard.txt`, `critic_eval.txt`
- [ ] `app/services/quiz_service.py` — persists `quiz_attempts`
- [ ] `app/api/routes/chat.py` — `POST /chat/quiz` handler only (coordinate, see R3 note)
- [ ] Wire into `graph.py`'s Quiz Generator/Critic nodes + conditional retry edge
      (coordinate merge with R5)

**Depends on**: R0, R2, R5 (skeleton). **Shares `chat.py` with R3, R6, R11.**

### R6 — Scoped Free-Question Mode
- [ ] `app/api/routes/chat.py` — `POST /chat/ask` handler only (coordinate, see R3 note)
- [ ] Extend `rag_service.py` with a topic-scope filter parameter (default: current topic;
      explicit `expand=true` → whole corpus) — **additive parameter, do not restructure
      the function signature R5 already shipped**

**Depends on**: R5. **Shares `chat.py` and `rag_service.py`** — land after R5's skeleton,
coordinate the `rag_service.py` diff with whoever touches it last before merge.

---

## Phase 4 — Cross-Cutting Concerns (parallel, mostly additive)

These wrap or gate Phase 3 code but live in their own files — safe to build in parallel
with Phase 3, merge order only matters for the final integration point noted per task.

### R10 — Generation Cache
- [ ] `app/models/generation_cache.py` + migration (schema fixed in ARCHITECTURE.md §6.5
      — do not redesign, implement as specified)
- [ ] `app/services/generation_cache_service.py` — `get_or_generate(topic_id, mode, level,
      prompt_version, generate_fn)`: `COUNT(*) < N=3` → call `generate_fn`, insert row;
      else → random `SELECT`
- [ ] Unit tests against a fake `generate_fn` (no LLM needed)
- [ ] **Integration task (do last, touches R3/R4 files)**: swap direct LLM calls in
      `mode_agent.py` and `quiz_agent.py` to go through
      `generation_cache_service.get_or_generate(...)` — small, additive diff once R3/R4
      are merged

**Depends on**: R0. Build the service + tests independently; only the final integration
step needs R3/R4 merged first.

### R11 — Domain-Scoping Guard
- [ ] `app/services/domain_guard_service.py` — `is_in_domain()` per ARCHITECTURE.md §9
- [ ] Unit tests with fake embedding provider (in-domain passes, out-of-domain
      short-circuits)
- [ ] **Integration task (touches `chat.py`)**: add as a guard clause at the top of the
      free-text handler (`POST /chat/ask` from R6) — one `if` statement, not a
      restructure

**Depends on**: R1 (corpus embeddings must exist to check against).

### R12 — Security & Secrets Hardening
- [ ] `app/core/security.py` — `X-API-Key` FastAPI dependency
- [ ] Apply dependency to all routes except `/health` (small edit per route file —
      coordinate timing so it lands after route files exist, i.e. after Phase 3)
- [ ] slowapi rate limiting, keyed per API key, matched to Gemini free-tier limits
- [ ] CORS config (env-driven origin, never `*`)
- [ ] `.gitignore` audit — confirm `.env`, `*.pem`, `*.key` excluded; `data/seed/` explicitly
      allowed

**Depends on**: R0. Apply-to-all-routes step should land after Phase 3 routes exist, or be
re-run as new routes are added.

---

## Phase 5 — Recommendation (sequential after Phase 3/4; Interview Mode deferred to v2)

### R8 — Recommendation Engine
- [ ] `app/services/recommendation_service.py` — `suggest_related_topic()` exactly as
      specified in ARCHITECTURE.md §7
- [ ] Similarity search is a live pgvector query (no precompute — must stay correct as
      user-submitted topics arrive at runtime, see ARCHITECTURE.md §8 item 2); filter
      candidates to `status='ready'`
- [ ] Wire trigger 1 (repeat-click completed topic) into `app/api/routes/topics.py`
      (coordinate with R2 owner)
- [ ] Wire trigger 2 (quiz resolved) into `graph.py`'s Recommendation step (coordinate
      with R5 owner)
- [ ] Wire trigger 3 (empty-state chat entry) into `chat.py` or a new `app/api/routes/chat.py`
      entry handler

**Depends on**: R2, R7.

### R9 — Interview-Prep Mode — **DEFERRED TO V2, skip for v1 build**
Not scheduled in any v1 phase. Task list kept here only so v2 can pick it up directly:
- [ ] `app/schemas/session_profile.py` — `domain`, `language`, `level` (transient, request
      params only, no new table)
- [ ] `app/prompts/quiz_interview.txt`
- [ ] Add `style="interview_question"` parameter to `quiz_agent.py` (additive — coordinate
      with R4 owner, small diff)
- [ ] Add profile-embedding anchor option to `recommendation_service.py` (additive —
      coordinate with R8 owner)
- [ ] New route or query param on existing chat/topics routes to accept the session profile

**Depends on**: R4, R8 (both must be merged — this is a thin parameterization, not new
logic).

---

## Phase 6 — Testing & CI (ongoing, not batched at the end)

### R13 — Testing & CI
- [ ] `app/core/fakes.py` — `FakeLLMProvider`, `FakeEmbeddingProvider` (build in Phase 0,
      right after the Protocol interfaces exist)
- [ ] `.github/workflows/ci.yml` — ruff + pytest on push (build in Phase 0)
- [ ] Per-requirement test files ship **with** that requirement's PR, not batched:
  - `tests/test_ingestion.py` (with R1) — cache hit/miss, `pipeline_version` invalidation, chunking
  - `tests/test_topics.py` (with R2)
  - `tests/test_url_submission.py` (with R1.5) — duplicate-URL cache hit, failure path,
    title fallback
  - `tests/test_progress.py` (with R7)
  - `tests/test_graph.py` (with R5) — Quiz Critic conditional edge, `retry_count` stop,
    LLM-failure fallback
  - `tests/test_mode_agent.py` (with R3)
  - `tests/test_quiz.py` (with R4)
  - `tests/test_rag_service.py` (with R6) — scope filter
  - `tests/test_generation_cache.py` (with R10)
  - `tests/test_domain_guard.py` (with R11)
  - `tests/test_recommendation.py` (with R8) — exclusion filter logic
  - `tests/test_interview_mode.py` (with R9)
  - `tests/test_routes_smoke.py` (`TestClient`, schema validation, error codes — grows
    incrementally as routes land)

**Depends on**: R0 (fakes need the Protocol interfaces). Not a separate phase in practice —
listed last only to name the CI job; each checkbox above is actually done inside its
sibling requirement's PR.

---

## Phase 7 — Secondary / Optional (schedule independently, does not block v1 demo)

### R14 — Email Re-engagement Hook (F9)
- [ ] `app/services/email_service.py`
- [ ] `scripts/send_digest.py` — scheduled entry point
- [ ] Trigger endpoint if needed (`app/api/routes/email.py`)

**Depends on**: R2, R4. Explicitly out of the critical path — PRD.md frames this as
secondary; do not let it block Phase 3-5 merges.

---

## Suggested execution order for a solo/small-team build

1. **Phase 0** solo, sequential (foundation must be right before anything else starts).
2. **Phase 1**: R1 and R7 in parallel (different files, R7 only needs R2's *shape* agreed,
   not merged — stub the FK).
3. **Phase 2**: R2 alone — short, sequential gate. R1.5 (URL submission) any time after
   R2 lands, in parallel with Phase 3 (no shared files with R3-R6).
4. **Phase 3**: R5 skeleton first (even a stub graph unblocks R3/R4/R6 to build against
   `TopicState`), then R3/R4/R6 in parallel with the `chat.py`/`rag_service.py`
   coordination notes above.
5. **Phase 4**: R10/R11/R12 in parallel with Phase 3 — they're additive and mostly
   independent; only their "integration task" checkboxes wait on Phase 3 merges.
6. **Phase 5**: R8 — needs Phase 3 fully merged. (R9 deferred to v2, skip.)
7. **Phase 6**: continuous, not a separate step — each PR carries its own tests.
8. **Phase 7**: anytime, does not gate the demo.

**v1 scope excludes R9 (Interview-Prep Mode)** — deferred to v2, see PRD.md §6.
