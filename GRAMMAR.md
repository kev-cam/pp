# Grammar Configuration

## Overview

pp does not contain hardcoded language grammars. Each source language is defined by a set of preprocessor pragmas that configure the parser's keyword table, operator table, rewrite rules, and context transitions. Switching languages is loading a different configuration. Defining a new DSL is writing a new one.

## Preprocessor Pragma Syntax

### Keyword Definition

```
#pragma keyword <token> <classification> [properties...]
```

Classifications:
- `scope_open` — Opens a structural scope (used by ripper and parser). E.g., `module`, `begin`, `{`
- `scope_close` — Closes a structural scope. E.g., `endmodule`, `end`, `}`
- `block_context` — Establishes a named parsing context that affects operator table selection. E.g., `always`, `constraint`
- `decl_type` — Type declaration keyword. E.g., `wire`, `int`, `logic`
- `soft` — Resolved as keyword or identifier depending on position

Properties:
- `context=<name>` — Names the context this keyword establishes
- `pair=<token>` — Matching close keyword for a scope_open

Examples:
```
#pragma keyword module    scope_open pair=endmodule
#pragma keyword endmodule scope_close
#pragma keyword begin     scope_open pair=end
#pragma keyword end       scope_close
#pragma keyword always    block_context context=always
#pragma keyword always_ff block_context context=always
#pragma keyword class     scope_open pair=endclass
#pragma keyword wire      decl_type
#pragma keyword logic     soft
```

### Operator Definition

```
#pragma operator <token> precedence=<N> assoc=<left|right|none> [properties...]
```

Properties:
- `arity=<unary|binary|ternary>` — Default is binary
- `fixity=<prefix|postfix|infix>` — Default is infix; relevant for unary operators
- `context=<ctx>:<meaning>` — Context-dependent semantics. Multiple context properties allowed
- `group=<name>` — Operator group for bulk precedence manipulation

Examples:
```
// Standard arithmetic
#pragma operator +    precedence=10 assoc=left
#pragma operator -    precedence=10 assoc=left
#pragma operator *    precedence=11 assoc=left
#pragma operator /    precedence=11 assoc=left

// Context-dependent operators
#pragma operator <=   precedence=2 assoc=right context=always:nba context=expr:comparison

// Unary operators
#pragma operator !    precedence=14 assoc=right arity=unary fixity=prefix
#pragma operator ~    precedence=14 assoc=right arity=unary fixity=prefix
#pragma operator ++   precedence=15 assoc=right arity=unary fixity=prefix
#pragma operator ++   precedence=16 assoc=left  arity=unary fixity=postfix

// SystemVerilog assertion operators
#pragma operator |->  precedence=3 assoc=right
#pragma operator |=>  precedence=3 assoc=right

// Verilog-AMS contribution operator
#pragma operator <+   precedence=2 assoc=right context=analog:contribution

// VHDL-style word operators
#pragma operator downto precedence=5 assoc=none
#pragma operator to     precedence=5 assoc=none
```

### Rewrite Rules

```
#pragma rewrite <pattern> { <expansion> }
```

Rewrite rules define how syntactic sugar is desugared into normalized IR. The expansion must use only IR primitives (call, label, goto, scope_enter, scope_exit) and may introduce temporaries.

Examples:
```
// C post-increment
#pragma rewrite ($a)++ {
  scope_enter
    __temp = $a
    $a = call("+", $a, 1)
    __result = __temp
  scope_exit
}

// C pre-increment
#pragma rewrite ++($a) {
  scope_enter
    $a = call("+", $a, 1)
    __result = $a
  scope_exit
}

// Ternary operator
#pragma rewrite ($cond) ? ($then) : ($else) {
  scope_enter
    call("branch_false", $cond, __L_else)
    __result = $then
    goto __L_end
    __L_else:
    __result = $else
    __L_end:
  scope_exit
}
```

### Parser Switching

```
#pragma parser "<language_name>"
```

Pushes the current grammar tables onto a stack and loads the named language's grammar configuration. Parsing continues with the new grammar until a matching `#pragma parser` directive or scope exit restores the previous configuration.

```
#pragma parser "c++"        // push SV tables, load C++ tables
// ... C++ code parsed with C++ grammar ...
#pragma parser "systemverilog"  // pop back to SV tables
```

### Include Grammar

```
#pragma grammar "<path>"
```

Loads a grammar definition file containing keyword, operator, and rewrite rule pragmas. This is how standard language definitions are distributed.

```
#pragma grammar "grammars/systemverilog-2017.pp"
#pragma grammar "grammars/uvm-extensions.pp"
```

## Standard Grammar Libraries

pp ships with grammar definitions for:

| File | Language |
|------|----------|
| `systemverilog.pp` | IEEE 1800-2017 SystemVerilog |
| `verilog-ams.pp`   | Verilog-AMS 2.4 |
| `vhdl.pp`          | IEEE 1076-2008 VHDL |
| `c.pp`             | C11 |
| `cpp.pp`           | C++17 subset |
| `spice.pp`         | SPICE netlist |
| `liberty.pp`       | Liberty timing format |
| `sdc.pp`           | Synopsys Design Constraints |

## Context Interaction

When a `block_context` keyword is encountered, the parser enters that named context. Operators with `context=<name>:<meaning>` annotations resolve differently within that context.

This enables:
- `<=` as non-blocking assignment inside `always` blocks, comparison elsewhere
- `<+` as contribution operator inside `analog` blocks, parse error elsewhere
- `inside` as set membership operator inside `constraint` blocks, identifier elsewhere

Contexts nest. A `constraint` block inside a `class` inside a `module` has three active contexts, and operator resolution checks from innermost to outermost.

## DSL Definition Example

A complete quantum circuit DSL defined in a single grammar file:

```
// quantum.pp — Quantum Circuit DSL

#pragma keyword qubit    decl_type
#pragma keyword measure  block_context context=measurement
#pragma keyword circuit  scope_open pair=endcircuit

#pragma operator |>  precedence=5  assoc=left    // pipeline
#pragma operator ⊗   precedence=8  assoc=left    // tensor product
#pragma operator †   precedence=12 arity=unary fixity=postfix  // adjoint
#pragma operator ◦   precedence=9  assoc=left    // composition

// Usage:
// #pragma grammar "quantum.pp"
// circuit bell_state
//   qubit a, b
//   a |> H |> CNOT ⊗ b |> measure
// endcircuit
```
