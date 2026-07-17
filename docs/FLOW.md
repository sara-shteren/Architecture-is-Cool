# System Flow Diagrams — Architecture-is-Cool

Companion to `TASKS.md` — each diagram maps to the task topics that implement it
(referenced as **T1-T13**). Diagrams are Mermaid; GitHub renders them inline.

---

## 1. Big Picture — User Journey (end to end)

What the user experiences, from entry to "next topic". Covers T3, T4, T6, T7, T9, T12.

```mermaid
flowchart TD
    Start([User opens chat]) --> HasTopic{Picked a topic?}

    HasTopic -- "Clicks 'Surprise me' button" --> Suggest["Suggest interesting topic<br/>(based on last completed)"]
    Suggest --> Pick
    HasTopic -- "Clicks topic button" --> Pick[Topic selected]
    HasTopic -- "Pastes article URL" --> Ingest["Live URL ingestion<br/>(~10-30s, loading state)"]
    Ingest -- "ready" --> Pick
    Ingest -- "failed" --> Err[Clear error message] --> HasTopic

    Pick --> Done{Already completed?}
    Done -- "Yes" --> Offer["Offer: refresh OR<br/>related new topic"] --> Mode
    Done -- "No" --> Mode

    Mode{Choose mode} -- "Joke" --> Explain[Grounded explanation]
    Mode -- "Analogy" --> Explain
    Mode -- "Code (jr/sr/architect)" --> Explain
    Mode -- "Quiz" --> Quiz

    Explain --> FreeQ{Free question?}
    FreeQ -- "Yes" --> Scoped["Answer scoped to current topic<br/>(or expand to whole corpus)"] --> Mode
    FreeQ -- "No" --> Mode

    Quiz[Quiz question grounded in source] --> Answer[User answers free-text]
    Answer --> Critic{"Critic verdict<br/>(vs. source chunk)"}
    Critic -- "Incorrect / unclear<br/>(retry_count guard)" --> ReExplain[Explanation + citation] --> Quiz
    Critic -- "Correct" --> Refs["Surface related refs<br/>(multi-hop, optional)"]
    Refs --> Complete[Mark topic completed + mastery score]
    Complete --> Next["Recommend next topic<br/>(similar, not yet learned)"]
    Next --> HasTopic
```

---

## 2. Request Path Through the Layers (Modular Monolith)

Every request follows this one-directional chain — no layer skips. Covers T1, T5, T12.

```mermaid
flowchart LR
    UI[Next.js<br/>Chat UI] --> Routes["api/routes<br/>HTTP only, no logic"]
    Routes --> Services["services<br/>rag / ingestion / topics /<br/>recommendation / cache / guard"]
    Routes --> Graph["agents (LangGraph)<br/>Retriever → Router →<br/>Quiz → Critic"]
    Graph --> Services
    Services --> Vector["vector / DB<br/>Postgres + pgvector"]
    Services --> Providers["LLMProvider / EmbeddingProvider<br/>(Gemini behind an interface)"]
```

Rule: `routes` never touch DB/LLM directly; `agents` never touch the DB directly.
Swapping Gemini for another provider = config change only.

---

## 3. Agent Graph — Core Loop (LangGraph)

The explicit state machine (`TopicState` shared between nodes). Covers T5, T6, T7, T9.

```mermaid
flowchart TD
    State([TopicState in]) --> Retriever["Retriever<br/>topic → chunks (deterministic)<br/>or scoped similarity search"]
    Retriever --> Router{Mode Router}
    Router -- "joke / analogy / code" --> ModeAgent["Mode Agent<br/>(via generation cache)"]
    ModeAgent --> Out1([Response + sources])
    Router -- "quiz" --> QuizGen["Quiz Generator<br/>(via generation cache)"]
    QuizGen --> CriticNode["Quiz Critic<br/>ALWAYS LIVE — never cached"]
    CriticNode -- "incorrect / unclear<br/>retry_count < max" --> ReExplain["Re-explain with citation"]
    ReExplain --> QuizGen
    CriticNode -- "resolved OR retries exhausted" --> Rec["Recommendation step<br/>suggest_related_topic()"]
    Rec --> Out2([Verdict + next-topic suggestion])

    LLMFail["LLM failure (429/5xx)"] -.->|"retry + backoff inside provider;<br/>final failure → clean 'system busy' state,<br/>never a stack trace"| CriticNode
```

**Critic rule (critical):** an answer that is plausibly right but not covered by the
source chunk gets verdict `unclear` with an honest "the source doesn't address this" —
**never** a false `incorrect`.

---

## 4. User-Submitted URL Ingestion (F1.5)

Same pipeline as the seed corpus, triggered over HTTP. Covers T2, T4.

