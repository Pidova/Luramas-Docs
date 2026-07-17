---
id: closures
title: Closures
sidebar_position: 3
---

# Closures

The bridge between the IL and the IR. 
The IR lifter does not walk that directly; it walks a **closure tree**. 
Turning one into the other is a single call, and it is the last thing that happens before `ir::lift` takes over.

## API

Every target does the same thing after lifting to IL:

```cpp
auto closure = luramas::closures::gen_closure(il_buffer);
```

`gen_closure` takes the `ilang` and returns a `closure`. 
That closure is what you hand to `ir::lift`. 
**Nothing is target-specific anymore.**

## What a closure is

A **closure** is the lexed, structured form of one `ilang`:

* **il** - the source `ilang` it was built from.
* **nodes** - the instruction list, but lexed. Each **node** wraps a `lexeme` (the IL `disassembly` plus its operand/instruction kinds from the lexer), so downstream code reads categories instead of re-deriving them from opcodes.
* **closures** - the child closures, one per nested `ilang`. The tree: an outer function with inner functions produces an outer closure with child closures.
* **shared_memory** - `program_memory` shared across the tree, for state that spans closures.

A node also carries small per-instruction extras the builder computes. For example the poparg-flag bookkeeping that records which registers a call pops off the stack.

## What gen_closure does

Building the closure is where the flat IL becomes a walkable structure:

1. **Lex** every instruction - attach its `operand_kinds` and `inst_kinds` so the IR lifter can branch on what an instruction *does* rather than which opcode it is.
2. **Nest** - recurse into the buffer's `closures`, building a child closure for each and linking it under its parent.
3. **Flag** - run `set_flags` / the flag processor (`closures/processor/`) to fill in the per-node analysis (like the call-pop sets) the IR relies on.

The result is a `closure` whose `nodes` are ready to lift and whose `closures` mirror the IL's nesting.

## Closure flags

The closure carries a `flags` block that tells the IR how to treat the whole tree. The x86 example sets one directly:

```cpp
auto closure = luramas::closures::gen_closure(buffer);
closure->flags.fassociated_args = true;
```

These are properties of the lifted program as a whole, distinct from the per-run **environment_flags** you pass to `lift`. 
Set them after `gen_closure` and before `lift`.

## Inspecting a closure

For debugging, a closure can print itself:

* **str()** - the disassembly of the main closure's nodes as text.
* **dump()** - prints the main closure and every nested closure, walking the tree.
* **extract(reg)** - prints each node as the `generate_opcode<...>` call that would recreate it, optionally substituting register names. Useful for turning an observed closure back into lifter code.

## Next

Now with a closure we can move into IR: `ir::lift(closure, env_flags)`. 
See **[IR Overview](../Intermediate-Representation/overview.md)** for what happens next, 
and **[Customization](../Intermediate-Representation/customization.md)** for environment flags.
