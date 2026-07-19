---
id: il-overview
title: Overview
sidebar_position: 0
---

# Intermediate Language

The **IL** is the architecture-independent instruction set that sits between a target and the IR. 
A lifter never emits x86 or any opcodes downstream. 
IR abstraction can be very involved so the IL uses generic and flexible opcodes which allows lifter development to be much simpler.
It emits IL, and everything past this point only ever sees IL. 
This is what lets one optimizer and one set of emitters serve unrelated targets, allow new front-ends to produce IL, and it utalize the entire pipeline.

This section has three parts. 
This page is the concept: the buffer a lifter fills and the pieces around it. 
The **[Instruction Set Architecture](./Instruction Set Architecture.md)** is the opcode reference. 
**[Lifting Assembly](./Lifting/disassemblers.md)** is how you actually produce IL from a target.

## Ilang Buffer

Every lifter's output is an **ilang** - the IL program. It holds three things:

* **dis** - the IL disassembly, the ordered stream of decoded IL instructions.
* **kval** - the constant pool. Booleans, integers, strings, tables, closures, upvalues; each `kvalue` knows its kind and can print itself.
* **closures** - nested `ilang`s. A function that defines inner functions carries them here, so the buffer is a tree, not a flat list.

One `disassembly` is one IL instruction: an address, an **opcode** (`arch::opcodes`), a binary-op kind where relevant, an operand list, and any cross-references (`xrefs` / `ref`) 
it resolved into other instructions: a jump target, a constant. 
The instruction can print itself through `disassemble()`, which is what the IL dumps you see are made of.

## Building IL

A lifter appends to the buffer through one emitter call per instruction:

```cpp
il::emitter::generate_opcode<arch::opcodes::OP_ADD>(il, pc, dest, source, value);
```

`generate_opcode<OP>` takes the buffer, the current address, and up to seven operand slots, decodes them into the right operand kinds for that opcode, 
resolves any constant or closure indices against the pool, and appends the `disassembly`. 
Bytecode lifters call it directly, one opcode at a time; native lifters go through the higher-level **build::expr** DSL, which calls `generate_opcode` underneath 
(see **[Writing a Lifter](./Lifting/writing-lifters.md)**).

## Lexer

Raw IL opcodes carry no classification like `OP_ADD` and `OP_SUB` are just two opcodes. 
The **lexer** (`il/lexer/`) attaches the categories the closure builder and IR need:

* **operand_kinds** - what each operand *is*: `reg`, `dest`, `source`, `value`, `integer`, `kvalue`, `jmpaddr`, etc. This is how a downstream stage reads an operand without knowing the opcode.
* **inst_kinds** - what the instruction *does*: `arith`, `branch`, `branch_condition`, `load`, `compare`, `unary`, etc. Passes and the closure builder branch on this instead of enumerating opcodes.

A lexed instruction is a `lexeme` wrapping the `disassembly` plus its kinds; that is what a `closure` node holds.

## Validation

The IL validates itself. `ilang::validate` and `validate_operands` check that the buffer is valid: operands match their opcode's expected kinds, 
references resolve which raise a `commit_error` when something is off. 
This is the IL-level equivalent of the IR's ECC: it catches a malformed lifter before the bad IL ever reaches the IR.

## Transformers

`transformers::kinds(il)` runs the IL-level normalization that has to happen before lifting to IR; resolving instruction kinds across the buffer so the closure builder sees a consistent stream. 
It is a small, fixed step; the heavy optimization all lives in the IR.

## Library

Some calls are known runtime functions, not user code. 
The **library** (`il/libary/`) names them: a `library_module` records an author, name, and version, and `naming_conventions::generate` produces 
a stable name for a function so the same runtime helper reads the same way across every decompile. 
Native targets tie declared externals to these; see **[Native Code](./Lifting/native-code.md)** and **[Customization](../Intermediate-Representation/customization.md)**.

## IL to IR

Before the IR can optimize it, the buffer is turned into a **closure tree** by `gen_closure`, which lexes each instruction and groups the nested 
`ilang`s into a structure the IR lifter walks. That bridge is its own step: see **[Closures](./closures.md)**. From there, `ir::lift` takes over (**[IR Overview](../Intermediate-Representation/overview.md)**).
