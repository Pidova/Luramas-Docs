---
id: overview
title: Overview
sidebar_position: 0
---

# Intermediate Representation

The **IR** is the middle of the decompiler and the part that does the real work. 
The IL that comes out of lifting is correct but sloppy - it says exactly what the machine does, one lowered operation at a time.
The IR is where that gets turned into something that reads like source: 
dead code disappears, constants collapse, control flow is rebuilt into loops and conditionals, and virtual functions are recovered.

Everything here operates on two types:
* **ir_stat** (a statement) 
* **ir_stat::ir_expr** (an expression) 

Everything downstream, every pass and every emitter, is written using them. This section is the reference for that model and the pipeline that runs over it.

## API

A whole IR run is a single call:

```cpp
ir_stat::space lift(std::shared_ptr<luramas::closures::closure> &closure,
                    const passes::environment_flags &env_flags = passes::environment_flags());
```

`lift` takes a `closure` (the tree the front-end produced from an `ilang` buffer) plus a set of **environment_flags** that describe what the target actually supports, 
and returns an optimized `ir_stat::space` ready to hand to an emitter. The closure came from the IL; nothing in this call knows which target produced it.

## What happens inside

`lift` runs the same sequence for every target:

1. **Build** - the `ilang` closure is lifted into an `ir_stat::space`, the raw statement stream.
2. **Reconstruct** - control flow is recovered into a **cfg**, dominators are computed, and the code is renamed into **SSA** so every value has exactly one definition. See **[Control Flow & SSA](./cfg-ssa.md)**.
3. **Optimize** - the **pass_manager** runs a fixed schedule of passes to fixpoint, reshaping the code and recomputing analysis between phases. See **[Passes](./passes.md)**.
4. **Finalize** - definitions and types are inferred so the emitter has variables and signatures to print.

The result is still `ir_stat` - the emitter is a separate stage. See **[Code Generation](../Code-Generation/emitters.md)**.

## Environment Flags

The IR is target-independent, but targets differ - some have pages (native code), some have real memory, some treat comparison results as integers. 
Those differences are passed in as **environment_flags**, and passes gate their behavior on them. 
A native x86 run sets `fhas_pages`, `fhas_memory`, `fhas_types`, `feliminate_flags`, and more; a Lua run leaves most of them off. The full set, and how a real run configures them, is in **[Customization](./customization.md)**.

## The rest of this section

* **[Data Model](./data-model.md)** - the `ir_stat` and `ir_expr` types, their kinds, and the emit API.
* **[Control Flow & SSA](./cfg-ssa.md)** - how the CFG, dominators, and SSA are built.
* **[Passes](./passes.md)** - the pass_manager, the commit model, and the schedule.
* **[Writing a Pass](./writing-passes.md)** - developing your own transform.
* **[Helper Functions](./helpers.md)** - the `tools::` toolkit passes are built from.
* **[Customization](./customization.md)** - environment flags, virtual-function tables, and output formatting.

> **Do not add or change primitive expression kinds.** Intrinsic handling across the whole pipeline depends on the existing `expr_kinds` set. Compose new ideas from existing kinds.
