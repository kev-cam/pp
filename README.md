# GPP — General Purpose Parser

A C++ language parsing framework with Perl/Python bindings, designed to parse any language through dynamically configurable grammars. The parser produces a normalized functional IR (intermediate representation) suitable for LLVM JIT compilation, formal verification, and static analysis.

The primary validation target is full SystemVerilog/UVM parsing — a capability currently limited to expensive commercial EDA tools.

## Core Principles

- **Languages are grammar configurations, not hardcoded parsers.** Keywords, operators, precedence, and associativity are defined via preprocessor pragmas, not baked into compiled grammar files. No lex/yacc, no ANTLR.
- **One uniform IR.** Every source language desugars to the same normalized functional representation: call nodes, labels, gotos, and scoped temporaries. No syntactic sugar survives into the IR.
- **No undefined behavior in the IR.** All source-level constructs with ambiguous or implementation-defined semantics receive deterministic rewrite rules. This makes the IR formally verifiable by construction.
- **Parallel by design.** Large codebases are split by namespace/package and parsed concurrently by forked processes communicating through memory-mapped IR files.

## Architecture Overview

```
Source Files
    │
    ▼
┌──────────────┐
│   Ripper     │  Fast lexical scan identifies module/package boundaries
└──────┬───────┘
       │  Chunks assigned to parser processes
       ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Parser P1   │  │  Parser P2   │  │  Parser P3   │
│  (namespace A)│  │  (namespace B)│  │  (namespace C)│
│              │◄─►│              │◄─►│              │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌──────────────────────────────────────────────┐
│         Memory-Mapped IR Files               │
│  (offset-based pointers, arena allocated)    │
└──────────────────────┬───────────────────────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
        ┌──────────┐    ┌─────────────┐
        │ LLVM JIT │    │ Tool Passes │
        │ Backend  │    │ (lint, cov, │
        │          │    │  formal)    │
        └──────────┘    └─────────────┘
```

See [ARCHITECTURE.md](ARCHITECTURE.md) for full details.

## Key Features

- **Dynamic grammar configuration** via preprocessor pragmas
- **Runtime operator definition** — any token not recognized as a keyword or variable is an operator; precedence and associativity are programmable
- **In-parse language switching** — swap grammar tables at block granularity (e.g., embed C++ inside SystemVerilog)
- **Parallel parsing** with forked processes, namespace-based work partitioning, and dependency-driven work reordering
- **Memory-mapped IR** with offset-based references for zero-copy sharing between processes
- **Perl/Python/C++ bindings** for IR traversal and manipulation
- **LLVM JIT backend** for native execution (following the approach proven by NVC for VHDL)

## Validation Target: UVM

The Universal Verification Methodology exercises every difficult corner of SystemVerilog: parameterized classes, factory macros, virtual interfaces, constraint blocks, covergroups. Open-source parsers like iverilog cannot handle UVM. GPP targets full UVM parsing as its existence proof.

## Building

*TODO: Build instructions*

## License

*TODO: License selection*

## Status

Early development. See [ROADMAP.md](ROADMAP.md) for the development plan.
