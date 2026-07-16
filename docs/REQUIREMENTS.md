# REQUIREMENTS — Architecture-is-Cool

Each requirement lists its **owned files/modules** — the paths a developer working on that
requirement will create or edit. Requirements with no overlapping owned files can be built
in parallel without merge conflicts; see TASKS.md for the concrete task breakdown and
dependency order.

## R0 — Foundation (prerequisite for everything else)

Project skeleton, config, DB connection, core interfaces, Docker. Not a product
requirement — the substrate every other requirement is built on.

- **Owns**: `app/main.py`, `app/core/`, `app/models/` (base + migrations setup),
  `docker-compose.yml`, `Dockerfile`, `.env.example`, `alembic/`, `app/vector/` (connection
  layer only, no queries yet), provider interfaces (`LLMProvider`, `EmbeddingProvider`,
  `StorageProvider` — ARCHITECTURE.md §4).
- **Depends on**: nothing.
- **Blocks**: all other requirements (they need config, DB session, provider interfaces to
  exist).

## R1 — Ingestion Pipeline (PRD §2, ARCHITECTURE §6 ingestion, §6.5 `source_files`/`chunks`)

Fetch → extract → chunk → embed → store the seed corpus.

- **Owns**: `app/models/source_file.py`, `app/models/chunk.py`, `app/services/ingestion_service.py`,
  `scripts/seed_embeddings.py`, `data/seed/`, `app/utils/chunking.py`, `app/utils/html_to_text.py`.
- **Depends on**: R0.
- **Blocks**: R2 (topics link to `source_files`/`chunks`), R3/R4/R6 (retrieval needs chunks
  to exist).

## R2 — Topic Domain & F1 (Topic selection & retrieval)

Deterministic topic → source chunk mapping; topic CRUD/listing.

- **Owns**: `app/models/topic.py`, `app/services/topic_service.py`, `app/api/routes/topics.py`,
  `app/schemas/topic.py`. Model/schema include `status` and `source` columns from day one
  (ARCHITECTURE.md §6.5) so R1.5 doesn't need a schema change later.
- **Depends on**: R1 (needs `source_files`/`chunks` tables to reference).
- **Blocks**: R3, R4, R5, R6, R8 (all read topic data), R1.5.

## R1.5 — User-Submitted URL Ingestion (F1.5, ARCHITECTURE §6.6)

User pastes a URL → live ingestion (~10-30s, polling-based loading state) → auto-titled
topic with the full mode/quiz loop. Reuses R1's `ingestion_service` unchanged; once a
topic row is `status='ready'` it's indistinguishable from a curated topic to R3-R8.

- **Owns**: `app/utils/title_extraction.py` (new), `create_topic_from_url()` in
  `app/services/topic_service.py` (additive function — coordinate with R2 owner),
  `POST /topics/from-url` handler in `app/api/routes/topics.py` (additive handler —
  coordinate with R2/R8 owners).
- **Depends on**: R1 (pipeline reused as-is, no changes to R1's files), R2 (topic
  model/routes exist, incl. `status`/`source` columns).
- **Blocks**: none. Does not touch R3/R4/R5/R6 files at all.

## R3 — Mode Generation & F2 (joke / analogy / leveled code)

Generate the three explanation modes grounded in a topic's retrieved chunk(s).

- **Owns**: `app/agents/mode_agent.py`, `app/prompts/joke.txt`, `app/prompts/analogy.txt`,
  `app/prompts/code_{junior,senior,architect}.txt`, `app/api/routes/chat.py` (mode endpoints
  only — see R6 for shared ownership note).
- **Depends on**: R0, R2.
- **Blocks**: none (leaf requirement).
- **Shares `app/api/routes/chat.py` with R6** — coordinate route names in advance (e.g.
  `POST /chat/mode`, `POST /chat/quiz`) to avoid editing the same route function.

## R4 — Quiz Generation & Critic Loop (F3, F4, F5)

The core product loop: generate a grounded question, evaluate the user's free-text answer
against source, multi-hop reference surfacing on resolution.

- **Owns**: `app/agents/quiz_agent.py`, `app/agents/quiz_critic_agent.py`,
  `app/prompts/quiz_standard.txt`, `app/prompts/critic_eval.txt`, `app/models/quiz_attempt.py`,
  `app/services/quiz_service.py`.
- **Depends on**: R0, R2, R5 (Retriever agent must exist for grounding + multi-hop lookup).
- **Blocks**: R9 (Interview-Prep reuses this loop), F9 email hook (out of current scope
  per TASKS.md phasing).

## R5 — Retriever Agent & Graph Orchestration (ARCHITECTURE §5)

LangGraph state machine wiring Retriever → Mode Router → Quiz Generator → Quiz Critic →
Recommendation step. Owns the shared `TopicState` and free-text similarity retrieval (F6).

- **Owns**: `app/agents/graph.py`, `app/agents/retriever_agent.py`, `app/agents/state.py`
  (`TopicState`), `app/services/rag_service.py`.
- **Depends on**: R0, R1, R2.
- **Blocks**: R3, R4 (both are nodes invoked by the graph), R6 (free-question retrieval).

**Note**: R3 and R4 own their *own* agent files (`mode_agent.py`, `quiz_agent.py`,
`quiz_critic_agent.py`) — R5 only owns graph wiring (`graph.py`) and the `Retriever` node
itself. This split lets R3/R4/R5 be developed in parallel against an agreed `TopicState`
shape (frozen at the start of Phase 2, see TASKS.md) with each node stubbed until wired.

## R6 — Scoped Free-Question Mode (F6)

User asks free-form questions mid-topic; retrieval filtered to current topic by default,
explicit "expand" widens to full corpus.

