# Architecture

## Pipeline Overview

GPP processes source code in three stages: ripping, parallel parsing, and backend consumption. Each stage is designed for a specific performance and correctness goal.

## Stage 1: The Ripper

A fast lexical scanner that identifies structural boundaries — module, package, class, and namespace delimiters — without performing a full parse. Its job is to carve large, non-modular codebases (typical of Verilog) into independent chunks that can be distributed to parser processes.

The ripper understands just enough syntax to avoid being fooled by strings, comments, and preprocessor directives. It recognizes scope-opening and scope-closing keywords as defined in the active grammar configuration.

For Verilog, this means identifying `module`/`endmodule`, `package`/`endpackage`, `class`/`endclass` boundaries. For C/C++, namespace and function boundaries delimited by `{`/`}`.

The ripper also pre-reads `include` files and caches them in memory before the parser processes are forked, so every child inherits the cached content via copy-on-write pages with no additional I/O.

## Stage 2: Parallel Parsing

### Process Model

The main program forks (without exec) to create parser processes. Fork-without-exec gives copy-on-write sharing of grammar tables, symbol infrastructure, pre-loaded include caches, and any context established by the parent. Each parser process is a separate address space — a rogue parse cannot corrupt other parsers.

Each parser process **owns** one or more namespaces and is the authoritative source for all declarations within them. This is not embarrassingly parallel parsing — the parsers form a distributed actor system that collaborates to resolve cross-namespace dependencies.

### Work Assignment and Reordering

Each parser receives a list of work items (modules, classes, packages) to parse. When a parser encounters a dependency on a namespace owned by another process (e.g., `import pkg::*` in SystemVerilog), and that dependency is not yet resolved, the parser **reorders its work list** rather than blocking:

1. Shelve the current work item, saving parse state (token position, partial IR, scope stack)
2. Pick up the next item from its list
3. Resume the shelved item when the dependency becomes available

This requires the parser to support suspension and resumption at dependency points — effectively coroutine-like behavior at the parse level.

### Cross-Process Communication

Parsers communicate through two mechanisms:

**Active messaging** (during parsing): When a parser needs a symbol from another namespace, it sends a resolution request to the owning process. The message vocabulary is small:
- "Resolve identifier X in your namespace"
- "Here is the type/value for X"
- "I'm done; my IR file is at path P"

Communication channels can be Unix domain sockets, named pipes, or shared-memory ring buffers.

**Passive shared memory** (after completion): Once a parser finishes, its memory-mapped IR file is complete and other processes can map it read-only. The communication channel gracefully transitions from active messaging to passive shared memory access.

### Deadlock Detection

If all parsers exhaust their work lists with items still shelved due to unresolved dependencies, a genuine circular dependency exists. This triggers a two-phase resolution: parsers export public declarations (signatures only) first, exchange them, then complete the full parse. This is analogous to a linker's symbol resolution pass.

### Shared Status Table

The parent process maintains (or the children share) an mmap'd status table where each parser atomically publishes namespace completion status. Children poll this table or receive signals to unshelve suspended work items.

## Stage 3: IR Consumption

Backend tools — LLVM JIT compiler, linters, static analyzers, formal verification engines, coverage tools — map in the relevant IR files and traverse the unified call graph. Because the IR is language-agnostic, tools work identically across all source languages without modification.

## Parsing Architecture

### Two-Tier Parsing

Parsing uses a two-tier approach:

**Hand-crafted recursive descent** for structural constructs: module declarations, port lists, class definitions, generate blocks, control flow statements. This tier recognizes language keywords and desugars structural syntax into the normalized IR (calls, labels, gotos, scopes).

**Table-driven expression parsing** for everything within an expression context. The expression parser uses dynamically loaded precedence/associativity tables to build call trees from operators. It makes no assumptions about what operators exist or what they mean.

The transition between tiers is clean: the recursive descent parser recognizes an expression context and hands off to the expression parser with the current grammar tables. The expression parser returns a call subtree.

### Operator Recognition

The expression parser follows one rule: **any token that is not a keyword and does not resolve as a variable or literal is an operator.** The token is looked up in the active operator table to determine precedence, associativity, and arity. If not found, it's an error.

This means operators can be:
- Traditional symbols: `+`, `-`, `*`, `<=`
- Multi-character symbols: `|->`, `|=>`, `<+`
- Unicode: `⊗`, `†`, `∇`
- Word-like: `inside`, `dist`, `downto`

All go through the same mechanism.

### Grammar Configuration via Preprocessor

The preprocessor is the grammar's bootstrap mechanism. Before parsing begins, preprocessor directives configure the parser:

```
#pragma keyword module    scope_open
#pragma keyword endmodule scope_close
#pragma keyword always    block_context

#pragma operator +   precedence=10 assoc=left
#pragma operator *   precedence=11 assoc=left
#pragma operator <=  precedence=2  assoc=right context=always:assignment context=expr:comparison
```

Keywords are classified into tiers:
- **Structural keywords** (e.g., `module`, `endmodule`) — used by the ripper and scope analysis
- **Contextual keywords** (e.g., `always`, `generate`) — establish parsing context affecting operator table selection
- **Soft keywords** (e.g., `wire`, `logic`) — resolved as keywords or identifiers depending on position

### In-Parse Language Switching

Grammar tables can be swapped at any scope boundary via pragma directives:

```
module my_chip(...);
  // SystemVerilog grammar active
  always_ff @(posedge clk) begin
    result <= a + b;
  end

  #pragma parser "c++"
  // C++ grammar tables now active
  inline int model(int a, int b) {
    return (a >> 2) | (b & 0xFF);
  }
  #pragma parser "systemverilog"
endmodule
```

The parser pushes the current grammar tables onto a stack, loads the new language's tables, and continues. On scope exit or a matching `#pragma parser` directive, the stack is popped. Operator precedence, keyword classification, and rewrite rules all change with the grammar context.

This enables block-level or even expression-level mixing of languages within a single source file, all producing the same uniform IR.

### DSL Definition

Defining a new DSL is equivalent to writing a grammar header file:

```
#pragma operator |>  precedence=5 assoc=left
#pragma operator ⊗   precedence=8 assoc=left
#pragma operator †   precedence=12 arity=unary postfix
#pragma keyword measure block_context
```

No parser recompilation required. The grammar is data, not code.
