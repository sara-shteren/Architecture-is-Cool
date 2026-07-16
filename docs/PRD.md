# PRD — Architecture-is-Cool

## 1. Goal

A RAG-powered chat that turns dense, long-form software architecture and design-pattern
material into a fast, topic-based learning experience — with real understanding checked,
not just content skimmed.

The user picks a topic (e.g. "Monolith", "Strategy Pattern", "CQRS"). The assistant
answers grounded in a curated knowledge corpus, offers three lightweight ways to make the
concept click (joke, analogy, leveled code), and — critically — quizzes the user and
evaluates their answer **against the source material itself**, not general LLM judgment.
Wrong answers get explained with a citation back to the corpus; right answers can surface
related material the user hasn't covered yet.

**Explicit contrast with "ChatGPT + uploaded PDF":** that pattern answers whatever you ask
about the one document you happened to upload, with no memory across sessions and no way
to know what you actually understood. This system retrieves across an accumulating corpus,
tracks what's been learned, and grounds every evaluation in a specific source chunk — the
guided, checked-understanding loop is the product, not the summarization.

## 2. Domain & Data

- Seed corpus: **v1 target is ~6-10 public, licensing-clean articles/guides** (reduced from
  an original 15-25 to cut curation/ingestion time and shrink the surface area for
  format/chunking edge cases before the first demo). Matches the button-driven topic list
  (e.g. ~6 design patterns). Beyond the seed, the corpus **grows organically at runtime**
  via user-submitted URLs (F1.5) — same ingestion pipeline, same cache mechanism
  (ARCHITECTURE.md §6.5/§6.6) — so a larger curated set (15-25) is a v2+ nice-to-have, not
  a v1 blocker.
- No proprietary or paywalled content in the demo corpus — this is a portfolio piece, and
  source legality must be unambiguous.
- Format choice (HTML now vs. PDF later): only affects the `extract_text()` step of
  ingestion (see ARCHITECTURE.md §6). Retrieval quality (chunking, embeddings, similarity
  search) is independent of source format — adding PDF ingestion later is a new extraction
  branch behind the existing interface, not a redesign.

## 3. Target User & Scenarios

**Target user**: an experienced developer (mid-to-senior) who wants to genuinely absorb a
specific body of knowledge (a design-patterns book's worth of material, an
architecture-style guide, an AWS/GCP architecture doc set) but doesn't have time to read
linearly, and — more importantly — has no way to check whether a five-minute skim actually
produced real understanding.

**Design note carried over from discovery**: a rigid, fixed daily reading order does not
fit how experienced developers actually learn — they encounter a concept when they need
it, not on a schedule. The product is therefore **topic-selection-driven**: the user picks
what to explore next (or accepts a suggestion), rather than following a mandated sequence.
The three generation modes (joke/analogy/code) are a light, optional way to make a concept
land quickly; they are not the core value. The quiz/critic loop is the core value.

**Key scenarios:**
1. User picks a topic they don't know yet → gets an explanation (joke/analogy/code, their
   choice) → gets quizzed → answer is evaluated against source, with a clear explanation
   of what was right or wrong and a citation.
2. User re-selects a topic they already completed → system tells them it's already done
   and offers either a refresher or a related, not-yet-covered topic.
3. User opens the chat with no topic in mind → system suggests a topic related to what
   they've already completed but haven't gone deeper on.
4. User asks a free-form question mid-topic ("what's the difference between this and X?")
   → answered via retrieval scoped to the current topic, not the whole corpus.

## 4. Functional Requirements

