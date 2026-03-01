# IR Specification

## Design Philosophy

The pp intermediate representation is **normalized functional code**. All syntactic sugar from source languages is removed during parsing. The IR has exactly five constructs, no more.

The IR contains **no undefined behavior, no implementation-defined behavior, and no unspecified behavior.** Every source-language construct receives deterministic rewrite rules that produce unambiguous IR. This property makes the IR suitable for formal verification by construction.

## IR Constructs

### 1. Call

```
result = call(operator_symbol, arg1, arg2, ...)
```

Every computation is a function application. There is no distinction between binary expressions, function calls, method invocations, or special operators. They are all call nodes.

A source expression like `x = a + b * c` becomes:

```
call("=", ref("x"), call("+", ref("a"), call("*", ref("b"), ref("c"))))
```

The nesting reflects the precedence and associativity that were active when the expression was parsed.

### 2. Label

```
L1:
```

A named target for control flow. Labels are first-class in the IR.

### 3. Goto

```
goto L1
```

An unconditional transfer of control.

### 4. Scope Enter

```
scope_enter [optional_label]
```

Opens a new scope. Variables declared within the scope (including temporaries introduced by rewrite rules) are not visible outside it. Named scopes support targeted control flow (e.g., Verilog `disable block_name`).

### 5. Scope Exit

```
scope_exit [optional_label]
```

Closes the current scope. Temporaries are released. Named scope exit provides a goto target for block-level exits.

## Everything Desugars

### Control Flow

All source-language control flow constructs desugar into labels, gotos, and conditional calls:

**if/else:**
```
call("branch_false", condition, L_else)
// then body
goto L_end
L_else:
// else body
L_end:
```

**for loop** (`for (i=0; i<10; i++)`):
```
call("=", i, 0)
L_top:
call("branch_false", call("<", i, 10), L_exit)
// loop body
call("=", i, call("+", i, 1))
goto L_top
L_exit:
```

**Verilog `disable block_name`:**
```
scope_enter block_name
// block body
// disable block_name becomes:
goto block_name_exit
scope_exit block_name
block_name_exit:
```

There are no `for`, `while`, `if`, or `switch` nodes in the IR. Backend tools that need to recognize loop structure can reconstruct it from the label/goto pattern.

### Operators as Calls

Every operator in every source language becomes a call node. The meaning of the operator is entirely determined by the backend that binds implementations to operator symbols.

The same IR call `call("<=", a, b)` might originate from a VHDL signal assignment, a Verilog non-blocking assignment, or a C comparison — depending on which grammar context was active when it was parsed. The call node carries a context annotation indicating its source grammar, enabling the backend to select the correct implementation.

### Rewrite Rules

Source constructs with ambiguous or undefined semantics receive deterministic rewrite rules that expand them into explicit IR sequences.

**C post-increment** (`i++`):
```
scope_enter
  temp_0 = i
  call("=", i, call("+", i, 1))
  // scope result: temp_0
scope_exit
```

The rewrite rule introduces a scoped temporary, makes the sequencing explicit, and eliminates the undefined behavior of C's sequence point rules. The expression `i++ + ++i` which is undefined in C becomes fully deterministic in the IR because each rewrite produces its own scope with explicit ordering.

**Rewrite rules are language-specific** and loaded as part of the grammar configuration. Different languages can rewrite the same syntactic construct differently while producing equally deterministic IR.

## Memory Layout

IR is stored in memory-mapped files with the following properties:

- **Offset-based references**: All internal pointers are file-relative offsets, not machine pointers. This makes IR files position-independent and shareable across processes via mmap.
- **Arena allocation**: Nodes are allocated sequentially in the file. Scope exits allow bulk deallocation by resetting the arena pointer for that scope's temporaries.
- **String interning**: Identifiers and operator symbols are stored once in a string table at the file header. Nodes reference strings by offset into this table.
- **Type table**: A header section enumerates node types and their layouts, allowing consumers to interpret the IR without compile-time coupling to the producer's binary.

### File Structure

```
┌─────────────────────┐
│ Header              │
│  - magic number     │
│  - version          │
│  - node type table  │
├─────────────────────┤
│ String Intern Table │
│  - identifiers      │
│  - operator symbols │
├─────────────────────┤
│ IR Nodes            │
│  - sequential arena │
│  - offset-based refs│
├─────────────────────┤
│ Scope Table         │
│  - scope ranges     │
│  - label targets    │
└─────────────────────┘
```

## Properties for Formal Verification

The IR is designed to be formally tractable:

1. **Deterministic**: No construct has more than one interpretation. Every rewrite rule produces exactly one IR expansion.
2. **Totally ordered within scopes**: Rewrite rules make sequencing explicit, eliminating the ambiguities that arise from C's sequence points or Verilog's scheduling semantics.
3. **SSA-friendly**: Each call produces a value. Converting to SSA form is mechanical — insert phi nodes at control flow merge points.
4. **No aliasing ambiguity**: Rewrite rules that introduce temporaries use scoped variables that cannot alias external state.
5. **Equivalence-checkable**: Two IR graphs from different source languages can be compared for functional equivalence because both have fully defined semantics.
