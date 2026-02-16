# Quickscope Architecture - Complete File Processing Diagram

This document provides a full and detailed diagram of how each file is processed, including all internal workings such as scope tracking, declaration/reference tracking, hoisting/visibility resolution, and code generation.

---

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           QUICKSCOPE FILE PROCESSING                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────┐     ┌────────────────┐     ┌─────────────────┐     ┌───────────┐ │
│  │   Input      │────▶│  Build-Time    │────▶│   Runtime       │────▶│  Output   │ │
│  │   File       │     │  Code Gen      │     │   Analysis      │     │  Result   │ │
│  └──────────────┘     └────────────────┘     └─────────────────┘     └───────────┘ │
│        │                     │                       │                     │        │
│        │                     │                       │                     │        │
│   .ts/.js/.tsx          Generates            O(1) Inline Analysis   AnalysisResult │
│   source code        optimized Rust             (see below)           - Scopes     │
│                         matchers                                      - Imports    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Build-Time Code Generation

At compile time, the build script transforms language definitions into optimized Rust code.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        BUILD-TIME CODE GENERATION                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  quickscope-languages/src/typescript.rs                                            │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │  language!(                                                                 │    │
│  │    name: "TypeScript",                                                      │    │
│  │    extensions: ["js", "ts", "tsx", ...],                                    │    │
│  │    language: tree_sitter_typescript::LANGUAGE_TYPESCRIPT,                   │    │
│  │    handlers: {                                                              │    │
│  │      ["function_declaration"] => { builder.create_scope(node, true); }      │    │
│  │      ["if_statement"] => { builder.increment_branch(); ... }                │    │
│  │      ...                                                                    │    │
│  │    }                                                                        │    │
│  │  )                                                                          │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                        │                                            │
│                                        ▼                                            │
│  quickscope-build/src/codegen/mod.rs                                               │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │  for language in LANGUAGES:                                                 │    │
│  │    1. Convert string node kinds → numeric kind_id                           │    │
│  │       "function_declaration" → 156 (tree-sitter ID)                         │    │
│  │                                                                             │    │
│  │    2. Generate visit_typescript() function with numeric match               │    │
│  │       match node.kind_id() {                                                │    │
│  │         156 => { builder.create_scope(node, true); }                        │    │
│  │         ...                                                                 │    │
│  │       }                                                                     │    │
│  │                                                                             │    │
│  │    3. Generate thread-local parser: PARSER_TYPESCRIPT                       │    │
│  │                                                                             │    │
│  │    4. Generate analyze() match arm for extensions                           │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                        │                                            │
│                                        ▼                                            │
│  $OUT_DIR/quickscope.rs (generated)                                                │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │  fn visit_typescript(builder, node, source_bytes) {                         │    │
│  │      match node.kind_id() {                                                 │    │
│  │          156 | 157 => { /* function handlers */ }                           │    │
│  │          23 | 45 | 67 => { /* branch handlers */ }                          │    │
│  │          ...                                                                │    │
│  │      }                                                                      │    │
│  │  }                                                                          │    │
│  │                                                                             │    │
│  │  thread_local! {                                                            │    │
│  │      static PARSER_TYPESCRIPT: UnsafeCell<Parser> = { ... };                │    │
│  │  }                                                                          │    │
│  │                                                                             │    │
│  │  pub fn analyze(file: &File) -> Option<AnalysisResult> {                    │    │
│  │      match extension {                                                      │    │
│  │          "ts" | "tsx" | "js" ... => { /* traversal code */ }                │    │
│  │          _ => None                                                          │    │
│  │      }                                                                      │    │
│  │  }                                                                          │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Why Numeric Kind IDs?

```
String Comparison:     ~60+ CPU cycles (hash computation, memory access)
Numeric Comparison:    ~2 CPU cycles (integer equality)

                       30x faster per node visit!
```

---

## 2. Runtime Analysis - O(1) Inline Resolution

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        O(1) INLINE RESOLUTION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  SINGLE-PASS TRAVERSAL WITH LIVE BINDING TABLE                              │   │
│  │  ═════════════════════════════════════════════                              │   │
│  │  • Walk tree-sitter AST depth-first                                         │   │
│  │  • Create scopes with locations & metrics                                   │   │
│  │  • Maintain live symbol binding table (2 bytes per symbol)                  │   │
│  │  • Resolve references O(1) via direct table lookup                          │   │
│  │  • Track import declarations and increment counts inline                    │   │
│  │                                                                             │   │
│  │  HOW IT HANDLES HOISTING:                                                   │   │
│  │  JavaScript hoisting means references can appear BEFORE declarations:       │   │
│  │                                                                             │   │
│  │    foo();                    // reference to 'foo' (line 1)                 │   │
│  │    function foo() {}         // declaration of 'foo' (line 2)               │   │
│  │                                                                             │   │
│  │  VAR HOISTING EDGE CASE:                                                    │   │
│  │  When `var foo` shadows an import inside a function, we must retroactively  │   │
│  │  correct any import counts that were incremented before the var was seen.   │   │
│  │  This is handled via pending_var_refs tracking within function boundaries.  │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                                        ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  O(1) RESOLUTION DATA STRUCTURES                                            │   │
│  │  ═══════════════════════════════                                            │   │
│  │  bindings: Vec<SymbolBinding>      // O(1) lookup, 2 bytes per symbol       │   │
│  │  shadow_symbols: Vec<u16>          // Shadow stack for scope restoration    │   │
│  │  shadow_bindings: Vec<SymbolBinding>                                        │   │
│  │  shadow_boundaries: Vec<u16>       // Scope boundaries for batch restore    │   │
│  │  pending_var_refs: Vec<PendingVarRef>   // For var hoisting correction      │   │
│  │  pending_var_boundaries: Vec<u16>  // Function boundaries                   │   │
│  │                                                                             │   │
│  │  COMPLEXITY: O(1) per reference vs old O(refs × depth × decls_per_scope)    │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. AST Traversal - Detailed Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            AST TRAVERSAL DETAIL                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  INPUT: File { path: "example.ts", content: "..." }                                │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 1: Parse Source → AST                                                  │  │
│  │                                                                              │  │
│  │  PARSER_TYPESCRIPT.with(|parser| {                                           │  │
│  │      let tree = parser.parse(&file.content, None);                           │  │
│  │      // tree-sitter builds concrete syntax tree                              │  │
│  │  })                                                                          │  │
│  │                                                                              │  │
│  │  Thread-local parser avoids:                                                 │  │
│  │  • Re-initialization overhead                                                │  │
│  │  • Memory allocation for parser state                                        │  │
│  │  • Language setting for each file                                            │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                            │
│                                        ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 2: Initialize ScopeBuilder                                             │  │
│  │                                                                              │  │
│  │  ScopeBuilder {                                                              │  │
│  │      scopes: Scopes::new(),           // Parallel arrays (SoA)               │  │
│  │      parents: Vec::new(),             // Parent relationships                │  │
│  │      names: AHashMap::new(),          // String interning table              │  │
│  │      imports: Vec::new(),             // ImportData storage                  │  │
│  │      current_function_scope: 0,       // For var hoisting                    │  │
│  │      scope_stack_indices: IndexStack, // Fixed-size [u16; 127]               │  │
│  │      scope_stack_created: CreatedStack, // Bitfield (256 bits)               │  │
│  │      // O(1) resolution data structures                                      │  │
│  │      bindings: Vec::new(),            // Symbol -> binding lookup table      │  │
│  │      shadow_symbols: Vec::new(),      // Shadow stack for scope restoration  │  │
│  │      shadow_bindings: Vec::new(),     // Parallel to shadow_symbols          │  │
│  │      shadow_boundaries: Vec::new(),   // Scope boundaries                    │  │
│  │      pending_var_refs: Vec::new(),    // For var hoisting correction         │  │
│  │      pending_var_boundaries: Vec::new() // Function boundaries               │  │
│  │  }                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                            │
│                                        ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 3: Depth-First Traversal Loop                                          │  │
│  │                                                                              │  │
│  │  let mut cursor = tree.walk();                                               │  │
│  │                                                                              │  │
│  │  loop {                                                                      │  │
│  │      let node = cursor.node();                                               │  │
│  │      let before = scopes.len();                                              │  │
│  │                                                                              │  │
│  │      visit_typescript(&mut builder, &node, source_bytes);  // Call visitor   │  │
│  │                                                                              │  │
│  │      let created = scopes.len() > before;  // Did visitor create a scope?    │  │
│  │      // ... (stack management, see below)                                    │  │
│  │                                                                              │  │
│  │      if cursor.goto_first_child() { continue; }  // Descend                  │  │
│  │      builder.leave();                                                        │  │
│  │      if cursor.goto_next_sibling() { continue; }  // Sibling                 │  │
│  │      // Walk up until we find a sibling                                      │  │
│  │      loop {                                                                  │  │
│  │          if !cursor.goto_parent() { return result; }  // Done                │  │
│  │          builder.leave();                                                    │  │
│  │          if cursor.goto_next_sibling() { break; }                            │  │
│  │      }                                                                       │  │
│  │  }                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Scope Tracking with Fixed-Size Stacks

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           SCOPE TRACKING INTERNALS                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  TWO STACKS WORK TOGETHER:                                                         │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  IndexStack: Tracks WHICH scopes are in the current path                    │   │
│  │  ══════════                                                                 │   │
│  │  struct IndexStack {                                                        │   │
│  │      indices: [u16; 127],  // Fixed array (no heap allocation)              │   │
│  │      len: u8,              // Current stack depth                           │   │
│  │  }                                                                          │   │
│  │                                                                             │   │
│  │  Example state during traversal:                                            │   │
│  │  ┌─────┬─────┬─────┬─────┬───────────────────────────┐                      │   │
│  │  │  0  │  2  │  5  │  8  │  ...  (unused)  ...       │                      │   │
│  │  └─────┴─────┴─────┴─────┴───────────────────────────┘                      │   │
│  │    ▲     ▲     ▲     ▲                                                      │   │
│  │    │     │     │     └── Current scope (innermost)                          │   │
│  │    │     │     └── Function scope                                           │   │
│  │    │     └── If block scope                                                 │   │
│  │    └── Module scope (root, index 0)                                         │   │
│  │                                                                             │   │
│  │  len = 4 (4 scopes in current path)                                         │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  CreatedStack: Tracks WHETHER each traversal depth created a scope          │   │
│  │  ════════════                                                               │   │
│  │  struct CreatedStack {                                                      │   │
│  │      bits: [u64; 4],  // 256 bits total (32 bytes)                          │   │
│  │      len: u8,         // Current traversal depth                            │   │
│  │  }                                                                          │   │
│  │                                                                             │   │
│  │  Example: bit N = 1 if depth N created a scope                              │   │
│  │  ┌───────────────────────────────────────────────────────────────────┐      │   │
│  │  │ bits[0]: 1 0 0 1 0 0 1 0 0 0 0 1 ...                              │      │   │
│  │  └───────────────────────────────────────────────────────────────────┘      │   │
│  │            ▲     ▲     ▲         ▲                                          │   │
│  │            │     │     │         └── depth 11: scope created (if block)     │   │
│  │            │     │     └── depth 6: scope created (function)                │   │
│  │            │     └── depth 3: scope created (if block)                      │   │
│  │            └── depth 0: scope created (program/module)                      │   │
│  │                                                                             │   │
│  │  WHY BITFIELD?                                                              │   │
│  │  • 32 bytes vs 256 bytes for bool array (8x smaller)                        │   │
│  │  • Single cache line fits entire structure                                  │   │
│  │  • Bitwise ops: 2 cycles vs 60+ cycles for division                         │   │
│  │                                                                             │   │
│  │  BIT ACCESS OPTIMIZATION:                                                   │   │
│  │  idx = len >> 6;      // depth / 64 (which u64)                             │   │
│  │  bit = len & 63;      // depth % 64 (which bit)                             │   │
│  │  // Bitwise is ~30x faster than integer division!                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  WHY TWO STACKS?                                                                   │
│  ═══════════════                                                                   │
│  Not every AST node creates a scope:                                               │
│                                                                                     │
│  AST: program                        │ IndexStack     │ CreatedStack               │
│       ├── function_declaration       │ push(0)        │ push(true)  [depth 0]      │
│       │   ├── identifier "foo"       │ push(1)        │ push(true)  [depth 1]      │
│       │   └── statement_block        │ (no scope)     │ push(false) [depth 2]      │
│       │       └── return_statement   │ (no scope)     │ push(false) [depth 3]      │
│       │           └── number "1"     │ (no scope)     │ push(false) [depth 4]      │
│                                                                                     │
│  On leave():                                                                        │
│  • Pop CreatedStack to check if we pushed to IndexStack                            │
│  • Only pop IndexStack if CreatedStack was true                                    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Declaration and Reference Tracking

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      DECLARATION & REFERENCE TRACKING                               │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  STEP 1: STRING INTERNING (during traversal)                                       │
│  ═══════════════════════════════════════════                                       │
│                                                                                     │
│  Names HashMap: Maps identifier strings → unique u16 indices                       │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Source: "import { foo } from 'bar'; const x = foo(); let foo = 1;"         │  │
│  │                                                                              │  │
│  │  names: AHashMap<&str, u16>                                                  │  │
│  │  ┌──────────────┬─────────┐                                                  │  │
│  │  │  Key (&str)  │  Value  │                                                  │  │
│  │  ├──────────────┼─────────┤                                                  │  │
│  │  │  "foo"       │    0    │  ◄── First occurrence, assigned index 0          │  │
│  │  │  "bar"       │    1    │  ◄── Import source                               │  │
│  │  │  "x"         │    2    │  ◄── Variable name                               │  │
│  │  └──────────────┴─────────┘                                                  │  │
│  │                                                                              │  │
│  │  WHY INTERN?                                                                 │  │
│  │  • Same name → same u16 index (deduplication)                                │  │
│  │  • References and declarations use index, not string                         │  │
│  │  • Borrow from source bytes (zero allocation during traversal)               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  STEP 2: COMPACT DECLARATION STORAGE (4 bytes each)                                │
│  ══════════════════════════════════════════════════                                │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  CompactDecl (4 bytes total)                                                 │  │
│  │  ┌─────────────────┬─────────────────────────────────────────────────────┐   │  │
│  │  │  symbol: u16    │  packed: u16                                        │   │  │
│  │  │  (name index)   │                                                     │   │  │
│  │  └─────────────────┴─────────────────────────────────────────────────────┘   │  │
│  │                          │                                                   │  │
│  │                          ▼                                                   │  │
│  │  Packed bit layout:                                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐     │  │
│  │  │ 15  14  13  12  11  10   9   8   7   6   5   4 │  3  │  2  │ 1   0 │     │  │
│  │  ├─────────────────────────────────────────────────┼─────┼─────┼───────┤     │  │
│  │  │            import_idx (12 bits)                 │ imp │ tim │target │     │  │
│  │  │            (0-4095 import index)                │     │ ing │(2 bit)│     │  │
│  │  └─────────────────────────────────────────────────┴─────┴─────┴───────┘     │  │
│  │                                                                              │  │
│  │  target (2 bits):  0=Scope, 1=Function, 2=Module                             │  │
│  │  timing (1 bit):   0=Point (TDZ), 1=Scope (hoisted)                          │  │
│  │  is_import (1 bit): 0=regular decl, 1=import                                 │  │
│  │  import_idx (12 bits): index into imports Vec (if is_import=1)               │  │
│  │                                                                              │  │
│  │  MEMORY SAVINGS:                                                             │  │
│  │  Enum approach: ~24 bytes per declaration                                    │  │
│  │  Packed approach: 4 bytes per declaration                                    │  │
│  │  Savings: 83% reduction!                                                     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  STEP 3: O(1) REFERENCE RESOLUTION (Inline)                                        │
│  ═══════════════════════════════════════════                                       │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  SymbolBinding (2 bytes packed):                                             │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐     │  │
│  │  │ [15:4] import_idx (12 bits)  [3] is_import  [2:1] unused  [0] present│     │  │
│  │  └─────────────────────────────────────────────────────────────────────┘     │  │
│  │                                                                              │  │
│  │  Resolution is O(1): bindings[symbol].is_import() → imports[idx].count++    │  │
│  │                                                                              │  │
│  │  Example during traversal:                                                   │  │
│  │                                                                              │  │
│  │  Source:                                                                     │  │
│  │  ```javascript                                                               │  │
│  │  import { foo } from 'bar';  // bindings[0] = import(idx=0)                  │  │
│  │  function test() {           // scope 1, enter_function_scope()              │  │
│  │      const x = foo();        // reference: bindings[0].is_import → count++   │  │
│  │      if (true) {             // scope 2, enter_scope()                       │  │
│  │          let foo = x;        // bindings[0] = local (shadow old to stack)    │  │
│  │          console.log(foo);   // reference: bindings[0] is local, no count    │  │
│  │      }                       // leave_scope(): restore bindings[0] = import  │  │
│  │      return foo;             // reference: bindings[0].is_import → count++   │  │
│  │  }                           // leave_function_scope()                       │  │
│  │  ```                                                                         │  │
│  │                                                                              │  │
│  │  Shadow stack enables scope restoration:                                     │  │
│  │  ┌───────────────────────────────────┐                                       │  │
│  │  │ When entering scope 2:            │                                       │  │
│  │  │   shadow_symbols.push(0)          │  ◄── symbol "foo"                     │  │
│  │  │   shadow_bindings.push(import(0)) │  ◄── old binding (import)             │  │
│  │  │   bindings[0] = local             │  ◄── new binding (local)              │  │
│  │  │ When leaving scope 2:             │                                       │  │
│  │  │   bindings[0] = shadow_bindings.pop() = import(0)                         │  │
│  │  └───────────────────────────────────┘                                       │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Hoisting and Visibility Resolution

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        HOISTING & VISIBILITY RESOLUTION                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  JAVASCRIPT SCOPING RULES:                                                         │
│  ═════════════════════════                                                         │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Target::Scope (let, const, class)                                           │  │
│  │  ─────────────────────────────────                                           │  │
│  │  • Block-scoped: visible only in declaring block and nested blocks           │  │
│  │  • Timing::Point: TDZ applies (ReferenceError before declaration)            │  │
│  │                                                                              │  │
│  │  Target::Function (var, function declarations)                               │  │
│  │  ────────────────────────────────────────────                                │  │
│  │  • Function-scoped: hoisted to enclosing function (or module)                │  │
│  │  • Timing::Scope: accessible from start of function (value undefined)        │  │
│  │                                                                              │  │
│  │  Target::Module (imports)                                                    │  │
│  │  ────────────────────────                                                    │  │
│  │  • Module-scoped: visible throughout entire module                           │  │
│  │  • Timing::Scope: imports are hoisted (available immediately)                │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  O(1) RESOLUTION VIA BINDING TABLE:                                                │
│  ══════════════════════════════════                                                │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Reference resolution is O(1) using the live binding table:                  │  │
│  │                                                                              │  │
│  │  fn reference(&mut self, name: &str) {                                       │  │
│  │      let symbol = self.names.entry(name).or_insert(next_idx);                │  │
│  │      if symbol >= bindings.len() { return; }  // Unbound (global/builtin)    │  │
│  │                                                                              │  │
│  │      let binding = self.bindings[symbol];                                    │  │
│  │      if binding.is_present() && binding.is_import() {                        │  │
│  │          self.imports[binding.import_idx()].count += 1;                      │  │
│  │                                                                              │  │
│  │          // Track for potential var shadowing within functions               │  │
│  │          if self.current_function_scope != 0 {                               │  │
│  │              self.pending_var_refs.push(PendingVarRef { symbol, import_idx });│  │
│  │          }                                                                   │  │
│  │      }                                                                       │  │
│  │  }                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  VAR HOISTING EDGE CASE:                                                           │
│  ═══════════════════════                                                           │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Problem: In JS, `var` declarations hoist to function scope, but we process  │  │
│  │  references BEFORE we see the var declaration:                               │  │
│  │                                                                              │  │
│  │    import { foo } from 'mod';                                                │  │
│  │    function test() {                                                         │  │
│  │        foo();      // We increment import count here...                      │  │
│  │        var foo = 1;// ...but var hoists, so foo() refers to local var!       │  │
│  │    }                                                                         │  │
│  │                                                                              │  │
│  │  Solution: Track pending import refs within each function. When we see a     │  │
│  │  `var` declaration that shadows an import, decrement the count and remove    │  │
│  │  the pending refs for that symbol.                                           │  │
│  │                                                                              │  │
│  │  fn handle_var_shadowing(&mut self, symbol: u16) {                           │  │
│  │      let binding = self.bindings[symbol];                                    │  │
│  │      if binding.is_import() {                                                │  │
│  │          // Find and remove pending refs for this symbol                     │  │
│  │          // Decrement import count accordingly                               │  │
│  │      }                                                                       │  │
│  │  }                                                                           │  │
│  │                                                                              │  │
│  │  // Unresolved references are ignored (global builtins, undeclared vars)     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  RESOLUTION EXAMPLE:                                                               │
│  ═══════════════════                                                               │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Source:                                                                     │  │
│  │  ```javascript                                                               │  │
│  │  import { helper } from './utils';  // scope 0, Target::Module               │  │
│  │  var globalVar = 1;                 // scope 0, Target::Function             │  │
│  │                                                                              │  │
│  │  function outer() {                 // scope 1 (function)                    │  │
│  │      let x = helper();              // scope 1, Target::Scope                │  │
│  │                                                                              │  │
│  │      if (true) {                    // scope 2 (block)                       │  │
│  │          var y = globalVar;         // scope 2, but hoists to scope 1!       │  │
│  │          let x = 10;                // scope 2, Target::Scope (shadows!)     │  │
│  │          console.log(x);            // references x → finds scope 2's x      │  │
│  │      }                                                                       │  │
│  │                                                                              │  │
│  │      console.log(x);                // references x → finds scope 1's x      │  │
│  │      console.log(y);                // references y → finds scope 1's y      │  │
│  │  }                                                                           │  │
│  │  ```                                                                         │  │
│  │                                                                              │  │
│  │  Scope structure:                                                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐         │  │
│  │  │ scope 0 (module):                                               │         │  │
│  │  │   declarations: [helper: Module, globalVar: Function]           │         │  │
│  │  │   function_scope: 0                                             │         │  │
│  │  │   ┌─────────────────────────────────────────────────────────┐   │         │  │
│  │  │   │ scope 1 (function outer):                               │   │         │  │
│  │  │   │   declarations: [x: Scope, y: Function (hoisted!)]      │   │         │  │
│  │  │   │   function_scope: 1                                     │   │         │  │
│  │  │   │   ┌─────────────────────────────────────────────────┐   │   │         │  │
│  │  │   │   │ scope 2 (if block):                             │   │   │         │  │
│  │  │   │   │   declarations: [x: Scope (shadows outer x!)]   │   │   │         │  │
│  │  │   │   │   function_scope: 1  (same as parent)           │   │   │         │  │
│  │  │   │   └─────────────────────────────────────────────────┘   │   │         │  │
│  │  │   └─────────────────────────────────────────────────────────┘   │         │  │
│  │  └─────────────────────────────────────────────────────────────────┘         │  │
│  │                                                                              │  │
│  │  Key insight: `var y` in scope 2 gets declared in scope 1 (function scope)  │  │
│  │  because Target::Function hoists to enclosing function.                      │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Data Structures - Struct-of-Arrays Layout

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        STRUCT-OF-ARRAYS DATA LAYOUT                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  TRADITIONAL (Array-of-Structs):                                                   │
│  ═══════════════════════════════                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────┐    │
│  │ [Scope0: loc+metrics+score] [Scope1: loc+metrics+score] [Scope2: ...]     │    │
│  └────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  Problem: Iterating over just metrics loads location and score data into cache    │
│                                                                                     │
│  QUICKSCOPE (Struct-of-Arrays):                                                    │
│  ═══════════════════════════════                                                   │
│                                                                                     │
│  Scopes {                                                                          │
│      locations:      [Loc0, Loc1, Loc2, ...]      // 4 bytes each                  │
│      metrics:        [Met0, Met1, Met2, ...]      // 16 bytes each (cache-aligned) │
│      scores:         [Scr0, Scr1, Scr2, ...]      // 16 bytes each                 │
│      declarations:   [Vec0, Vec1, Vec2, ...]      // Per-scope declaration lists   │
│      function_scope: [0,    1,    1,    ...]      // u16 per scope                 │
│  }                                                                                  │
│                                                                                     │
│  Benefits:                                                                          │
│  • Iterate metrics: only metrics data in cache (no pollution)                      │
│  • Metrics packed to 16 bytes: 4 fit in one 64-byte cache line                     │
│  • Sequential access pattern: CPU prefetcher works optimally                       │
│                                                                                     │
│  METRICS STRUCTURE (16 bytes, cache-aligned):                                      │
│  ══════════════════════════════════════════════                                    │
│  ┌───────────────────────────────────────────────────────────────────────────┐     │
│  │  argument_count:      u8   │  4 Metrics instances fit                     │     │
│  │  depth:               u8   │  perfectly in one                            │     │
│  │  is_async:            bool │  64-byte cache line                          │     │
│  │  depth_limited:       bool │                                              │     │
│  │  binding_count:       u8   │  ┌─────────────────────────────────────────┐ │     │
│  │  return_count:        u8   │  │ Cache Line (64 bytes):                  │ │     │
│  │  throw_count:         u8   │  │ [Metrics0][Metrics1][Metrics2][Metrics3]│ │     │
│  │  branch_count:        u8   │  └─────────────────────────────────────────┘ │     │
│  │  jump_count:          u8   │                                              │     │
│  │  try_count:           u8   │                                              │     │
│  │  call_count:          u8   │                                              │     │
│  │  await_count:         u8   │                                              │     │
│  │  nested_scope_count:  u8   │                                              │     │
│  │  assignment_count:    u8   │                                              │     │
│  │  is_function:         bool │  ◄── For hoisting semantics only             │     │
│  │  _pad:                u8   │  ◄── Padding to exactly 16 bytes             │     │
│  └───────────────────────────────────────────────────────────────────────────┘     │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Import Tracking