```mermaid
flowchart TD
    Submit["POST /topics/from-url"] --> Create["Create topic row<br/>status='processing', source='user_submitted'<br/>→ return topic id immediately"]
    Create --> BG["Background task starts"]
    Create --> Poll["Frontend polls GET /topics/{id}<br/>shows loading state"]

    BG --> Fetch["Fetch URL<br/>(http/s only, ≤2MB, timeout)"]
    Fetch -- "error" --> Failed["status='failed' + short reason"]
    Fetch --> Hash{"content_hash +<br/>pipeline_version<br/>already cached?"}
    Hash -- "HIT (URL seen before)" --> Link["Link topic to existing<br/>source_file + chunks"] --> Ready
    Hash -- "MISS" --> Extract["html_to_text()"]
    Extract --> Chunk["chunk()<br/>heading-aware, no mid-concept cuts"]
    Chunk --> Embed["embed() → pgvector upsert"]
    Embed --> Title{"Title from<br/>&lt;title&gt; / &lt;h1&gt;?"}
    Title -- "Yes" --> Finalize
    Title -- "No" --> LLMTitle["One-line LLM summary"] --> Finalize
    Finalize["chunk_range = all chunks<br/>topic_embedding = centroid"] --> Ready["status='ready'"]

    Ready --> Same(["Topic now indistinguishable<br/>from a curated topic (T3 onward)"])
```

Invariant: a topic is never left in `processing` after the task ends — it's `ready` or
`failed`, nothing in between.

---

## 5. Generation Cache — Lazy, Capped, Rotating (cost control)

Applies to explanations and quiz questions. The Critic is **never** cached. Covers T10.

```mermaid
flowchart TD
    Req["Request: (topic, mode, level)"] --> Count{"Cached variants for<br/>(topic, mode, level,<br/>prompt_version)?"}
    Count -- "< 3" --> Gen["Call LLM (live generation)"]
    Gen --> Store["Insert new variant row"]
    Store --> Serve1([Serve fresh variant])
    Count -- "= 3" --> Rand["SELECT one variant at random<br/>— zero LLM calls"]
    Rand --> Serve2([Serve cached variant])

    Note["prompt_version bump →<br/>fresh variant set (natural invalidation)"] -.-> Count
```

Effect: LLM cost is bounded by `topics × modes × levels × 3` regardless of user count —
a ~100-user demo costs under $1 (only live Critic calls scale with users).

---

## 6. Free-Text Message Guard (security / quota protection)

Runs before any free-text input reaches the LLM. Button clicks skip it. Covers T11.

```mermaid
flowchart TD
    Msg["Free-text message<br/>(question or quiz answer)"] --> Auth{"X-API-Key valid?<br/>+ per-key rate limit OK?"}
    Auth -- "No" --> Deny([401 / 429])
    Auth -- "Yes" --> Embed["Embed message<br/>(one embedding call)"]
    Embed --> Sim{"Max similarity to topic corpus<br/>≥ threshold?<br/>(threshold calibrated empirically,<br/>lives in env config)"}
    Sim -- "No (out of domain)" --> Polite(["'This assistant only answers<br/>architecture & design-patterns questions'<br/>— LLM never called"])
    Sim -- "Yes" --> Pass([Proceed to agent graph])
```

---

## 7. Data Model at a Glance

Which table answers which question. Covers T1, T2, T3, T9, T10.

```mermaid
flowchart LR
    subgraph CACHE["Cache layer (keyed by content)"]
        SF["source_files<br/>content_hash + pipeline_version"]
        GC["generation_cache<br/>(topic, mode, level, prompt_version)<br/>up to 3 variants"]
    end
    subgraph DOMAIN["Domain layer"]
        T["topics<br/>title, chunk_range, embedding,<br/>status, source"]
    end
    subgraph RAG["RAG unit"]
        C["chunks<br/>text + embedding<br/>(no topic FK — chunk_range is<br/>the single source of truth)"]
    end
    subgraph STATE["Plain SQL state — NO embeddings"]
        UP["user_progress<br/>completed? mastery?"]
        QA["quiz_attempts<br/>audit of every attempt"]
    end

    T --> SF
    C --> SF
    T -.->|chunk_range| C
    GC --> T
    UP --> T
    QA --> T
```

**The separation to protect:** `user_progress` answers a lookup question ("done?
yes/no") — recommendations are the *only* place embedding similarity is used.

---

## Diagram → Task Topic Map

| Diagram | Implements | TASKS.md topics |
|---|---|---|
| 1. User journey | The whole product experience | T3, T4, T6, T7, T8, T9, T12 |
| 2. Layer path | Module boundaries | T1, T12 |
| 3. Agent graph | Core loop + failure handling | T5, T6, T7, T9 |
| 4. URL ingestion | F1.5 end to end | T2, T4 |
| 5. Generation cache | Cost control | T10 |
| 6. Free-text guard | Security + quota | T11 |
| 7. Data model | All persistence decisions | T1, T2, T3, T9, T10 |
