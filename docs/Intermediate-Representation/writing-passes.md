---
id: writing-passes
title: Writing a Pass
sidebar_position: 4
---

# Writing a Pass

Practical guide to developing a pass, the signature, the iteration model, the mutation rules, and the one invariant that keeps sixty passes from corrupting each other. 
The mechanism behind these rules is in **[Passes](./passes.md)**; this page is how to use it.

## Signature

Every pass is a free function with one definition, declared in `ir/lifter/passes.hpp`, defined under `ir/lifter/passes/`:

```cpp
void my_pass(pass_manager &pm, shared &s);
```

* **pm** - the pass_manager: the live IR plus the only mutation API.
* **s** - shared per-run state the manager threads between passes.

Register it in `passes::setups::normal` with `pm.add(my_pass, flags, "My pass")`, in the phase where its preconditions hold.

## Iterating

Walk statements through the manager, **never over a raw vector**:

```cpp
void my_pass(pass_manager &pm, shared &s) {
      for (const auto &i : pm.iter()) {   /* forward addresses; riter() for reverse */
            auto &p = pm[i];              /* the ir_stat at address i */
            switch (p->k) {
                  case keywords::goto_label: {
                        /* ... */
                        break;
                  }
                  default: {
						break;
				  }
            }
      }
}
```

* `pm.iter()` / `pm.riter()` - forward / reverse address ranges.
* `pm[i]` - the statement at address `i`; **pm.amount()** - the count; **pm.contains(i)** - bounds-check a computed address before indexing.
* `pm.valid_next<N>(i)` / `pm.valid_prev<N>(i)` - safe to look `N` statements ahead / behind.

## Mutating through the manager

**The invariant: a pass never edits the statement list directly.** 
All structural change is staged and applied atomically at commit (see [Passes: how changes are applied](./passes#how-the-pass_manager-applies-changes), 
so your iteration stays valid even as you queue edits.

* **pm.remove(stat, safe?)** - remove a statement (or space, set, index range, block range); variadic form removes several.
* **pm.insert(where, v)** - insert `v` after statement `where` (single stat or space; variadic for several).
* **pm.insert_front / push_front / push_back** - positional inserts.
* **pm.move(where, stat)** / **pm.move(where, range)** - relocate.
* **pm.mut(LURAMAS_DEBUG_LINE)** - **report a change.** Fixpoint scheduling depends on this; forget it and a `fmodified` pass stops early.

## Safety

Before removing or reordering, ask whether it is safe, a statement is unsafe to touch if something depends on its effect:

* **pm.safe(stat)** - safe to remove/move (no live dependents); variadic checks several.
* **pm.is_safe(stat)** / **pm.is_safe(range)** - flag-level query.
* **pm.set_safe(stat)** - mark safe after satisfying dependents.

### Example
Canonical removal, from `dead_code_elimination` - check safety, preserve side effects, remove, report:

```cpp
if (tools::stat::branch::is_cond_goto_label(p, executable) && pm.safe(p)) {
      for (const auto &v : tools::stat::mutate::extract_volatiles_stats(p)) {
            pm.insert(p, v);            /* keep any side effects the branch had */
      }
      pm.remove(p);
      pm.mut(LURAMAS_DEBUG_LINE);       /* tell the manager we changed something */
}
```

## Building replacements

You rarely construct `ir_stat` / `ir_expr` by hand. The generate helpers do it correctly. In a pass you typically:

1. Recognize a shape with an `is_*` predicate (`tools::stat::is_*`, `tools::exprs::is_*`).
2. Build the replacement with `tools::stat::generate::*` / `tools::exprs::generate::*`.
3. Swap it in via `pm.insert` + `pm.remove`, then `pm.mut`.

See **[Helper Functions](./helpers.md)** for the full toolkit and **[Data Model](./data-model.md)** for the structures.

## Run flags recap

You do not loop to fixpoint yourself; the flags you register with control that: `fmodified` (until no change), `fsingle_pass` (once), `fafter_single` (fixpoint then one extra). 
Write the pass to do one sweep and report; let the scheduler decide the rest.

## Checklist

* One responsibility per pass. Compose, don't combine.
* Iterate with `pm.iter()`, index with `pm[i]`, bounds-check computed addresses.
* Mutate only through `pm`; never touch the underlying vector.
* Call `pm.mut(...)` on every change.
* Gate target-specific behavior behind an **environment_flag** (`pm.env_flags`).
* Register in `setups::normal` at the correct phase.