Import tracking builds a **file-level dependency graph** where:
- **Nodes** = files in the repository
- **Edges** = import relationships between files
- **Edge weight** = total count of symbol usages from the imported file

This is intentionally **per-source-path, not per-specifier**. We don't track whether `foo` was used 5 times and `bar` was unused—we track that `'./utils'` was referenced 6 times total. This provides the connection strength between files for the dependency graph without the overhead of per-symbol bookkeeping.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                             IMPORT TRACKING                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  PURPOSE: Build weighted dependency graph (file A → file B, weight = usage count)  │
│                                                                                     │
│  IMPORT TYPES HANDLED:                                                             │
│  ═════════════════════                                                             │
│                                                                                     │
│  1. Named imports:     import { foo, bar } from 'module'                           │
│  2. Default imports:   import React from 'react'                                   │
│  3. Namespace imports: import * as utils from './utils'                            │
│                                                                                     │
│  STORAGE STRUCTURE:                                                                │
│  ══════════════════                                                                │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  During Traversal:                                                           │  │
│  │                                                                              │  │
│  │  imports: Vec<ImportData<'a>>                                                │  │
│  │  ┌──────────────────────────────────────────┐                                │  │
│  │  │ [0] ImportData { source: "react", count: 0 }                              │  │
│  │  │ [1] ImportData { source: "./utils", count: 0 }                            │  │
│  │  │ [2] ImportData { source: "lodash", count: 0 }                             │  │
│  │  └──────────────────────────────────────────┘                                │  │
│  │                        ▲                                                     │  │
│  │                        │                                                     │  │
│  │  CompactDecl in declarations[0]:                                             │  │
│  │  ┌─────────────────────┐                                                     │  │
│  │  │ symbol: 0 ("React") │───────────────────┐                                 │  │
│  │  │ packed: 0b...1010   │                   │                                 │  │
│  │  │   target: Module    │                   │                                 │  │
│  │  │   timing: Scope     │                   │                                 │  │
│  │  │   is_import: true   │                   │                                 │  │
│  │  │   import_idx: 0     │───────────────────┘ (links to imports[0])           │  │
│  │  └─────────────────────┘                                                     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  After Traversal (counts updated inline):                                    │  │
│  │                                                                              │  │
│  │  imports: Vec<ImportData<'a>>                                                │  │
│  │  ┌───────────────────────────────────────────┐                               │  │
│  │  │ [0] ImportData { source: "react", count: 5 }  ◄── 5 refs to react symbols │  │
│  │  │ [1] ImportData { source: "./utils", count: 12 } ◄── 12 refs (edge weight) │  │
│  │  │ [2] ImportData { source: "lodash", count: 0 }  ◄── No refs (dead edge)    │  │
│  │  └───────────────────────────────────────────┘                               │  │
│  │                                                                              │  │
│  │  Converted to owned data in into_result():                                   │  │
│  │                                                                              │  │
│  │  imports: Vec<Import>                                                        │  │
│  │  ┌───────────────────────────────────────────┐                               │  │
│  │  │ [0] Import { source: String::from("react"), count: 5 }                    │  │
│  │  │ [1] Import { source: String::from("./utils"), count: 12 }                 │  │
│  │  │ [2] Import { source: String::from("lodash"), count: 0 }                   │  │
│  │  └───────────────────────────────────────────┘                               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE FILE PROCESSING FLOW                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  INPUT                                                                              │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  File { path: "example.ts", content: "..." }                                │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  EXTENSION MATCHING                                                         │   │
│  │  match extension {                                                          │   │
│  │      "ts" | "tsx" | "js" | ... => use TypeScript handler                    │   │
│  │      _ => return None                                                       │   │
│  │  }                                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  PARSING (tree-sitter)                                                      │   │
│  │  • Get thread-local parser (PARSER_TYPESCRIPT)                              │   │
│  │  • Parse source → AST (tree)                                                │   │
│  │  • Create cursor for traversal                                              │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  PASS 1: TRAVERSAL                                                          │   │
│  │  ┌───────────────────────────────────────────────────────────────────────┐  │   │
│  │  │  For each AST node (depth-first):                                     │  │   │
│  │  │                                                                       │  │   │
│  │  │  1. visit_typescript(builder, node, source_bytes)                     │  │   │
│  │  │     │                                                                 │  │   │
│  │  │     ├─ match node.kind_id() {                                         │  │   │
│  │  │     │    156 => builder.create_scope(node, true)  // function         │  │   │
│  │  │     │    23 => builder.increment_branch()         // if               │  │   │
│  │  │     │    89 => builder.declare_symbol(...)        // variable         │  │   │
│  │  │     │    42 => builder.declare_import(...)        // import           │  │   │
│  │  │     │    17 => builder.reference(name)            // identifier       │  │   │
│  │  │     │    _ => {}                                                      │  │   │
│  │  │     │  }                                                              │  │   │
│  │  │     │                                                                 │  │   │
│  │  │  2. Track scope creation (CreatedStack.push)                          │  │   │
│  │  │                                                                       │  │   │
│  │  │  3. If scope created:                                                 │  │   │
│  │  │     ├─ Update parent's nested_scope_count                             │  │   │
│  │  │     ├─ Track function scope if is_function                            │  │   │
│  │  │     └─ Push to IndexStack                                             │  │   │
│  │  │                                                                       │  │   │
│  │  │  4. Descend/Sibling/Ascend with leave() calls                         │  │   │
│  │  └───────────────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│    │                                                                                │
│    │  Collected Data (with import counts already resolved inline):                 │
│    │  ├─ scopes.locations: [Loc0, Loc1, ...]                                       │
│    │  ├─ scopes.metrics: [Met0, Met1, ...]                                         │
│    │  ├─ scopes.declarations: [Vec<CompactDecl>, ...]                              │
│    │  ├─ scopes.function_scope: [0, 1, 1, 2, ...]                                  │
│    │  ├─ parents: [MAX, 0, 1, 2, ...]                                              │
│    │  ├─ names: {"foo": 0, "bar": 1, ...}                                          │
│    │  └─ imports: [ImportData{source, count: N}, ...]  ◄── counts already computed │
│    │                                                                                │
│    ▼                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  FINALIZATION (into_result)                                                 │   │
│  │  • Convert ImportData<'a> (borrowed) → Import (owned)                       │   │
│  │  • Return AnalysisResult { scopes, imports }                                │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│    │                                                                                │
│    ▼                                                                                │
│  OUTPUT                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  AnalysisResult {                                                           │   │
│  │      scopes: Scopes {                                                       │   │
│  │          locations: Vec<Location>,      // [start_row, end_row] per scope   │   │
│  │          metrics: Vec<Metrics>,         // 16 fields per scope              │   │
│  │          scores: Vec<Score>,            // [own, max] per scope             │   │
│  │          declarations: Vec<ScopeDecls>, // symbols per scope                │   │
│  │          function_scope: Vec<u16>,      // enclosing function index         │   │
│  │      },                                                                     │   │
│  │      imports: Vec<Import>,              // source path + usage count        │   │
│  │  }                                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. TypeScript Handler Details

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         TYPESCRIPT HANDLER BREAKDOWN                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  SCOPE-CREATING NODES:                                                             │
│  ═════════════════════                                                             │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Node Kind                        │ is_function │ Additional Actions          │  │
│  ├───────────────────────────────────┼─────────────┼─────────────────────────────┤  │
│  │  program                          │    false    │ (module root)               │  │
│  │  else_clause                      │    false    │                             │  │
│  │  finally_clause                   │    false    │                             │  │
│  │  function_declaration             │    true     │ declare name, check async   │  │
│  │  generator_function_declaration   │    true     │ declare name, check async   │  │
│  │  arrow_function                   │    true     │ check async                 │  │
│  │  function_expression              │    true     │ check async                 │  │
│  │  method_definition                │    true     │ check async                 │  │
│  │  if_statement                     │    false    │ increment_branch            │  │
│  │  for_statement                    │    false    │ increment_branch            │  │
│  │  for_in_statement                 │    false    │ increment_branch            │  │
│  │  while_statement                  │    false    │ increment_branch            │  │
│  │  do_statement                     │    false    │ increment_branch            │  │
│  │  catch_clause                     │    false    │ increment_branch            │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  DECLARATION HANDLERS:                                                             │
│  ═════════════════════                                                             │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Node Kind              │ Target    │ Timing │ Notes                         │  │
│  ├─────────────────────────┼───────────┼────────┼───────────────────────────────┤  │
│  │  variable_declarator    │ Scope     │ Point  │ if parent is lexical_decl     │  │
│  │  variable_declarator    │ Function  │ Scope  │ if parent is variable_decl    │  │
│  │  function_declaration   │ Function  │ Scope  │ hoisted to function           │  │
│  │  class_declaration      │ Scope     │ Point  │ TDZ applies                   │  │
│  │  required_parameter     │ Scope     │ Scope  │ function parameter            │  │
│  │  optional_parameter     │ Scope     │ Scope  │ function parameter            │  │
│  │  import_specifier       │ Module    │ Scope  │ named import                  │  │
│  │  namespace_import       │ Module    │ Scope  │ import * as x                 │  │
│  │  import_clause          │ Module    │ Scope  │ default import                │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  METRIC-ONLY HANDLERS:                                                             │
│  ═════════════════════                                                             │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  Node Kind                        │ Metric Incremented                       │  │
│  ├───────────────────────────────────┼──────────────────────────────────────────┤  │
│  │  try_statement                    │ increment_try                            │  │
│  │  switch_case                      │ increment_branch                         │  │
│  │  ternary_expression               │ increment_branch                         │  │
│  │  return_statement                 │ increment_return                         │  │
│  │  throw_statement                  │ increment_throw                          │  │
│  │  break_statement                  │ increment_jump                           │  │
│  │  continue_statement               │ increment_jump                           │  │
│  │  call_expression                  │ increment_call                           │  │
│  │  new_expression                   │ increment_call                           │  │
│  │  await_expression                 │ increment_await                          │  │
│  │  yield_expression                 │ increment_return                         │  │
│  │  assignment_expression            │ increment_assignment                     │  │
│  │  augmented_assignment_expression  │ increment_assignment                     │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
│  REFERENCE HANDLER:                                                                │
│  ══════════════════                                                                │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │  ["identifier"] => {                                                         │  │
│  │      if let Ok(name) = node.utf8_text(source_bytes) {                        │  │
│  │          builder.reference(name);  // O(1) inline resolution via bindings    │  │
│  │      }                                                                       │  │
│  │  }                                                                           │  │
│  │                                                                              │  │
│  │  NOTE: Every identifier is processed as a reference, including:              │  │
│  │  • Variable uses:     foo()                                                  │  │
│  │  • Property access:   obj.foo  (both obj and foo)                            │  │
│  │  • Declaration names: const foo = 1  (foo recorded here too)                 │  │
│  │                                                                              │  │
│  │  This is fine because unbound references are simply ignored.                 │  │
│  │  The reference to 'foo' in a declaration resolves to the binding just set.   │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Performance Characteristics

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        PERFORMANCE CHARACTERISTICS                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ZERO ALLOCATION IN HOT PATH:                                                      │
│  ════════════════════════════                                                      │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Component            │ Allocation Strategy                                 │   │
│  ├───────────────────────┼─────────────────────────────────────────────────────┤   │
│  │  IndexStack           │ Fixed [u16; 127] on struct (no heap)                │   │
│  │  CreatedStack         │ Fixed [u64; 4] on struct (no heap)                  │   │
│  │  String interning     │ Borrows &str from source (no allocation)            │   │
│  │  CompactDecl          │ 4 bytes inline (no pointer chasing)                 │   │
│  │  Thread-local parser  │ One-time init per thread (reused)                   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  Only allocations:                                                                 │
│  • Result vectors (scopes, metrics, etc.) - unavoidable for output                 │
│  • Tree-sitter parse tree - external dependency                                    │
│                                                                                     │
│  CACHE EFFICIENCY:                                                                 │
│  ═════════════════                                                                 │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Structure            │ Size     │ Cache Line Fit                           │   │
│  ├───────────────────────┼──────────┼──────────────────────────────────────────┤   │
│  │  Metrics              │ 16 bytes │ 4 per cache line (64 / 16 = 4)           │   │
│  │  Location             │ 4 bytes  │ 16 per cache line                        │   │
│  │  Score                │ 16 bytes │ 4 per cache line                         │   │
│  │  CompactDecl          │ 4 bytes  │ 16 per cache line                        │   │
│  │  CreatedStack         │ 33 bytes │ Fits in single cache line                │   │
│  │  IndexStack           │ 255 bytes│ 4 cache lines (acceptable)               │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  INLINED METHODS (21 total):                                                       │
│  ══════════════════════════                                                        │
│                                                                                     │
│  All marked #[inline(always)]:                                                     │
│  • IndexStack: push, pop, last                                                     │
│  • CreatedStack: push, pop                                                         │
│  • ScopeBuilder: create_scope, leave, set_async                                    │
│  • All increment_* methods (12 total)                                              │
│  • declare_symbol, declare_import, reference                                       │
│  • CompactDecl: new, new_import, target, timing, is_import, import_idx             │
│  • Metrics::default, Score::default                                                │
│                                                                                     │
│  NUMERIC VS STRING MATCHING:                                                       │
│  ═══════════════════════════                                                       │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  Operation              │ Cycles │ Notes                                    │   │
│  ├─────────────────────────┼────────┼──────────────────────────────────────────┤   │
│  │  String comparison      │ ~60+   │ Hash compute + memory access             │   │
│  │  Numeric comparison     │ ~2     │ Single integer equality                  │   │
│  │  Speedup                │ ~30x   │ Per node visit!                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  LINEAR SCAN VS HASHMAP:                                                           │
│  ═══════════════════════                                                           │
│                                                                                     │
│  For scope declarations (typically 0-10 entries):                                  │
│  • Linear scan: sequential memory access, no hashing                               │
│  • HashMap: hash computation, potential cache misses                               │
│  • Winner: Linear scan for small N (better cache locality)                         │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Error Handling and Edge Cases

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        ERROR HANDLING & EDGE CASES                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  DEPTH LIMITING:                                                                   │
│  ═══════════════                                                                   │
│                                                                                     │
│  If scope nesting exceeds MAX_SCOPE_DEPTH (127):                                   │
│  • Scope is still created (for metrics)                                            │
│  • depth_limited flag is set to true                                               │
│  • Scope is NOT tracked on stacks (no stack overflow)                              │
│  • Nested references may not resolve correctly                                     │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  if created && !can_track {                                                 │   │
│  │      builder.scopes.metrics[before as usize].depth_limited = true;          │   │
│  │  }                                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  COUNTER OVERFLOW:                                                                 │
│  ═════════════════                                                                 │
│                                                                                     │
│  All metrics use u8 (max 255). If exceeded:                                        │
│  • Debug builds: assertion fails (catch during development)                        │
│  • Release builds: wraps around via wrapping_add()                                 │
│  • Rationale: >255 of any metric indicates pathological code                       │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  #[cfg(debug_assertions)]                                                   │   │
│  │  debug_assert!(m.branch_count < u8::MAX, "branch_count overflow");          │   │
│  │  m.branch_count = m.branch_count.wrapping_add(1);                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  IMPORT INDEX OVERFLOW:                                                            │
│  ══════════════════════                                                            │
│                                                                                     │
│  import_idx is 12 bits (max 4095 imports per file):                                │
│  • Debug assertion if exceeded                                                     │
│  • Extremely rare in practice (no file has 4096 imports)                           │
│                                                                                     │
│  UNSUPPORTED FILE TYPES:                                                           │
│  ═══════════════════════                                                           │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │  match extension {                                                          │   │
│  │      "ts" | "tsx" | "js" ... => { /* process */ }                           │   │
│  │      _ => None  // Gracefully return None, no panic                         │   │
│  │  }                                                                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  MALFORMED SOURCE:                                                                 │
│  ═════════════════                                                                 │
│                                                                                     │
│  Tree-sitter handles syntax errors gracefully:                                     │
│  • Returns partial AST with ERROR nodes                                            │
│  • Quickscope ignores ERROR nodes (no specific handler)                            │
│  • Analysis continues on valid portions                                            │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

The quickscope library processes files through a highly optimized single-pass analysis with O(1) reference resolution:

1. **Build-time**: Language definitions are compiled into numeric-ID-based match statements
2. **Traversal with inline resolution**: Single depth-first walk creating scopes, declarations, and resolving references O(1) via live binding table

Key optimizations:
- **O(1) reference resolution** via symbol binding table with shadow stack
- **Zero allocations** in traversal hot path (fixed-size stacks)
- **Numeric node kind matching** (30x faster than string comparison)
- **Struct-of-arrays layout** (cache-friendly iteration)
- **4-byte packed declarations** (83% memory reduction)
- **Thread-local parsers** (no re-initialization)
- **Aggressive inlining** (21 hot-path methods)

The design correctly handles JavaScript's complex scoping rules including:
- **var hoisting** to function scope (via pending ref correction)
- **let/const TDZ** (temporal dead zone)
- **Import hoisting** to module scope
- **Shadowing** at block boundaries (via shadow stack restoration)

---

## 13. Architecture Review

### What's Excellent

**O(1) inline resolution with shadow stack.** The binding table provides O(1) lookup per reference, and the shadow stack handles scope restoration correctly. Var hoisting edge cases are handled via pending ref tracking.

**Fixed-size stacks avoid allocation.** IndexStack [u16; 127] and CreatedStack as a bitfield are solid choices for the hot path.

**Struct-of-Arrays layout** is correct for our access patterns (iterating all metrics together).

**String interning with borrowed &str** from source bytes eliminates allocation during traversal.

**CompactDecl bit packing** at 4 bytes is memory-efficient.

---

### Architectural Concerns

#### 1. Declaration Storage Creates Fragmentation

Each scope has its own `Vec<CompactDecl>`:
```
declarations: [Vec0, Vec1, Vec2, ...]
```

This means:
- Many small heap allocations (one Vec per scope)
- Pointer chasing when iterating a scope's declarations
- Poor cache locality across scopes

**Alternative:** Single flat `Vec<CompactDecl>` with scope start/end indices stored in the Scopes struct:
```rust
decl_data: Vec<CompactDecl>,      // All declarations, contiguous
decl_ranges: Vec<(u16, u16)>,     // (start, end) per scope
```

This gives zero fragmentation and cache-friendly iteration.

#### 2. Reference Resolution is Now O(1)

~~For each reference, we walk up the parent chain checking declarations at each level. With deep nesting (127 scopes max) and many references, this compounds.~~

**RESOLVED:** The implementation now uses a live binding table with shadow stack for O(1) reference resolution. Each reference is resolved immediately via `bindings[symbol]` lookup. The shadow stack handles scope restoration when leaving blocks, and pending var refs handle the hoisting edge case.

#### 3. Missing Incremental Parsing Leverage

Tree-sitter supports incremental parsing—reusing unchanged parts of the AST when source changes slightly. For commit-by-commit analysis where files often change by a few lines, this could be a major win.

If we're re-parsing entire files on every change, we're leaving performance on the table for the stated use case.

**NOT PLANNED:** Incremental parsing requires keeping previous `Tree` objects in an in-memory per-file cache on the server. The server architecture processes files statelessly — there is no per-file state retained between commits. The `analyze()` API would also need to accept optional previous trees and edit descriptions, significantly complicating the interface for all consumers. While the theoretical speedup is 5–10x for changed files (since parsing dominates runtime), the architectural cost is not justified for the current server design. Do not propose this change.

#### 4. No Error Type Definition

The documents promise "graceful degradation" and "explicit error handling," but error types aren't defined. What happens when:
- Tree-sitter parsing fails?
- A scope exceeds depth limits?
- Import index overflows 12 bits?

For a library that "cannot crash the server," we need explicit `Result<AnalysisResult, AnalysisError>` with documented error variants.

#### 5. Language Abstraction Gap

The TypeScript handler is shown inline, but there's no abstraction for language-specific scoping rules. Python has LEGB. Ruby has different block semantics. Go has package-level visibility.

