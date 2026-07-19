---
id: passes
title: Passes
sidebar_position: 3
---

# Passes

The optimization pipeline that turns raw lifted IR into something that reads like source. 
Passes are where dead code disappears, constants collapse, control flow is rebuilt, and virtual functions are recovered. 
They all run through one manager: the **pass_manager**, under a fixed schedule in `passes::setups::normal`.

## What a Pass is

Passes are pure transforms over an `ir_stat::space` that report whether anything changed. 
No pass edits the statement vector directly; each stages every removal, insertion, and mutation through the **pass_manager**, which applies them atomically and recomputes the analysis the next pass depends on.

## Pass Safety
To insure correct optimization each pass follows a strict guideline:
* Each pass is pre-proven to be completely safe with no unintended side-effects.
* Passes do not have any oscillating behaviours (e.g. `R1 << 1; <-> R1 * 2`)
* Each pass has one goal 

(When in doubt **DO NOTHING**, correct behaviour is better then incorrect behaviour)

## How the pass_manager Applies Changes

Passes call `remove`, `insert`, `push_front`, `push_back`, and `mut`; but none of those touch `ir.data` immediately. 
They record intent into four staging sets: **removals**, **insertions** (keyed by the statement to insert after), **front**, and **back**.

When the pass finishes, the manager **commits**. 
Commit rebuilds `ir.data` in one pass: it emits the `front` set, then walks the old data emitting every statement that was not removed and is not a `nothing` no-op, 
splicing in that statement's staged insertions right after it, then appends the `back` set. The rebuilt vector replaces `ir.data` in a single assignment.

* Commit - one linear rebuild, O(N + inserts).
* Staging - mutation is order-independent within a pass; the rebuild imposes the final order.

Because everything is staged and applied at once, a pass can freely remove statement *i* and insert near statement *j* without invalidating its own iteration.

## Re-analysis and the consistency check

After commit, the manager calls `update()`, which recomputes the analysis the passes read against the new `ir.data`. 
This is where the **ECC** (equivalence/consistency check) runs: it re-validates the IR after a pass, catching a pass that left the code inconsistent 
(for example, `validate()` errors on a `goto` whose label no longer exists). Passes can disable ECC for one run with the `fignore_ecc` flag when it knows it leaves the IR transiently inconsistent.

## The pass_manager API

* **Iterate** - `pm.iter()` (forward addresses), `pm.riter()` (reverse), `pm[i]` (the statement at address `i`), `pm.amount()`, `pm.contains(i)`.
* **Look around** - `pm.valid_next<N>(i)` / `pm.valid_prev<N>(i)` - is it safe to look `N` statements ahead / behind.
* **Mutate** - `pm.remove(...)`, `pm.insert(where, v)`, `pm.insert_front`, `pm.push_front`, `pm.push_back`, `pm.move(...)`.
* **Report** - `pm.mut(LURAMAS_DEBUG_LINE)` on every change; fixpoint scheduling depends on it.
* **Safety** - `pm.safe(stat)` / `pm.is_safe(...)` - is it safe to remove or move a statement (no live dependents).

## Run Flags

How often a pass runs is controlled by **passes::flags** at registration, not by the pass itself:

* **fmodified** - re-run until it stops making changes (fixpoint).
* **fsingle_pass** - run exactly once.
* **fafter_single** - run to fixpoint, then once more after it can no longer change anything.
* **ffast_folder** - constant folding may also perform dead-label elimination inline.
* **fignore_ecc** - skip the consistency check for this pass.

So a pass is written to do *one* sweep and report whether it changed anything; the scheduler decides how many times to call it.

## Example

Passes recognize a huerisic and rewrite it. `constant_fold` merges chained conditional gotos. Given two branches to the same label:

```
if (A) then goto l; end
if (B) then goto l; end
```

the pass detects the second `condition_goto` sits directly after another that jumps to the same label, checks both are safe, folds them with `append_cond<expr_logical::or_>`, and removes the second:

