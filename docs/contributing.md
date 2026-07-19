---
id: contributing
title: Contributing
sidebar_position: 7
---

# Contributing

How to build **Luramas**, where every part lives, and the conventions a patch is expected to follow.

## Building

Building instructions can be found [Building Luramas](./Framework/building.md)

## Repository layout

Everything sits under **framework/**.

* **framework/disassembler/** - Per-target decoders and opcode tables (**Lifting Assembly**). One sub-directory per target.
* **framework/il/** - The **Intermediate Language**: the **ilang** core (**il/il/**), the per-target lifters (**il/lifter/langs/**), the IL lexer, the IL library.
* **framework/ir/** - The **Intermediate Representation**: the **ir_stat** core (**ir/ir_defs.hpp**), the lifter and its pass pipeline (**ir/lifter/**), analysis, types, symbols, virtual-function recovery.
* **framework/ir/code/** - **Code Generation**: the **generate()** entry and every language emitter under **generation/lang/**.
* **framework/closures/** - The bridge that turns an **ilang** buffer into a **closure** tree.
* **Shared/luramas/** - Shared utilities: **profile** native block-builder, formatting, profiles, etc.

## Naming Conventions

The codebase is consistent. Match what is already there.

* **Naming prefixes** Flag fields start **f** (**fhas_pages**), option fields **o**, safety fields **s**. Opcode string constants start **OP_**.
* **Containers.** Reach for **boost::unordered_flat_map** / **_set** before **std::** equivalents unless ordering is required. Hot paths assume flat containers.
* **Formatting.** A **.clang-format** sits at the repo root and in each vendored tree. Run it before submitting.
* **No new primitive expression kinds.** **ir_stat::ir_expr** carries the warning in its own definition: changing or adding primitive kinds breaks intrinsic handling downstream. Be creative with existing kinds.
* **Passes are pure transforms.** A pass mutates only through the **pass_manager** API and reports whether it changed anything. Never touch the statement vector directly. See **[Writing a Pass](./Framework/Intermediate-Representation/writing-passes.md)**.

## Adding a target

To add a new target architecture:

1. Write a decoder under `framework/disassembler/<target>/` with an opcode table. See **[Disassemblers](./Framework/Intermediate-Language/Lifting/disassemblers.md)**.
2. Write a lifter under `framework/il/lifter/langs/<target>/` that emits **IL disassembly** into an **ilang** buffer. See **[Writing a Lifter](./Framework/Intermediate-Language/Lifting/writing-lifters.md)**.
3. Nothing in the IR or any pass changes. The middle is already written.

For a **native** target there is one extra front-end step: describe the instruction stream with the **[profile](./Framework/Intermediate-Language/Lifting/native-code.md)** block-builder before disassembling and lifting. 
Bytecode targets skip it; the VM's `Proto` already has the structure.

To add a new output language, add an emitter under `framework/ir/code/generation/lang/<language>/` and a case in the **emitter_syntax** dispatch. 
See **[Emitters](./Framework/Code-Generation/emitters.md)**.

## Testing

Concepts use to validate if decompilations and passes are correct:

* **ECC (Error Consistency Check)** - after every pass commits, the manager re-validates the IR and raises on inconsistency (a `goto` to a label that no longer exists, and so on). It is on by default; a pass that leaves the IR transiently inconsistent opts out for one run with `fignore_ecc`. Build with the **LURAMAS_ECC** flag to enable the structural checks; see **[Building: flags](./Framework/building.md#macros)**.
* **The IR fuzzer** - `ir::fuzzer::generate(seed)` produces a random `ir_stat::space` from a seed. Running the pipeline over fuzzed IR passes that mutate incorrectly or fail the consistency check. When you add a pass, fuzzing it is the cheapest way to find the case you did not think of.

When in doubt, **do nothing**. It is better to be correct then incorrect. See **[Passes: pass safety](./Framework/Intermediate-Representation/passes.md#pass-safety)**.

## Patch Versioning

Patches follow the standard verisioning system: **Major.Minor.Patch (xx.xx.xx)**