The current design embeds JS semantics (Target::Scope, Target::Function, Target::Module) into the core resolution algorithm. Adding Python will require either:
- Generalizing this abstraction
- Duplicating the resolution logic per language

Neither is addressed in the architecture.

#### 6. README vs ARCHITECTURE Terminology

README says: "Single-pass tree traversal — One walk over the tree-sitter AST, period."

ARCHITECTURE shows: Single-pass with O(1) inline reference resolution.

**RESOLVED:** The implementation now matches the README claim. Reference resolution happens inline during the single AST walk via the binding table, not in a separate pass.

---

### Open Questions

1. **What's the typical reference-to-declaration ratio?** If files average 500 references and 50 declarations, resolution cost matters more than traversal cost.

2. **How often do files change by <10 lines between commits?** This determines whether incremental parsing is worth pursuing.

---

### Verdict

The core design is sound and well-reasoned. The main issues are:
- Declaration fragmentation (easily fixable)
- Missing incremental parsing consideration
- Incomplete error handling story
- Language abstraction needs future-proofing

The biggest gap is the lack of explicit error types for a "cannot crash" library.

---

## 14. Performance & Developer Experience Improvements

### Overview

The macro-level architecture is fundamentally correct — single-pass DFS traversal, SoA layout, numeric kind dispatch, O(1) binding resolution, fixed-size stacks. The issues identified below are all in the implementation details, where several design choices undermine the stated "zero allocation in hot path" goal, and where the tree-sitter Node API is used in ways that tree-sitter documentation explicitly warns against.

**7 performance issues** (2 critical, 3 medium, 2 low) and **5 developer experience issues** were identified. The critical performance issues are (1) per-scope heap allocations on every `create_scope()` call, and (2) pervasive use of `node.parent()` / `child_by_field_name()` / `kind()` string comparison in the visitor — all of which tree-sitter documentation identifies as performance pitfalls.

---

### Performance Issues

#### P0-1: Per-Scope Heap Allocation in Hot Path

**Severity:** Critical
**Location:** `builder.rs:236` (`create_scope` method)

```
CURRENT CODE:
self.build_declarations.push(Vec::new());
```

Every `create_scope()` call pushes a new empty `Vec<CompactDecl>`. The first `declare_symbol()` into that scope triggers a heap allocation via the system allocator. For a file with 50 scopes, that is up to 50 `malloc` calls during traversal.

This directly violates the stated constraint: "No heap allocation during traversal."

Compounding this, `clear()` at `builder.rs:156` calls `self.build_declarations.clear()` which **drops all inner Vecs**, destroying their allocated capacity. The thread-local reuse pattern does not help — every analysis starts from zero capacity.

```
ALLOCATION FLOW (current):

  create_scope()
      └─ build_declarations.push(Vec::new())   ← Vec struct on outer Vec, 0 capacity
          └─ declare_symbol() later
              └─ inner_vec.push(decl)           ← malloc() for inner Vec buffer
                  └─ (repeated for each scope)

  clear()
      └─ build_declarations.clear()             ← DROPS all inner Vecs, capacity LOST
```

**Root cause:** `Vec<Vec<CompactDecl>>` is a fragmented data structure. Each inner Vec is an independent heap allocation. There is no mechanism to retain inner Vec capacity across analysis calls.

**Fix — Flat build buffer:**

Replace `Vec<Vec<CompactDecl>>` with:

```rust
build_decl_data: Vec<(u16, CompactDecl)>,  // (scope_index, decl) pairs, flat
```

During traversal, push `(scope_idx, decl)` to the flat vec. In `take_result()`, group by scope_index to build `FlatDeclarations`. This eliminates all inner Vec allocations. The grouping step in `take_result()` is O(n) with a single pass using counting sort since scope indices are dense integers.

```
ALLOCATION FLOW (after fix):

  create_scope()
      └─ (no allocation — just record scope start index)

  declare_symbol()
      └─ build_decl_data.push((scope_idx, decl))  ← amortized O(1), no malloc

  clear()
      └─ build_decl_data.clear()                   ← capacity RETAINED
```

**Impact:** Eliminates up to N heap allocations per file (where N = number of scopes). For the batch processing use case (millions of files), this compounds to millions of eliminated malloc/free pairs.

---

#### P0-2: `node.parent()` in Hot Path Visitor

**Severity:** Critical
**Location:** `typescript.rs:101–102`

```rust
let is_var = node.parent()
    .is_some_and(|p| p.kind() == "variable_declaration");
```

