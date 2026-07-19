---
id: customization
title: Customization
sidebar_position: 6
---

# Customization

The IR is target-independent, but real targets are not identical. 
Native x86 has pages, memory, and CPU flags; Lua bytecode has none of those. 
Those differences are handed to `lift` as data. 
This page covers 3 fundamental structures: 

* **environment_flags** that tune the passes
* **virtual-function tables** that fold known idioms into named calls
* **format** that shapes the emitted text.

## Environment Flags

`lift` takes a `passes::environment_flags` describing what the target supports. 
Passes gate their behavior on it, so the same pass does the right thing on a stack VM and on native code. The flags are defined in `ir/ir_defs.hpp`.

```cpp
luramas::ir::passes::environment_flags env_flags;
env_flags.fhas_pages   = true;   /* native code is organized into pages   */
env_flags.fhas_memory  = true;   /* target has real addressable memory     */
env_flags.fhas_types   = true;   /* run type inference for the emitter     */
env_flags.feliminate_flags = true; /* erase CPU-flag bookkeeping once modeled */
/* ... */
auto ir = luramas::ir::lift(closure, env_flags);
```

### Capability Flags

These say what the target *has*, and unlock the matching passes:

* **fhas_pages** - native code is a set of pages (functions). Enables the whole page pipeline: promotion, inlining, linkage, organization.
* **fhas_memory** - the target has addressable memory; enables memory reads/writes and their folding.
* **fhas_references** - the target has references.
* **fhas_types** - run definition/type inference so the emitter has real signatures.
* **fhas_nan** - NaN can be deduced from arithmetic.

### Behavior Flags

These say how the target *behaves*, so passes fold correctly:

* **feliminate_flags** - erase CPU-flag bookkeeping once its meaning is captured in expressions.
* **fcomparative_results_binvals** - comparison results are integers, not booleans.
* **fprimitive_object** - primitives are objects.
* **fuse_bitwise** - allow optimizations down to bitwise ops.
* **fexprcanon_use_table** - let expression canonicalization use the table path.
* **fallow_definition_flattening** / **fallow_advance_constant_prop** - enable the heavier propagation and flattening passes.
* **fremove_page_dead_args** / **fremove_main_dead_args** / **fremove_dead_synthetics** - how aggressive dead-argument and dead-synthetic removal is allowed to be.
* **fpromote_safety** - make the safety rules stricter (forbid more).

### Safety and Options

Two nested structs carry the fine-grained knobs:

* **safety** - per-operator arithmetic guarantees. For example `sarith_rvalue_zero.insert({bin_kinds::mod_})` tells the optimizer the right side of a `mod` is never zero, so it can fold without guarding a divide-by-zero. There are matching sets for negative and zero on each side, plus stack-related safety for page calls.
* **options** - defaults and callbacks: `odefault_bits` and `odefault_type` (the width and type to wrap to), `osign_dominance`, `oforce_propagation_depth_threshold`, the `ostat_virtual_functions` / `oexpr_virtual_functions` tables (below), and the page-call callbacks (`opage_call_action_s`, `opage_call_action_e`, `opage_return_read`, `opage_return_action`) that describe how a page call reads and writes the stack.

Everything defaults to the conservative choice, so a bytecode target can pass an almost-empty `environment_flags` and get a correct result; native targets use more aggresive options.

## Virtual-function Tables

Some lowered IR are really a known call: a runtime helper, an inlined intrinsic, a compiler idiom. 
The virtual-function tables let you register "this IR chunk means this call," and the reconstruction pass rewrites every match.

The matcher is **tools::replace::match_wild_cards(value, match, dest)**: 

* `match` - pattern with wildcard holes
* `value` - code under inspection
* `dest` - what to emit when it matches, with the captured holes filled in. 

You supply the pattern/replacement pairs through the options:

* **options.oexpr_virtual_functions** - expression-shaped pairs.
* **options.ostat_virtual_functions** - statement-space-shaped pairs.

The recovery and inlining of these is done by the virtual-function pass; see the pass family in 
**[Passes -> pass families](./passes.md#pass-families)** and the analysis in **[Control Flow & SSA](./cfg-ssa.md)**. 
The `virtual_functions::emanager` (in `ir/virtual_functions/`) is the container that holds the registered key/value expression pairs, with a 
`foptional_cast` flag controlling whether casts in the pattern must be explicit or only need a matching referenced type.

## Formatting

The environment flags decide *what* the IR becomes; the **format** decides how the final text *looks*. `generate` takes a 
`std::shared_ptr<ir::data::format::format>` (defined in `Shared/luramas/formatting/`), and it controls indentation, spacing, line breaks, semicolons, and variable naming 
independently of the emitter language.

```cpp
auto format = std::make_shared<luramas::ir::data::format::format>();
format->linebreak.page_function_end_post = 1u;
auto text = luramas::ir::code::generation::generate(syntax, ir, format);
```

The `format` struct is organized by concern:

* **indent** - per-statement indent deltas (pre/post) for every construct, plus a `collapse` sub-struct that aligns and collapses small tables, parameter lists, and ternaries.
* **spacing** - spacing around args, parameters, table delimiters, and unary operators.
* **linebreak** - blank-line counts before and after each construct.
* **stats** - statement-level choices: whether to emit semicolons (and after which keywords), whether to fold `a = a + b` into `a += b`, whether to drop useless returns.
* **vars** - variable naming: the case convention (`snake_`, `pascal_`, `camel_`, `flat_`, `screaming_snake_`), prefixes and suffixes per register role (arg, upvalue, flag), and the **smart** naming heuristics.

Because formatting is separate from emission, one emitter can produce many styles, and the same style can be applied across languages. 
See **[Emitters](../Code-Generation/emitters.md)** for how the emitter and format interact.