```cpp
if (tools::stat::branch::same_cond_goto_labels(p, prev) && pm.safe(prev, p)) {
      prev->append_cond<expr_logical::or_>(p->l, p->b, p->r);   /* (A || B) */
      pm.remove(p);
      pm.mut(LURAMAS_DEBUG_LINE);                               /* report the change */
}
```

The result:

```
if (A || B) then goto l; end
```

Every pass is a family of these recognize-and-rewrite rules, and the ASCII diagrams in the source comments show the hueristic rule matches.

## Schedule

`passes::setups::normal` runs the pipeline in phases. 
Each phase queues a batch, runs to fixpoint, clears, and moves on; ordering matters because later phases assume earlier ones ran.

* **Pages** (if `fhas_pages`, native only) - generate the main page, complete and close pages, merge labels into pages, promote and align, adjust linkage. Establishes the page structure the rest of the run maintains.
* **Initial control flow** - `label_flatten`, `branch_redirection`, `unreachable_code_elimination`. Gets the CFG into canonical shape.
* **Core** - the workhorse batch, repeated with page maintenance interleaved: `constant_propagation`, `virtual_function_reconstruction`, `dead_code_elimination`, `dead_label_elimination`, `dead_store_elimination`, `constant_fold`, `control_flow_simplification`, `expression_canonicalization_elimination`, `virtual_function_inline`, `jump_threading`.
* **Loops** - `loop_winding`, `branch_optimization`, `branch_threading`, `loop_simplification`, `loop_unroll`. Reconstructs loops from raw branches.
* **Control flow** - advanced propagation enabled: `dead_store_elimination`, `expression_canonicalization_elimination`, `constant_propagation`, `loop_canonicalize_exits`.
* **Page organization** (if `fhas_pages`) - organize, align, promote, inline pages; remove dead controllers and orphans; separate pages into final layout.
* **Expressions** - definition flattening enabled: `sub_expression_reordering`, `dead_store_elimination`, `constant_propagation`. Merges expression trees toward source density.
* **Finalization** - `constant_fold`, `expression_canonicalization_elimination`, `branch_optimization`, `flag_optimization` (if `feliminate_flags`), `branch_redirection`, `branch_simplification`, `variadic_function`.
* **Definitions** - `definition_flattening`, then `definition_inference`, `static_definition_inference`, and (if `fhas_types`) `set_descriptor_types`. Establishes variable definitions and their types for the emitter.

After the entry closure, any **unanalyzed pages** discovered during the run are lifted with the same `pass` function, recursively, until none remain.

## Pass families

Grouped by target, the sources live under `ir/lifter/passes/`:

* **Constant** (`constant.cpp`, `propagations/constant.cpp`) - folding and propagation, backed by **gmp**/**mpfr** so wide math never loses precision.
* **Eliminations** (`eliminations/`) - dead code, dead stores, dead labels, dead pages, redundant bit read/writes.
* **Control flow** (`control_flow.cpp`, `branches.cpp`, `loops.cpp`) - branch redirection/threading/simplification, loop winding/unrolling/canonicalization.
* **Patterns** (`patterns/`) - foldable if-conditions, compilable idioms, repeatable rewrites. Recognizes source-level shapes hidden in lowered code.
* **Pages** (`page.cpp`, `merger/`) - the native page model: promotion, inlining, linkage, organization.
* **Virtual functions** (`virtual_functions.cpp`) - reconstructs and inlines dispatched virtual calls. See **[Control Flow & SSA](./cfg-ssa.md)** and **[Customization](./customization.md#virtual-function-tables)**.
* **Flags** (`flags.cpp`) - erases CPU-flag bookkeeping once its semantics are captured in expressions.
* **Type** (`type.cpp`) - type-directed rewrites feeding definition inference.

To write your own, see **[Writing a Pass](./writing-passes.md)**.