**Background:** Tree-sitter removed the parent node cache to achieve full immutability for safe concurrent tree access. Since that change, `ts_node_parent()` must **traverse from the root of the tree every time** — it is O(depth). This is documented in [tree-sitter discussion #2018](https://github.com/tree-sitter/tree-sitter/discussions/2018):

> `goto_parent()` is O(1) because the cursor maintains an internal path from the root. By contrast, `ts_node_parent` (the Node API) must start from the top of the tree every time.

For every `variable_declarator` node in the source (every `let`, `const`, `var` statement), the current code pays O(depth) to walk from the root. For a 200-line file with 50 variable declarations at average depth 8, that is 50 × 8 = 400 node traversals just to check var-vs-let.

Compounding this: `.kind() == "variable_declaration"` then performs:
1. FFI call to `ts_node_kind()` to get a C string pointer
2. `CStr::from_ptr()` conversion to Rust `&str`
3. String comparison against "variable_declaration"

Meanwhile, the main dispatch already uses numeric `kind_id()` specifically to avoid this exact overhead.

**Fix option A — Pass parent kind_id from traversal (recommended):**

The traversal loop already has cursor context. Track the parent node's `kind_id` and pass it to the visitor function:

```
CURRENT visitor signature:
  fn visit_typescript(builder, node, source_bytes)

PROPOSED visitor signature:
  fn visit_typescript(builder, node, source_bytes, parent_kind_id: u16)
```

The traversal loop can cheaply maintain parent_kind_id by recording `cursor.node().kind_id()` before descending. The codegen in `traversal.rs` and `visitor.rs` would need updates to thread this value through.

In the visitor, the variable_declarator handler becomes:

```rust
// parent_kind_id is free — no FFI, no string comparison, no tree walk
const KIND_VARIABLE_DECLARATION: u16 = /* resolved at build time */;

if parent_kind_id == KIND_VARIABLE_DECLARATION {
    builder.declare_symbol(name, Target::Function, Timing::Scope);
} else {
    builder.declare_symbol(name, Target::Scope, Timing::Point);
}
```

**Fix option B — Handle at parent node level:**

Instead of checking the parent from the child, handle `variable_declaration` and `lexical_declaration` as distinct handler entries. Set a flag on the builder (e.g., `current_var_kind: VarKind`) when entering these nodes. Child `variable_declarator` nodes read the flag instead of calling `node.parent()`.

**Impact:** Eliminates O(depth) traversal + FFI + string comparison for every variable declaration. For files heavy in variable declarations (which is most real-world JS/TS), this is a significant win.

---

#### P1-1: `child_by_field_name()` String Lookup Throughout Visitor

**Severity:** High
**Location:** `typescript.rs` lines 32, 82–83, 96, 118, 162–163, 182, 200

The visitor calls `child_by_field_name()` with string literals in multiple handlers:

```
child_by_field_name("name")      — 4 call sites
child_by_field_name("pattern")   — 1 call site
child_by_field_name("alias")     — 1 call site
child_by_field_name("source")    — 3 call sites
```

Internally, `ts_node_child_by_field_name()` must:
1. Resolve the field name string to a field ID via `ts_language_field_id_for_name()`
2. Iterate through children to find the one with that field ID
3. Return the matching child node

**Step 1 is worse than a hash lookup — it is a LINEAR SCAN.** Examination of tree-sitter's C implementation (`language.c:232`) reveals `ts_language_field_id_for_name` iterates: `for (TSSymbol i = 1; i < count + 1; i++)` with `strncmp` against every field name in the grammar. The TypeScript grammar has dozens of field names. This is O(num_fields) string comparisons per call, not O(1).

With 9+ call sites across high-frequency handlers (`variable_declarator`, `required_parameter`, `function_declaration`, import handlers), the total linear-scan cost is substantial. `child_by_field_id(field_id)` skips this entire scan because the field ID is already resolved.

Tree-sitter provides `Language::field_id_for_name()` which can be called at build time — exactly like the existing `id_for_node_kind()` call in the codegen pipeline.

**Fix:**

In `codegen/mod.rs`, resolve field IDs alongside kind IDs:

```
CURRENT codegen resolves:
  "function_declaration" → 156  (via id_for_node_kind)

PROPOSED codegen also resolves:
  "name"    → 42  (via field_id_for_name)
  "pattern" → 17  (via field_id_for_name)
  "alias"   → 8   (via field_id_for_name)
  "source"  → 31  (via field_id_for_name)
```

Generate constants in the output:

```rust
const FIELD_NAME: u16 = 42;
const FIELD_PATTERN: u16 = 17;
const FIELD_ALIAS: u16 = 8;
const FIELD_SOURCE: u16 = 31;
```

Replace all `child_by_field_name("name")` with `child_by_field_id(FIELD_NAME)` in handler bodies. This requires either:
- Making the constants available in the generated code (straightforward)
- Updating the `language!` macro DSL to support field references

**Impact:** Eliminates string lookup overhead on 9+ call sites per node visit. The cost is one integer comparison instead of a string table scan.

---

#### P1-2: `node.kind() == "string"` Comparisons in Handlers

**Severity:** Medium
**Location:** `typescript.rs` lines 40, 50, 59, 84, 102

The visitor uses `node.kind()` string comparison for inline checks:

```rust
// Async detection (4 occurrences):
node.child(0).is_some_and(|c| c.kind() == "async")
node.child(1).is_some_and(|c| c.kind() == "async")

// Parameter pattern filtering:
.filter(|n| n.kind() == "identifier")

// Parent kind check:
.is_some_and(|p| p.kind() == "variable_declaration")
```

Each `node.kind()` call performs:
1. FFI call to `ts_node_kind()` → returns `const char *`
2. `CStr::from_ptr()` → scans for null terminator
3. `.to_str().unwrap()` → validates UTF-8
4. String equality comparison

Meanwhile, the main dispatch uses `node.kind_id()` which is a single FFI call returning `u16` — no string conversion, no comparison overhead. The main visitor match already proved this optimization is 30x faster per node.

**Fix:**

Generate numeric constants for all node kinds referenced in handler bodies, not just the ones used as match arms:

```rust
const KIND_ASYNC: u16 = /* resolved at build time */;
const KIND_IDENTIFIER: u16 = /* resolved at build time */;
const KIND_VARIABLE_DECLARATION: u16 = /* resolved at build time */;
```

Replace all `c.kind() == "async"` with `c.kind_id() == KIND_ASYNC`.

This requires the codegen pipeline to scan handler body strings for `kind()` comparisons and generate the corresponding constants. Alternatively, extend the `language!` macro to accept a `constants` block listing additional kind names to resolve.

**Impact:** Eliminates FFI + CStr + string comparison on every async detection, parameter filtering, and parent kind check. These run on high-frequency node types (parameters, variable declarators).

---

#### P1-3: `take_result()` Defeats Thread-Local Vec Capacity Reuse

**Severity:** Medium
**Location:** `builder.rs:124–128`

```rust
locations: std::mem::take(&mut self.scopes.locations),
metrics: std::mem::take(&mut self.scopes.metrics),
scores: std::mem::take(&mut self.scopes.scores),
...
function_scope: std::mem::take(&mut self.scopes.function_scope),
```

`std::mem::take()` on a `Vec<T>` replaces it with `Vec::new()` — a Vec with **zero capacity and zero allocation**. After `take_result()`, the builder holds 5+ empty Vecs with no allocated memory. The subsequent `clear()` call is a no-op on these empty Vecs. The next analysis must re-grow every Vec from scratch, triggering multiple reallocations as each Vec grows through capacities 1 → 2 → 4 → 8 → 16 → ...

```
CURRENT steady-state behavior (per file):

  clear()          — no-op (all Vecs are empty after previous take_result)
  create_scope()×N — Vecs grow: 0→1→2→4→8→... (log N reallocations each)
  take_result()    — Vecs moved out, builder left with 0-capacity Vecs
  (repeat)

  Result: Every file analysis pays O(log N) reallocations per Vec
```

The MEMORY.md states `take_result()` leaves "empty-but-allocated Vecs" — this is incorrect. `std::mem::take()` leaves empty-with-zero-allocation Vecs. The distinction is critical: "empty-but-allocated" would mean `len=0, capacity>0` (retaining buffer), while the actual behavior is `len=0, capacity=0` (no buffer).

**Fix — Swap-buffer pattern:**

Keep secondary "result buffers" in the builder. On `take_result()`:

```rust
// Move builder's active Vecs into result
let result_locations = std::mem::take(&mut self.scopes.locations);
// ... (build result)

// Swap in the previous result's buffers (which the caller returned)
// Or: just accept the allocation cost for result Vecs and instead
// clone data out, keeping the builder's capacity intact
```

Simpler alternative — clone-and-clear:

```rust
pub fn take_result(&mut self) -> AnalysisResult {
    let scopes = Scopes {
        locations: self.scopes.locations.clone(),
        metrics: self.scopes.metrics.clone(),
        // ... etc
    };
    // Builder retains its allocated capacity
    // clear() will set len=0 but keep capacity
    return AnalysisResult { scopes, imports };
}
```

The clone approach duplicates memory briefly, but eliminates all reallocation on subsequent calls. For a tight loop processing millions of files, the amortized cost of retaining capacity vastly outweighs one extra memcpy per file.

**Best alternative — Collect into fresh Vecs for result, keep builder Vecs:**

```rust
pub fn take_result(&mut self) -> AnalysisResult {
    // Build FlatDeclarations (already allocates fresh Vecs)
    let declarations = self.flatten_declarations();

    // Collect into new Vecs for the result — these are fresh allocations
    // but unavoidable since the result needs to own its data
    let scopes = Scopes {
        locations: self.scopes.locations.drain(..).collect(),
        metrics: self.scopes.metrics.drain(..).collect(),
        scores: self.scopes.scores.drain(..).collect(),
        declarations,
        function_scope: self.scopes.function_scope.drain(..).collect(),
    };
    // ...
}
```

`drain(..).collect()` creates a new Vec from the elements, then the `drain` leaves the original Vec empty **but with its capacity intact**. This is the key difference from `mem::take`.

```
FIXED steady-state behavior (per file):

  clear()          — sets len=0, capacity RETAINED from previous run
  create_scope()×N — Vecs already have capacity ≥ N (no reallocation)
  take_result()    — drain into fresh result Vecs, builder keeps capacity
  (repeat)

  Result: After warmup, ZERO reallocations in the builder's internal Vecs
```

**Impact:** Eliminates O(log N) reallocations per Vec per file after the first analysis. For 5+ Vecs growing to typical sizes of 20–100 elements, this saves 25–35 reallocations per file. At millions of files per second, this is significant.

---

#### P2-1: Import Handlers Chain `parent().parent()`

**Severity:** Low
**Location:** `typescript.rs:168–169, 185–186, 201–202`

```rust
let source = node.parent()
    .and_then(|p| p.parent()) // import_clause -> import_statement
    .and_then(|p| p.child_by_field_name("source"))
```

Three import handlers (`import_specifier`, `namespace_import`, `import_clause`) each chain two `parent()` calls to reach the `import_statement` node and get its `source` field. Each `parent()` is O(depth) as discussed in P0-2. For files with many imports (React components commonly have 15–30 import statements with multiple specifiers each), this is 2 × O(depth) × specifier_count tree traversals.

**Fix:** Same approach as P0-2. If the traversal loop passes parent context (parent_kind_id, or a small ancestor stack), import handlers can access the grandparent's data without calling `node.parent()`. Alternatively, handle imports at the `import_statement` level and iterate children using the cursor, avoiding upward tree navigation entirely.

**Impact:** Low in isolation (imports are a small fraction of total nodes), but the fix is free if P0-2 is already implemented.

---

#### P2-2: No Release Profile Optimizations

**Severity:** Low (but trivial to fix)
**Location:** Root `Cargo.toml` — missing `[profile.release]` section

No `[profile.release]` exists in any Cargo.toml in the workspace. For a library whose stated goal is "absolute fastest in its category," the default release profile leaves significant performance on the table.

**Missing settings:**

```toml
[profile.release]
lto = "fat"           # Cross-crate link-time optimization
codegen-units = 1     # Single LLVM module — better optimization
strip = "symbols"     # Smaller binary, no debug info overhead
panic = "abort"       # Eliminates unwinding tables and landing pads
```

**Why LTO is critical for this specific codebase:**

The generated code (emitted by `quickscope-build`, compiled into the `quickscope` crate) calls methods on `ScopeBuilder` which is defined in a different compilation unit. Without LTO, the Rust compiler treats crate boundaries as optimization barriers. `#[inline(always)]` on `ScopeBuilder` methods is a **hint that requires LTO to work across crates** — without LTO, the linker may not inline these calls even with the annotation.

```
WITHOUT LTO:
  generated visitor → call increment_branch()  ← function call (not inlined)
                    → call create_scope()       ← function call (not inlined)
                    → call reference()          ← function call (not inlined)

WITH LTO ("fat"):
  generated visitor → [increment_branch body inlined]  ← zero call overhead
                    → [create_scope body inlined]       ← zero call overhead
                    → [reference body inlined]          ← zero call overhead
```

Since the traversal loop calls these methods millions of times per second, eliminating function call overhead (push/pop frame, argument passing, return) compounds massively.

**Why `codegen-units = 1`:**

The default is 16 codegen units for parallel compilation. Each unit is optimized independently, preventing cross-unit inlining and constant propagation. Setting to 1 allows LLVM to see the entire crate as a single module, enabling global optimization passes.

**Why `panic = "abort"`:**

Eliminates unwinding tables (`.eh_frame` section) and landing pads at every function that could panic. Since the library states "cannot crash the server" and handles all errors explicitly, unwinding is unnecessary overhead.

**Benchmark profile should also be optimized:**

```toml
[profile.bench]
lto = "fat"
codegen-units = 1
```

Without this, benchmark results do not reflect real-world release performance, potentially leading to incorrect optimization decisions.

**Impact:** Typically 10–25% improvement from LTO alone on crates with heavy cross-crate inlining. Combined with codegen-units=1, improvements of 15–30% are commonly reported for compute-bound Rust code.

---

### Developer Experience Issues

#### DX1: Handler Bodies Are Opaque Strings

**Severity:** High
**Location:** `quickscope-core/src/lib.rs:85` (the `language!` macro)

The `language!` macro uses `stringify!` to capture handler bodies as strings:

```rust
body: stringify!($body),  // stored as &'static str, parsed later by codegen
```

The handler bodies in `typescript.rs` are real Rust syntax that the compiler parses as part of the macro invocation — but they are immediately serialized to strings and discarded. The codegen pipeline in `quickscope-build` later parses these strings back into token streams for emission.

**Consequences:**

1. **No IDE support.** Language servers (rust-analyzer) cannot provide autocomplete, type checking, go-to-definition, or hover information inside handler bodies. The code appears as an opaque block to tooling. Every language definition author works effectively blind.

2. **Error messages point to generated code.** If a handler body has a type error (e.g., calling a nonexistent method on `builder`), the compiler error points to the generated `$OUT_DIR/quickscope.rs` file, not to the handler in `typescript.rs`. Developers must manually correlate generated code back to the source.

3. **Refactoring tools cannot see into handlers.** Renaming `builder.create_scope()` to `builder.open_scope()` across the codebase will miss every handler body, because they are strings at the macro level.

4. **No syntax highlighting** in most editors. The handler bodies appear as macro arguments, not as Rust code blocks.

**Fix — Proc-macro approach:**

Replace the declarative `language!` macro with a proc-macro that:
1. Type-checks the handler bodies against the real `ScopeBuilder` API at the definition site
2. Extracts the handler bodies as token streams for codegen
3. Emits a dummy function (gated behind `#[cfg(never)]`) that references all types used in handlers, giving IDEs something to analyze

This is a significant refactor but would transform the language definition experience from "write code in a string" to "write normal Rust code with full tooling support."

**Lower-effort alternative — Generated code inspection:**

Add a cargo feature or build-time environment variable that writes the generated code to a known, version-controlled location:

```
QUICKSCOPE_DUMP_GENERATED=1 cargo build
→ writes to crates/quickscope/src/generated_debug.rs
```

This does not fix the IDE problem but makes debugging significantly easier.

---

#### DX2: No Error Types

**Severity:** High
**Location:** `lib.rs:56` (generated `analyze()` return type)

```rust
pub fn analyze(file: &File) -> Option<AnalysisResult>
```

`Option<AnalysisResult>` provides zero information about failure. For a library that states it "cannot crash the server" and requires "explicit error handling" with "all error paths typed and documented" (AGENTS.md, README), returning `None` for all failure modes is insufficient.

**Failure modes that currently all return `None`:**

| Failure | Current behavior | What the caller needs |
|---------|-----------------|----------------------|
| Unsupported file extension | `None` | "Extension '.xyz' not supported" |
| Tree-sitter parse failure | `None` | "Parser returned no tree" (rare, indicates OOM or corrupt input) |
| Depth limit exceeded | Silent truncation | Warning that metrics are incomplete |
| Scope count exceeds u16::MAX | `debug_assert` (release: silent) | Error with scope count |

**Fix:**

```rust
pub enum AnalysisError {
    /// File extension is not supported by any registered language.
    UnsupportedExtension,
    /// Tree-sitter failed to produce a parse tree.
    ParseFailed,
}

pub fn analyze(file: &File) -> Result<AnalysisResult, AnalysisError>
```

Depth limiting is not an error (it is graceful degradation with the `depth_limited` flag), but the other failure modes should be distinguishable for monitoring and debugging.

**Impact on callers:**

```rust
// CURRENT (caller cannot distinguish failure modes):
if let Some(result) = analyze(&file) { ... }

// PROPOSED (caller can handle appropriately):
match analyze(&file) {
    Ok(result) => { /* process */ },
    Err(AnalysisError::UnsupportedExtension) => { /* skip, expected */ },
    Err(AnalysisError::ParseFailed) => { /* log warning, investigate */ },
}
```

---

#### DX3: `File` Forces Allocation

**Severity:** Medium
**Location:** `types.rs:275–278`

```rust
pub struct File {
    pub path: String,
    pub content: String,
}
```

Every call to `analyze()` requires the caller to own `String` values for both path and content. In the server use case (processing files from git), the content is typically available as `&[u8]` or `&str` from a memory-mapped file or git blob. Requiring `String` ownership forces an unnecessary allocation and copy of potentially large file contents.

**Fix:**

```rust
pub struct File<'a> {
    pub path: &'a str,
    pub content: &'a str,
}
```

The parser needs `&str` internally (`parser.parse(&file.content, None)` takes `impl AsRef<[u8]>`). Borrowing from the caller eliminates one allocation per file — for a library processing millions of files per second, this removes millions of unnecessary string copies.

If some callers do have owned strings, they can pass `&owned_string` trivially — going from borrowed to owned requires allocation, but going from owned to borrowed is free.

---

#### DX4: Thin Test Coverage

**Severity:** Medium
**Location:** `lib.rs:69–173`

Only 6 tests exist, all basic happy-path assertions:

```
test_analyze_simple_function      — checks scopes.len() >= 2
test_analyze_nested_scopes        — checks scopes.len() >= 3
test_analyze_branches             — checks total branches >= 3
test_analyze_async_function       — checks any is_async == true
test_analyze_tsx_file             — checks scopes.len() >= 2
test_analyze_unknown_extension    — checks result.is_none()
```

**Not tested (critical gaps):**

| Category | What is missing |
|----------|----------------|
| Import tracking | Import source paths, usage counts, multiple specifiers from same source |
| Reference resolution | Does `bindings[symbol]` resolve correctly? Are import counts accurate? |
| Var hoisting | `var` inside function shadowing an import — does `pending_var_refs` correction work? |
| Shadowing | `let` shadowing import in block, restoration on scope exit |
| FlatDeclarations | Do flattened ranges match the original per-scope declarations? |
| Edge cases | Empty files, files with only comments, files with 128+ nested scopes |
| Declaration counts | Does output declaration count match actual declarations in source? |
| Depth limiting | Is `depth_limited` flag set correctly at MAX_SCOPE_DEPTH? |
| Metrics accuracy | Are specific metric counts (call_count, branch_count) exact, not just >= N? |

The README states "Thorough testing — Comprehensive test coverage for all languages and edge cases." The current tests do not meet this bar.

**Fix:**

Add targeted tests for each subsystem:

1. **Import tracking tests:** Verify exact import counts after analysis. Test `import { a, b } from 'x'; a(); a(); b();` yields source "x" with count 3.

2. **Hoisting tests:** Test `import { foo } from 'x'; function f() { foo(); var foo = 1; }` yields source "x" with count 0 (var shadows the import, pending refs corrected).

3. **Shadowing tests:** Test `import { x } from 'y'; if (true) { let x = 1; x; } x;` yields count 1 (only the outer `x` reference counts, the inner one is shadowed by `let`).

4. **Declaration tests:** Verify exact FlatDeclarations output matches expected per-scope declarations.

5. **Depth limit tests:** Construct input with 130+ nested blocks, verify `depth_limited` is set on the appropriate scope.

---

#### DX5: No Way to Inspect Generated Code Easily

**Severity:** Low
**Location:** `quickscope-build/src/lib.rs:27` (writes to `$OUT_DIR`)

The generated `quickscope.rs` is written to `$OUT_DIR` which resolves to a deep path inside `target/`:

```
target/debug/build/quickscope-<hash>/out/quickscope.rs
```

For debugging visitor dispatch, understanding the generated traversal loop, or verifying that kind_id values are correct, developers must manually find this file. The hash changes on rebuilds, making it harder to locate.

**Fix:**

Add a build-time environment variable that copies the generated code to a known location:

```rust
// In quickscope-build/src/lib.rs::run()
if std::env::var("QUICKSCOPE_DUMP_GENERATED").is_ok() {
    std::fs::copy(&dest_path, "crates/quickscope/src/generated_debug.rs").ok();
}
```

Usage: `QUICKSCOPE_DUMP_GENERATED=1 cargo build`

Add `generated_debug.rs` to `.gitignore` so it is not committed.

---

### Architectural Observations

#### Score Struct Is Unused

`Score { own: f32, max: f32 }` is initialized to zero in `Score::default()` and **never modified** during traversal. No code computes or sets scores. The `scores: Vec<Score>` in `Scopes` is allocated and pushed to on every `create_scope()` but contains only zeros.

```
CURRENT: Every create_scope() pushes Score { own: 0.0, max: 0.0 }
         8 bytes allocated, written, and returned — containing no information
```

If scoring is a planned future feature, it should be behind a feature flag or computed post-hoc. Currently it wastes 8 bytes per scope in allocation, in the hot path push, and in the output data. It also pollutes the cache when iterating adjacent arrays in the SoA layout.

#### BuildDeclarations vs FlatDeclarations: Two Representations for One Concept

The codebase maintains `BuildDeclarations` (`Vec<Vec<CompactDecl>>`) during traversal, then converts to `FlatDeclarations` (flat `Vec<CompactDecl>` + `Vec<DeclRange>`) in `take_result()`. This conversion iterates all declarations, counts total size, allocates two new Vecs, and copies everything.

A unified approach — building directly into the flat format — would eliminate this entire conversion step. The challenge is interleaved declarations across scopes (declarations for scope 0 and scope 1 are temporally interleaved), but this can be solved with the flat build buffer approach described in P0-1.

#### Language Abstraction Is JS-Specific

`Target::Scope / Function / Module` and the hoisting/TDZ semantics in the builder are hardcoded to JavaScript's scoping model. Other languages have fundamentally different scope resolution rules:

| Language | Scoping model | Mismatch with current Target enum |
|----------|--------------|-----------------------------------|
| Python | LEGB (Local, Enclosing, Global, Built-in) | No "Enclosing" or "Global" target |
| Go | Package-level + block scope | No "Package" target |
| Ruby | Method scope + block scope (different rules) | Block semantics differ from JS |
| Rust | Module + function + block (no hoisting) | No hoisting needed, Module means something different |
| C/C++ | File scope + function + block | No module concept |
| Java/Kotlin | Class scope + method + block | No class-level target |

When adding the 16 planned languages, the builder will need either:
- **Generalized scope resolution** with per-language scope rules
- **Per-language builder variants** (duplication)

Neither approach is addressed in the current architecture. Designing a more abstract scope resolution interface before adding the second language would avoid costly refactoring later.

#### The `is_function` Flag Mixes Metadata with Metrics

`Metrics.is_function` is checked on every `leave()` call (hot path) to decide whether to restore function scope. But it lives in the `Metrics` struct alongside collected counters like `branch_count` and `call_count`.

This mixes "scope classification" (structural metadata about what kind of scope this is) with "collected measurements" (runtime analysis data). A separate per-scope flags byte or bitfield — stored in its own parallel array — would:
1. Be semantically cleaner
2. Free 1 byte in Metrics for a future counter without changing alignment
3. Avoid loading the entire 16-byte Metrics struct when only checking `is_function` during `leave()`

---

### Prioritized Implementation Order

See [Unified Priority Table](#unified-priority-table) at the end of the document for the merged table across all sections.

---

---

## 15. Additional Performance, Correctness & Reliability Improvements

### Overview

The issues identified in Section 14 cover the tree-sitter API misuse (node.parent(), child_by_field_name(), kind() strings), per-scope allocation, Vec capacity loss, missing release profile, and DX gaps. This section identifies a second tier of improvements discovered through deeper analysis of the hot path, the traversal loop structure, and tree-sitter grammar semantics.

**7 performance issues** (2 high, 5 medium/low), **1 correctness bug**, and **3 reliability/operational improvements** were identified.

---

### Performance Issues

#### NP-1: Skip Unnamed AST Nodes in Traversal

**Severity:** High (comparable to P0-2)
**Location:** `quickscope-build/src/codegen/traversal.rs:65–144` (generated traversal loop)

In a typical TypeScript AST, **~60–70% of nodes are unnamed** — punctuation (`{`, `}`, `;`, `,`, `(`), keywords (`function`, `if`, `return`), operators (`=`, `+`). The current traversal visits every single one:

```
traversal.rs:68–82 (generated):
    let node = cursor.node();                              // FFI call
    let before = builder.scopes.locations.len() as u16;    // Vec::len()
    visit_fn(builder, &node, source_bytes);                // kind_id() FFI + match → _ => {}
    let created = builder.scopes.locations.len() as u16 > before;  // Vec::len()
    let can_track = ...;
    let tracked = created && can_track;
    builder.scope_stack_created.push(tracked);             // bitfield write
```

For unnamed nodes, the visitor always falls through to `_ => {}`. No handler in `typescript.rs` targets unnamed kinds. The overhead per unnamed node: 2 FFI calls (`cursor.node()`, `kind_id()`), 2 `Vec::len()` reads, a match dispatch, and a `CreatedStack::push(false)`.

**Background:** `ts_node_is_named()` is O(1) — a single metadata lookup from the node's type data. It is cheaper than `kind_id()` + match dispatch. The TreeCursor API does not provide a named-only traversal mode; checking must be done manually after `cursor.node()`.

**Fix:**

Check `node.is_named()` before the visitor call. The `scope_stack_created.push(false)` is still required to maintain depth tracking balance, but everything else is skipped:

```rust
let node = cursor.node();
if node.is_named() {
    let before = builder.scopes.locations.len() as u16;
    #visit_fn_name(builder, &node, source_bytes);
    let created = /* ... */;
    // ... tracking logic
} else {
    builder.scope_stack_created.push(false);
}
```

This eliminates: the visitor dispatch, both `len()` reads, the `created`/`tracked` computation, and the `kind_id()` FFI call for all unnamed nodes.

**Impact:** For a file with 1000 nodes where ~650 are unnamed, this eliminates ~650 visitor dispatches + ~1300 FFI calls + ~1300 Vec reads. At millions of files/second, this is substantial.

---

#### NP-2: Eliminate Redundant `Option` Check in All `increment_*` Methods

**Severity:** High
**Location:** `builder.rs:271–368` (all 12 `increment_*` methods + `set_async`)

All 12 `increment_*` methods plus `set_async()` repeat this pattern:

```rust
#[inline(always)]
pub fn increment_argument(&mut self) {
    if let Some(idx) = self.scope_stack_indices.last() {  // branch every call
        let m = &mut self.scopes.metrics[idx as usize];
        m.argument_count = m.argument_count.wrapping_add(1);
    }
}
```

After the `program` node creates the root scope (the very first thing the traversal does), the IndexStack is **never empty** for the remainder of traversal. Every handler that calls `increment_*` executes inside a scope. The `if let Some` branch is taken 100% of the time after initialization — it is a dead branch consuming a branch prediction slot millions of times.

**Fix:**

Store `current_scope_idx: u16` directly on `ScopeBuilder`. Update it on `IndexStack::push()` and `IndexStack::pop()`. Replace all `if let Some(idx)`:

```rust
#[inline(always)]
pub fn increment_argument(&mut self) {
    let m = &mut self.scopes.metrics[self.current_scope_idx as usize];
    m.argument_count = m.argument_count.wrapping_add(1);
}
```

This eliminates 1 branch + 1 function call (`last()`) per increment, across 13 methods. In a file with 200 calls, 100 branches, 50 assignments, etc., that is hundreds of eliminated branches per file.

**Trade-off:** Slightly more state to maintain (update `current_scope_idx` on push/pop). But push/pop are far less frequent than increments.

---

#### NP-3: `reference()` Pollutes Names HashMap for Unbound Identifiers

**Severity:** Medium-High
**Location:** `builder.rs:428–431`

```rust
pub fn reference(&mut self, name: &'a str) {
    let next_idx = self.names.len() as u16;
    let symbol = *self.names.entry(name).or_insert(next_idx);  // ALWAYS inserts
    // ...
    if idx >= self.bindings.len() { return; }  // Immediately bails for unbound
}
```

For every identifier — including builtins (`console`, `Math`, `JSON`, `undefined`, `Promise`, `Array`...), globals (`window`, `document`, `process`), and any referenced-but-never-declared name — this:
1. Hashes the name string
2. Looks up in AHashMap
3. **Inserts a new entry** if not found
4. Then immediately returns because `idx >= self.bindings.len()`

The HashMap grows with useless entries, degrading future lookups via increased load factor. In a typical React component with 30 imports but 200 identifier references, perhaps 100+ are to builtins/globals that are never declared.

**Correctness analysis:** Changing `reference()` to not insert is semantically identical. If a name hasn't been declared (not in `names`), it is unbound. Forward references before declaration (JS hoisting) already don't count in the current code for the same reason — the binding table has no entry. The declaration later creates the names entry via `declare_symbol()`.

**Fix:**

Use `names.get()` for references. Only `declare_symbol()` and `declare_import()` should insert:

```rust
pub fn reference(&mut self, name: &'a str) {
    let Some(&symbol) = self.names.get(name) else { return };  // No insert
    let idx = symbol as usize;
    if idx >= self.bindings.len() { return; }
    // ... rest unchanged
}
```

**Impact:** Eliminates HashMap insert for every unbound identifier. Also keeps the HashMap smaller, improving lookup speed for bound identifiers via lower load factor.

---

#### NP-4: Redundant UTF-8 Validation in Every `utf8_text()` Call

**Severity:** Medium
**Location:** `typescript.rs` — 11+ call sites across all handlers

Each `node.utf8_text(source_bytes)` call executes:

```rust
// tree-sitter's implementation:
pub fn utf8_text<'a>(&self, source: &'a [u8]) -> Result<&'a str, Utf8Error> {
    std::str::from_utf8(&source[self.start_byte()..self.end_byte()])
}
```

`from_utf8()` scans every byte for UTF-8 validity. But `source_bytes` came from `file.content.as_bytes()` where `content: String` — it is **already valid UTF-8**. Any byte range within a valid UTF-8 string is also valid UTF-8 (tree-sitter returns byte offsets at codepoint boundaries for named nodes).

For a file with 500 identifier references averaging 8 bytes each, that is 4000 bytes of unnecessary byte-by-byte validation per file.

**Fix:**

In the codegen, replace `node.utf8_text(source_bytes)` with a direct unchecked slice:

```rust
// Safety: source_bytes originates from a String (&str), which is always valid UTF-8.
// tree-sitter byte offsets for named nodes align to codepoint boundaries.
unsafe {
    std::str::from_utf8_unchecked(&source_bytes[node.start_byte()..node.end_byte()])
}
```

This could be implemented as a helper function emitted by the codegen pipeline, or by rewriting `utf8_text` usages in handler body strings during code generation. The helper approach is cleaner:

```rust
// Emitted once in generated code:
#[inline(always)]
fn node_text<'a>(node: &tree_sitter::Node, source: &'a [u8]) -> &'a str {
    unsafe { std::str::from_utf8_unchecked(&source[node.start_byte()..node.end_byte()]) }
}
```

**Impact:** Eliminates per-byte validation for every identifier, name, parameter, and import source string access. The savings scale linearly with identifier density.

---

#### NP-5: Merge Parallel Shadow Vecs into Single Vec

**Severity:** Low-Medium
**Location:** `builder.rs:58–60, 535–543`

```rust
shadow_symbols: Vec<u16>,
shadow_bindings: Vec<SymbolBinding>,
```

These are **always** pushed together, popped together, and iterated together in `leave_scope()`:

```rust
while self.shadow_symbols.len() > boundary {
    let symbol = self.shadow_symbols.pop().unwrap();
    let old_binding = self.shadow_bindings.pop().unwrap();
    self.bindings[symbol as usize] = old_binding;
}
```

Two parallel Vecs means: 2 pushes per declaration, 2 pops per scope-exit restore, 2 capacity checks per push, 2 `len()` reads per loop iteration, and 2 `pop().unwrap()` calls with Option overhead per iteration.

**Fix:**

Single `Vec<(u16, SymbolBinding)>`:

```rust
shadow_stack: Vec<(u16, SymbolBinding)>,  // (symbol, old_binding)
```

`(u16, SymbolBinding)` = 4 bytes total — same as the separate entries but now contiguous. Each push/pop is a single operation on one Vec. The restore loop becomes:

```rust
while self.shadow_stack.len() > boundary {
    let (symbol, old_binding) = self.shadow_stack.pop().unwrap();
    self.bindings[symbol as usize] = old_binding;
}
```

**Impact:** Halves push/pop overhead and improves spatial locality during the leave_scope() restore loop.

---

#### NP-6: Pre-Allocate Parallel Vecs Based on File Size

**Severity:** Medium
**Location:** `builder.rs:80–103` (ScopeBuilder::new), codegen traversal

All parallel arrays in `Scopes` (locations, metrics, scores, function_scope) plus `parents` and `build_declarations` start from 0 capacity on every analysis. Even after P1-3 (Vec capacity fix) is implemented, the **first analysis** on each thread still pays full reallocation cost as Vecs grow through capacities 0 → 1 → 2 → 4 → 8 → 16 → ...

**Fix:**

Add a `reserve_estimate()` method called once at start of traversal based on source size:

```rust
fn reserve_estimate(&mut self, byte_count: usize) {
    // Empirical: ~1 scope per 150 bytes in typical JS/TS
    let estimate = (byte_count / 150).max(4);
    self.scopes.locations.reserve(estimate);
    self.scopes.metrics.reserve(estimate);
    self.scopes.scores.reserve(estimate);
    self.scopes.function_scope.reserve(estimate);
    self.parents.reserve(estimate);
    self.build_declarations.reserve(estimate);
}
```

Call from generated traversal code after `builder.clear()`:

```rust
builder.clear();
builder.reserve_estimate(file.content.len());
```

After the first file, the thread-local builder retains capacity (assuming P1-3 is fixed), so this primarily helps the cold-start case and files significantly larger than the previous one.

**Impact:** Eliminates O(log N) reallocations per Vec on cold start. For 6 parallel Vecs growing to typical size 30, that is ~30 eliminated reallocations on first use per thread.

---

#### NP-7: Add `#[cold]` on Unlikely Code Paths

**Severity:** Low
**Location:** `builder.rs:198, 253`, `traversal.rs:85–87`

Several functions are rarely executed but live alongside hot code:
- `mark_depth_limited()` — only at depth 127+ (pathological)
- `restore_function_scope()` — only when leaving function scopes (parent chain walk)
- The `if created && !tracked` path in traversal — only at depth overflow

Without `#[cold]`, LLVM may interleave their machine code with hot-path instructions, polluting the instruction cache.

**Fix:**

Add `#[cold]` attribute to these functions:

```rust
#[cold]
pub fn mark_depth_limited(&mut self) { ... }

#[cold]
fn restore_function_scope(&mut self, leaving_scope: u16) { ... }
```

This tells LLVM to place them in a separate text section, improving I-cache utilization for the main traversal loop.

**Complementary: `core::hint::cold_path()` for inline cold branches.**

`core::hint::cold_path()` ([stabilization PR #151576](https://github.com/rust-lang/rust/pull/151576), in Final Comment Period as of Feb 2026) provides inline cold-path marking without requiring function extraction. This is especially valuable in `#[inline(always)]` methods where extracting a `#[cold]` function would break the inlining benefit:

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.pop() {
        let raw = self.scope_stack_indices.pop();
        if raw & 0x8000 != 0 {  // is_function (per NI-2)
            core::hint::cold_path();  // ← LLVM moves this block out of line
            self.leave_function_scope();
        } else {
            self.leave_scope();
        }
    }
}
```

LLVM moves the cold block out of the hot path's instruction stream, improving I-cache utilization without the code-organization overhead of extracting cold functions. Use `#[cold]` on standalone functions (stable today) and `cold_path()` inside branches of inlined methods (stabilizing soon).

**Impact:** Modest I-cache improvement. Trivial effort.

---

#### BC-1: Bounds Check Elimination via `assert_unchecked`

**Severity:** High
**Location:** `builder.rs:296–341` (all `increment_*` methods), generated traversal loop
**Depends on:** NP-2 (guaranteed valid scope index)

Every `increment_*` method indexes into `self.scopes.metrics[]` with a runtime bounds check:

```rust
pub fn increment_call(&mut self) {
    if let Some(scope_idx) = self.scope_stack_indices.last() {
        self.scopes.metrics[scope_idx as usize].call_count =
            self.scopes.metrics[scope_idx as usize].call_count.wrapping_add(1);
    }
}
```

After NP-2 eliminates the `Option` check by caching the scope index, the code becomes:

```rust
pub fn increment_call(&mut self) {
    self.scopes.metrics[self.current_scope as usize].call_count =
        self.scopes.metrics[self.current_scope as usize].call_count.wrapping_add(1);
}
```

LLVM emits a bounds check on every `metrics[idx]` access — a compare + conditional branch to a panic path. In the traversal hot loop, this adds a branch per `increment_*` call (5–15 per node depending on node type).

The priority table originally suggested `get_unchecked()`. A safer alternative is `core::hint::assert_unchecked` (stabilized in Rust 1.81), which communicates the invariant to LLVM via `llvm.assume` without switching to unsafe indexing:

**Fix:**

Add `assert_unchecked` at the start of each `increment_*` method (or once in the traversal loop after the scope index is known to be valid):

```rust
pub fn increment_call(&mut self) {
    let idx = self.current_scope as usize;
    // Safety: scope_stack_indices only contains indices that were pushed
    // by create_scope(), which appends to metrics first. The index is
    // always in bounds while the scope is on the stack.
    unsafe { core::hint::assert_unchecked(idx < self.scopes.metrics.len()) };
    self.scopes.metrics[idx].call_count =
        self.scopes.metrics[idx].call_count.wrapping_add(1);
}
```

Or, more efficiently, assert once at the start of the visitor call in the traversal loop (after `enter_scope` ensures `current_scope` is valid):

```rust
// In generated traversal, after scope tracking:
unsafe {
    core::hint::assert_unchecked(
        (builder.current_scope as usize) < builder.scopes.metrics.len()
    );
}
visit_fn(builder, &node, source_bytes);
```

This communicates to LLVM that the index is always in bounds for the duration of the visitor call, allowing it to elide ALL bounds checks in the `increment_*` methods without any unsafe indexing.

**Why `assert_unchecked` over `get_unchecked`:**

| Approach | Safety | Effect | Code change |
|---|---|---|---|
| `get_unchecked()` | Requires `unsafe` at every call site | Bypasses bounds check entirely — UB if wrong | Pervasive unsafe |
| `assert_unchecked()` | Single `unsafe` at proof point | LLVM elides checks via `llvm.assume` — same codegen | Localized unsafe |

`assert_unchecked` keeps safe indexing syntax (`metrics[idx]`) while achieving the same codegen. The unsafe boundary is at the invariant assertion, not at every array access. In debug builds, the assertion is still checked (it becomes a regular assert), catching bugs early.

**Impact:** Eliminates one compare + conditional branch per `increment_*` call. For a typical named node that triggers 2–3 increments, this removes 2–3 branches per node. Combined with NP-2 (which removes the Option check), each increment becomes a single `wrapping_add` with no branching.

---

### Correctness Issue

#### NC-1: Wrong Grammar for JSX/TSX Files

**Severity:** High (correctness bug)
**Location:** `quickscope-languages/src/typescript.rs:22`

```rust
language: tree_sitter_typescript::LANGUAGE_TYPESCRIPT,
```

This grammar is used for ALL 8 extensions including `.tsx` and `.jsx`. But tree-sitter-typescript provides **two separate grammars** because, quoting the [tree-sitter-typescript README](https://github.com/tree-sitter/tree-sitter-typescript): "Because TSX and TypeScript are actually two different dialects, this module defines two grammars."

`LANGUAGE_TYPESCRIPT` does **not** parse JSX syntax — angle brackets are interpreted as type assertions/comparison operators, not JSX elements. The `test_analyze_tsx_file` test passes by accident because tree-sitter produces a partial AST with error nodes for the JSX portion while the surrounding `function_declaration` is still recognized.

**Fix:**

Split into two language definitions — one for TypeScript files, one for TSX/JSX files:

```rust
pub static LANGUAGE_TS: Language = language!(
    name: "TypeScript",
    extensions: ["ts", "cts", "mts"],
    language: tree_sitter_typescript::LANGUAGE_TYPESCRIPT,
    handlers: { /* ... same handlers ... */ }
);

pub static LANGUAGE_TSX: Language = language!(
    name: "TSX",
    extensions: ["tsx", "jsx", "js", "cjs", "mjs"],
    language: tree_sitter_typescript::LANGUAGE_TSX,
    handlers: { /* ... same handlers ... */ }
);
```

Plain `.js` files should use the TSX grammar since JSX is common in JS build systems and the TSX grammar is a superset. The handler bodies are identical — only the grammar differs.

**Impact:** Correct AST parsing for all JSX/TSX files. Without this fix, any JSX syntax in `.tsx`/`.jsx` files produces error nodes, causing missed scopes, missed declarations, and incorrect metrics.

---

### Reliability Improvements

#### NR-1: No Parser Timeout for Pathological Input

**Severity:** Medium
**Location:** Generated traversal code (parser usage)

tree-sitter provides `Parser::set_timeout_micros()`. For the stated requirement "cannot crash the server" (README, AGENTS.md), unbounded parse time on malicious or pathological input (extremely deeply nested code, massive files, adversarial grammar exploits) is a risk. A single file could block a worker thread indefinitely.

**Fix:**

Set a timeout during parser initialization in the generated thread-local:

```rust
static PARSER_TYPESCRIPT: UnsafeCell<Parser> = {
    let mut parser = Parser::new();
    parser.set_language(...).expect("Failed to set language");
    parser.set_timeout_micros(100_000);  // 100ms max parse time
    UnsafeCell::new(parser)
};
```

When the timeout is exceeded, `parser.parse()` returns `None`, which the generated code already handles by returning `None` from `analyze()`. With the proposed error types from DX2, this would become `Err(AnalysisError::ParseTimeout)`.

**Impact:** Prevents any single file from blocking a worker thread. Aligns with the "cannot crash the server" guarantee.

---

#### NR-2: No Cancellation Support for Graceful Shutdown

**Severity:** Low
**Location:** Generated traversal code (parser usage)

tree-sitter provides `Parser::set_cancellation_flag()` which accepts a reference to an `AtomicUsize`. The server could set this flag during shutdown to cooperatively cancel in-progress parses rather than killing threads or waiting for them to finish.

**Fix:**

Accept an optional `&AtomicUsize` in the `analyze()` function signature (or set it per-thread during initialization). When the flag is set, `parser.parse()` returns `None`.

**Impact:** Enables graceful shutdown of worker threads processing large files.

---

#### NO-1: Profile-Guided Optimization (PGO) Build Process

**Severity:** Medium (operational)

For a performance-critical library at this optimization level, PGO can yield 10–20% additional improvement by:
- Optimizing branch prediction based on real workload profiles
- Laying out hot code contiguously in memory for I-cache efficiency
- Making inlining decisions informed by actual call frequency

This is complementary to the release profile (P2-2). Document the PGO build process:

```bash
# Step 1: Build with instrumentation
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# Step 2: Run representative workload (benchmark or batch of real files)
./target/release/quickscope-bench ...

# Step 3: Merge profile data
llvm-profdata merge -o /tmp/pgo-data/merged.profdata /tmp/pgo-data/

# Step 4: Rebuild with profile data
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

**Impact:** Typically 10–20% improvement on branch-heavy code. Compounds with all other optimizations.

---

### Prioritized Implementation Order

See [Unified Priority Table](#unified-priority-table) at the end of the document for the merged table across all sections.

---

### References

- [tree-sitter Discussion #878: Most optimized traversal pattern](https://github.com/tree-sitter/tree-sitter/discussions/878) — Do not mix Node API and Cursor API
- [tree-sitter Discussion #2018: Cursor vs Node speed](https://github.com/tree-sitter/tree-sitter/discussions/2018) — `node.parent()` traverses from root
- [tree-sitter TreeCursor docs](https://docs.rs/tree-sitter/latest/tree_sitter/struct.TreeCursor.html) — `goto_parent()` is O(1), `node.parent()` is O(depth)
- [tree-sitter-typescript README](https://github.com/tree-sitter/tree-sitter-typescript) — TSX and TypeScript are separate grammars
- [tree-sitter Node::is_named()](https://docs.rs/tree-sitter/latest/tree_sitter/struct.Node.html#method.is_named) — O(1) metadata lookup
- [tree-sitter Node::utf8_text()](https://docs.rs/tree-sitter/latest/tree_sitter/struct.Node.html#method.utf8_text) — Performs `from_utf8()` validation
- [oxc Performance](https://oxc.rs/docs/learn/performance) — Arena allocation, parse-order traversal for cache locality
- [Algorithmica: AoS and SoA](https://en.algorithmica.org/hpc/cpu-cache/aos-soa/) — When SoA vs AoS wins
- [Speeding up tree-sitter-haskell 50x](https://owen.cafe/posts/tree-sitter-haskell-perf/) — Allocations in external scanners dominate cost
- [Rust issue #80630: Loop/match state machine optimization](https://github.com/rust-lang/rust/issues/80630) — LLVM can miss optimizations in loop/match patterns
- [Nick Wilcox: Auto-Vectorization in Rust](https://www.nickwilcox.com/blog/autovec/) — Compiler matches hand-written SIMD in simple cases
- [SIMD Performance Guide: Vertical vs Horizontal](https://rust-lang.github.io/packed_simd/perf-guide/vert-hor-ops.html) — Vertical ops are fast, horizontal are slow
- [Rust PGO documentation](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) — Profile-guided optimization build process
- [`core::hint::assert_unchecked` docs](https://doc.rust-lang.org/std/hint/fn.assert_unchecked.html) — Communicates invariants to LLVM via `llvm.assume`, stabilized Rust 1.81
- [`core::hint::cold_path()` stabilization PR #151576](https://github.com/rust-lang/rust/pull/151576) — Inline cold-path hint, in FCP as of Feb 2026

---

---

## 16. Further Performance, Correctness & DX Improvements

### Overview

Sections 14 and 15 identified 23 issues covering tree-sitter API misuse, per-scope allocation, Vec capacity loss, missing release profile, DX gaps, unnamed node skipping, Option elimination, HashMap pollution, UTF-8 validation, shadow Vec merging, pre-allocation, cold paths, JSX grammar, parser timeout, cancellation, and PGO. This section identifies a third tier of improvements discovered through deeper analysis of the generated traversal loop structure, the visitor calling convention, tree-sitter cursor capabilities, import semantics, and deployment toolchain.

**7 performance issues** (2 high, 5 medium/low), **1 correctness issue**, and **3 DX/operational improvements** were identified. The most architecturally significant are (1) eliminating the `len()` comparison for scope-creation detection by changing the visitor return type, (2) exploiting `cursor.field_id()` to eliminate ALL child navigation from visitors entirely, (3) LT-1 const lookup table for O(1) early-exit on unhandled kind_ids, and (4) FP-7 extension extraction overhead eliminated by pre-extracting or bypassing `Path::extension()`.

---

### Performance Issues

#### FP-1: Visitor Return Value Instead of `len()` Comparison

**Severity:** High
**Location:** `quickscope-build/src/codegen/traversal.rs:68–76`, `quickscope-build/src/codegen/visitor.rs:22–44`

The traversal loop detects scope creation by reading `locations.len()` before and after the visitor call:

```rust
let before = builder.scopes.locations.len() as u16;
visit_fn(builder, &node, source_bytes);
let created = builder.scopes.locations.len() as u16 > before;
```

This is 2 `Vec::len()` reads + 1 comparison on **every node** in the AST — including the ~60–70% unnamed nodes that NP-1 would skip (but for the remaining ~30–40% named nodes, this is still overhead). For a file with 400 named nodes, that is 800 memory reads + 400 comparisons just to detect a side effect.

The pattern is also fragile: it relies on the invariant that `create_scope()` is the ONLY method that pushes to `locations`. Any future method that modifies `locations.len()` would silently break scope detection.

**Fix:**

Change the visitor function to return `Option<u16>` — the scope index if a scope was created, `None` otherwise:

```
CURRENT visitor signature:
  fn visit_typescript(builder, node, source_bytes)

PROPOSED visitor signature:
  fn visit_typescript(builder, node, source_bytes) -> Option<u16>
```

Inside the visitor, handlers that create scopes return the index:

```rust
fn visit_typescript<'src>(...) -> Option<u16> {
    match node.kind_id() {
        KIND_FUNCTION_DECL => {
            // ...
            return Some(builder.create_scope(node, true));
        }
        KIND_IDENTIFIER => {
            builder.reference(name);
            return None;
        }
        _ => return None,
    }
}
```

The traversal loop simplifies to:

```rust
let scope_idx = visit_fn(builder, &node, source_bytes);
let created = scope_idx.is_some();
```

This requires updating `visitor.rs` to generate `-> Option<u16>` and ensuring all handler bodies that call `create_scope()` return `Some(idx)` while all others return `None`. The `language!` macro DSL may need adjustment — handlers that don't create scopes would implicitly return `None`, while scope-creating handlers would use a `return Some(builder.create_scope(...))` pattern.

**Impact:** Eliminates 2 Vec reads + 1 comparison per named node visit. Also makes the control flow explicit — scope creation is communicated via return value rather than detected via side-effect observation. This prevents an entire class of future bugs.

---

#### FP-2: Cursor `field_id()` Propagation — Eliminates ALL Child Navigation

**Severity:** High (architectural)
**Location:** `quickscope-build/src/codegen/traversal.rs`, `visitor.rs`, `typescript.rs`

Section 14's P1-1 suggests converting `child_by_field_name("name")` to `child_by_field_id(FIELD_NAME)`. This eliminates the string lookup but still requires parent-to-child navigation: the parent handler iterates through its children to find the one with the matching field ID.

There is a fundamentally better approach. The tree-sitter `TreeCursor` already tracks the field ID of the node it is currently positioned on via `cursor.field_id()`. This is O(1) — a struct field read on the cursor's internal state (confirmed by tree-sitter's cursor design and [TreeCursor docs](https://docs.rs/tree-sitter/latest/tree_sitter/struct.TreeCursor.html)). During traversal, the cursor visits every child node anyway. Instead of the parent handler navigating DOWN to children, the child handler can check its own field role by looking at the field_id — no navigation required.

**Background:**

Every current handler that accesses child nodes follows this pattern:

```
Parent handler          Action
variable_declarator  → child_by_field_name("name")
function_declaration → child_by_field_name("name")
class_declaration    → child_by_field_name("name")
required_parameter   → child_by_field_name("pattern") or child_by_field_name("name")
import_specifier     → child_by_field_name("alias") or child_by_field_name("name")
import_specifier     → parent().parent().child_by_field_name("source")
namespace_import     → parent().parent().child_by_field_name("source")
import_clause        → parent().child_by_field_name("source")
```

All of these navigate FROM a parent node TO a child node to extract text. But the traversal already visits those child nodes. The information is available passively — it just needs to be communicated.

**Fix:**

Track `field_id` and `parent_kind_id` in the traversal loop (parent_kind_id is from P0-2) and pass both to the visitor:

```
CURRENT visitor signature:
  fn visit_typescript(builder, node, source_bytes)

PROPOSED visitor signature:
  fn visit_typescript(builder, node, source_bytes, parent_kind_id: u16, field_id: u16)
```

Traversal loop changes:

```rust
let field_id = cursor.field_id().unwrap_or(0);
// parent_kind_id tracked by recording kind_id before descending (per P0-2)
visit_fn(builder, &node, source_bytes, parent_kind_id, field_id);
```

**Visitor transformation:**

Handlers that currently navigate to children are restructured so the CHILD handles itself based on context:

```
BEFORE (parent navigates to child):

  ["variable_declarator"] => {
      if let Some(name_node) = node.child_by_field_name("name")
          .filter(|n| n.kind() == "identifier")
      {
          if let Ok(name) = name_node.utf8_text(source_bytes) {
              let is_var = node.parent()
                  .is_some_and(|p| p.kind() == "variable_declaration");
              if is_var {
                  builder.declare_symbol(name, Target::Function, Timing::Scope);
              } else {
                  builder.declare_symbol(name, Target::Scope, Timing::Point);
              }
          }
      }
      builder.increment_binding();
  }

AFTER (child handles itself via context):

  ["variable_declarator"] => {
      builder.increment_binding();
      // Name handling moved to identifier handler below
  }

  ["identifier"] => {
      let name = node_text(node, source_bytes);
      if field_id == FIELD_NAME {
          match parent_kind_id {
              KIND_VARIABLE_DECLARATOR => {
                  // grandparent kind tracked via builder state flag (set by
                  // variable_declaration/lexical_declaration handlers)
                  if builder.current_var_is_hoisted() {
                      builder.declare_symbol(name, Target::Function, Timing::Scope);
                  } else {
                      builder.declare_symbol(name, Target::Scope, Timing::Point);
                  }
              }
              KIND_FUNCTION_DECLARATION | KIND_GENERATOR_FUNCTION_DECLARATION => {
                  builder.declare_symbol(name, Target::Function, Timing::Scope);
              }
              KIND_CLASS_DECLARATION => {
                  builder.declare_symbol(name, Target::Scope, Timing::Point);
              }
              _ => builder.reference(name),
          }
      } else if field_id == FIELD_PATTERN {
          // Parameter name
          if parent_kind_id == KIND_REQUIRED_PARAMETER || parent_kind_id == KIND_OPTIONAL_PARAMETER {
              builder.declare_symbol(name, Target::Scope, Timing::Scope);
          }
      } else if field_id == FIELD_ALIAS {
          // Import alias — handled by import_specifier context
          // (import source tracked via builder state set by import_statement handler)
          builder.declare_import(name, builder.current_import_source(), Target::Module, Timing::Scope);
      } else {
          builder.reference(name);
      }
  }
```

For import source tracking, the `import_statement` handler sets builder state with the source string when it's visited (the "source" field child will be visited by the traversal):

```
  ["import_statement"] => {
      // Source will be captured when the string child is visited
  }

  ["string"] if field_id == FIELD_SOURCE => {
      if parent_kind_id == KIND_IMPORT_STATEMENT {
          let source = node_text(node, source_bytes)
              .trim_matches(|c| c == '"' || c == '\'');
          builder.set_current_import_source(source);
      }
  }
```

**What this eliminates:**

| Call site | Current cost | After fix |
|-----------|-------------|-----------|
| `child_by_field_name("name")` × 4 | 4 × (string lookup + child iteration) | 0 — field_id check is integer comparison |
| `child_by_field_name("pattern")` × 1 | string lookup + child iteration | 0 |
| `child_by_field_name("alias")` × 1 | string lookup + child iteration | 0 |
| `child_by_field_name("source")` × 3 | string lookup + child iteration | 0 |
| `node.parent()` × 1 (variable_declarator) | O(depth) from root | 0 — parent_kind_id is free |
| `node.parent().parent()` × 3 (imports) | 2 × O(depth) from root | 0 — builder state tracks source |
| `node.child(0)` × 5 (async, imports) | O(1) per child | 0 — async keyword handled as child visitor |
| `node.child(1)` × 2 (async, namespace) | O(1) per child | 0 |
| `.kind() == "string"` × 6 | FFI + CStr + comparison | 0 — kind_id checked by match |
| `.filter(\|n\| n.kind() == "identifier")` × 3 | FFI + CStr + comparison | 0 — node kind already known |

Total: eliminates ~25+ FFI calls and string operations per scope-creating or declaration-related node. The visitor becomes a pure computation on integers and byte slices.

**Requirements for `language!` macro DSL:**

The macro needs to support field_id matching. The codegen pipeline would:
1. Resolve field names to IDs at build time via `Language::field_id_for_name()` (already used for kind IDs)
2. Generate `FIELD_NAME`, `FIELD_PATTERN`, `FIELD_ALIAS`, `FIELD_SOURCE` constants
3. Pass `parent_kind_id` and `field_id` as visitor function parameters

**Trade-off:** This changes the visitor authoring model from "parent handles children by navigating to them" to "children handle themselves based on context." The latter is more aligned with the traversal's actual execution pattern (it visits every node depth-first anyway) and eliminates all navigation calls, but requires more context to be threaded through the visitor.

**Impact:** Combined with P0-2 (parent_kind_id) and NP-1 (skip unnamed nodes), this eliminates every tree-sitter Node API call from visitors. The only FFI calls remaining are `cursor.node()`, `cursor.field_id()`, `node.kind_id()`, `node.is_named()`, and `node.start_byte()`/`node.end_byte()` for text extraction — all O(1). The visitor becomes essentially a lookup table over integers.

---

#### FP-3: Extension Dispatch via Integer Encoding

**Severity:** Medium
**Location:** `quickscope-build/src/codegen/mod.rs:82–88`, generated `analyze()` function

The generated `analyze()` function matches file extensions as string patterns:

```rust
match extension {
    "ts" | "tsx" | "js" | "cjs" | "mjs" | "jsx" | "cts" | "mts" => { ... }
    _ => None,
}
```

Rust compiles string `match` as sequential length check + byte comparison chains. The compiler does NOT generate perfect hash functions for string matches ([rust-lang/rust#39525](https://github.com/rust-lang/rust/issues/39525), [internals discussion](https://internals.rust-lang.org/t/what-if-match-statetement-could-generate-perfect-hash-function/19222)). With a single language and 8 extensions, this is 8 sequential comparisons in the worst case. With 17 languages and ~50+ total extensions, this becomes a significant linear scan on every `analyze()` call.

**Fix:**

Encode short extensions (all target language extensions are 1–4 bytes) as packed `u32` at build time. LLVM generates a jump table for dense integer `match` with >3 arms ([LLVM Switch Lowering](https://llvm.org/pubs/2007-05-31-Switch-Lowering.pdf)).

Generate an inline helper in the codegen output:

```rust
#[inline(always)]
fn ext_to_u32(ext: &[u8]) -> u32 {
    match ext.len() {
        1 => ext[0] as u32,
        2 => u32::from_le_bytes([ext[0], ext[1], 0, 0]),
        3 => u32::from_le_bytes([ext[0], ext[1], ext[2], 0]),
        4 => u32::from_le_bytes([ext[0], ext[1], ext[2], ext[3]]),
        _ => 0,
    }
}
```

Then generate the match on pre-computed u32 constants:

```rust
// Build time: "ts" → 0x00007374, "tsx" → 0x00787374, "js" → 0x0000736A, etc.
match ext_to_u32(extension.as_bytes()) {
    0x00007374 | 0x00787374 | 0x0000736A | ... => { /* TypeScript */ }
    _ => None,
}
```

This replaces N string comparisons (each involving length check + byte-by-byte comparison) with a single integer conversion + jump table lookup. The integer encoding is computed once at build time by the codegen pipeline.

**Impact:** Reduces extension dispatch from O(extensions) string comparisons to O(1) integer match. Matters more as languages scale from 1 to 17.

---

#### FP-4: Function Scope Stack for O(1) Restore

**Severity:** Medium
**Location:** `builder.rs:198–211`

`restore_function_scope()` walks the parent chain O(depth) to find the nearest enclosing function when leaving a function scope:

```rust
fn restore_function_scope(&mut self, leaving_scope: u16) {
    let mut ancestor = self.parents[leaving_scope as usize];
    while ancestor != u16::MAX {
        if self.scopes.metrics[ancestor as usize].is_function {
            self.current_function_scope = ancestor;
            return;
        }
        ancestor = self.parents[ancestor as usize];
    }
    self.current_function_scope = 0;
}
```

Section 15's NP-7 suggests marking this `#[cold]`, which improves I-cache layout but doesn't fix the algorithmic cost. The parent chain walk accesses `parents[i]` and `metrics[i].is_function` at potentially scattered indices, causing cache misses for deeply nested functions.

**Fix:**

Maintain a dedicated function scope stack in `ScopeBuilder`:

```rust
// New field:
function_scope_stack: Vec<u16>,
```

Push/pop alongside the existing function scope tracking:

```rust
pub fn enter_function_scope(&mut self) {
    self.enter_scope();
    // Save current function scope before we overwrite it
    self.function_scope_stack.push(self.current_function_scope);
    self.pending_var_boundaries.push(self.pending_var_refs.len() as u16);
}

fn leave_function_scope(&mut self) {
    if let Some(prev) = self.function_scope_stack.pop() {
        self.current_function_scope = prev;
    }
    if let Some(boundary) = self.pending_var_boundaries.pop() {
        self.pending_var_refs.truncate(boundary as usize);
    }
    self.leave_scope();
}
```

The traversal loop's `leave()` method no longer needs the parent chain walk:

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.pop() {
        let leaving_scope = self.scope_stack_indices.pop();
        if self.scopes.metrics[leaving_scope as usize].is_function {
            self.leave_function_scope();
        } else {
            self.leave_scope();
        }
    }
}
```

This eliminates `restore_function_scope()` entirely. The `function_scope_stack` Vec retains capacity across analyses via thread-local reuse and typically holds ≤10 entries (function nesting depth).

**Impact:** O(1) function scope restoration instead of O(depth). Eliminates the only remaining parent chain traversal in the hot path. Also removes the dependency on `parents` and `metrics` arrays during scope exit, reducing cache pressure.

---

#### FP-5: Redundant `len > 0` Guard in `leave()`

**Severity:** Low
**Location:** `builder.rs:177–178`

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.len > 0 && self.scope_stack_created.pop() {
```

The traversal loop guarantees that every `leave()` call has a matching `push()` — the `scope_stack_created.push(tracked)` at line 82 of `traversal.rs` always executes before any `leave()` can be reached. The `len > 0` check evaluates to `true` on 100% of calls after the root node's first push.

The guard exists as defensive programming. But `pop()` already contains `debug_assert!(self.len > 0)` which catches bugs in debug builds. In release, the redundant check wastes a branch prediction slot on every node exit — millions of times per second.

**Fix:**

Remove the guard:

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.pop() {
```

The `debug_assert` inside `pop()` provides safety during development. In release, the branch is eliminated.

**Impact:** One fewer branch per node exit. Trivial effort, minor but free performance improvement.

---

#### FP-6: Use `grammar_id()` Instead of `kind_id()` for Node Matching

**Severity:** Medium
**Location:** `quickscope-build/src/codegen/mod.rs:66–78`, generated `visit_*` functions

The build-time codegen resolves string node kinds to numeric IDs:

```rust
// codegen/mod.rs — current approach:
let kind_id = (0..language.node_kind_count())
    .find(|&i| language.node_kind_for_id(i as u16) == Some(kind))
    .unwrap_or_else(|| panic!("Unknown node kind: {kind}"));
```

At runtime, the generated visitor matches on `node.kind_id()`. Tree-sitter's `kind_id()` implementation (`ts_node_symbol()` in the C API) resolves through `ts_language_public_symbol()`:

```c
// tree-sitter/lib/src/language.c
TSSymbol ts_language_public_symbol(const TSLanguage *self, TSSymbol symbol) {
    if (symbol == ts_builtin_sym_error) return symbol;  // ← branch
    return self->public_symbol_map[symbol];              // ← array index
}
```

This is a branch + array dereference on **every node visit**. In contrast, `grammar_id()` (`ts_node_grammar_symbol()`) returns the internal symbol directly — a single struct field read:

```c
// tree-sitter/lib/src/node.c
TSSymbol ts_node_grammar_symbol(const TSNode *self) {
    return ts_subtree_symbol(ts_node__subtree(self));  // ← direct field access
}
```

The `public_symbol_map` lookup exists because tree-sitter supports node aliases in grammar rules. For quickscope's use case (matching specific AST node kinds), grammar-level symbols are stable and correct — the codegen already looks up IDs against the concrete grammar at build time.

**Fix:**

Change the codegen to resolve `grammar_id`s instead of `kind_id`s, and match on `node.grammar_id()` in the generated visitor:

```rust
// codegen/mod.rs — use grammar_id resolution
let grammar_id = /* resolve via grammar symbol tables */;

// Generated visitor — match on grammar_id:
fn visit_typescript<'src>(...) {
    match node.grammar_id() {  // ← one field read, no map lookup
        // ...
    }
}
```

This requires using the internal symbol IDs from tree-sitter's grammar rather than the public symbol map. The build-time codegen already has access to the `Language` object which exposes these.

**Impact:** Eliminates one branch + one array dereference per node visit. On the critical path for every AST node. The improvement is small per node but multiplied by total node count (thousands per file, millions in batch processing).

---

#### LT-1: Const Lookup Table for Visitor Early-Exit

**Severity:** Medium-High
**Location:** `quickscope-build/src/codegen/visitor.rs`, generated `visit_*` functions

The generated visitor function matches `node.kind_id()` against ~30 handled kind IDs out of ~350 total kind IDs in the TypeScript grammar. For the ~320 unhandled IDs (~91% of all kind IDs), the match falls through to `_ => {}`. LLVM compiles this sparse match as a balanced binary search tree, requiring ~5 comparisons (`log2(30) ≈ 5`) before reaching the default arm for unhandled nodes.

Since NP-1 skips unnamed nodes, the visitor only sees named nodes (~30–40% of AST). Of those named nodes, roughly half are handled (scope-creating, declaration, reference) and half are not (expression nodes, type nodes before NI-1, etc.). For unhandled named nodes, those ~5 binary search comparisons are pure waste.

**Fix:**

Generate a `const` lookup table at build time — a `[bool; N]` array indexed by `kind_id`, where `true` means "this kind has a handler":

```rust
// Generated at build time:
const HANDLED_TYPESCRIPT: [bool; 350] = {
    let mut table = [false; 350];
    table[166] = true;  // statement_block
    table[187] = true;  // for_statement
    // ... all handled kind_ids
    table
};
```

In the generated visitor, check the table before the match:

```rust
#[inline(always)]
fn visit_typescript<'src>(...) {
    let kid = node.kind_id() as usize;
    if kid >= HANDLED_TYPESCRIPT.len() || !HANDLED_TYPESCRIPT[kid] {
        // Identifier (kind_id 1) is always handled — check separately
        if kid == 1 {
            if let Ok(name) = node.utf8_text(source_bytes) {
                builder.reference(name);
            }
        }
        return;
    }
    match node.kind_id() {
        // ... only reached for handled kinds
    }
}
```

The `const` array is embedded in the binary's `.rodata` section. At 350 bytes, it fits in ~6 cache lines. The single array index + bool check replaces ~5 branch comparisons for unhandled nodes.

**Alternative:** Instead of a bool table, use a `const [u8; 350]` that maps kind_id to a handler index (0 = unhandled, 1..N = handler). This replaces both the early-exit check AND the binary search match with a single table lookup + jump table on the handler index. More complex codegen but eliminates the binary search entirely.

**Impact:** For unhandled named nodes (~50% of visitor calls after NP-1), replaces ~5 binary search comparisons with 1 array index + 1 bool check. For handled nodes, adds 1 array check but the branch is well-predicted (always true for the match path). Net positive for the common case. Needs benchmarking to confirm the cache cost of the 350-byte table doesn't offset the branch elimination.

---

#### FP-7: Extension Extraction Overhead in `analyze()`

**Severity:** Medium
**Location:** Generated `analyze()` function, `quickscope-build/src/codegen/mod.rs:82–88`

Before extension dispatch (FP-3), the generated `analyze()` function extracts the extension from the file path:

```rust
let extension = std::path::Path::new(&file.path)
    .extension()
    .and_then(|e| e.to_str())?;
```

`Path::new()` creates a wrapper around the string. `.extension()` scans backwards from the end to find the last `.`, then returns an `OsStr`. `.to_str()` validates UTF-8 on the `OsStr`. This is 3 operations (reverse scan, OsStr creation, UTF-8 validation) repeated on every `analyze()` call, even though the path doesn't change.

For the batch processing use case (millions of files), the caller typically already knows the extension (it was used to filter which files to analyze). Repeating the extraction inside `analyze()` is redundant work.

**Fix:**

This combines with DX3 (borrow `File<'a>` instead of owning String). When `File` is restructured, pre-extract the extension:

```rust
pub struct File<'a> {
    pub path: &'a str,
    pub content: &'a str,
    pub extension: &'a str,  // pre-extracted by caller
}
```

Or, keep the current `File` struct but add a separate `analyze_with_extension(file, extension)` entry point that skips extraction. The public `analyze()` remains for convenience but delegates:

```rust
pub fn analyze(file: &File) -> Option<AnalysisResult> {
    let ext = std::path::Path::new(&file.path)
        .extension()
        .and_then(|e| e.to_str())?;
    analyze_with_extension(file, ext)
}

pub fn analyze_with_extension(file: &File, extension: &str) -> Option<AnalysisResult> {
    // ... dispatch on extension
}
```

The server code, which processes files in batches grouped by extension, can call `analyze_with_extension()` directly.

**Impact:** Eliminates reverse path scan + OsStr creation + UTF-8 validation per file. Minor for single files, meaningful at millions of files/second. Naturally combines with DX3 and FP-3.

---

### Correctness Issue

#### FC-1: Import Source Deduplication — Implementation Contradicts Documented Semantics

**Severity:** High (correctness)
**Location:** `builder.rs:400–421`, `typescript.rs:160–210`

The ARCHITECTURE.md explicitly states in Section 8 (Import Tracking):

> "This is intentionally **per-source-path, not per-specifier**. We don't track whether `foo` was used 5 times and `bar` was unused—we track that `'./utils'` was referenced 6 times total."

But the implementation creates **one `ImportData` per specifier**:

```rust
// declare_import() — called once per import specifier
pub fn declare_import(&mut self, name: &'a str, source: &'a str, ...) {
    let import_idx = self.imports.len() as u16;
    self.imports.push(ImportData { source, count: 0 });
    // ...
}
```

For `import { a, b, c } from 'react'`, this creates 3 separate `ImportData` entries, all with `source: "react"` and independent `count` fields. References to `a`, `b`, and `c` increment different counters. The output contains 3 separate `Import { source: "react", count: N }` entries that the consumer must aggregate — contradicting the documented per-source-path design.

**Consequences:**

1. **Incorrect edge weights:** The dependency graph gets multiple edges from the same file to the same source, each with partial counts, instead of a single edge with the total count.
2. **Wasted memory:** N ImportData entries where 1 would suffice (typical React component: 5–20 sources but 30–100 specifiers).
3. **Extra output allocation:** `take_result()` creates N `Import` structs with N `String` allocations for the same source path, instead of 1.
4. **Consumer burden:** Every consumer must group-by source and sum counts — work that should be done once in the builder.

**Fix:**

Add source deduplication in `declare_import()` using a source-path lookup map:

```rust
// New field on ScopeBuilder:
import_sources: AHashMap<&'a str, u16>,  // source_path → import_idx

pub fn declare_import(&mut self, name: &'a str, source: &'a str, ...) {
    if let Some(scope_idx) = self.scope_stack_indices.last() {
        let next_idx = self.names.len() as u16;
        let symbol = *self.names.entry(name).or_insert(next_idx);

        // Deduplicate by source path
        let import_idx = match self.import_sources.get(source) {
            Some(&idx) => idx,
            None => {
                let idx = self.imports.len() as u16;
                self.imports.push(ImportData { source, count: 0 });
                self.import_sources.insert(source, idx);
                idx
            }
        };

        let decl = CompactDecl::new_import(symbol, target, timing, import_idx);
        self.build_declarations[scope_idx as usize].push(decl);
        self.declare_binding(symbol, SymbolBinding::new_import(import_idx));
    }
}
```

Add `self.import_sources.clear()` in `clear()` to reset between analyses (capacity retained via thread-local reuse).

**After fix:**

`import { a, b, c } from 'react'` creates 1 `ImportData { source: "react", count: 0 }`. All three symbols' `SymbolBinding` point to the same `import_idx`. References to `a()`, `b()`, `c()` all increment the same counter. Output contains 1 `Import { source: "react", count: 8 }` — the correct per-source-path total.

**Impact:** Fixes the semantic mismatch with the documented design. Reduces ImportData entries by ~3–5x for typical files. Eliminates duplicate String allocations in output. Makes the dependency graph edge weights correct without consumer aggregation.

---

### DX & Operational Improvements

#### FO-1: BOLT Post-Link Optimization

**Severity:** Medium (operational)
**Location:** Build/deployment documentation

Section 15's NO-1 documents PGO (profile-guided optimization) but not BOLT (Binary Optimization and Layout Tool). PGO optimizes compiler decisions (inlining, branch prediction hints, register allocation). BOLT operates at a different level — it reorders functions and basic blocks in the final binary based on runtime profile data, optimizing instruction cache and branch target buffer utilization.

Research shows PGO + BOLT **combined** yields 34–68% improvement for compute-bound code — significantly more than PGO alone ([Arm PGO+BOLT benchmarks](https://www.phoronix.com/news/Arm-Fast-PGO-BOLT-LLVM-Clang), [cargo-pgo benchmarks](https://kobzol.github.io/rust/cargo/2023/07/28/rust-cargo-pgo.html)). The Rust compiler itself uses BOLT for its release builds.

For quickscope specifically, BOLT is valuable because the traversal loop calls many `#[inline(always)]` methods. Even with inlining, LLVM may place cold code (error paths, depth-limiting) adjacent to hot code, polluting the instruction cache. BOLT uses runtime profile data to move cold blocks out of the hot path.

```bash
# Install
cargo install cargo-pgo

# Step 1: Build with BOLT instrumentation
cargo pgo bolt build

# Step 2: Run representative workload
cargo pgo bolt run -- bench

# Step 3: Optimize binary layout
cargo pgo bolt optimize
```

For maximum effect, combine with PGO:

```bash
# PGO + BOLT combined
cargo pgo run -- bench           # PGO profiling
cargo pgo bolt build             # Build with PGO + BOLT instrumentation
cargo pgo bolt run -- bench      # BOLT profiling
cargo pgo bolt optimize          # Final optimized binary
```

**Impact:** Typically 10–20% additional improvement on top of PGO, from improved I-cache utilization and branch target prediction. Compounds with all Section 14 optimizations (particularly P2-2 LTO and NP-7 `#[cold]` annotations).

---

#### FO-2: `target-cpu=native` for Deployment

**Severity:** Low (operational)
**Location:** Build/deployment documentation

Not mentioned anywhere in the codebase. For deployment on known hardware (the stated Rust Axum server), enabling CPU-specific instructions unlocks:

- **AES-NI** for aHash (the hasher used by `AHashMap`) — aHash detects AES-NI at runtime but the compiler can eliminate the feature-detection branch with `target-cpu=native`
- **BMI2** for bit manipulation in `CreatedStack` and `CompactDecl` packed fields
- **POPCNT** and other bit-level instructions
- **AVX2** for potential auto-vectorization in data processing

```toml
# In .cargo/config.toml for the deployment target:
[build]
rustflags = ["-C", "target-cpu=native"]
```

Or for cross-compilation to a specific server architecture:

```toml
[build]
rustflags = ["-C", "target-cpu=neoverse-n2"]  # Example: AWS Graviton3
```

**Caveat:** This makes the binary non-portable. Only use for deployment builds, not for distributed libraries. The library's Cargo.toml should NOT set this — it should be set by the consuming application's build configuration.

**Impact:** Typically 2–5% improvement from CPU-specific instruction selection. Free to enable, zero code changes.

---

#### FD-1: README/API Documentation Drift

**Severity:** Medium (DX)
**Location:** `README.md` API table and usage examples

Several discrepancies between the README API documentation and the actual implementation:

| README claims | Actual code | Status |
|---|---|---|
| `Score { own: f64, max: f64 }` | `Score { own: f32, max: f32 }` | Changed in optimization, README not updated |
| `Metrics` includes `total_lines: u16`, `code_lines: u16`, `comment_lines: u16`, `whitespace_lines: u16` | These fields do not exist in the `Metrics` struct | Documented but never implemented |
| `Walker` type with `add()` and `walk()` methods | Does not exist; API is `analyze(&File) -> Option<AnalysisResult>` | README describes old/planned API |
| `Language` trait with `mappings()` and `visit()` methods | Does not exist; uses `language!` macro + build-time codegen | README describes old/planned API |
| `ParserMapping` struct | Does not exist | README describes old/planned API |
| `Context` type passed to visitors | Does not exist; visitors receive `ScopeBuilder` | README describes old/planned API |
| `TypeScript` as a type to pass to `Walker::add()` | Does not exist | README describes old/planned API |

The entire "Usage" section with `Walker::new()`, `walker.add(TypeScript)`, and `walker.walk(".ts", source)` does not correspond to any code in the repository. The "Defining a custom language" section describes a `Language` trait that does not exist.

The "Project structure" section shows a `quickscope/src/languages/` directory structure that does not match the actual workspace layout (`crates/quickscope/`, `crates/quickscope-build/`, `crates/quickscope-core/`, `crates/quickscope-languages/`).

**Fix:** Update README.md to reflect the actual API surface:

```rust
// Actual usage:
use quickscope::types::File;
use quickscope::analyze;

let file = File {
    path: "example.ts".to_string(),
    content: "function greet(name) { return name; }".to_string(),
};

if let Some(result) = analyze(&file) {
    for (i, metrics) in result.scopes.metrics.iter().enumerate() {
        let loc = &result.scopes.locations[i];
        println!(
            "rows {}-{}: {} args, {} bindings",
            loc.start_row, loc.end_row,
            metrics.argument_count, metrics.binding_count,
        );
    }
}
```

**Impact:** Prevents confusion for anyone attempting to use the library. The current README would cause immediate compilation failures for any user following its examples.

---

### Prioritized Implementation Order

See [Unified Priority Table](#unified-priority-table) at the end of the document for the merged table across all sections.

---

### References (Section 16)

- [rust-lang/rust#39525: String/slice matching efficiency](https://github.com/rust-lang/rust/issues/39525) — Rust does not generate PHF for string matches
- [LLVM Switch Lowering (PDF)](https://llvm.org/pubs/2007-05-31-Switch-Lowering.pdf) — Jump table vs binary search thresholds
- [cargo-pgo: PGO and BOLT for Rust](https://kobzol.github.io/rust/cargo/2023/07/28/rust-cargo-pgo.html) — Combined PGO+BOLT benchmarks
- [Arm PGO+BOLT results](https://www.phoronix.com/news/Arm-Fast-PGO-BOLT-LLVM-Clang) — 34–68% improvement documented
- [TreeCursor::field_id() docs](https://docs.rs/tree-sitter/latest/tree_sitter/struct.TreeCursor.html#method.field_id) — O(1) field ID from cursor state
- [tree-sitter Discussion #878: Optimized traversal](https://github.com/tree-sitter/tree-sitter/discussions/878) — Cursor API is stateful and efficient
- [internals.rust-lang.org: PHF for match statements](https://internals.rust-lang.org/t/what-if-match-statetement-could-generate-perfect-hash-function/19222) — Not currently implemented
- [Bounds Checks in Rust](https://readyset.io/blog/bounds-checks) — 64.5% of cases show <1% cost, compiler often elides
- [Vec::pop optimization issues](https://github.com/rust-lang/rust/issues/85365) — `unwrap_unchecked` gains are context-dependent (0.5–3%)
- [tree-sitter `ts_node_grammar_symbol`](https://github.com/tree-sitter/tree-sitter/blob/master/lib/src/node.c) — Direct field access, no alias map lookup
- [tree-sitter `ts_language_public_symbol`](https://github.com/tree-sitter/tree-sitter/blob/master/lib/src/language.c) — `kind_id()` goes through `public_symbol_map[]` with alias branch
- [tree-sitter `ts_language_field_id_for_name` linear scan](https://github.com/tree-sitter/tree-sitter/blob/master/lib/src/language.c) — `for` loop with `strncmp` per field name (P1-1 severity upgrade)

---

---

## 17. Structural & Deployment Improvements

### Overview

Sections 14–16 identified 30+ issues covering tree-sitter API misuse, per-scope allocation, Vec capacity loss, missing release profile, DX gaps, unnamed node skipping, Option elimination, HashMap pollution, UTF-8 validation, shadow Vec merging, pre-allocation, cold paths, JSX grammar, parser timeout, cancellation, PGO, visitor return values, cursor field_id propagation, extension dispatch, function scope stack, import deduplication, BOLT, target-cpu, and documentation drift.

This section identifies a final tier of improvements discovered through cross-cutting analysis of how the existing suggestions interact, deep research into tree-sitter cursor internals (v0.26.x), arena allocation patterns used by OXC (the Rust JavaScript toolchain), and SIMD applicability analysis for the Metrics struct.

**7 improvements** (1 high-impact performance, 2 medium-high, 2 medium, 1 low, 1 trivial) and **1 DX improvement** were identified. The most architecturally significant is subtree skipping for analysis-irrelevant AST regions, which changes *how much* of the AST is visited — a fundamentally different lever from Sections 14–16 which optimize *how efficiently* each node is processed. NI-6 (Vec::push capacity check elimination) documents a confirmed LLVM limitation where `Vec::push` always emits a capacity check branch even after `reserve()`. NI-7 identifies 286 bytes of unnecessary `memset` per file in `clear()` from re-initializing already-unwound fixed-size stacks. DX6 addresses benchmark representativeness — the single fixture prevents informed optimization decisions.

**Research findings that inform these improvements:**

- **tree-sitter v0.26.x cursor API:** No `goto_first_named_child()` or `goto_next_named_sibling()` exist. `cursor.node()` copies a 32-byte `TSNode` struct (4×u32 context + 2 pointers). No `cursor.node_kind_id()` exists — must construct Node first. `cursor.field_id()` returns `Option<NonZeroU16>` (niche-optimized, zero-cost Option). No way to get byte range without constructing a Node.
- **Arena allocation (bumpalo/OXC):** OXC achieves 84x faster than ESLint using bumpalo-based arena allocation. However, quickscope's thread-local reuse pattern already addresses the same problem differently — arenas would add lifetime complexity without clear benefit given that the builder retains capacity across files (after P1-3 is fixed).
- **SIMD for Metrics:** SIMD is **not** beneficial for individual u8 field increments (extract/insert overhead exceeds scalar cost). LLVM already optimizes `Metrics::default()` to a single 128-bit zero write (`pxor xmm0, xmm0` + `movdqa`). No code changes needed.

---

### Performance Issues

#### NI-1: Subtree Skipping for Analysis-Irrelevant AST Regions

**Severity:** High
**Location:** `quickscope-build/src/codegen/traversal.rs`, `quickscope-languages/src/typescript.rs`

Sections 14–16 optimize how each node is processed: NP-1 skips unnamed nodes (~60–70% of AST), FP-1 replaces `len()` comparison with return value, FP-2 eliminates child navigation. None address skipping entire named subtrees that are irrelevant to scope analysis.

In TypeScript, **30–50% of named AST nodes** can live inside type annotations, interface bodies, type alias definitions, and similar constructs that contain **zero** scopes, declarations, or references:

```
TypeScript type annotations:

  function processData(
      input: {                              ← 15+ named AST nodes in this type annotation
          users: Array<{                       contain NO scopes, NO declarations,
              name: string;                    NO references, NO metrics to collect
              metadata: Record<string, {
                  score: number;
                  tags: Set<string>;
              }>;
          }>;
          config: Readonly<Config>;          ← all of this is traversed for nothing
      },
  ): Promise<Result<void, Error>> {          ← return type: more wasted traversal
      // actual runtime code...
  }

  interface UserConfig {                     ← entire interface body: zero useful data
      theme: 'light' | 'dark';
      locale: string;
      notifications: {
          email: boolean;
          push: boolean;
          sms: boolean;
      };
  }

  type DeepPartial<T> = {                    ← type alias body: zero useful data
      [P in keyof T]?: DeepPartial<T[P]>;
  };
```

The traversal currently descends into every child of every node. There is no mechanism for the visitor to signal "skip this subtree."

**Why this compounds with NP-1:** NP-1 skips *unnamed* nodes (punctuation, keywords). This skips *named* nodes in irrelevant subtrees. Together, they could eliminate processing for 70–85% of all AST nodes in type-heavy TypeScript files. The improvements operate on orthogonal node populations.

**Fix — Extend the visitor return type (builds on FP-1):**

FP-1 proposes `Option<u16>`. Instead, use a return enum that also encodes subtree skipping:

```rust
#[repr(u8)]
enum VisitResult {
    /// No scope created, descend into children normally.
    None = 0,
    /// Scope created at the given index, descend into children.
    Scope(u16) = 1,
    /// No scope created, skip entire subtree (don't descend).
    SkipChildren = 2,
}
```

In the traversal loop, when `SkipChildren` is returned, skip `cursor.goto_first_child()` and proceed directly to `leave()` + sibling navigation:

```
CURRENT traversal (simplified):

    visit_fn(builder, &node, source_bytes);
    // ... tracking logic ...
    if cursor.goto_first_child() { continue; }  // ALWAYS attempts descent

PROPOSED traversal:

    let result = visit_fn(builder, &node, source_bytes);
    match result {
        VisitResult::SkipChildren => {
            builder.scope_stack_created.push(false);
            // Skip goto_first_child — fall through to leave() + sibling
        }
        VisitResult::Scope(idx) => {
            // ... tracking logic (push to stacks, enter scope) ...
            if cursor.goto_first_child() { continue; }
        }
        VisitResult::None => {
            builder.scope_stack_created.push(false);
            if cursor.goto_first_child() { continue; }
        }
    }
    builder.leave();
    // ... sibling/parent navigation ...
```

The `scope_stack_created.push(false)` still happens for `SkipChildren` to maintain depth balance with subsequent `leave()` calls.

In the TypeScript visitor, type-only node kinds return `SkipChildren`:

```rust
["type_annotation", "type_parameters", "type_arguments",
 "predefined_type", "generic_type", "union_type",
 "intersection_type", "mapped_type", "conditional_type",
 "index_type_query", "lookup_type"] => {
    return VisitResult::SkipChildren;
}
```

For `interface_declaration` and `type_alias_declaration`, the handler can extract the name for declaration tracking before signaling skip:

```rust
["interface_declaration", "type_alias_declaration"] => {
    // Extract name for declaration tracking (via field_id per FP-2)
    // ... name extraction ...
    builder.increment_binding();
    return VisitResult::SkipChildren;
}
```

**Per-language applicability:**

| Language | Skippable subtree types | Estimated node reduction |
|----------|------------------------|-------------------------|
| TypeScript | Type annotations, interfaces, type aliases, enums, decorators | 30–50% of named nodes |
| Java | Generics, annotations, throws clauses | 10–20% |
| C++ | Template parameters, requires clauses | 10–15% |
| Go | Type definitions (when only tracking scopes) | 5–10% |
| Python | Type hints (PEP 484+) | 5–15% |
| Rust | Lifetime annotations, where clauses, trait bounds | 10–20% |

The optimization is most impactful for TypeScript — the current and primary target language — where enterprise codebases routinely have 40%+ type annotation density.

**Impact:** For type-heavy TypeScript (the primary target), eliminates traversal of 30–50% of named nodes — the exact nodes that survive NP-1's unnamed filter and reach the visitor. Combined with NP-1 (skip unnamed ~65%), the traversal processes only ~18–25% of total AST nodes (vs 100% currently). At millions of files/second, this is a multiplicative improvement on walker throughput.

---

#### NI-2: Pack `is_function` Flag into IndexStack Entries

**Severity:** Medium-High
**Location:** `builder.rs:177–191`, `stacks.rs:25–84`

The `leave()` method is called on every node exit — the single most frequently executed code path. It currently loads a 16-byte `Metrics` struct from a potentially cold cache line just to read a single bool:

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.pop() {
        let leaving_scope = self.scope_stack_indices.pop();
        // THIS loads Metrics[leaving_scope] — 16 bytes from metrics array
        // just to read 1 bit (is_function)
        if self.scopes.metrics[leaving_scope as usize].is_function {
            self.restore_function_scope(leaving_scope);
            self.leave_function_scope();
        } else {
            self.leave_scope();
        }
    }
}
```

Section 14 notes that `is_function` "mixes metadata with metrics" and suggests a separate parallel flags array. A separate array would add another Vec to maintain (allocation, push, capacity tracking). There is a zero-cost alternative.

**Fix — Tag the high bit of the scope index on IndexStack:**

The IndexStack stores `u16` scope indices. Use the high bit (bit 15) as the `is_function` flag:

```
Bit layout of tagged scope index:
┌─────────────────────────────────────────────────────────────┐
│ 15       │ 14  13  12  11  10   9   8   7   6   5   4   3   2   1   0 │
├──────────┼────────────────────────────────────────────────────────────┤
│is_function│                  scope_index (15 bits)                     │
└──────────┴────────────────────────────────────────────────────────────┘
```

In `create_scope()`, tag the index before pushing:

```rust
let tagged_idx = idx | ((is_function as u16) << 15);
self.scope_stack_indices.push(tagged_idx);
```

In `leave()`, extract both fields from the popped value:

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.pop() {
        let raw = self.scope_stack_indices.pop();
        let leaving_scope = raw & 0x7FFF;
        let is_function = raw & 0x8000 != 0;

        if is_function {
            self.leave_function_scope();  // O(1) with FP-4
        } else {
            self.leave_scope();
        }
    }
}
```

**Scope index limit:** 15 bits → max 32767 scopes per file. No real-world file has 32K scopes (the architecture already assumes u16 suffices for scope indices, and the debug_assert catches overflow). The limit is conservative.

**What this eliminates:** The `self.scopes.metrics[leaving_scope as usize].is_function` access causes a load from the Metrics parallel array. With SoA layout, each Metrics is 16 bytes — so this loads an entire cache line (64 bytes, containing 4 Metrics entries) just to read 1 bit. The tagged index keeps the information on the IndexStack itself, which is already in L1 cache (the stack is accessed on every scope push/pop).

**Combined with FP-4 (function scope stack), `leave()` becomes:**

```rust
pub fn leave(&mut self) {
    if self.scope_stack_created.pop() {
        let raw = self.scope_stack_indices.pop();
        if raw & 0x8000 != 0 {
            self.leave_function_scope();
        } else {
            self.leave_scope();
        }
    }
}
```

Zero array accesses. Pure register operations on stack-local data.

**Impact:** Eliminates one 16-byte array load per scope exit on the hottest code path. For a file with 50 scopes, that is 50 cache-polluting loads removed. Combines multiplicatively with FP-4 (which eliminates `restore_function_scope`'s parent chain walk) to make `leave()` allocation-free and array-access-free.

---

#### NI-3: Eliminate `parents: Vec<u16>` After FP-4

**Severity:** Medium
**Location:** `builder.rs:27, 157, 199–211, 242`

This is a consequence of FP-4 (function scope stack) that Section 16 does not identify. Currently, `parents: Vec<u16>` has exactly two roles:

1. **Traversal:** Used only by `restore_function_scope()` to walk the parent chain.
2. **Output:** NOT part of `AnalysisResult`. Looking at `take_result()`, parents is never transferred to the result — it is purely builder-internal.

```rust
// take_result() — parents is NOT included:
let scopes = Scopes {
    locations: std::mem::take(&mut self.scopes.locations),
    metrics: std::mem::take(&mut self.scopes.metrics),
    scores: std::mem::take(&mut self.scopes.scores),
    declarations: FlatDeclarations { decls, ranges },
    function_scope: std::mem::take(&mut self.scopes.function_scope),
    // ← parents: nowhere
};
```

After FP-4 replaces `restore_function_scope()` with a dedicated stack, `parents` has **zero reads** during traversal. It is written on every `create_scope()` call and discarded on `clear()`.

**Fix:** Remove `parents` entirely from `ScopeBuilder`:

```rust
// REMOVE from ScopeBuilder:
//   pub parents: Parents,

// REMOVE from create_scope():
//   let parent_idx = self.scope_stack_indices.last().unwrap_or(u16::MAX);
//   self.parents.push(parent_idx);
//   debug_assert!(self.parents.len() == self.scopes.locations.len(), ...);

// REMOVE from clear():
//   self.parents.clear();
```

Also remove the `Parents` type alias from `types.rs` if no longer used.

**Impact:** Eliminates one Vec<u16> allocation (capacity tracking, growth), one push (2 bytes + bounds check) per `create_scope()`, one debug_assert per `create_scope()`, and one `clear()` call per file. For a file with 50 scopes: 100 bytes less write traffic, 50 fewer Vec pushes, one fewer data structure in the hot path. The primary benefit is reduced cache pressure during `create_scope()` — one fewer Vec to push to means one fewer potential cache miss.

---

### Deployment & Operational Improvements

#### NI-4: Use `mimalloc` as Global Allocator in Server Binary

**Severity:** Medium
**Location:** Server binary (consumer of the library), not the library itself

Sections 14–16 cover release profile (LTO, codegen-units), PGO, BOLT, and `target-cpu=native`. None mention the global allocator, which is one of the highest-impact zero-code-change optimizations for allocation-heavy Rust workloads.

The library performs many small allocations during analysis: HashMap entries (`names`, `import_sources` after FC-1), Vec growth (even with thread-local capacity reuse, the first file per thread triggers growth), shadow stack pushes, and output construction (AnalysisResult Vecs, Import Strings). Tree-sitter itself allocates internally for the parse tree.

Published benchmarks for Rust workloads with many small allocations:

```
┌──────────────┬───────────────────┬──────────────┬──────────────────────┐
│ Allocator    │ Small alloc (<64B)│ Throughput   │ Multi-thread scaling │
├──────────────┼───────────────────┼──────────────┼──────────────────────┤
│ glibc malloc │ baseline          │ baseline     │ poor (global lock)   │
│ jemalloc     │ 1.5–3x faster    │ 1.3–2x       │ good (arenas)        │
│ mimalloc     │ 2–7x faster      │ 1.5–3x       │ excellent (per-heap) │
└──────────────┴───────────────────┴──────────────┴──────────────────────┘
```

For the stated use case (many threads each processing files independently), mimalloc's per-thread heap design is ideal: zero cross-thread contention on allocation. Each analysis thread has its own heap, allocations are purely local pointer bumps.

**Why not in the library:** The library should not dictate the allocator. The server binary sets it. This is a deployment optimization, like LTO and `target-cpu`.

**Fix:** In the server binary (not in the quickscope library):

```rust
#[global_allocator]
static ALLOC: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

Server `Cargo.toml`:

```toml
[dependencies]
mimalloc = { version = "0.1", default-features = false }
```

**When this matters most:** Before the allocation-reducing optimizations from Sections 14–16 are implemented (P0-1, P1-3, NP-6), the library performs many more allocations per file. mimalloc reduces the per-allocation cost. After those optimizations reduce allocation count, mimalloc still benefits tree-sitter's internal allocations and the inherent output allocations.

**Impact:** Typically 10–30% reduction in total allocation overhead. Most impactful for the multi-threaded server scenario where glibc malloc's global lock becomes a contention bottleneck across analysis threads.

---

#### NI-5: Borrowed `AnalysisResult<'a>` for Zero-Copy Import Output

**Severity:** Low-Medium
**Location:** `builder.rs:111–143`, `types.rs:264–273`

DX3 in Section 14 proposes `File<'a>` with borrowed `&'a str` for path and content. The natural extension — making `AnalysisResult` borrow from the source bytes — is not explicitly called out. It eliminates a specific class of allocations in `take_result()`:

```rust
// CURRENT: allocates one String per import
let imports: Vec<Import> = self.imports.drain(..)
    .map(|imp| Import {
        source: imp.source.to_owned(),  // ← heap allocation per import
        count: imp.count,
    })
    .collect();
```

For a React component with 20 import statements (common in enterprise codebases), that is 20 `String::from()` heap allocations in `take_result()`. After FC-1 (import source deduplication), this drops to unique source count (typically 10–15), but each still allocates.

**Fix:** Return borrowed import data:

```rust
pub struct AnalysisResult<'a> {
    pub scopes: Scopes,
    pub imports: Vec<ImportData<'a>>,  // borrowed source strings, zero allocation
}
```

`take_result()` for imports becomes:

```rust
// AFTER: zero allocation — drain moves ImportData<'a> without cloning strings
let imports: Vec<ImportData<'a>> = self.imports.drain(..).collect();
```

The caller holds the source bytes and the result:

```rust
// In server code:
let source: &str = git_blob.as_str();     // borrowed from git
let file = File { path: "app.tsx", content: source };
let result = analyze(&file);               // result borrows from source

// Process result (send to ClickHouse batch)
batch.add_imports(&result.imports);        // reads borrowed strings

// Drop result, then source — natural lifetime ordering
drop(result);
drop(source);
```

**Trade-off:** Lifetimes propagate to consumers. The server code needs `fn process<'a>(source: &'a str) -> AnalysisResult<'a>`. This is natural in the "process file → batch insert → drop" pattern stated in the README, but prevents storing results beyond the source lifetime. For the ClickHouse batch-insert architecture, this is not a problem — results are consumed immediately.

**Impact:** Eliminates N String heap allocations per file (N = unique import sources, typically 5–15 after FC-1). Each String allocation involves: size computation, malloc call, memcpy of source path, and later free. At millions of files/second, this removes millions of malloc/memcpy/free triples.

---

#### NI-6: Vec::push Capacity Check Elimination

**Severity:** Low
**Location:** `builder.rs` (all `Vec::push` call sites), generated traversal loop

`Vec::push` always emits a capacity check branch — even immediately after `reserve()`. This is a confirmed Rust compiler limitation ([rust-lang/rust#105156](https://github.com/rust-lang/rust/issues/105156)): LLVM cannot prove that the capacity reserved by `reserve(n)` is sufficient for subsequent `push()` calls because the `reserve` → `push` invariant is not communicated through LLVM IR.

In the traversal hot path, several `push` calls occur per node:

```rust
builder.scope_stack_created.push(tracked);     // ← capacity check branch
self.scopes.locations.push(loc);                // ← capacity check branch
self.scopes.metrics.push(Metrics::default());   // ← capacity check branch
self.build_declarations.push(Vec::new());       // ← capacity check branch
```

Each capacity check is a compare + conditional branch to a cold reallocation path. For the fixed-size stacks (`CreatedStack`, `IndexStack`), this is already avoided via the inline array design. But `Vec`-based storage still pays the cost.

**Fix (targeted, not pervasive):**

For hot-path pushes where the capacity invariant is known, use `push_within_capacity` (stabilized Rust 1.86) combined with `assert_unchecked`:

```rust
// After NP-6 reserve_estimate() guarantees capacity:
unsafe {
    core::hint::assert_unchecked(self.scopes.locations.len() < self.scopes.locations.capacity());
}
// Now LLVM can elide the capacity check in push:
self.scopes.locations.push(loc);
```

Or use the lower-level `spare_capacity_mut` + `MaybeUninit::write` pattern to bypass the check entirely:

```rust
let slot = self.scopes.locations.spare_capacity_mut().first_mut().unwrap();
slot.write(loc);
unsafe { self.scopes.locations.set_len(self.scopes.locations.len() + 1) };
```

**Recommendation:** This is a micro-optimization. Apply only to the 3–4 hottest `push` sites (in `create_scope` and `scope_stack_created.push`) after benchmarking confirms the capacity check branch is measurable. The `assert_unchecked` approach is preferred as it keeps safe `push()` syntax. Do not apply pervasively — the code complexity cost is not justified for cold paths.

**Impact:** Eliminates 3–4 branch instructions per `create_scope()` call. Since only ~10–15% of visited nodes create scopes, the absolute impact is modest. Worth pursuing only after higher-priority items are implemented and profiling confirms this as a remaining bottleneck.

---

#### NI-7: Unnecessary Zero-Fill in `clear()`

**Severity:** Low (trivial fix)
**Location:** `builder.rs:148–156`, `stacks.rs`

`ScopeBuilder::clear()` resets the fixed-size stacks by assigning fresh instances:

```rust
self.scope_stack_indices = IndexStack::new();
self.scope_stack_created = CreatedStack::new();
```

`IndexStack::new()` zero-initializes a `[u16; 127]` array (254 bytes). `CreatedStack::new()` zero-initializes a `[u64; 4]` array (32 bytes). That is 286 bytes of `memset` on every `clear()` call — every file analyzed.

However, after a successful traversal, both stacks are fully unwound: every `push()` has a matching `pop()`, so `len == 0` at the end. The array contents are stale but irrelevant — only elements below `len` are ever read, and all reads are preceded by writes (pushes). Setting `len = 0` is sufficient.

**Fix:**

Add a `reset()` method to both stacks that only zeroes the length:

```rust
impl IndexStack {
    #[inline(always)]
    pub fn reset(&mut self) {
        self.len = 0;
    }
}

impl CreatedStack {
    #[inline(always)]
    pub fn reset(&mut self) {
        self.len = 0;
    }
}
```

In `clear()`:

```rust
self.scope_stack_indices.reset();
self.scope_stack_created.reset();
```

**Impact:** Eliminates 286 bytes of unnecessary `memset` per file. At millions of files/second, this is measurable in aggregate. Trivial fix with zero risk — the stacks are push/pop only, so stale array contents are never observable.

---

### DX & Operational Improvements

#### DX6: Benchmark Representativeness

**Severity:** Medium (DX)
**Location:** `crates/quickscope-bench/benches/analysis.rs`, `crates/quickscope-bench/fixtures/`

The benchmark suite uses a single fixture (`complex.ts`). Optimization decisions based on a single file risk overfitting — an improvement on one file structure may regress on another. Confident performance claims require diverse test inputs.

**Missing fixtures:**

| Fixture | Purpose | Characteristic |
|---------|---------|---------------|
| `small.ts` | Minimum viable file (~10 lines) | Measures fixed overhead (setup, clear, take_result) |
| `large.ts` | Stress test (~2000+ lines) | Reveals O(n) vs O(n²) scaling |
| `type-heavy.ts` | Dense type annotations, interfaces | Validates NI-1 subtree skipping impact |
| `import-heavy.ts` | 50+ import statements, re-exports | Validates FC-1 dedup and import tracking |
| `deeply-nested.ts` | 20+ levels of nesting | Stress-tests IndexStack/CreatedStack and depth limiting |
| `declaration-heavy.ts` | Hundreds of variable/function declarations | Stresses build_declarations and shadow stack |

Each fixture should be benchmarked individually so regressions on specific file shapes are immediately visible. The current `bench_analysis` and `bench_baseline` groups should be parameterized over all fixtures.

**Impact:** Prevents optimization blind spots. Enables informed prioritization — if type-heavy files dominate the actual workload, NI-1 should be prioritized higher; if most files are small, fixed overhead matters more.

---

### Prioritized Implementation Order

See [Unified Priority Table](#unified-priority-table) at the end of the document for the merged table across all sections.

---

### Research Notes

#### Tree-Sitter v0.26.x Cursor API Limitations

The following were investigated and confirmed as **not available** in tree-sitter 0.26:

| Desired capability | Available? | Workaround |
|---|---|---|
| `cursor.goto_first_named_child()` | No | Check `cursor.node().is_named()` after `goto_first_child()` (NP-1) |
| `cursor.goto_next_named_sibling()` | No | Check `cursor.node().is_named()` after `goto_next_sibling()` |
| `cursor.node_kind_id()` | No | Must construct Node via `cursor.node()` then call `.kind_id()` |
| Byte range from cursor directly | No | Must construct Node first: `cursor.node().start_byte()` |
| `cursor.field_id()` type | `Option<NonZeroU16>` | Niche-optimized — `Option<NonZeroU16>` is same size as `u16` |

`cursor.node()` copies a 32-byte `TSNode` struct to the stack (4×u32 context + 2 pointers on 64-bit). This is cheap (one memcpy) but not free. Since there is no way to get `kind_id` or byte ranges without constructing the Node, this copy is unavoidable for every visited node.

#### Arena Allocation Assessment

OXC (the Rust JavaScript toolchain, 84x faster than ESLint) uses `bumpalo`-based arena allocation for all AST nodes and traversal data. Their key insight: allocate in traversal order, so traversal accesses memory linearly — perfect cache locality.

For quickscope's architecture, arena allocation was evaluated and **not recommended** because:

1. The thread-local reuse pattern (clear + take_result) already eliminates repeated allocation — the builder retains Vec capacity across files (after P1-3 fix).
2. Arena allocation would require lifetime propagation through all builder methods, adding complexity without clear benefit.
3. The remaining inherent allocations (HashMap internals, output Vecs) are better addressed by mimalloc (NI-4) which requires zero code changes.

If the architecture ever moves to an arena model, bumpalo with 8-byte alignment (per OXC's approach) would be the right choice. But the current thread-local reuse architecture achieves similar amortized allocation cost with simpler code.

#### SIMD for Metrics Struct

The Metrics struct is exactly 16 bytes — fits in an `__m128i` register. SIMD was evaluated for:

1. **Individual field increments:** Not beneficial. SIMD extract/insert for a single lane costs 3–5 cycles vs 1 cycle for scalar `wrapping_add`. The access pattern (one field at a time, different fields for different node types) has zero data parallelism.

2. **Zero-initialization (`Default::default()`):** Already optimized by LLVM. The compiler emits `pxor xmm0, xmm0` (zero a 128-bit register, 0 latency on modern CPUs due to register renaming) followed by a single aligned store. No code changes needed.

---

### References (Section 17)

- [tree-sitter v0.26 TreeCursor API](https://docs.rs/tree-sitter/0.26.0/tree_sitter/struct.TreeCursor.html) — Confirmed no `goto_first_named_child`, no `node_kind_id` on cursor
- [tree-sitter TSNode struct layout](https://github.com/tree-sitter/tree-sitter/blob/master/lib/include/tree_sitter/api.h) — 32 bytes on 64-bit: 4×u32 context + 2 pointers
- [OXC Performance Architecture](https://oxc.rs/docs/learn/performance) — Arena allocation, parse-order traversal, 84x ESLint
- [OXC AST Allocator](https://docs.rs/oxc_allocator/latest/oxc_allocator/) — bumpalo wrapper with 8-byte alignment optimization
- [bumpalo crate](https://github.com/fitzgen/bumpalo) — 11 fast-path instructions per allocation, order of magnitude speedup for small allocations
- [mimalloc benchmarks](https://github.com/microsoft/mimalloc) — 2–7x faster small allocations, per-thread heaps for zero contention
- [SIMD extract/insert overhead](https://stackoverflow.blog/2020/07/08/improving-performance-with-simd-intrinsics-in-three-use-cases/) — Setup/cleanup can exceed inner loop cost for small operations
- [Register zeroing: pxor optimization](https://randomascii.wordpress.com/2012/12/29/the-surprising-subtleties-of-zeroing-a-register/) — Zero-latency via register renaming on modern CPUs
- [Rust Heap Allocations — Performance Book](https://nnethercote.github.io/perf-book/heap-allocations.html) — Vec reuse via clear() is recommended pattern
- [rust-lang/rust#105156: Vec::push capacity check not elided after reserve](https://github.com/rust-lang/rust/issues/105156) — Confirmed LLVM limitation for Vec::push after reserve()

---

---

## 18. Additional Performance & Build Improvements

### Overview

Sections 14–17 focused on tree-sitter API optimization, per-node cost reduction, allocation patterns, and deployment. This section identifies 4 cross-cutting improvements discovered through benchmarking alternative hash implementations, analyzing tree-sitter's `ParseOptions` API (v0.26.3), and profiling build-time overhead.

**6 improvements** (1 medium-high performance, 2 low-medium performance, 1 low performance, 1 medium reliability, 1 low build-time DX) are identified. The most impactful is RP-1 (rapidhash), which replaces ahash for all HashMap operations with a faster hash function optimized for short strings — the dominant key type in scope analysis.

---

### RP-1: Switch from `ahash` to `rapidhash` for HashMap Operations

**Severity:** Medium-High
**Location:** `types.rs:7,283`, `quickscope/Cargo.toml`

**Current behavior:** `Names<'a>` is defined as `AHashMap<&'a [u8], NameData>` using the `ahash` crate. ahash requires AES-NI hardware intrinsics for its fast path and falls back to a slower implementation on CPUs without it.

**Why this matters:** The `names` HashMap is the single most accessed data structure during analysis — every `declaration()` and `reference()` call hits it. Identifier keys are short strings (4–30 bytes), a range where hash function choice has outsized impact because the hash computation itself dominates over memory access.

**Fix:** Replace `ahash::AHashMap` with `rapidhash::RapidHashMap` from the `rapidhash` crate (v4.2.2).

```rust
// Before (types.rs)
use ahash::AHashMap;
pub type Names<'a> = AHashMap<&'a [u8], NameData>;

// After
use rapidhash::RapidHashMap;
pub type Names<'a> = RapidHashMap<&'a [u8], NameData>;
```

Also update `Cargo.toml`: remove `ahash` dependency, add `rapidhash = "4.2"`.

**Evidence:** The foldhash benchmark suite (comprehensive hash function comparison) ranks rapidhash at average rank 2.11 vs ahash at 5.05 across all input sizes. For short strings specifically (the identifier use case), rapidhash consistently outperforms. Unlike ahash, rapidhash does not require AES-NI — it achieves speed through a mathematically optimized mixing function.

**Impact:** 15–25% faster HashMap operations on the single most accessed hot-path data structure. Also applies to future `import_sources` HashMap (FC-1).

---

### RP-2: Pre-reserve HashMap Capacity Using Tree-sitter Node Count

**Severity:** Low-Medium (extends NP-6)
**Location:** `builder.rs` (reserve_estimate), generated traversal code

**Current behavior:** NP-6 proposes Vec pre-allocation using a byte-length heuristic (e.g., `source.len() / 100` for scopes). The `names` HashMap has no pre-allocation — it starts at default capacity and grows via O(n) rehash cycles as symbols are discovered during traversal.

**Why this matters:** HashMap growth is expensive: each rehash copies all entries, recomputes hashes, and re-probes. For a typical 500-line file with ~100 unique symbols, the HashMap grows through capacities 0→8→16→32→64→128, triggering 5 rehash cycles. Each rehash is O(n) — the cumulative cost grows quadratically with symbol count.

**Fix:** After parsing, call `tree.root_node().descendant_count()` to get the exact node count. Use this for HashMap capacity estimation alongside Vec pre-allocation (NP-6):

```rust
let node_count = tree.root_node().descendant_count();
// ~1 unique symbol per 5-8 named nodes (empirically measured)
let estimated_symbols = node_count / 6;
builder.reserve_estimate(source.len(), estimated_symbols);
```

In `reserve_estimate()`, add HashMap pre-allocation:
```rust
pub fn reserve_estimate(&mut self, byte_len: usize, symbol_estimate: usize) {
    // Existing Vec reserves from NP-6...
    self.names.reserve(symbol_estimate);
}
```

**Why node count instead of byte length:** `descendant_count()` returns the exact number of nodes in the parse tree — a direct proxy for code complexity. Byte length conflates comments, whitespace, and string literals with actual code. A 10KB file that's mostly comments has far fewer symbols than a 10KB file of dense logic. Node count captures this distinction. The call itself is O(1) — it reads a pre-computed field from the tree-sitter node struct.

**Impact:** Eliminates O(n) rehash cycles on cold start (first file per thread). On subsequent files (thread-local reuse), the HashMap retains capacity from the previous file — but if the new file is significantly larger, rehashing still occurs. Pre-reservation handles both cases.

---

### RP-3: Gate `prettyplease` Formatting to Debug Builds Only

**Severity:** Low (build-time DX)
**Location:** `quickscope-build/src/lib.rs:70-77`

**Current behavior:** The build script formats generated code through `prettyplease::unparse(syn::parse_file(...))` on every build. This parses the generated token stream into a `syn` AST and pretty-prints it — a non-trivial operation that adds measurable build-time overhead.

**Why this matters:** The formatted output is only useful when debugging the generated code (inspecting `$OUT_DIR/quickscope.rs`). In release builds and CI, no one reads the generated file — the formatting is pure waste.

**Fix:** Gate formatting behind `cfg!(debug_assertions)`:

```rust
let code = if cfg!(debug_assertions) {
    // Pretty-print for readable debugging
    prettyplease::unparse(&syn::parse_file(&all_tokens.to_string()).unwrap())
} else {
    // Raw token stream — faster build, same compilation result
    all_tokens.to_string()
};
std::fs::write(&dest_path, code).unwrap();
```

This allows `syn` and `prettyplease` to be gated as debug-only build dependencies (via `[build-dependencies]` with optional features or conditional compilation).

**Impact:** Faster release builds. Zero runtime impact — the generated Rust code compiles identically regardless of formatting.

---

### RP-4: Use Tree-sitter Progress Callback Instead of Timeout (Supersedes NR-1)

**Severity:** Medium (reliability)
**Location:** Generated traversal code (`traversal.rs`)

**Current behavior:** NR-1 proposes `Parser::set_timeout_micros()` for safety bounds on parsing. This uses wall-clock time, which is platform-dependent, affected by CPU load, and non-deterministic — a parse that succeeds on an idle machine may fail under load.

**Why this matters:** For a server processing untrusted input, deterministic safety bounds are essential. A wallclock timeout creates flaky behavior: the same file may parse successfully or fail depending on system load, CPU frequency scaling, or VM scheduling. This makes debugging and testing unreliable.

**Fix:** tree-sitter 0.26.3 provides `ParseOptions` with a `progress_callback` that receives `ParseState` with `current_byte_offset()` — deterministic bounds based on parse progress rather than elapsed time:

```rust
use tree_sitter::{ParseOptions, ParseState};
use std::ops::ControlFlow;

let max_bytes: usize = 10 * 1024 * 1024; // 10MB safety limit

let tree = parser.parse_with_options(
    source_bytes,
    None,
    &ParseOptions {
        progress_callback: Some(&mut |state: ParseState| {
            if state.current_byte_offset() > max_bytes {
                ControlFlow::Break(())  // Cancel parsing deterministically
            } else {
                ControlFlow::Continue(())
            }
        }),
    },
)?;
```

The callback is invoked periodically during parsing. `current_byte_offset()` reflects how far the parser has progressed through the source — a deterministic metric that doesn't depend on CPU speed or system load. `ControlFlow::Break(())` cancels parsing immediately.

**Why this supersedes NR-1:** Same goal (prevent runaway parsing), strictly better mechanism:
- **Deterministic:** Based on byte offset, not wall clock
- **Platform-independent:** Same behavior on fast and slow machines
- **Testable:** Can write unit tests with exact byte limits
- **Configurable:** Limit can be per-language or per-request

**Impact:** Deterministic safety bound vs wallclock timeout. Stronger "cannot crash the server" guarantee. Replaces NR-1 entirely.

---

### RP-5: Pre-size `bindings` Vec in `reserve_estimate()` (Extends RP-2)

**Severity:** Low
**Location:** `builder.rs:96` (field), `builder.rs:478-491` (declare_binding resize), `builder.rs:428-457` (reference bounds check), `builder.rs:495-531` (handle_var_shadowing bounds check)

**Current behavior:** The `bindings: Vec<SymbolBinding>` vector starts empty and grows on demand. In `declare_binding()`, every new symbol triggers `if idx >= self.bindings.len() { self.bindings.resize(idx + 1, SymbolBinding::EMPTY); }`. In `reference()` (the hottest function — called for every identifier node), the first operation is `if idx >= self.bindings.len() { return; }` — a bounds check on every call.

**Why this matters:** `reference()` is called more frequently than any other builder method. The bounds check `idx >= self.bindings.len()` loads `self.bindings.len` from memory on every call. If `bindings` were pre-sized to cover all symbol indices, this check would always pass and the branch predictor would learn the pattern — but the Vec could also be pre-allocated to eliminate the check entirely via `assert_unchecked`.

**Fix:** When RP-2's `reserve_estimate()` is implemented, extend it to also pre-size `bindings`:

```rust
pub fn reserve_estimate(&mut self, byte_len: usize, symbol_estimate: usize) {
    // Existing reserves from NP-6 and RP-2...
    self.names.reserve(symbol_estimate);
    // Pre-size bindings to cover expected symbol indices
    self.bindings.resize(symbol_estimate, SymbolBinding::EMPTY);
}
```

This eliminates the per-symbol `resize()` in `declare_binding()` for the common case (only files with more symbols than estimated would still trigger growth), and makes the bounds check in `reference()` predictable.

**Impact:** Eliminates per-symbol Vec resize in `declare_binding()` and makes `reference()` bounds check branch-predictor-friendly. Low standalone impact since the branch predictor likely learns the pattern quickly, but it removes a data dependency (load `self.bindings.len` from memory) from the hottest code path.

---

### RP-6: Combine Parser and Builder into Single Thread-Local

**Severity:** Low
**Location:** Generated code in `codegen/mod.rs:69-76` (thread-local declarations), `codegen/traversal.rs:38-39` (two `.with()` calls)

**Current behavior:** Each language gets two separate thread-locals: `PARSER_<LANG>` and `BUILDER_<LANG>`. The `analyze()` function accesses both via two separate `.with()` calls:

```rust
let parser_ptr = PARSER_TYPESCRIPT.with(|cell| cell.get());
let builder_ptr = BUILDER_TYPESCRIPT.with(|cell| cell.get() as *mut u8);
```

Each `.with()` call on a thread-local involves a TLS lookup (typically 2–3 instructions on x86-64: `fs` segment load + offset calculation).

**Fix:** Combine parser and builder into a single struct:

```rust
struct AnalysisState {
    parser: tree_sitter::Parser,
    builder: ScopeBuilder<'static>,
}

std::thread_local! {
    static STATE_TYPESCRIPT: std::cell::UnsafeCell<AnalysisState> = {
        let mut parser = tree_sitter::Parser::new();
        parser.set_language(&tree_sitter_typescript::LANGUAGE_TYPESCRIPT.into())
            .expect("Failed to set language");
        std::cell::UnsafeCell::new(AnalysisState {
            parser,
            builder: ScopeBuilder::new(),
        })
    };
}
```

Then access both via a single `.with()`:

```rust
let state_ptr = STATE_TYPESCRIPT.with(|cell| cell.get());
let state = unsafe { &mut *state_ptr };
// state.parser and state.builder accessed directly
```

**Impact:** Saves one TLS `.with()` access per `analyze()` call (~2–3 instructions). Marginal in isolation, but slightly cleaner codegen. The builder still needs lifetime erasure via `*mut u8` cast for the `ScopeBuilder<'static>` → `ScopeBuilder<'_>` transition.

---

### References (Section 18)

- [rapidhash crate](https://crates.io/crates/rapidhash) — RapidHashMap, 15-25% faster than ahash for short strings
- [rapidhash benchmarks (GitHub)](https://github.com/hoxxep/rapidhash) — Average rank 2.11 vs ahash 5.05
- [tree-sitter ParseOptions docs](https://docs.rs/tree-sitter/0.26.3/tree_sitter/struct.ParseOptions.html) — Progress callback with ControlFlow
- [tree-sitter Node::descendant_count()](https://docs.rs/tree-sitter/0.26.3/tree_sitter/struct.Node.html#method.descendant_count) — Exact subtree node count

---

---

## Unified Priority Table

All improvements from Sections 14–18, merged and re-prioritized holistically. Items are grouped by priority tier (P0 = do first, P1 = do next, P2 = do when convenient), then sorted by impact within each tier. Category tags: **Perf** = performance, **Correct** = correctness bug, **DX** = developer experience, **Ops** = deployment/operational.

```
┌──────────┬───────┬──────────────────────────────────────────────────┬────────┬────────┬─────┐
│ Priority │  ID   │ Issue                                            │ Impact │ Effort │ Cat │
├──────────┼───────┼──────────────────────────────────────────────────┼────────┼────────┼─────┤
│ P0       │ NC-1  │ Wrong grammar for JSX/TSX files                  │ High   │ Low    │ Cor │
│ P0       │ FC-1  │ Import source deduplication                      │ High   │ Low    │ Cor │
│ P0       │ P2-2  │ Add [profile.release] with LTO + codegen-units=1│ High   │ Trivial│ Ops │
│ P0       │ NP-1  │ Skip unnamed nodes in traversal                  │ High   │ Low    │Perf │
│ P0       │ NI-1  │ Subtree skipping (type annotations)              │ High   │ Medium │Perf │
│ P0       │ P0-1  │ Eliminate per-scope Vec allocation (flat buffer)  │ High   │ Low    │Perf │
│ P0       │ P0-2  │ Remove node.parent() from visitor                │ High   │ Medium │Perf │
│ P0       │ NP-2  │ Eliminate Option check in increment_* methods    │ High   │ Low    │Perf │
│ P0       │ FP-1  │ Visitor return value (elim len() reads)          │ High   │ Low    │Perf │
│ P0       │ BC-1  │ Bounds check elim via assert_unchecked (after NP-2)│ High  │ Low    │Perf │
│ P0       │ P1-1  │ Pre-resolve field IDs (child_by_field_id)        │ High   │ Medium │Perf │
│ P0       │ NI-2  │ Pack is_function into IndexStack                 │ Med-Hi │ Low    │Perf │
├──────────┼───────┼──────────────────────────────────────────────────┼────────┼────────┼─────┤
│ P1       │ FP-2  │ Cursor field_id() propagation                    │ High   │ Medium │Perf │
│ P1       │ NP-3  │ reference() HashMap pollution                    │ Med-Hi │ Low    │Perf │
│ P1       │ RP-1  │ Switch ahash → rapidhash for HashMaps            │ Med-Hi │ Trivial│Perf │
│ P1       │ LT-1  │ Const lookup table for visitor early-exit        │ Med-Hi │ Low    │Perf │
│ P1       │ FP-6  │ Use grammar_id() instead of kind_id()            │ Medium │ Low    │Perf │
│ P1       │ P1-2  │ Replace kind() == "str" with kind_id() == ID     │ Medium │ Low    │Perf │
│ P1       │ P1-3  │ Fix take_result to preserve Vec capacity         │ Medium │ Low    │Perf │
│ P1       │ NP-4  │ Redundant UTF-8 validation                       │ Medium │ Low    │Perf │
│ P1       │ NP-6  │ Pre-allocate Vecs by file size                   │ Medium │ Low    │Perf │
│ P1       │ RP-2  │ Pre-reserve HashMap via descendant_count()       │ Low-Med│ Low    │Perf │
│ P1       │ FP-4  │ Function scope stack (O(1) restore)              │ Medium │ Low    │Perf │
│ P1       │ FP-3  │ Extension dispatch via integer encoding          │ Medium │ Low    │Perf │
│ P1       │ FP-7  │ Extension extraction overhead in analyze()       │ Medium │ Low    │Perf │
│ P1       │ NP-5  │ Merge parallel shadow Vecs                       │ Low-Med│ Low    │Perf │
│ P1       │ NI-3  │ Remove parents Vec (after FP-4)                  │ Medium │ Trivial│Perf │
│ P1       │ NI-4  │ mimalloc in server binary                        │ Medium │ Trivial│ Ops │
│ P1       │ NR-1  │ Parser timeout for safety (→ superseded by RP-4)  │ Med/DX │ Trivial│ DX  │
│ P1       │ RP-4  │ Progress callback instead of timeout (sup. NR-1) │ Medium │ Low    │ DX  │
│ P1       │ DX2   │ Add proper error types (AnalysisError enum)      │ Med/DX │ Low    │ DX  │
│ P1       │ DX3   │ Borrow File<'a> instead of owning String         │ Med/DX │ Low    │ DX  │
│ P1       │ FD-1  │ README/API documentation drift                   │ Med/DX │ Low    │ DX  │
├──────────┼───────┼──────────────────────────────────────────────────┼────────┼────────┼─────┤
│ P2       │ RP-3  │ Gate prettyplease to debug builds                 │ Low    │ Low    │ DX  │
│ P2       │ NO-1  │ PGO build documentation                          │ Medium │ Low    │ Ops │
│ P2       │ FO-1  │ BOLT post-link optimization                      │ Medium │ Low    │ Ops │
│ P2       │ DX4   │ Comprehensive test coverage                      │ Med/DX │ Medium │ DX  │
│ P2       │ DX1   │ Improve handler body DX (proc-macro)             │ Med/DX │ High   │ DX  │
│ P2       │ NI-5  │ Borrowed AnalysisResult<'a>                      │ Low-Med│ Low    │Perf │
│ P2       │ NP-7  │ #[cold] on unlikely paths + cold_path()           │ Low    │ Trivial│Perf │
│ P2       │ FP-5  │ Redundant len > 0 guard in leave()               │ Low    │ Trivial│Perf │
│ P2       │ NI-6  │ Vec::push capacity check elimination             │ Low    │ Low    │Perf │
│ P2       │ NI-7  │ Unnecessary zero-fill in clear() stacks          │ Low    │ Trivial│Perf │
│ P2       │ FO-2  │ target-cpu=native for deployment                 │ Low    │ Trivial│ Ops │
│ P2       │ NR-2  │ Cancellation flag support                        │ Low/DX │ Trivial│ DX  │
│ P2       │       │ Remove unused Score allocation                    │ Low    │ Low    │Perf │
│ P2       │       │ Generalize scope model for multi-language*        │ Low*   │ High   │ DX  │
│ P2       │ DX5   │ Inspect generated code easily                     │ Low    │ Low    │ DX  │
│ P2       │ DX6   │ Benchmark representativeness (more fixtures)      │ Med/DX │ Low    │ DX  │
│ P2       │       │ Remove unused phf_codegen build dependency        │ Low    │ Trivial│ Ops │
│ P2       │       │ Replace #[allow] with #[expect] (Rust 2024)       │ Low    │ Low    │ DX  │
│ P2       │       │ Audit function_scope Vec necessity in output      │ Low    │ Trivial│ DX  │
│ P2       │ RP-5  │ Pre-size bindings Vec in reserve_estimate()       │ Low    │ Trivial│Perf │
│ P2       │ RP-6  │ Combine parser+builder into single thread-local   │ Low    │ Low    │Perf │
└──────────┴───────┴──────────────────────────────────────────────────┴────────┴────────┴─────┘

  * Low impact now, high impact when adding second language
```

### Implementation Notes

**P0 tier (do first):** 12 items. Correctness bugs (NC-1, FC-1) should be fixed before any performance work — wrong results fast is worse than correct results slower. The release profile (P2-2) is trivial and unlocks cross-crate inlining that makes all other `#[inline(always)]` annotations effective. The remaining P0 items collectively eliminate the majority of wasted work: NP-1 skips ~65% of nodes (unnamed), NI-1 skips ~30–50% of remaining named nodes (type annotations), P0-1/P0-2/NP-2/FP-1/NI-2 optimize the surviving ~18–25% of nodes. BC-1 eliminates bounds checks on all `increment_*` methods via `assert_unchecked` once NP-2 guarantees a valid scope index. P1-1 is elevated to P0 because `child_by_field_name()` performs a linear scan (`strncmp` loop in `language.c:232`) — more expensive than the original "Medium" assessment suggested.

**P1 tier (do next):** 21 items. These refine the hot path further (FP-2 eliminates all child navigation, NP-3/NP-4 reduce hash/validation overhead, LT-1 provides O(1) early-exit for unhandled node kinds via const lookup table instead of LLVM's binary search tree, FP-6 replaces `kind_id()` with `grammar_id()` to save a branch + array lookup per node, FP-7 eliminates redundant path extension extraction per file) and address batch throughput (P1-3, NP-6, NI-3 for Vec reuse). RP-1 (rapidhash) replaces ahash for all HashMap usage — 15–25% faster for short-string keys. RP-2 uses tree-sitter's `descendant_count()` for accurate HashMap pre-allocation. RP-4 supersedes NR-1 with deterministic progress-callback-based safety bounds. DX items (DX2, DX3, FD-1) improve the API surface for server integration.

**P2 tier (do when convenient):** 21 items. RP-3 gates `prettyplease` formatting to debug builds only, eliminating wasted build-time overhead in release/CI. Deployment optimizations (PGO, BOLT, target-cpu) yield 10–30% additional gains but require build infrastructure. NI-6 (Vec::push capacity check elimination), NI-7 (clear() memset waste), and NP-7 (`#[cold]` + `cold_path()`) are micro-optimizations to apply after profiling confirms remaining bottlenecks. RP-5 extends RP-2 by also pre-sizing the `bindings` Vec in `reserve_estimate()`, removing a data dependency from the hottest code path (`reference()`). RP-6 combines the per-language parser and builder thread-locals into a single struct, saving one TLS access per `analyze()` call. DX items (tests, proc-macro, generated code inspection, DX6 benchmark fixtures) improve maintainability. Housekeeping: remove dead phf_codegen dependency, migrate `#[allow]` to `#[expect]` (Rust 2024 edition), audit whether function_scope Vec is needed by consumers.

**Estimated combined impact:** P0 items alone are estimated to yield **30–60% improvement** on the analysis portion (excluding tree-sitter parse time). P1 items add another **10–20%**. Deployment optimizations (P0 release profile + P2 PGO/BOLT) add **15–30%** on top. The multiplicative effect of reducing both node count (NP-1 + NI-1) and per-node cost (all other P0 items) is greater than the sum of individual improvements.

**Dependency graph:**

```
NC-1 (JSX grammar)          ← standalone, do first
FC-1 (import dedup)          ← standalone, do first
P2-2 (release profile)       ← standalone, do first

NP-1 (skip unnamed)          ← standalone
NI-1 (subtree skip)          ← benefits from FP-1 (visitor return value)
P0-1 (per-scope alloc)       ← standalone
P0-2 (remove node.parent)    ← enables FP-2 (parent_kind_id propagation)
NP-2 (elim Option check)     ← standalone
BC-1 (bounds check elim)     ← depends on NP-2 (guaranteed valid index)
FP-1 (visitor return value)  ← enables NI-1 (subtree skip signal)
P1-1 (field_id resolution)   ← standalone (linear scan makes this P0)
NI-2 (tagged IndexStack)     ← standalone
LT-1 (lookup table)          ← standalone, needs benchmarking

FP-2 (cursor field_id)       ← depends on P0-2 (parent_kind_id)
FP-4 (fn scope stack)        ← enables NI-3 (remove parents Vec)
FP-6 (grammar_id)            ← standalone, build-time codegen change
NI-3 (remove parents)        ← depends on FP-4

DX3 (File<'a>)               ← enables NI-5 (borrowed AnalysisResult), enables FP-7
FP-7 (extension extraction)  ← benefits from DX3 (File<'a>), combines with FP-3
NI-5 (borrowed result)       ← depends on DX3

NI-6 (Vec::push elim)        ← benefits from NP-6 (pre-allocation guarantees capacity)
NI-7 (clear() memset)         ← standalone, trivial
DX6 (benchmark fixtures)      ← standalone, informs optimization priorities

RP-1 (rapidhash)             ← standalone, trivial
RP-2 (HashMap pre-reserve)   ← extends NP-6, uses descendant_count after parse
RP-3 (prettyplease gating)   ← standalone, build-time only
RP-4 (progress callback)     ← supersedes NR-1, requires parse_with_options
RP-5 (bindings pre-size)     ← extends RP-2 (shares reserve_estimate)
RP-6 (combined thread-local) ← standalone, codegen change
```

### Non-Actionable Notes

**`-C jump-tables=n` (stabilized Rust 1.93):** Controls whether LLVM lowers `match` arms to jump tables or binary search trees. For sparse matches on `u16` kind_ids (~30 handled arms out of ~350 possible values, density ~8%), LLVM defaults to a balanced binary search tree (~5 comparisons). Forcing `-C jump-tables=n` ensures this stays a binary search tree even if LLVM's density heuristics change. Worth experimenting with once the visitor is stable — measure both directions.
