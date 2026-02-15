  1. Code Property Graph (CPG)

  Start here — this is the most concrete concept and the foundation everything else builds on.

  The Problem

  When you analyze code for bugs or vulnerabilities, there are three classic representations:

  - AST (Abstract Syntax Tree) — the structure of your code. You know this from tools like Babel, tree-sitter, or zig ast-check. It tells you what the code looks like
  syntactically.
  - CFG (Control Flow Graph) — the execution paths. "If this condition is true, go here; otherwise go there." Think of it like a flowchart.
  - PDG (Program Dependence Graph) — data and control dependencies. "Variable y depends on x because x was used to compute y." And "this statement only runs if that condition
   was true."

  Each one captures something important, but none alone is enough. A vulnerability like a buffer overflow involves the syntax (how the buffer is declared), the control flow
  (which path reaches the dangerous write), and the data flow (where the unchecked size value came from) — all at once.

  The Solution

  Fabian Yamaguchi (2014) said: merge all three into one graph. That's a CPG.

  The trick is simple — statements and predicates appear as nodes in all three representations. So you overlay them:

  Source Code
      |
      v
     AST  ----+
     CFG  ----+--->  Single unified graph (CPG)
     PDG  ----+

  Same nodes, different edge types layered on top.

  Concrete Example

  void foo() {
      int x = source();      // node A
      if (x < MAX) {         // node B
          int y = 2 * x;     // node C
          sink(y);           // node D
      }
  }

  Node B (x < MAX) simultaneously has:
  - AST edges down to its child nodes (x, MAX, the < operator)
  - CFG edges going to node C (true branch) and to the function return (false branch)
  - CDG edges (control dependence) to nodes C and D — they only execute because B was true
  - Reaching-def edges (data dependence) from node A — that's where x was defined

  So you can ask questions like: "Can data from source() reach sink()?" by traversing data-flow edges. Or "Is there a bounds check on the path to this memcpy?" by combining
  control-flow and data-flow traversals.

  Real Tools

  - Joern (open source) — the original CPG tool. You query it with a Scala DSL:
  // Find if tainted data reaches a dangerous function
  def source = cpg.call("ntohl")       // network byte order = untrusted input
  def sink = cpg.call("memcpy").argument(3)  // the size argument
  sink.reachableByFlows(source).p       // trace the data flow
  - CodeQL (GitHub) — similar concept but uses a relational database model instead of a property graph
  - Semgrep — pattern-matching on ASTs (simpler, less powerful)

  Think of CPG as the data structure — the next concepts are about learning patterns from it automatically using neural networks, instead of writing queries by hand.

  ---
  2. Gated Graph Neural Networks (GGNN)

  The Problem

  Writing CPG queries by hand is like writing regex — it works for known patterns, but it doesn't generalize. What if you could train a model to learn vulnerability patterns
  from examples?

  To do that, you need a neural network that can process graph-structured data. That's what GNNs are — but vanilla GNNs have a critical flaw.

  GNNs in 30 Seconds

  Think of it like a gossip network. Every node in the graph has some initial knowledge (its features — for code, this might be "I'm a function call to memcpy"). Then, in
  each round:

  3. Every node sends its current state to all its neighbors (message passing)
  4. Every node combines what it received into a new state (aggregation)
  5. Repeat for T rounds

  After T rounds, each node's state encodes info from up to T hops away in the graph.

  The Flaw: Over-Smoothing

  Here's the problem — it's like a game of telephone crossed with averaging. After enough rounds, every node's state converges to roughly the same value. All distinguishing
  information gets washed out. This is called over-smoothing.

  In code terms: after too many rounds, the node for malloc and the node for free end up with nearly identical representations. That's useless for detecting use-after-free
  bugs.

  The Fix: Gating (the "G" in GGNN)

  Li et al. (2015) borrowed the GRU (Gated Recurrent Unit) mechanism from sequence models (like the ones behind early machine translation). If you've seen LSTMs, GRUs are the
   simpler cousin.

  The key idea: instead of blindly overwriting a node's state each round, give it two learned gates:

  Update gate (z): "How much of this incoming message should I actually absorb?"
  - z ≈ 0 → "I'm keeping my current state, thanks" (the node is confident/stable)
  - z ≈ 1 → "I'll adopt this new info" (the node needs updating)

  Reset gate (r): "Before computing my new candidate state, how much of my old state should I forget?"
  - r ≈ 0 → "Start fresh, ignore what I previously knew"
  - r ≈ 1 → "My old knowledge is relevant, factor it in"

  In pseudocode (think of it as per-node, per-round):

  fn update_node(node, messages_from_neighbors) {
      let aggregated = sum(messages_from_neighbors);  // combine neighbor info

      let z = sigmoid(W_z * aggregated + U_z * node.state);  // update gate
      let r = sigmoid(W_r * aggregated + U_r * node.state);  // reset gate

      let candidate = tanh(W * aggregated + U * (r * node.state));  // new proposal

      node.state = (1 - z) * node.state + z * candidate;  // blend old and new
  }

  The W_* and U_* matrices are learned during training. The network learns which nodes should be sticky (preserving their identity) and which should be permeable (passing
  information through).

  Why This Matters for Code

  Microsoft Research (Allamanis et al., 2018) applied GGNNs to program graphs — ASTs augmented with data-flow edges like LastRead, LastWrite, ComputedFrom. They could:

  - Predict variable names with 33% accuracy (way above baselines)
  - Detect variable misuse bugs with 85.5% accuracy — and actually found real bugs in open-source C# projects

  The Devign system (2019) ran GGNNs directly on Code Property Graphs for vulnerability detection.

  Limitations

  GGNNs process edges pairwise — each edge connects exactly two nodes. If a function call has 5 arguments, that's 5 separate edges. The relationship "these 5 things are all
  arguments to the same call" is implicit, not explicit. This is where hypergraphs come in.

  ---
  3. Heterogeneous Directed Hypergraph (HDHG)

  This is a data structure — a way to represent code that's richer than a regular graph. Three words, three concepts:

  "Hyper" — Edges Can Connect More Than 2 Nodes

  In a regular graph, an edge is always between exactly 2 nodes: A --edge--> B.

  A hyperedge can connect any number of nodes simultaneously.

  Why this matters for code: Consider this AST node:

  def foo(a, b, c):
      x = a + b
      y = b * c
      return x + y

  The body field of foo contains three statements. In a regular graph, you'd need three separate edges:
  foo --body--> x = a + b
  foo --body--> y = b * c
  foo --body--> return x + y

  These look like three independent relationships. But they're not — they're all part of the same body block. A hyperedge captures this as one relationship:

  hyperedge "body": {x = a + b, y = b * c, return x + y} --> foo

  This preserves the group structure — the fact that these three statements belong together in the same field. Think of it like the difference between "these items are in the
   same array" vs. "each item individually points to the parent."

  "Directed" — Edges Have a Direction

  Each hyperedge has a tail (source nodes) and a head (target node). In the AST context:
  - Tail: the child nodes (the statements in the body)
  - Head: the parent node (the function definition)

  This preserves the parent-child hierarchy of the AST. Without direction, you lose the information about which nodes are children and which is the parent.

  "Heterogeneous" — Nodes and Edges Have Types

  Not all nodes are the same, and not all edges are the same:

  - Node types: FunctionDef, BinOp, Name, Return, If, Call, etc. (Python has ~93 AST node types)
  - Edge types: body, args, targets, value, test, orelse, etc. (Python has ~58 field types)

  A body edge means something fundamentally different from an args edge. Heterogeneity preserves this semantic information.

  All Together

  An HDHG node is: (type, value) — e.g., (Name, "x") or (FunctionDef, "foo")

  An HDHG hyperedge is: (edge_type, {child_nodes...}, parent_node) — e.g., ("body", {stmt1, stmt2, stmt3}, func_node)

  Think of it as a strictly more expressive version of what a CPG does — but for the AST layer specifically. Instead of flattening everything into pairwise edges, it
  preserves:
  - Group relationships (hyperedges)
  - Hierarchy (direction)
  - Semantics (heterogeneous types)

  ---
  4. Heterogeneous Directed Hypergraph Neural Network (HDHGN)

  This is the neural network that learns on HDHG-structured data. It's the GGNN equivalent, but for hypergraphs instead of regular graphs.

  The Pipeline

  Source Code → Parse AST → Build HDHG → HDHGN → Classification

  How It Works (Two-Phase Message Passing)

  Remember how GGNNs do message passing node-to-node along pairwise edges? HDHGN does it in two phases because hyperedges are first-class citizens:

  Phase 1: Nodes → Hyperedge
  All nodes connected by a hyperedge send messages to that hyperedge. The hyperedge aggregates them using transformer-style attention (think: it learns which child nodes
  matter most).

  Critically, direction matters here: child nodes (tail) are transformed with one weight matrix, the parent node (head) with a different one. The network knows who's a child
  and who's a parent.

              ┌─ child_1 ──┐
              │  child_2 ──┤──→ hyperedge representation
              │  child_3 ──┤     (via attention)
              └─ parent  ──┘
                 (different weights for parent vs children)

  Phase 2: Hyperedge → Nodes
  Each hyperedge broadcasts its aggregated representation back to all connected nodes, again using attention and direction-aware weights.

  hyperedge representation ──→ updated child_1
                            ──→ updated child_2
                            ──→ updated child_3
                            ──→ updated parent

  After one round, all siblings that share a parent field know about each other. In a regular GNN, this would take two rounds (child → parent → other child). HDHGN is
  architecturally more efficient at propagating sibling-level information.

  Node Update

  After the two-phase aggregation, each node blends its new state with its previous state (residual connection), normalizes with GraphNorm, and applies an ELU activation.
  This is simpler than GRU gating — no explicit gates, just normalization + residual.

  Graph Readout

  To classify an entire code snippet, HDHGN computes a learned attention-weighted sum over all node states, producing a single vector that represents the whole program. This
  goes through an MLP for final classification.

  Performance

  On code classification (IBM's CodeNet dataset):
  ┌───────┬───────────┬─────────┐
  │ Model │ Python800 │ Java250 │
  ├───────┼───────────┼─────────┤
  │ GCN   │ 93.86%    │ 91.17%  │
  ├───────┼───────────┼─────────┤
  │ GGNN  │ 92.17%    │ 92.68%  │
  ├───────┼───────────┼─────────┤
  │ HDHGN │ 97.87%    │ 96.42%  │
  └───────┴───────────┴─────────┘
  The ablation study shows each "H-D-H" property matters:
  - Remove hyperedges (use pairwise edges): -3.08%
  - Remove heterogeneity (treat all types the same): -2.64%
  - Remove direction (ignore parent/child distinction): -2.38%

  ---
  How They All Fit Together

  CPG = the data structure for representing code
        (merges AST + CFG + data flow into one graph)
           │
           ▼
  GGNN = a neural network that learns on graphs like CPGs
         (uses GRU gates to prevent over-smoothing)
           │
           ▼
  HDHG = a richer data structure (hypergraph, directed, typed)
         that captures group relationships the CPG misses
           │
           ▼
  HDHGN = a neural network that learns on HDHGs
          (two-phase message passing with attention)

  The evolution is: we went from hand-written graph queries (CPG + Joern), to learning on pairwise graphs (GGNN on CPGs), to learning on richer hypergraph representations
  (HDHGN on HDHGs) that preserve more structural information from the source code.

---

What Graphs You Actually Need

  1. Code Structure Graph (per-change analysis)

  This is where CPG lives. You need it for the "trace back all variables/functions/classes across all levels" requirement. But you don't need to build this yourself — use
  existing tools:

  - tree-sitter — fast, incremental AST parsing for ~40 languages. You already said you'll use language-specific parsers. This is your foundation.
  - LSP (Language Server Protocol) — gives you go-to-definition, find-references, call hierarchies, type hierarchies. Most languages have mature LSP servers. This is your
  "trace across all levels" tool.
  - Joern / CodeQL — if you need deep data-flow tracing (taint analysis, reachability). Heavier, but gives you CPG-level analysis.

  You don't need HDHG/HDHGN here. Those give you ~3% accuracy improvement on code classification (identifying algorithms). Your task is code understanding — you have LLMs for
   reasoning and static analysis tools for facts. The LLM doesn't need a hypergraph; it needs the right context fed to it.

  2. Repository Dependency Graph (cross-file impact)

  FileA imports ModuleB
  ModuleB exports FunctionC
  FunctionC calls FunctionD (in FileE)
  FunctionD uses TypeF (defined in FileG)

  This answers: "this PR changed FunctionC — what else in the repo is affected?"

  You can build this from LSP data + import analysis. It's a directed heterogeneous graph (different node types: files, modules, functions, types, variables; different edge
  types: imports, calls, inherits, implements). Store it in something queryable — Neo4j, or even a simpler adjacency representation if the repo isn't massive.

  This is critical for your "don't annoy humans" goal. If you can't trace impact, you'll either miss real issues (false negatives) or flag things that don't matter (false
  positives).

  3. Temporal Knowledge Graph (project memory)

  This is the big one for your vision. Not a code graph — a knowledge graph.

  Nodes are entities like:
  - Decisions ("we chose PostgreSQL over MongoDB because...")
  - Conventions ("we use snake_case in Python, camelCase in TS")
  - Architecture patterns ("auth goes through middleware X")
  - Known quirks ("this function looks wrong but it's intentional, see PR #423")
  - People ("Alice owns the billing module")
  - External context ("JIRA-1234 required this workaround")

  Edges are relationships: decided_by, supersedes, relates_to, authored_by, referenced_in, etc.

  Your non-destructive memory requirement maps directly to a bitemporal model:

  ┌─────────────────────────────────────────────────┐
  │ Fact: "Auth uses JWT"                           │
  │ valid_from: 2024-03-15  valid_to: 2025-01-10    │
  │ recorded_at: 2024-03-15                         │
  │ source: PR #287                                 │
  ├─────────────────────────────────────────────────┤
  │ Fact: "Auth uses session tokens"                │
  │ valid_from: 2025-01-10  valid_to: NULL (current) │
  │ recorded_at: 2025-01-10                         │
  │ source: PR #891                                 │
  │ supersedes: "Auth uses JWT"                     │
  └─────────────────────────────────────────────────┘

  Two time axes:
  - Valid time — when the fact was true in the real world
  - Transaction time — when the system recorded it

  This lets you answer "what did we believe was true on March 1st?" and "when did we learn that auth changed?" — both are important for your temporal understanding
  requirement.

  This is a database design problem, not a GNN problem. Look at property graphs (Neo4j) with temporal extensions, or even a relational model with bitemporal columns if your
  query patterns are predictable.

  4. Semantic Code Index (for IDE search)

  Your "React popup/modal component" search requirement needs vector embeddings, not graph traversal.

  The approach:
  5. Parse code into semantic scopes (functions, classes, blocks — you mentioned this)
  6. Embed each scope using a code embedding model (CodeBERT, StarCoder embeddings, or similar)
  7. Store in a vector database (Qdrant, Pinecone, pgvector)
  8. At query time, embed the natural language query and find nearest neighbors

  You could augment this with a graph: "this function is semantically similar to X AND is called by Y AND lives in module Z" — hybrid retrieval. But the core is vector
  search, not graph neural networks.

  9. Developer Profile Model (for personalization)

  For the Socratic method / experience-based personalization, you need to model per-developer:
  - Which parts of the codebase they've touched (from git history)
  - Which types of issues they've been flagged on before
  - Their seniority / experience with specific patterns
  - Their stated preferences

  This could be a subgraph within your knowledge graph, or a simpler per-developer profile document. It doesn't need its own graph structure.

  6. Change Semantics Graph (for logical grouping)

  For your "group uncommitted changes into logical groups" feature, you need to understand which hunks are semantically related. This is closer to:

  Hunk A (modifies function X) ──uses_same_type──→ Hunk B (modifies type Y used by X)
  Hunk A ──same_feature──→ Hunk C (adds test for function X)
  Hunk D (updates README) ──documents──→ Hunk A

  This is a clustering problem informed by the dependency graph (#2). Not a separate graph type — it's derived from other graphs at analysis time.

  ---
  Where GNNs/HDHGN Actually Fit (Maybe Later)

  For your v1, LLMs + static analysis + graph queries will outperform any GNN approach. Here's why:

  - GNNs are good at pattern classification ("is this function vulnerable? yes/no"). Your tool needs reasoning ("why is this problematic, and how should it be fixed?"). LLMs
  do reasoning; GNNs do classification.
  - GNNs need training data. You don't have labeled datasets of "good review vs bad review" yet.
  - Your quality-over-quantity principle means you can afford to spend tokens on LLM reasoning rather than trying to save tokens with a GNN classifier.

  Where GNNs might help in later versions:

  - Pre-filtering: A trained GNN could score every function in a diff for "likelihood of containing a bug" and let the LLM focus its expensive reasoning on high-scoring
  functions. This is the "spend tokens wisely" optimization.
  - The minions/hotspot detection: Scanning the entire codebase periodically is expensive with LLMs. A GNN could cheaply score every function for code smell / complexity /
  vulnerability risk, and only send the top N to the LLM agents.
  - Symbolic reasoning layer: You mentioned this. A GNN on top of the CPG could serve as the structured reasoning component that complements the LLM's natural language
  reasoning.

  What I'd Prioritize

  v1:  tree-sitter + LSP + linters → feed context to LLM agents
       ├── Dependency graph (from LSP/imports) for impact analysis
       ├── Bitemporal knowledge graph for project memory
       └── Vector index for semantic search

  v2:  Add CPG-level analysis (Joern) for data-flow tracing
       ├── Git history analysis for temporal understanding
       └── PR history mining for tribal knowledge

  v3:  Consider GNN-based pre-filtering for hotspot detection
       ├── HDHG-style representations if classification accuracy matters
       └── Symbolic reasoning layer on CPG

  The graph research (CPG, GGNN, HDHG) is relevant to your long-term architecture — understanding these concepts will help you make good design decisions now. But building or
   training them is not where your engineering time should go first.