- **Owns**: `app/api/routes/chat.py` (free-question endpoint — coordinate with R3, see
  above), `app/services/rag_service.py` (shared with R5 — R6 only adds the
  topic-scope-filter parameter, does not restructure the function).
- **Depends on**: R5.
- **Blocks**: none.

## R7 — Progress Tracking (F7)

Plain SQL completion/mastery state — explicitly not embeddings-based.

- **Owns**: `app/models/user_progress.py`, `app/services/progress_service.py`,
  `app/api/routes/progress.py`, `app/schemas/progress.py`.
- **Depends on**: R0, R2.
- **Blocks**: R8 (recommendation filters by completion state).

## R8 — Recommendation Engine (F8)

`suggest_related_topic()` — similarity search over topic embeddings, filtered to
not-yet-completed topics, called from three UX entry points.

- **Owns**: `app/services/recommendation_service.py`, wiring into
  `app/api/routes/topics.py` (repeat-click trigger — coordinate with R2) and
  `app/agents/graph.py` (post-quiz trigger — coordinate with R5).
- **Depends on**: R2, R7.
- **Blocks**: R9 (reuses this mechanism with a different anchor).

## R9 — Interview-Prep Mode (F10, ARCHITECTURE §7.5) — **DEFERRED TO V2, not in v1 scope**

Thin parameterization of the existing graph + recommendation mechanism — no new tables, no
new agents. Kept documented (not deleted) so the v1 `TopicState`/graph/schema design
doesn't accidentally preclude adding it later. Do not schedule against v1 phases.

- **Owns** (when built): `app/prompts/quiz_interview.txt`, `app/schemas/session_profile.py`,
  a `style="interview_question"` parameter threaded through `quiz_agent.py` (coordinate
  with R4 owner — additive parameter, not a rewrite) and an anchor-source parameter in
  `recommendation_service.py` (coordinate with R8 owner).
- **Depends on**: R4, R8.
- **Blocks**: none.

## R10 — Generation Cache (NFR "Demo-scale by lazy caching", ARCHITECTURE §6.5/§8)

Lazy, capped, rotating-variant cache for mode/quiz generations. Cross-cutting: wraps calls
made by R3 and R4, but lives in its own module so it can be built and tested independently
against a fake generator function.

- **Owns**: `app/models/generation_cache.py`, `app/services/generation_cache_service.py`
  (the `COUNT(*) < N → generate, else → random SELECT` logic from ARCHITECTURE.md §8).
- **Depends on**: R0.
- **Blocks**: nothing structurally, but R3/R4 should call through
  `generation_cache_service` rather than the LLM directly once both exist — coordinate
  merge order (see TASKS.md Phase 3).

## R11 — Domain-Scoping Guard (ARCHITECTURE §9)

Embed-and-check guard clause before any free-text message reaches the LLM.

- **Owns**: `app/services/domain_guard_service.py`, wiring as a dependency/guard clause in
  `app/api/routes/chat.py` (additive check at the top of the free-question handler —
  coordinate with R3/R6, do not restructure their handlers).
- **Depends on**: R1 (needs corpus embeddings to check against).
- **Blocks**: none.

## R12 — Security & Secrets Hardening (ARCHITECTURE §9)

API key auth, rate limiting, CORS.

- **Owns**: `app/core/security.py`, `app/main.py` (middleware registration only —
  coordinate with R0), `.gitignore`.
- **Depends on**: R0.
- **Blocks**: none (but should land before any public deploy).

## R13 — Testing & CI (ARCHITECTURE §10)

Fake providers, per-requirement test suites, CI job.

- **Owns**: `tests/` (subdivided by requirement — `tests/test_ingestion.py`,
  `tests/test_recommendation.py`, etc., each owned by the same developer as the
  corresponding R-number), `app/**/fakes.py` (`FakeLLMProvider`, `FakeEmbeddingProvider`),
  `.github/workflows/ci.yml`.
- **Depends on**: R0 (fakes depend on the Protocol interfaces).
- **Blocks**: nothing structurally — each requirement's tests should ship with that
  requirement's PR, not be batched separately. `tests/` is listed as its own requirement
  only to name the CI job and the fakes; individual test files are owned by whoever owns
  the corresponding R-number's code.

## R14 — Email Re-engagement Hook (F9)

Secondary, optional feature — deliberately last / independently schedulable.

- **Owns**: `app/services/email_service.py`, `app/api/routes/` (new `email.py` if a
  trigger endpoint is needed), a scheduled job entry point (`scripts/send_digest.py`).
- **Depends on**: R2, R4 (needs a recent quiz question to reference).
- **Blocks**: none. Can be built anytime after its dependencies land, including after v1
  demo — see PRD.md §6 Out of Scope framing (secondary surface, not core).

---

## Parallelization summary

| Can run fully in parallel (no shared files) | Requires coordination on shared files |
|---|---|
| R1, R7 (after R0) | R3 & R6 (`chat.py`) |
| R1.5 vs. R3/R4/R5/R6 (after R1+R2) | R1.5 & R2/R8 (`topics.py`, `topic_service.py`) |
| R3 & R4 (once R5's `TopicState` is frozen) | R3/R4 & R10 (cache wrapping, merge order) |
| R11 & R12 | R2/R5 & R8 (trigger wiring in existing files) |
| R9 (once R4+R8 land) | R4 & R9 (`quiz_agent.py` parameter) |
| R14 (anytime after R2+R4) | R0 & R12 (`main.py` middleware) |

Rule of thumb: a requirement that only **adds new files** is safe to parallelize
unconditionally. A requirement that **edits a file another requirement also edits** needs
either sequencing (one lands first) or an upfront interface agreement (e.g. `TopicState`,
route names) frozen before both start.
