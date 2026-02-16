# Boid: The Autonomous Software Agency

## Vision

Boid is an **LLM-powered agency** that replaces an entire company. Someone with an idea (and budget) can pour their vision into Boid, and it will:

*   Ask the right questions to refine the idea
*   Do all the work — design, development, content, operations
*   Never sleep, scale instantly by spinning up more agents

**The Dream:** As LLMs become smarter and cheaper, Boid becomes affordable for anyone with a normal job to launch their side-business and make their dreams real.

---

## Starting Point: Minions

Before building the full agency, we start with **Minions** — a swarm of specialized LLM agents that continuously improve your codebase.

**Target Users:** Dev teams with mature, revenue-generating products. You're past the "rewrite everything" phase and want to focus on stability, security, performance, and maintainability.

**The Problem:** Your backlog has 200 "someday" tickets — technical debt accumulates while the team ships features.

**The Solution:** Connect your repo and receive PRs with improvements. Minions perform a **Historical Code Forensic** analysis using a custom Rust-native engine to identify exactly where code quality is decaying.

### The Boid Engine (Rust-Native)

Instead of relying on heavy compilers or fragile scripting, Boid uses a high-performance Rust core to build a "Forensic DNA" of your repository:

1.  **Build-Less Analysis (Tree-sitter):** We don't need `npm install` or a working build system. We parse the Concrete Syntax Tree (CST) directly using Tree-sitter to extract metrics (Complexity, Nesting Depth, NOM) and resolve internal dependencies (Fan-In/Out) with 100% fidelity. The analyzer outputs data in a **columnar (SoA) layout** — parallel arrays of locations and metrics — matching ClickHouse's columnar storage model for efficient in-memory analytics and future zero-copy insertion.
2.  **Incremental Replay:** The engine walks the Git history using `git2-rs`, performing incremental parsing to calculate the "Entropy" of every file over time. This allows us to track metrics at commit-level resolution.
3.  **Entity-Based Identity (CUID2):** We use CUID2s to track file "souls" across renames and moves. History is never lost, even if a file is refactored into a different directory.
4.  **Temporal Storage:** Metrics are streamed into a high-performance time-series stack (**VictoriaMetrics**) for sub-millisecond analytical queries.

### Minion Types

| Minion | Focus |
|--------|-------|
| **Security** | Vulnerabilities, input validation, auth issues, OWASP top 10 |
| **Performance** | Hot paths, memory leaks, N+1 queries, caching opportunities |
| **Maintainability** | Code smells, complexity, dead code, coupling |
| **Documentation** | Missing docs, outdated comments, API documentation |
| **Accessibility** | WCAG compliance, screen reader support, keyboard navigation |
| **Testing** | Coverage gaps, flaky tests, missing edge cases |

---

## The Consensus Mechanism

Unlike typical multi-agent systems that force agreement, Minions use **adversarial critique** — they stress-test each other's proposals rather than trying to reach consensus.

#### Phase 1: Independent Analysis
Each minion analyzes the codebase through its lens. No cross-talk, preventing anchoring bias.

#### Phase 2: Adversarial Critique
Minions receive all proposals. Their job is NOT to agree, but to attack flaws from their domain's perspective (e.g., Performance Minion critiquing a Security fix for adding too much latency).

#### Phase 3: Orchestrator Decision
The orchestrator analyzes the debate. Proposals that survive critique or are successfully refined get merged. Dissent is logged for human review.

---

## Roadmap

### Phase 0: Minions (Current Focus)
*   **Rust Analysis Core:** Build-less Tree-sitter engine for high-fidelity code forensics.
*   **Adversarial Critique:** Specialized agents debating refactors via OpenRouter.
*   **Time-Series Foundation:** Integration of VictoriaMetrics for historical tracking.

### Phase 1a: Smart PM — Tickets ("Jira")
*   Ingest view: dump thoughts → LLM PM extracts issues, tasks, priorities.
*   Automatic RFC generation from goals.
*   Triggers targeted Minion exploration based on user input.

### Phase 1b: Smart PM — Monitoring ("Boid Chat")
*   **Built-in Chat:** Holistic team communication replacing Slack.
*   **Autonomous Ops:** Detects production incidents via VictoriaLogs/Traces and deploys fixes.

### Phase 6: The Nest (Universal Reality Engine)
*   **Universal Materialization:** The Graph is the only reality. Code, Design, and Content are synchronized materializations of the same knowledge nodes.

---

## Tech Stack (100% Rust)

| Component | Technology |
|-----------|------------|
| **Backend / API** | **Rust (Axum)** |
| **Type Safety** | **rspc** (Rust to TS end-to-end types like tRPC) |
| **Agent Logic** | **Rig** (Rust Agent Framework via OpenRouter) |
| **Analysis** | **Tree-sitter** + **git2-rs** (columnar SoA output) |
| **Analytical DB** | **ClickHouse** (Scope metrics, columnar queries) |
| **Graph DB** | **Dgraph** (Structural relationships) |
| **KV / Raw Text** | **Garnet** (Immutable text storage) |
| **Time-Series** | **VictoriaMetrics Suite** (Metrics, Logs, Traces) |
| **Vector Store** | **Milvus** (Semantic search) |
| **Identity** | **CUID2** (Collision-resistant IDs) |

---

## The Pipeline

Boid processes raw text through a Multi-Pass Reasoning Pipeline:

1.  **Text Cleaning:** ftfy + context-aware contraction expansion.
2.  **Grammar & Spelling:** GECToR + GRECO.
3.  **Coreference Resolution:** xCoRe.
4.  **Sentence Splitting:** SaT (Segment Any Text).
5.  **Extraction (The Scribe):** Sentences → Atomic Claims.
6.  **Assumption (The Detective):** Inferring implicit context.
7.  **Synthesis (The Architect):** Constructing the Knowledge Graph in **Dgraph**.
8.  **Knowledge Storage:** Raw text stored in **Garnet**, linked via CUID2.

---

## Storage Architecture

### Dgraph (Graph Database)
Stores **structure only** — the "Soul" of the company. Nodes (Claims, Tasks, Topics) and Edges (requires, supersedes, authors).

### Garnet (KV Store)
Stores **immutable text content** and raw metadata. Keys are CUID2s referencing Dgraph nodes. Provides sub-millisecond lookups for raw ticket text and agent thoughts.

### VictoriaMetrics Suite
*   **VictoriaMetrics:** Temporal code health metrics (Complexity, NOM, Churn).
*   **VictoriaLogs:** High-cardinality agent execution logs tagged by commit/file.
*   **VictoriaTraces:** OTLP tracing of multi-agent refactor workflows.

### Milvus (Vector Store)
Stores embeddings for semantic search, topic clustering, and context retrieval.

---

## Strategic Outlook

### The Moat: The "Workflow Singularity"
Once a team experiences **Universal Materialization** (Change Button -> Code & Design & Docs update instantly), going back to manual tools (Jira + Git + Figma) feels archaic. Leaving Boid doesn't cost data; it costs **Labor**. To replace Boid, you have to hire humans to manually sync your tools.
