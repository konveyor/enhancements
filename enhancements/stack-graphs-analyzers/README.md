---
title: stack-graphs-based-language-analyzers
authors:
  - "@JonahSussman"
reviewers:
  - "@shawn-hurley"
  - "@djzager"
  - "@eemcmullan"
  - "@pranavgaikwad"
  - "@jmle"
approvers:
  - TBD
creation-date: 2026-04-14
last-updated: 2026-04-17
status: provisional
see-also:
  - "/enhancements/dotnet-provider-treesitter/README.md"
  - "https://github.com/jmle/java-analyzer-provider"
  - "https://github.com/github/stack-graphs"
  - "https://arxiv.org/pdf/2211.01224"
replaces: []
superseded-by: []
---

<!-- NOTE: This enhancement is intentionally loose with the standard format.
     The problem space here spans multiple existing providers, a forked upstream
     library, and design questions that need community input. I felt a narrative
     structure — background, discoveries, open questions, proposal — was more
     effective than the usual section-by-section template for communicating the
     full picture. -->

# Stack-Graphs-Based Language Analyzers

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

We propose building Konveyor's language analysis on top of a maintained fork of [GitHub's stack-graphs](https://github.com/github/stack-graphs) — a Rust library for cross-file name resolution using tree-sitter. The fork would consolidate language-specific grammars (starting with C# and Java) into a single repository, fix gaps in the core library, and extend stack graphs beyond name resolution to support introspection queries like "find all references to `javax.servlet.http.HttpServlet`" or "what depends on namespace `NerdDinner.Models`."

This replaces the approach of maintaining separate Go-based language providers that depend on heavyweight language servers (JDTLS for Java, csharp-ls for C#).

## The Problem

Konveyor needs to analyze codebases to support application migration. Today that analysis is done by language-specific Go providers that shell out to language servers.

### The language servers don't work well

- **`dotnet-external-provider`** uses `csharp-ls`, which fails to reliably resolve references. Analysis results are incomplete or wrong. The [`c-sharp-analyzer-provider`](https://github.com/konveyor/c-sharp-analyzer-provider) uses a stack-graphs-based C# grammar, but the grammar itself has correctness issues — the underlying stack-graphs approach is sound, but the grammar needs significant work to produce correct results. The [tree-sitter-only enhancement](/enhancements/dotnet-provider-treesitter/README.md) was proposed in 2024 as a simpler alternative, but explicitly excluded cross-file analysis — making it insufficient for migration rules that track dependencies across files.
- **`java-external-provider`** uses JDTLS. This works better, but requires a running JVM, is slow to start, memory-hungry, and creates duplicate processes when used alongside IDE extensions.

These language servers are also *very large*. JDTLS pulls in the entire Eclipse JDT infrastructure. csharp-ls pulls in the .NET SDK tooling. Containerizing them means shipping enormous images. And when they break, debugging requires deep knowledge of the language server internals — not the analysis domain.

### No shared infrastructure or standardized queries

Each provider is built independently with its own architecture, query capabilities, and rule format. There is no way to write a query that works across languages. Adding a new language means building a new provider from scratch — new Go code, new language server dependency, new containerization, new bugs.

A migration rule like "find all references to `com.example.OldClass`" should be expressible the same way regardless of whether the codebase is Java, C#, or Python. Today, it can't be.

## What Are Stack Graphs?

Stack graphs are a language-agnostic framework for cross-file name resolution. They build on the *scope graphs* formalism from programming language theory ([Néron et al., 2015](https://doi.org/10.1007/978-3-662-46669-8_9); [van Antwerpen et al., 2016](https://doi.org/10.1145/2847538.2847543)) and were developed at GitHub for code navigation at scale. The [stack graphs paper](https://arxiv.org/pdf/2211.01224) (Creager & van Antwerpen, 2023) describes the full formalism.

### The core idea

In scope graphs, name binding information is encoded as a graph. Definitions and references are nodes; scoping rules (lexical scope, imports, inheritance) are edges. Resolving a reference means finding a path through the graph from a reference node to a definition node.

Stack graphs extend this with two key innovations:

> **A note on terminology:** Tree-sitter produces a *Concrete Syntax Tree* (CST) — a parse tree that preserves every token, including punctuation and whitespace. This is distinct from an *Abstract Syntax Tree* (AST), which strips syntactic noise. This document uses "CST" throughout since that's what tree-sitter actually produces. It also uses "stack graphs" (unhyphenated) for the formalism and "stack-graphs" (hyphenated) for the GitHub library and repository.

1. **Push and pop symbol nodes.** Instead of simple definition/reference nodes, stack graphs use *push symbol nodes* (↓x, which prepend a symbol onto a stack) and *pop symbol nodes* (↑x, which require and remove a matching symbol from the stack). A reference `x` becomes a push node ↓x — it *pushes* the name it's looking for onto the stack. A definition `x` becomes a pop node ↑x — it *pops* the name it provides, consuming the request. Name resolution is finding a path where all pushes and pops cancel out, leaving an empty stack. This mechanism also handles *type-dependent* lookups — see the worked example below.

2. **File incrementality via root nodes.** Each source file produces a disjoint subgraph with no edges crossing file boundaries. Files connect to each other only through *root nodes* — special nodes that act as entry/exit points. At query time, the path-finding algorithm creates virtual edges between root nodes in different files, allowing cross-file resolution. This means each file can be analyzed independently: no knowledge of other files is needed at index time.

The practical consequence: the name binding rules for a language are encoded entirely in the graph structure. The resolution algorithm is language-independent — the same algorithm resolves names in Python, Java, C#, or any other language.

### Worked example: name resolution

Push/pop is the mechanism for *all* name resolution in stack graphs — even the simplest intra-file reference.

**Intra-file resolution** — the simplest case:

```csharp
class Foo {
    string title;
    void Bar() {
        title;  // <-- resolve this
    }
}
```

The TSG grammar rules process this file and create graph nodes:
- ↑title (pop node — the field *definition*)
- ↓title (push node — the *reference* inside `Bar`)
- Scope nodes connected by edges encoding lexical scoping: Bar's scope → Foo's member scope

The TSG grammar rules create edges that encode the scoping rules. For this example, the edges look like:

```
  ↑title      ↑Bar           <-- pop nodes (definitions in this scope)
    ^           ^
    |           |
  [Foo's member scope]       <-- "Bar is nested in Foo, so Foo's members are visible"
    ^
    |
  [Bar's scope]              <-- "resolve references in Bar here"
    ^
    |
  ↓title                     <-- push node (reference to title)
```

The edge from ↓title to Bar's scope says "to resolve references in Bar, look in Bar's scope." The edge from Bar's scope to Foo's member scope says "Bar's scope is nested inside Foo — names from Foo's members are visible here." The edges from Foo's member scope to ↑title and ↑Bar say "title and Bar are defined in this scope."

Resolution:
1. Start at ↓title. Push `title` onto the stack. Stack: `⟨title⟩`
2. Walk edges. The algorithm follows every outgoing edge — it's a breadth-first search, not a targeted lookup. ↓title → Bar's scope → Foo's member scope.
3. From Foo's member scope, there are two outgoing edges: one to ↑title, one to ↑Bar. The algorithm tries both. Pop nodes act as *gates*: you can only pass through if the symbol matches the top of the stack. ↑Bar requires `Bar` on top — doesn't match `title`, path pruned. ↑title requires `title` — matches, so pop it. Stack: `⟨⟩`
4. Stack is empty, we're at a definition node. **Path is complete.** The reference `title` resolves to the field definition.

(If there were also a ↑other node in the same scope, the algorithm would try that edge too — but ↑other requires `other` on top of the stack, `title` doesn't match, so that path is pruned.)

Every name resolution — no matter how complex — reduces to this: push symbols onto the stack, walk edges, pop matching symbols off. When the stack is empty at a definition node, you've found the answer.

**Cross-file resolution** — adding namespaces and imports:

```csharp
// Models.cs
namespace NerdDinner.Models {
    class Dinner {
        public string Title;
    }
}

// App.cs
using NerdDinner.Models;
class App {
    void Run() {
        Dinner d = new Dinner();
    }
}
```

Each file produces an independent subgraph:

- **`Models.cs`** creates pop nodes for its namespace chain: ↑NerdDinner, ↑`.`, ↑Models, ↑`.`, ↑Dinner — connected through scope nodes back to a *root node*. This is the "export" side: it says "I define something reachable via `NerdDinner.Models.Dinner`."

- **`App.cs`** creates a push node ↓Dinner for the reference. The `using NerdDinner.Models` directive creates a chain of push nodes — ↓NerdDinner, ↓`.`, ↓Models, ↓`.` — that lead to a root node. These push nodes are wired so that when you start at ↓Dinner, the path first passes through the `using` directive's push chain, building up the full namespace on the stack before reaching the root node.

Resolution of `Dinner` in `App.cs`:
1. Start at ↓Dinner. Stack: `⟨Dinner⟩`
2. Follow edges through the `using` directive's push chain: ↓`.`, ↓Models, ↓`.`, ↓NerdDinner. Each push node prepends its symbol onto the stack. Stack is now: `⟨NerdDinner.Models.Dinner⟩`
3. Reach a root node. The algorithm creates a *virtual edge* to root nodes in other files — this is the only way paths cross file boundaries.
4. Enter `Models.cs`. Pop ↑NerdDinner, ↑`.`, ↑Models, ↑`.`, ↑Dinner — each pop matches and removes the top of the stack. Stack: `⟨⟩`
5. Path complete. `Dinner` resolves to the class definition in `Models.cs`.

No file needs to know about any other file at index time. The virtual edges between root nodes are created at query time.

**Member access** — resolving `d.Title`:

```csharp
Dinner d = new Dinner();
d.Title;  // <-- resolve this
```

The expression `d.Title` involves multiple lookups: resolve `d`, then resolve `Title` within `d`'s type. The grammar encodes this as a chain of push nodes. The algorithm starts at ↓Title and walks through the chain, building up the stack:

1. Start at ↓Title. Push `Title`. Stack: `⟨Title⟩`
2. Walk to ↓`.`. Push `.`. Stack: `⟨.Title⟩`
3. Walk to ↓d. Push `d`. Stack: `⟨d.Title⟩`
4. Reach ↑d (the local variable definition). Pop `d`. Stack: `⟨.Title⟩`
5. The grammar has placed edges from `d`'s definition scope *through the type's scope*. The details of how this routing works depend on the grammar's conventions — the Java grammar uses `":"` pop edges to model type-of relationships, while other grammars may wire scope edges directly. Either way, the path enters Dinner's member scope.
6. Pop `.`. Stack: `⟨Title⟩`
7. Reach ↑Title (the field definition). Pop `Title`. Stack: `⟨⟩`. Done.

The key insight: **the stack handles nested lookups.** While resolving `d`'s type (step 3), `Title` stays on the stack, waiting. Once the type is resolved and we're in the right scope, the stack "resumes" with the remaining lookup. This is why they're called *stack* graphs — the stack of pending symbols is the core data structure that makes type-dependent resolution work without eager cross-file lookups.

### Partial paths: precomputing work

A naive implementation would do all path-finding at query time — expensive for large codebases. Stack graphs address this with *partial paths*: precomputed path segments within a single file that are calculated at index time and stored. Each partial path has a *precondition* (what must be on the symbol stack when entering) and a *postcondition* (what will be on the stack when leaving).

For example, within `Models.cs`, there's a partial path that says: "if you arrive at my root node with `⟨NerdDinner.Models.Dinner.Title⟩` on the stack, I can resolve it down to the field definition with an empty stack." At query time, the algorithm concatenates compatible partial paths across files rather than walking individual edges. This shifts most of the computational work to index time while keeping file incrementality.

### The stack-graphs library

The [stack-graphs](https://github.com/github/stack-graphs) repository is a Rust workspace implementing this formalism:

- **`stack-graphs`** — The core library. Defines the graph data model (`StackGraph`), node types (root, scope, push symbol, pop symbol), the partial path data structures, and the path-stitching algorithm (`ForwardPartialPathStitcher`). Includes an **SQLite-based storage layer** for persisting graphs and partial paths across sessions. Each file's subgraph and partial paths are stored independently, keyed by a content hash — if a file hasn't changed, its stored data is reused without re-indexing. Nodes carry metadata via a `SourceInfo` struct (source span, syntax type, containing line, definiens span, fully qualified name).

- **`tree-sitter-stack-graphs`** — The bridge between tree-sitter and stack graphs. Defines a declarative DSL called TSG (tree-sitter-graph) for writing *rules* that transform tree-sitter parse trees into stack graph nodes and edges. Provides a CLI for indexing source files, querying the graph, and running resolution tests. Handles loading source files, running TSG rules, building the graph, computing partial paths, and persisting everything to SQLite.

- **`languages/`** — Per-language grammar crates. Each contains TSG rules (`.tsg` files), a Rust wrapper crate, and test cases. GitHub shipped grammars for Python, Java, JavaScript, and TypeScript. The repo is now archived and unmaintained.

The key thing for Konveyor: this gives us a single Rust binary that can parse source files, build a semantic graph with cross-file name resolution, persist it to SQLite, and answer "what does this reference resolve to?" — without a language server, JVM, or .NET runtime.

## What's Already Been Built

### C# Grammar

Shawn Hurley started a C# stack-graphs grammar (`tree-sitter-stack-graph-csharp`) in early 2025. It has since been expanded with:

- Name resolution for namespaces (simple, qualified, file-scoped), classes, structs, interfaces, enums, records, methods, constructors, fields, properties, parameters, local variables, delegates, events, type parameters, and local functions
- Using directives (simple, qualified, aliased)
- Inheritance and interface implementation
- FQN (fully qualified name) reconstruction — `NerdDinner.Models.Dinner.Title`
- Definiens tracking — knowing where a definition's body starts and ends
- A `find-node` subcommand (of the `tree-sitter-stack-graphs` CLI) for querying definitions by FQN regex
- ~40 resolution test files, ~15 FQN/definiens test files

### Java Analyzer (jmle)

[jmle's java-analyzer-provider](https://github.com/jmle/java-analyzer-provider) is a Rust-based Java analyzer that already uses stack-graphs + tree-sitter. It includes:

- A gRPC provider service compatible with analyzer-lsp
- A custom `TypeResolver` for Java inheritance
- Dependency analysis for Maven/Gradle
- 191+ tests, 34/42 existing analyzer-lsp rules passing

This is strong evidence that stack graphs can replace JDTLS for Konveyor's needs, though the 34/42 pass rate (81%) means there's still work to do.

### Moving Into a Fork

The C# grammar has been moved into [a fork of the stack-graphs repo](https://github.com/JonahSussman/stack-graphs) (eventually `konveyor/stack-graphs`) as a workspace member alongside the existing language grammars. The plan is to absorb jmle's Java grammar the same way, creating a single repository with shared infrastructure.

## What We Discovered Along the Way

Building the C# grammar and introspection tooling revealed gaps in the stack-graphs library and important design questions that need to be resolved. The discoveries below are interconnected — they all point toward the same underlying need: **stack graphs need a standardized metadata schema and the infrastructure to persist and query it.**

### The Serde Gap

`SourceInfo` — the struct that holds metadata about each graph node — has five fields in the core library:

| Field | Purpose | Survives SQLite? |
|-------|---------|:---:|
| `span` | Source location | Yes |
| `syntax_type` | Kind of definition (class, method, etc.) | Yes |
| `containing_line` | Full text of the source line | **No** |
| `definiens_span` | Span of the definition body | **No** |
| `fully_qualified_name` | FQN like `NerdDinner.Models.Dinner` | **No** |

The serialization layer only writes `span` and `syntax_type` to SQLite. The other three fields exist in the data model, are populated at build time by TSG rules, but are silently dropped when the graph is persisted. This is a ~20-line fix in `stack-graphs/src/serde/graph.rs`.

This matters because `SourceInfo` is the primary mechanism for attaching metadata to graph nodes — and it's the metadata we need for introspection. Without fixing this, we can't persist FQNs, definition body spans, or any other node-level data through the SQLite round-trip.

We worked around this by storing definiens information as `debug_` attributes (the only escape hatch TSG provides for custom metadata) and reconstructing FQNs at query time by walking `debug_fqdn = "parent"` edges between definition nodes. Both are hacks.

### The Need for Schema Standardization

The serde gap is really a symptom of a deeper problem: **stack graphs has no standardized schema for what metadata definitions should carry.** Each grammar is free to define whatever `syntax_type` values it wants, use whatever edge conventions it likes, and attach whatever `debug_` attributes it needs. There's no contract between grammars and the tools that query them.

Comparing the C# and Java grammars makes this concrete:

| Concept | Java | C# |
|---------|------|----|
| Static vs instance | Separate `.defs` / `.static_defs` scope chains | Not distinguished |
| Type-of relationship | `":"` pop edges | Not modeled |
| `this` / `super` | Explicit pop nodes | Not modeled |
| FQN containment | Not modeled | `debug_fqdn = "parent"` edges |
| Definiens tracking | Not modeled | `definiens_node` attribute |
| Wildcard imports | Partially implemented | N/A |

Both grammars "work" for basic name resolution, but they produce graphs with different metadata, different edge conventions, and different capabilities. Cross-language queries are impossible without standardization.

This connects to the serde fix: fixing `SourceInfo` serialization is necessary but not sufficient. We also need to define what metadata every grammar *must* produce — a schema that the query API can rely on. This includes not just `syntax_type` values but also how type relationships, containment hierarchies, and definition metadata are represented in the graph.

### FQN Computation

When processing `class Dinner` inside `namespace NerdDinner.Models`, we need the fully qualified name `NerdDinner.Models.Dinner`. We initially resorted to a hack: `debug_fqdn = "parent"` edges between definition nodes, walked at query time to reconstruct the FQN.

TSG does support string concatenation, and the [`c-sharp-analyzer-provider`](https://github.com/konveyor/c-sharp-analyzer-provider) attempted to compute FQNs this way — but the result was fragile and hard to maintain. Alternatively, the FQN is a property of the graph structure itself: it's encoded in the chain of pop nodes from the root to the definition. A post-processing pass could extract it by walking the graph without any special edges.

**This needs more investigation.** The `debug_fqdn` hack works but is ugly, and we shouldn't bake it into the architecture without exploring cleaner alternatives. See Open Question 5 below.

### The Resolution Engine Is Forward-Only

The path stitcher (`ForwardPartialPathStitcher`) resolves references to definitions — ref → def. There is no backward stitcher. "Find all references to class Dinner" requires resolving *every* reference in the graph and checking which ones land on `Dinner`. For a large codebase this is expensive.

This is directly related to the incrementality question below. Indexing is per-file, but answering "who references X?" is inherently a full-project question. One approach: after indexing all files, do a single resolution pass that resolves every reference and persists the ref→def mappings in a SQLite table. Then "find all references" is just a SQL query. This resolution pass is full-project but cheap — it's only path stitching over already-indexed partial paths, no re-parsing.

### Stack Graphs Don't Store the CST

The stack graph is a *derived* structure — tree-sitter parses source into a CST, TSG rules transform relevant parts of that CST into graph nodes and edges, and then the CST is discarded. The original parse tree is not persisted.

This matters for introspection. When a query asks "what type is this field?" or "what modifiers does this method have?", that information lives in the CST, not in the graph. Currently, answering these questions requires the original source files to be on disk so we can re-parse them. If the source is gone — or if we want a self-contained analysis database — we're stuck.

### Incrementality vs. Full-Project Queries

Stack graphs are designed to be *file-incremental*: each file produces an independent subgraph, and files connect only through root nodes at query time. This is powerful for indexing — change one file, re-index only that file.

But introspection queries like "find all references to `The.Namespace.*`" require resolving against the *entire* project graph, not just one file. This creates a tension: indexing is incremental, but the queries we want to answer are not.

The simplest approach is a full resolution pass after indexing: resolve every reference, persist the mappings, answer queries from the persisted table. This definitely works but re-resolves everything even if only one file changed.

A more interesting possibility: **exploit partial path pre/postconditions for incremental resolution.** Partial paths already encode what each file can resolve — the preconditions say "I need these symbols on the stack to enter" and postconditions say "these symbols will be on the stack when leaving." When a file changes:

1. Re-index that file (new partial paths — fast, file-incremental)
2. Invalidate reverse index entries involving the changed file
3. Use the preconditions/postconditions to identify which partial paths from *other* files could route through the changed file's root nodes, and re-resolve only those

This could avoid the full resolution pass entirely. But there are complications: intra-file references in the changed file all need re-resolution, and references from other files that resolved *through* the changed file also need re-resolution. It's not clear whether the bookkeeping to track "which resolutions are affected" is cheaper than just re-resolving everything. This needs prototyping.

## Open Questions

These are the questions that need discussion before the design is finalized.

### 1. TSG vs Lua

The TSG DSL has significant limitations: no real conditionals, no debugging, cryptic errors. It's a custom DSL that's harder to work with than a real language. A Lua-based alternative would let grammar authors use normal programming constructs:

```lua
on_match([[ (class_declaration name: (identifier) @name) @cls ]], function(m)
  local parent_fqdn = m.cls:inherited("parent_fqdn") or ""
  local my_fqdn = parent_fqdn .. "." .. text(m.name)
  define(m.name, { syntax_type = "class", fqdn = my_fqdn })
end)
```

But TSG's tree-sitter query pattern matching is good, and every existing grammar (Python, Java, JavaScript, TypeScript, C#) would need rewriting. A middle ground — keeping TSG for pattern matching but adding Lua as a computation layer — might be worth exploring.

**Should we invest in a Lua-based alternative, or work within TSG's constraints for now?** The C# grammar already works with TSG (plus hacks). The question is whether the hacks accumulate to the point where a better DSL pays for itself.

### 2. CST/Source Storage

The stack graph persists to SQLite, but the original parse tree (CST) is discarded after graph construction. Introspection queries that need syntactic detail (types, modifiers, parameter lists) currently require re-parsing the original source files from disk.

There are several options:

- **Store nothing extra** — require source files on disk during analysis. This is fine for Konveyor's batch analysis model (source is always available during `kantra` runs) but limits portability of the analysis database.
- **Store source text** — persist the raw source text in SQLite alongside the graph. The database becomes self-contained; re-parse on demand with tree-sitter when syntactic detail is needed. Larger database, but source text compresses well.
- **Store the CST** — persist a serialized form of the tree-sitter CST. Avoids re-parsing entirely, but CSTs are large and the serialization format would need to be defined.

The right answer may depend on how often `inspect()`-style queries are used in practice and whether we lean toward encoding more information in the graph via type edges (see question 4).

### 3. Graduated Replacement Criteria

The stack-graphs providers would initially run alongside existing Go providers, not replace them. At what point do they become the default?

**What benchmark should trigger switching?** Options include:
- X% of existing analyzer-lsp rules passing against the stack-graphs provider
- Specific rule categories passing (e.g., all `referenced` rules)
- Performance parity (indexing time, query time, memory)
- Manual sign-off from rule authors

The most practical starting point is probably rule pass rate — it's concrete and measurable. But the threshold needs to be agreed on: is 90% enough, or do we need 100% parity?

### 4. Type Information: Graph Edges vs. Re-parsing

Stack graphs handle name resolution — "this reference points to that definition." They don't model types. When a rule needs to know "what type is this field?" or "does this class implement interface X?", we need a separate mechanism.

There are two approaches, and we're currently leaning toward the first:

- **Type edges in the graph.** The Java grammar already uses `":"` pop edges for type-of relationships, resolved through graph traversal (as shown in the worked example above). This encodes type information *in* the graph, making it queryable after indexing without re-parsing source. The resolution engine already supports it — `":"` and `"()"` are just symbols on the stack, handled by the same push/pop mechanism. The tradeoff: every grammar must model types consistently, which means the schema must define how type edges work across languages. This is more upfront work per grammar but produces a richer, more self-contained analysis.

- **Re-parse on demand.** The stack graph tells us *where* the definition is (file + span). We re-parse the source file with tree-sitter and read the local syntactic detail (type annotations, modifiers, parameter lists). Simpler per grammar, no graph changes needed, but requires source files on disk and can't answer cross-file type questions (e.g., "what does `var x = GetDinner()` return?").

The split between what goes in the graph vs. what comes from re-parsing needs to be defined clearly in the schema.

### 5. How Should FQN Work?

We currently compute FQNs using `debug_fqdn = "parent"` edges — a hack that uses the `debug_` escape hatch in TSG to create edges between definition nodes, then walks those edges at query time to reconstruct the dotted name.

This works, but `debug_` attributes are a workaround, not a supported mechanism. Before building more infrastructure on top of this pattern, we need to decide:

- **Can FQN be derived from the graph structure directly?** The chain of pop nodes from root to definition *encodes* the fully qualified name. A post-processing pass could walk the graph to extract it without any special edges.
- **Should `fully_qualified_name` be populated by a post-build pass?** If so, what's the input — `debug_fqdn` edges, or something cleaner?
- **Should FQN be a first-class TSG attribute?** Could we add a new attribute type to `tree-sitter-stack-graphs` that tells the builder "this definition's FQN includes its parent's FQN"?
- **Can TSG compute FQN inline?** TSG has string concatenation, but the [`c-sharp-analyzer-provider`](https://github.com/konveyor/c-sharp-analyzer-provider) tried this approach and found it fragile. Is there a better way to use TSG's capabilities, or is this inherently awkward in the DSL?

The `debug_fqdn` hack should not become the permanent mechanism. This needs design work.

### 6. How jmle's Java Work Fits In

jmle's java-analyzer-provider is a working system with its own architecture: gRPC service, TypeResolver, dependency analysis. Absorbing it means either:

- **Adapting it** to use the shared infrastructure (FQN from SourceInfo, reverse index from SQLite, shared query API) while keeping its Java-specific features (TypeResolver, Maven/Gradle analysis)
- **Using it as reference** while rewriting the Java grammar to follow the standardized schema

The first approach is faster but may leave architectural inconsistencies. The second is cleaner but risks losing working functionality.

### 7. Incrementality and Full-Project Queries

File-incremental indexing vs. full-project queries is a fundamental tension (see context in "Incrementality vs. Full-Project Queries" and "The Resolution Engine Is Forward-Only" above). The approaches under consideration:

- **Full resolution pass** after indexing: resolve every reference, persist the mappings. Simple and definitely correct, but re-resolves everything even when only one file changed.
- **Incremental resolution via partial paths**: use pre/postconditions to identify which resolutions are affected by a file change and only re-resolve those. Potentially much faster, but the bookkeeping may be complex — intra-file references, transitive dependencies through root nodes, etc.
- **Wildcard queries as graph references**: model queries like `The.Namespace.*` as references through the root scope, using the existing partial path machinery to resolve them. This could make some "full-project" queries incremental by construction.

This needs prototyping to understand the performance characteristics and whether the incremental approaches are worth the complexity.

### 8. External Dependencies (JARs, NuGet Packages, Third-Party Libraries)

Real projects reference types from libraries that aren't part of the source tree. Stack graphs are file-incremental over *source files*, but library types don't have source files to index.

The [`c-sharp-analyzer-provider`](https://github.com/konveyor/c-sharp-analyzer-provider) handles this well: it uses ILSpy to decompile .NET assemblies into C# source, then indexes that decompiled source alongside the project source. Everything ends up in the same graph, and cross-file resolution works naturally — the decompiled source is just more files to index.

This approach could generalize:
- **C#**: ILSpy decompilation of NuGet packages / .NET assemblies (already proven)
- **Java**: Decompile JARs with CFR or Procyon, index the decompiled source
- **Pre-indexed library graphs**: Ship pre-built SQLite databases for common libraries (e.g., the Java standard library), load them alongside the project graph at query time

The decompilation approach is appealing because it requires no special handling in the stack-graphs library — decompiled source is just source. But it adds indexing time and may produce imperfect source for obfuscated or optimized bytecode.

### 9. Resolution Failure and Diagnostics

The current TSG DSL provides almost no diagnostic information when resolution fails. When a reference can't be resolved, you don't know why: Is the grammar incomplete? Is a dependency missing? Is there a scoping bug? You're left staring at graph dumps trying to figure out which edge is wrong or missing.

For Konveyor's migration analysis, knowing that a reference *couldn't* be resolved is important — it might indicate a missing dependency, an incomplete classpath, or a grammar gap. The system needs to distinguish between:
- **Intentionally unresolved**: reference to a type from an external library that wasn't indexed
- **Grammar gap**: the grammar doesn't handle this language construct yet
- **Ambiguous**: multiple definitions match (overloading, shadowing)
- **Broken**: the grammar produced incorrect graph structure

This connects to the TSG vs Lua question (Open Question 1) — a real programming language would make diagnostic instrumentation much easier. It also connects to the schema question — if the schema defines what every grammar must produce, missing metadata becomes detectable.

## What Needs to Change in Stack Graphs

The work proposed here requires changes to the stack-graphs library itself. Some of these are straightforward fixes; others depend on the open questions above and need further design discussion.

### Clear: Fix `SourceInfo` serialization

**Where:** `stack-graphs/src/serde/graph.rs`
**Scope:** ~20 lines

The serde `SourceInfo` struct only has two fields (`span`, `syntax_type`). Expand it to include all five fields from the core `SourceInfo`. Update the serializer and deserializer to round-trip all fields through SQLite.

This is unambiguously needed and unblocks everything else.

### Clear: Add reverse reference index

**Where:** `stack-graphs/src/storage.rs` (or new module)

Add a `resolved_references` table to SQLite. After indexing all files, resolve every reference using `ForwardPartialPathStitcher` and persist the ref-to-def mappings. This makes "find all references to X" a SQL lookup instead of a full graph traversal.

The table structure, when to rebuild it, and how it interacts with incremental indexing need design work, but the concept is straightforward.

### Needs design: FQN mechanism

How should fully qualified names be computed and stored? Options discussed in Open Question 5. The current `debug_fqdn` hack works but shouldn't be the long-term answer. We're leaning toward deriving FQNs from the graph structure via a post-build pass, since the information is already encoded in the pop node chains.

### Needs design: Schema and metadata model

What metadata must every grammar produce? What goes in `SourceInfo` vs. graph edges vs. external tables? This is the central design question — see the schema standardization discussion above and Open Questions 4 and 5. We're leaning toward adopting the Java grammar's `":"` pop edge convention for type-of relationships (see Open Question 4), which would reduce the need for re-parsing.

`SourceInfo` may need additional fields depending on these decisions:
- **Parent definition handle** — for FQN computation
- **Type reference** — for type-of relationships
- **Visibility/modifiers** — public, private, static, abstract, etc.

Every field added to `SourceInfo` is a contract that all grammars must honor.

### Needs design: Source/CST storage

See Open Question 2. If we decide to store source text or the CST in SQLite, the storage layer needs new tables and the indexing pipeline needs to persist additional data.

## Proposed Architecture

```
            Source files (.cs, .java, ...)
                    |
                    v
             tree-sitter (parsing)
                    |
                    v
          TSG rules (per-language)
          Following standardized schema:
          - syntax_type vocabulary
          - FQN containment (mechanism TBD)
          - definiens_node
          - type edges (mechanism TBD)
                    |
                    v
          Stack Graph (per-file, incremental)
                    |
                    v
         SQLite Database (persisted)
         - Partial paths (per-file)
         - SourceInfo with all metadata
         - Source text / CST (TBD, see Open Question 2)
                    |
            (post-indexing, full project)
                    |
                    v
         Reverse Reference Index
         - resolved_references table
         - ref → def mappings for all references
                    |
                    v
        Introspection Query API (Rust)
        - find_definitions(regex) — FQN lookup
        - references_to(def) — reverse index lookup
        - file_dependencies(file) — outgoing references
        - inspect(def) — re-parse source for CST detail
                    |
                    v
         gRPC Provider Service
         (replaces java-external-provider,
          dotnet-external-provider)
```

### Introspection Query API

```rust
// Note: DefinitionInfo, ReferenceInfo, and NodeDetail are sketch types
// illustrating the API shape. Final types will be defined during implementation.

pub struct Querier {
    db: SQLiteReader,
}

impl Querier {
    /// Find definitions whose FQN matches a regex pattern.
    /// FQNs are dot-separated: "NerdDinner.Models.Dinner.Title"
    /// Example: find_definitions(".*\\.Dinner\\..*") returns all members of class Dinner.
    pub fn find_definitions(&self, pattern: &Regex) -> Vec<DefinitionInfo>;

    /// Find all references to a given definition using the reverse index.
    pub fn references_to(&self, def: &DefinitionInfo) -> Vec<ReferenceInfo>;

    /// Find all files that a given file depends on (outgoing references).
    pub fn file_dependencies(&self, file: &str) -> Vec<String>;

    /// Re-parse the source file and return full CST detail for a definition
    /// (type annotations, modifiers, parameter lists, etc.).
    /// Requires source text to be available (on disk or in database).
    pub fn inspect(&self, def: &DefinitionInfo) -> NodeDetail;
}
```

### Grammar Schema

For introspection to work uniformly across languages, all grammars must follow a standardized schema. This schema defines the contract between grammars and the query API.

**Required `syntax_type` values:**
- Container types: `namespace`, `class`, `struct`, `interface`, `enum`, `record`
- Members: `method`, `constructor`, `field`, `property`, `event`, `enum_member`
- Other: `parameter`, `local_var`, `function`, `type_parameter`, `delegate`

**Required edge conventions (tentative — see open questions):**
- FQN containment: mechanism TBD (currently `debug_fqdn = "parent"`, but this is a hack)
- `pop(".")` for member access
- `pop(":")` for type-of relationships (leaning toward adopting the Java grammar's pattern — see Open Question 4)
- `definiens_node` on all definitions that have a body

**Required node attributes:**
- `syntax_type` on every definition node
- `source_node` on every definition and reference node

This is the minimum. The schema will likely grow as we resolve the open questions around type information, FQN computation, and visibility modifiers. Each addition is a commitment that all grammars must implement — so we should be deliberate about what goes in.

### Provider Integration

The stack-graphs analyzer exposes a gRPC service implementing the `provider.ServiceClient` interface from analyzer-lsp:

```
analyzer-lsp  --gRPC-->  stack-graphs-provider  (Rust binary)
                              |
                              +-- C# grammar
                              +-- Java grammar
                              +-- SQLite database
```

The provider:
1. Indexes the source tree using the appropriate grammar (incremental — skips unchanged files)
2. Builds the reverse reference index (full project — resolves all references)
3. Serves `provider.Evaluate()` calls by querying the introspection API
4. Returns incidents in the format analyzer-lsp expects

## Test Plan

- **Per-grammar resolution tests**: Annotation-based tests (`// ^ defined: ...`) using the tree-sitter-stack-graphs test harness. C# currently has ~40 test files.
- **FQN tests**: Verify FQN reconstruction for all definition types across both grammars.
- **Serde round-trip tests**: Build graph, serialize to SQLite, deserialize, verify all `SourceInfo` fields survived.
- **Query API tests**: Index a known codebase, run each query type, verify results.
- **Provider integration tests**: Run analyzer-lsp rules against the stack-graphs provider and compare results against the existing Go provider. Pass rate is the graduation metric.
- **Cross-language schema tests**: Verify that C# and Java grammars produce consistent `syntax_type` values and FQN formats.

## Risks

- **Maintenance burden**: Forking stack-graphs means we own it. The upstream repo is archived, so there are no upstream fixes coming regardless.
- **Grammar authoring difficulty**: TSG is hard to learn and debug. Better documentation helps, but the learning curve is steep. (See Open Question 1 about Lua alternatives.)
- **Approximation**: Stack graphs approximate language semantics. Complex features (overload resolution, generic type inference, implicit conversions) may not be fully modeled. For migration rules, this approximation is usually sufficient.
- **Transition period**: During graduation, both Go and Rust providers exist, increasing maintenance surface.

## Alternatives Considered

### Language servers (JDTLS, csharp-ls, Roslyn)

Continue with heavyweight language servers. Most accurate, but slow, memory-hungry, platform-dependent, and unreliable for some languages (csharp-ls). Cannot provide standardized cross-language queries.

### Tree-sitter only (no stack graphs)

CST queries without cross-file resolution — the [dotnet-provider-treesitter](/enhancements/dotnet-provider-treesitter/README.md) approach. Simple and fast, but cannot resolve cross-file references. Insufficient for most migration rules.

### Separate analyzers per language

Keep java-analyzer-provider and future language analyzers as independent projects. Avoids coordination overhead but duplicates infrastructure, produces inconsistent query capabilities, and makes cross-language standardization impossible.

## Implementation History

- 2024-08: [Tree-sitter C# enhancement](/enhancements/dotnet-provider-treesitter/README.md) proposed (CST-only, no cross-file resolution)
- 2025-04: Shawn Hurley begins C# stack-graphs grammar
- 2026-04: C# grammar expanded with FQN reconstruction, definiens tracking, `find-node` subcommand, 40+ resolution tests
- 2026-04: Serde gap, schema standardization needs, and other core issues discovered and documented
- 2026-04-14: This enhancement proposal created
- 2026-04-16: @jmle's Rust-based Java analyzer with stack-graphs brought to the team's attention

## Infrastructure Needed

- **GitHub repository**: `konveyor/stack-graphs` (fork of `github/stack-graphs`)
- **CI**: Rust build + test matrix for the workspace (all language crates)
- **Container image**: For the gRPC provider binary (used by kantra and analyzer-lsp)