| # | Capability | Description |
|---|---|---|
| F1 | Topic selection & retrieval | User selects a topic; system retrieves the grounding chunk(s) for it (deterministic, topic → source mapping) |
| F1.5 | User-submitted URL topic | User pastes a URL of an article they want to learn from; the system ingests it live (~10-30s with a visible loading state, same fetch→extract→chunk→embed pipeline as the seed corpus), auto-derives a topic title from the content, and the new topic gets the full F2-F5 loop — indistinguishable from a curated topic once ready. **Additive** to the curated topic-button list, not a replacement. Happy path only in v1 (see §6). |
| F2 | Three generation modes | Joke/story, analogy, and leveled code (junior/senior/architect) — each grounded in the same retrieved chunk(s), for consistency across modes |
| F3 | Quiz generation | System generates a question grounded in the current topic's retrieved content |
| F4 | Grounded answer evaluation (Critic) | User's answer is evaluated against the source chunk, not general LLM knowledge; explains what was correct/incorrect and why |
| F5 | Cross-corpus reference surfacing | When useful, evaluation surfaces related chunks from elsewhere in the corpus (multi-hop retrieval), not just the current topic |
| F6 | Scoped free-question mode | User can ask free-form questions during a topic session; retrieval is filtered to the current topic (or explicitly widened on request) |
| F7 | Progress tracking | Plain state tracking of topic completion status and mastery — not an RAG/embeddings concern (see ARCHITECTURE.md §6.5) |
| F8 | Related-topic recommendation | Suggests a next topic via similarity search over topic embeddings, filtered to topics not yet completed; triggered on: repeat-click of a completed topic, topic completion, or empty-state chat entry |
| F9 | Email re-engagement hook | Optional daily/periodic email: a short teaser + one quiz question tied to a recently retrieved chunk, linking back into the chat. Secondary feature, not the primary product surface. |

**F10 (Interview-prep mode) moved to v2** — see §6 Out of Scope. Design kept in
ARCHITECTURE.md §7.5 as a documented extension point (thin parameterization of F3-F5/F8,
no new tables/agents needed when built), but not implemented or scheduled for v1.

## 5. Non-Functional Requirements

- **Cloud-agnostic**: local Docker run architecturally identical to a future cloud
  deployment (12-factor config).
- **Provider-agnostic**: LLM and embeddings behind interfaces — swapping providers is a
  config change, not a rewrite.
- **Groundedness over speed**: the Critic evaluation loop may add latency; this is an
  intentional product trade-off (a wrong-but-confident evaluation is worse than a slower,
  correct one).
- **Demo-scale by lazy caching, not pre-generation**: explanations and quiz questions are
  generated live on first request per (topic, mode, level) and cached (a few rotating
  variants, not one frozen answer) — this applies uniformly to curated button topics and
  user-submitted URL topics (F1.5), since the cache is keyed by topic id either way.
  Later users get the same fast/cheap response without
  the experience ever looking like canned, non-LLM content. Only the Critic evaluation is
  always fully live (per-user, uncacheable). This makes a ~100-user demo a sub-$1 cost
  question (paid tier + budget cap for demo day), not a rate-limit problem (see
  ARCHITECTURE.md §8).
- **Small, fast-to-ship scope** — this is a portfolio/demo project, not a production
  learning platform. Prefer a working end-to-end slice over broad feature coverage.
- Short, readable code; no speculative abstraction for hypothetical future needs.

## 6. Out of Scope (v1)

- Autonomous crawling/scraping (following links, indexing whole sites) — v1 ingests
  exactly two ways: the curated seed corpus, and a single user-submitted URL at a time
  (F1.5). No crawler.
- Moderation/abuse-hardening for open URL input (F1.5) — content filtering, size/type
  validation beyond basics, and per-user submission quotas beyond the existing global
  API rate limit are deferred to v2. v1 ships the happy path, guarded only by the
  existing API-key auth + rate limiting (ARCHITECTURE.md §9).
- Real PDF ingestion — documented as a clean v2 extension point (new `extract_text()`
  branch), not implemented in v1.
- Multi-user auth/accounts beyond a single static API key (single-tenant demo, same as
  the reference project's security model).
- Gamification beyond the completion/mastery tracking described in F7 (no leaderboards,
  streak badges, etc.).
- Full daily-digest email as a primary surface — email is a lightweight, secondary
  re-engagement hook only (F9).
- **Interview-prep mode (F10)** — moved to v2. Design documented in ARCHITECTURE.md §7.5
  as a clean extension point (parameterizes the existing quiz/recommendation mechanism,
  no new pipeline); not built in v1 to keep scope to the core learning loop.

## 7. Success Criteria (demo-readiness, not business KPIs)

- End-to-end demoable flow: pick a topic → try all three generation modes → enter quiz
  mode → answer incorrectly → see a grounded explanation with a source citation → see a
  related-topic suggestion surface.
- Re-selecting a completed topic visibly offers "refresh vs. related new topic" — provable
  live, not just in code.
- Pasting a URL of a real article (F1.5) produces a working new topic within ~30s — with a
  visible loading state, an auto-derived title, and the full mode/quiz loop working on it.
- A clean-machine run (`git clone` → `docker compose up` → working UI) succeeds without
  manual fixes.
