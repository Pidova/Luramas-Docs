---
id: helpers
title: Helper Functions
sidebar_position: 5
---

# Helper Functions

The toolkit a pass is built from. 
You rarely inspect an `ir_stat` field by hand or construct one from scratch: the `tools::` helpers recognize hueristics, answer analysis questions, and build correct replacements for you.
This page is the map of what lives where.

Two families cover almost everything a pass writes:

* **tools::stat** / **tools::exprs** - predicates (`is_*`), mutators, and generators over statements and expressions. Declared under `ir/lifter/tools/extras/`.
* **tools::** analysis namespaces - the questions passes ask about the surrounding code: what dominates what, what is safe, what a value resolves to. Declared in `ir/lifter/tools/tools.hpp`, defined under `ir/lifter/tools/code/`.

> Prefer a helper over hand-written field access. The predicates encode the exact field layout each kind uses, so a helper stays correct even when a hueristic has edge cases the obvious check would miss.

## Recognizing Hueristics

Every kind has an `is_*` predicate, and most take optional arguments to narrow the match. 
They are the first line of almost every pass.

### tools::stat (statements)

* **Kinds** - `is_goto_label`, `is_label`, `is_end`, `is_condition`, `is_call`, `is_return`, and one per keyword. Prefer these over a raw `p->is_k<...>()` when the named predicate exists.
* **tools::stat::branch** - branch-specific queries: `is_cond_goto_label`, `same_cond_goto_labels`, `is_single_label_ref`, and the family that reasons about conditional gotos and their targets.
* **tools::stat::flags** - Flag-statement predicates.
* **tools::stat::assignment** - Assignment-hueristic queries.
* **tools::stat::mutate** - In-place statement rewrites, including `extract_volatiles_stats` (pull the side-effecting parts out of a statement before you remove it) and `mimic_compare`.

### tools::exprs (expressions)

Under **tools::exprs::values**, one predicate per expression hueristic: 
`is_arith`, `is_reg`, `is_arg`, `is_upvalue`, `is_call`, `is_closure`, `is_condition`, and more. 
Most are overloaded to also match a specific value example: `is_reg(e, r)` matches register `r`, `is_integer(e, n)` matches literal `n`, `is_boolean(e, b)` matches a specific bool.

The **tools::exprs::values::types** sub-namespace answers type questions on an expression: 
`is_basic`, `is_same_implicit`, `is_restricted`, `is_resulting_type`, `is_under_signed` / `is_under_unsigned`, `is_reg_cast`, `is_boolean_cast`.

## Building Replacements: Generate Helpers

When a pass produces new code, it builds through the generators, never by adding stuff to `ir_stat`.

### tools::stat::generate

One builder per statement kind, each returning a ready
`shared_ptr<ir_stat>`: `end()`, `label(loc)`, `goto_label(loc)`, `cond_goto_label(l, b, loc, r)`, `while_stat(...)`, `until(...)`, `repeat()`, `else_stat()`, `break_stat()`,
`continue_stat()`, `assignment(l, r)`, `table_assignment(t, idx, v)`, `create_stack(l)`, and more.

### tools::exprs::generate

The matching expression builders: `memoryread(target, bits)` and the rest plus the raw `emit_*` API on `ir_expr` itself. 
See **[Data Model -> the emit API](./data-model.md#building-expressions-the-emit-api)** for the full list.

## SSA (tools::ssa)

Everything a pass needs to reason about definitions and uses without rebuilding SSA.
 See **[Control Flow & SSA](./cfg-ssa.md#using-ssa-in-a-pass)** for the model.

* **defined_scope(pm, ssa, start, target)** - Is the register defined in scope at `start`.
* **used(pm, ssa, start, end, target)** - Is the register read anywhere in the range.
* **use_def_chain(ssa, target, limit)** - The definition chain of a value, scalar or phi. Returns `use_def_result` entries carrying the `assignment_kind`, the scalar version, and any phi members.
* **is_placeholder_variable(pm, ssa, target, start)** - A value assigned before every branch overwrites it, so it never affects execution.
* **same_highlevel_scope_id(pm, ssa, l, r)** - Do two addresses share a high-level scope.
* **tools::ssa::extract** - the heavier queries: `dominant_define`, `same_assignments`, `block_assignment` (with `hit_type::all` / `first` / `dominant`), `next_assignment_same_scope_assignment`, `all_dominant_singletons`, `parent_page`, and `linked`.

## Analysis namespaces

Grouped by the question they answer, all under `luramas::ir::tools`:

* **accumulate** - Gather things across a block or range: label refs, dominant addresses, jump-outs, break-outs, keywords in a block. The **accumulate::orphans** sub-namespace gathers page starts/ends and implicit gotos.
* **find** - Locate a statement by pointer or predicate (`find` returns `pm.amount()` when nothing matches), or find an expression by callback.
* **violations** - Does a block break the branch rules; `violations::accumulate` collects them.
* **contains** - Membership queries over statements and expressions, with `orphans` and `implicit` sub-namespaces.
* **types** - Type queries over expressions and definitions.
* **extract** - Pull sub-structures out (`space_stat`, `stats`, `exprs`, `ir`).
* **count** - Counts by callback, kind, or tkind: `insts` in a range, `definition_parameters`, `refs` to a label, and the `tk<>` / `keyword<>` templates.
* **dominant** - Dominance queries and `dominant::extract`.
* **control_flow** - Control-flow-level helpers, with a `block` sub-namespace.
* **loops** - Loop recognition and structure.
* **paging** - The native page model helpers (the largest analysis namespace).
* **compute** - Evaluate expressions and statements, including `compute::integrals` and `compute::strings`, backed by **gmp**/**mpfr** so wide math never loses precision.
* **simulate** - Reach a target through basic loop threading when direct control flow cannot.
* **safety** - Safety queries, including `safety::arith`.
* **guarantee** - What an expression or statement is guaranteed to do.
* **match** / **replace** - Hueristic matching and rewriting; **replace::match_wild_cards** powers the virtual-function tables (see **[Customization](./customization.md#virtual-function-tables)**).
* **inliner** - Inlining support, with a `cva` sub-namespace.
* **mutations** - Higher-level mutations like `pop_cond` and `safe_if_dupe`.
* **Remaining helpers, each named for what it does.**

## When You Need Something That is Not Here

If you find yourself reading `ir_stat` fields directly or building an `ir_expr` by hand, check for an `is_*` predicate and a `generate::*` builder first.
Adding a new helper to the right namespace is almost always better than inlining the logic into a pass: the next pass will want it too. See **[Writing a Pass](./writing-passes.md)**.
