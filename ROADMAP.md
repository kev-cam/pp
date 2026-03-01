# Roadmap

## Phase 1: Foundation

**Goal:** Core parsing engine with dynamic grammar configuration.

- [ ] C++ IR node library (call, label, goto, scope_enter, scope_exit)
- [ ] Arena allocator targeting mmap'd files with offset-based references
- [ ] String interning table
- [ ] IR file format: header, string table, node arena, scope table
- [ ] Preprocessor engine supporting `#pragma keyword`, `#pragma operator`, `#pragma rewrite`
- [ ] Table-driven expression parser with dynamic precedence/associativity
- [ ] Hand-crafted recursive descent framework for structural parsing
- [ ] Grammar table stack for in-parse language switching (`#pragma parser`)
- [ ] Basic grammar file for C subset (validation of expression parsing and rewrite rules)

## Phase 2: Parallel Parsing

**Goal:** Fork-based parallel parser pool with namespace ownership.

- [ ] Ripper: fast lexical scanner for structural boundary detection
- [ ] Fork-without-exec process creation from parent
- [ ] Namespace/package assignment to parser processes
- [ ] Cross-process communication (resolution requests/responses)
- [ ] Shared mmap'd status table for namespace completion signaling
- [ ] Work list reordering on unresolved dependencies
- [ ] Parse state suspension and resumption (coroutine-like checkpointing)
- [ ] Deadlock detection and two-phase declaration export fallback
- [ ] Include file pre-caching before fork (COW sharing)

## Phase 3: SystemVerilog / UVM

**Goal:** Parse real UVM testbenches that iverilog rejects.

- [ ] SystemVerilog grammar file (`systemverilog.pp`)
  - [ ] Module/interface/class/package declarations
  - [ ] Parameterized classes and type specialization
  - [ ] Constraint blocks with `inside`, `dist`, implication operators
  - [ ] Covergroups and coverage expressions
  - [ ] Assertion sequences and properties (`|->`, `|=>`)
  - [ ] Virtual interfaces
  - [ ] DPI import/export declarations
- [ ] UVM macro expansion through preprocessor
  - [ ] `uvm_component_utils` / `uvm_object_utils` family
  - [ ] Factory registration patterns
  - [ ] Field automation macros
- [ ] Validation: parse UVM base library source
- [ ] Validation: parse non-trivial UVM testbench end to end

## Phase 4: LLVM JIT Backend

**Goal:** Execute SystemVerilog/UVM through LLVM JIT, analogous to NVC's approach for VHDL.

- [ ] IR to LLVM IR lowering
  - [ ] Call nodes → LLVM function calls
  - [ ] Labels → LLVM basic blocks
  - [ ] Gotos → LLVM branches
  - [ ] Scopes → LLVM allocas with lifetime markers
- [ ] Operator implementation library (SystemVerilog 4-state logic, etc.)
- [ ] Event scheduling runtime (delta cycles, NBA regions)
- [ ] Basic simulation harness: load IR, JIT compile, execute
- [ ] Validation: run a UVM testbench to completion

## Phase 5: Bindings and Tooling

**Goal:** Perl/Python access to the IR, enabling scripted analysis and transformation.

- [ ] Perl XS / FFI bindings for IR traversal
- [ ] Python bindings (pybind11) for IR traversal
- [ ] IR visitor/walker API
- [ ] Example tools:
  - [ ] Lint pass (walk IR, check patterns)
  - [ ] Cross-reference / documentation extractor
  - [ ] Coverage analysis
  - [ ] Equivalence checking harness (compare two IR graphs)

## Phase 6: Expanded Language Support

**Goal:** Demonstrate generality beyond SystemVerilog.

- [ ] VHDL grammar file (`vhdl.pp`)
- [ ] Verilog-AMS grammar file (`verilog-ams.pp`)
- [ ] C11 grammar file (`c.pp`)
- [ ] C++17 subset grammar file (`cpp.pp`)
- [ ] Mixed-language test: SystemVerilog + C++ in one parse
- [ ] SPICE netlist grammar
- [ ] Liberty / SDC grammars
- [ ] User-defined DSL walkthrough and documentation

## Future Directions

- Formal verification integration (SMT solver interface over the deterministic IR)
- SSA conversion pass
- Optimization passes operating on the call graph
- Language server protocol (LSP) implementation for editor support
- Distributed parsing across networked machines (extending the mmap'd file model)
