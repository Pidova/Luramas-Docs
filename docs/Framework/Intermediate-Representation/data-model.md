---
id: data-model
title: Data Model
sidebar_position: 1
---

# Data Model

The two types every pass and every emitter operates on: **ir_stat** (a statement) and **ir_stat::ir_expr** (an expression). 
This page is the reference for their fields, their kind enums, and the emit API that builds them. Defined in `ir/ir_defs.hpp` and `ir/definitions.hpp`.

> **Do not add or change primitive expression kinds.** Intrinsic handling across the pipeline depends on the existing `expr_kinds` set. Compose new logic from existing kinds.

## ir_stat: statements

Function bodies are `ir_stat::space` = `vector<shared_ptr<ir_stat>>`. 
Each statement has a **keyword** kind (`p->k`) and a set of expression/space fields whose meaning depends on that kind. 
Common fields: `l` / `r` (l-value / r-value), `v` (value), `members` (an expression space - call args or multi-assign targets), `smembers`, `tmembers`, `label` / `jlabel` (label ids), plus flags.

### Keywords (statement kinds)

* **Assignment / data** - `assignment`, `table_assign`, `table_setlist`, `memoryset`, `bitwrite`, `create_stack`, `set_flag`, `globals_preset`.
* **Control flow** - `condition`, `condition_goto`, `goto_label`, `label`, `end`, `break_`, `continue_`, `while_`, `repeat`, `until`, `switch_`, `switch_case`, `switch_default`.
* **Loops** - `forloop_generic`, `forloop_numeric`, `for_loop_init`.
* **Calls / returns** - `call`, `retn`.
* **Stack** - `stack_push`, `stack_pop`, `stack_read`.
* **Definitions / structure** - `definition` (always the first statement of a closure or page: args, upvalues, flag ids, arg casts, controller), `entry_point`, `isolate`, `metadata`, `tag_start`, `tag_end`, `command`.
* **Pages** - `page_function_start`, `page_function_end`, `page_function_goto`, `page_function_pass`, `page_function_closure`.
* **nothing** - no-op (skipped at commit).

Check a kind with `p->is_k<keywords::goto_label>()` or the named `tools::stat::is_goto_label(p)` predicates. Each keyword's exact field layout is documented inline in `definitions.hpp` - for example `retn` uses `members` for return values and `smembers` for page-return locations.

## ir_stat::ir_expr - expressions

Expressions form recursive trees. An `ir_expr::space` is `vector<shared_ptr<ir_expr>>`.

### The tree fields

* **l** - l-value / first source.
* **r** - r-value / second source.
* **ev** - extra value (bitread max, ternary then).
* **xv** - extended value (bitwrite max, ternary else).
* **members** - object members / call args.
* **tmembers** - table member pairs (key, value).
* **captures** / **closure** - captured data and closure body for calls/closures.

### The kind tags

* **k** (`expr_kinds`) - what the expression *is*.
* **b** (`bin_kinds`) - the binary op, for `arith` / `condition`.
* **u** (`bin_kinds`) - the unary op, for `unary`.
* **e** (`expr_logical`) - logical op, for `condition`.
* **tk** (`tkind`) - the type/value kind.
* **rk** (`expr_reg_kinds`) - register role (`reg` / `arg` / …).

### The value fields

* `n` (numeric)
* `bv` (boolean)
* `v` (string)
* `reg` / `vreg` (physical / virtual register)
* `amount` (count)
* `non_native` (resulting/castable type)
* `xtype` (extended analysis type).

### expr_kinds

`nothing`, `bitread`, `bitwrite`, `memoryread`, `call`, `arith`, `condition`, `unpack`, `concat`, `idx`, `unary`, 
`reg`, `self`, `closure`, `upvalue`, `ternary`, `cast`, `flag`, `blank_lvalue`, `page_function_call`.

Each has a documented field layout in `definitions.hpp`: for example `bitread` uses `l` (value), `r`/`ev` (min/max), `non_native` (result type); `ternary` is `(l CMP r) ? ev : xv`; `memoryread` uses `l` (target), `r` (offset), `non_native` (read type).

### tkind (value kinds)

`nothing`, `none_obj`, `variadic`, `table`, `string`, `lura_int`, `global`, `boolean`, `kvalue`, `stack`, `object`, `controller`, `extpr`.

### expr_logical

Logical connectors used when `k == condition`: the `e` field distinguishes a plain comparison from an `and` / `or` chain.

## Building expressions: The emit API

`ir_expr` has an `emit_*` method for every type, and the `tools::exprs::generate::*` helpers call them. 
Prefer the generate helpers in passes; the raw API is:

* **Leaves** - `emit_int(i)`, `emit_boolean(b)`, `emit_str(s)`, `emit_reg(r)`, `emit_reg_arg(r)`, `emit_global(g)`, `emit_table()`, `emit_variadic()`, `emit_stack(id)`, `emit_none()`, `emit_nothing()`, `emit_controller()`.
* **Operators** - `emit_arith(l, r, b)`, `emit_unary(l, u)`, `emit_cond(l, b, r)`, `emit_logical<e>(l, r)`, `emit_ternary(...)`.
* **Access** - `emit_idx(l, r)`, `emit_table_get(v, i)`, `emit_table_set(v, i)`, `emit_concat(v)`, `emit_self(l, r)`.
* **Calls** - `emit_call(call)`, `emit_arg(arg)`, `emit_vcall(vcall)`, `emit_varg(varg)`.
* **Memory / bits** - `emit_memoryread(target, bits, offset)`, `emit_bitread(...)`, `emit_bitwrite(...)`.
* **Types** - `emit_cast(object, value)`, `emit_object(obj)`.
* **Closures / upvalues / pages** - `emit_closure(space)`, `emit_upvalue(r, vreg)`, `emit_capture(l, kind)`, `emit_flag(id)`, `emit_page_function_call(...)`.

## Operations on both types

* **clone(deep, regs)** - Deep or shallow copy; `regs` controls whether registers are copied.
* **transform()** (expr -> stat) - Lift an expression into its statement form.
* **operator==** on the pointee (`*a == *b`) - Value comparison, used throughout the predicates.

## Type Queries

Template predicates read the kind tags directly:

```cpp
e->is_k<expr_kinds::arith>();    /* is it arithmetic?          */
e->is_tk<tkind::reg>();          /* is it a register value?    */
e->is_b<bin_kinds::add_>();      /* is the binop an add?       */
```

The `bin_kinds` set (shared with the IL) is the operator vocabulary: 
* **Arithmetic** - `add_ sub_ mul_ div_ idiv_ mod_ pow_ and_ or_ xor_ shl_ shr_` 
* **Comparison** - `eq_ ne_ lt_ lte_ gt_ gte_ et_ nt_`
* **Unary** - `len_ minus_ not_ bitnot_ plus_ ref_` 

These are the same kinds the lifter DSL emits and the code generator spells out.
