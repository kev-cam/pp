# Design Rationale

## Why This Exists

The EDA industry's parsers are built on 30+ years of lex/yacc legacy. These tools bake language grammars into compiled C code. You cannot swap grammars at runtime, stack them, or add operators dynamically. This architectural limitation cascades into every downstream tool: each simulator, linter, and formal tool reinvents its own parser, its own AST, its own language quirks. Mixed-language designs require bridging layers between incompatible representations.

GPP replaces this with a single parsing engine where languages are grammar configurations, not hardcoded parsers, and all languages produce the same IR.

## Why No Undefined Behavior

Undefined behavior in source languages (C's sequence points, Verilog's scheduling ambiguities) creates holes in the semantic model. Formal verification tools must either consider all possible interpretations (state space explosion) or assume one (unsoundness). By assigning deterministic rewrite rules to every construct, the IR is formally verifiable by construction. There is exactly one interpretation for every IR sequence.

This is a deliberate departure from the C philosophy where undefined behavior enables compiler optimization. GPP is not a compiler — it's a verification-friendly representation. Determinism is more valuable than optimization freedom.

## Why Uniform Call Graph IR

The SystemVerilog DPI (Direct Programming Interface) standard was designed to bridge SystemVerilog and C by marshalling data across a language boundary. The result is wrapper functions, restricted type mappings, implementation-divergent behavior across simulators, and performance overhead for what should be a simple function call.

The fundamental problem was treating languages as separate domains requiring a bridge. GPP eliminates the boundary entirely. A SystemVerilog module calling a C++ function is one call node referencing another in the same IR. There is no marshalling, no wrappers, no impedance mismatch.

This is what a proper cross-language ABI would have looked like: a shared representation where "foreign" calls are indistinguishable from native calls.

## Why Fork Without Exec

The parser pool uses `fork()` without `exec()` for several reasons:

- **Copy-on-write sharing**: Grammar tables, include file caches, and preprocessor state are established in the parent before forking. Every child inherits them through COW pages with zero copy overhead.
- **Process isolation**: A parser crash (segfault from malformed input, runaway memory) is contained to one process. Other parsers and the coordinator continue.
- **Simple coordination**: Separate address spaces mean no lock contention, no shared mutable state except through explicit communication channels and the mmap'd status table.

The alternative — threads with shared memory — offers lower communication latency but introduces lock complexity and means one bad parse can corrupt the entire address space.

## Why Reorder Rather Than Block

When a parser hits an unresolved cross-namespace dependency, it shelves the current work item and picks up another. This avoids the pathological case where all parsers are blocked waiting for each other in a dependency chain.

The insight is that most modules are largely self-contained. Cross-namespace references are the exception. A parser with 10 work items might hit dependencies on 2-3 of them. By reordering, the other 7-8 proceed at full speed while dependencies resolve in the background.

Blocking would serialize the parse along the critical dependency path. Reordering keeps all parsers productive.

## Why Preprocessor-Driven Grammar Configuration

Every language GPP targets already has a preprocessor. C has `#define`/`#include`/`#pragma`. Verilog has `` `define``/`` `include``. Rather than inventing a separate metalanguage for grammar specification, GPP extends the preprocessor mechanism that users already understand.

This also means grammar definitions compose naturally. A base SystemVerilog grammar can be extended with UVM-specific operator definitions via `#pragma grammar "uvm-extensions.gpp"` — the same inclusion mechanism used for regular code.

## Why Dynamic Operators

Static operator sets force language designers to enumerate every possible operator at grammar-definition time. This fails for:

- **Verilog-AMS**: Contribution operators (`<+`) only exist in analog contexts
- **SystemVerilog assertions**: Sequence operators (`|->`, `|=>`) only exist in property contexts  
- **VHDL**: User-defined operators are part of the language standard
- **DSLs**: By definition, their operators are not known in advance

Dynamic operator definition means the parser never needs modification to support new operators. The grammar file defines them, the expression parser handles them, and the backend provides implementations. The parser is permanently complete.

## Why Memory-Mapped IR Files

Alternatives considered:

- **In-process AST**: Fast but not shareable across parser processes without serialization
- **Protobuf/Flatbuf serialization**: Adds encoding/decoding overhead and a schema dependency
- **Database (SQLite, etc.)**: Too much overhead for the access patterns (sequential write, random-access read)

Memory-mapped files with offset-based pointers provide:
- Zero-copy sharing between processes (just `mmap` the file)
- Persistence (the IR survives process exit, useful for incremental rebuilds)
- Sequential write performance (arena allocation into the mapped region)
- Random-access read performance (pointer arithmetic on the mapped region)
- Position independence (offsets, not pointers, so any process can map at any address)
