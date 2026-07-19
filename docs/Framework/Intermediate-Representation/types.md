---
id: types
title: Types
sidebar_position: 7
---

# Types

Lowered code has no types: a register is just bits, and `add eax, ecx` says nothing about whether those bits are a signed int, an unsigned count, or a pointer. 
Emitting readable source in a typed language like: C, C++, Rust means reconstructing that information. 
The **types** system is where it represents and infers it. It runs only when the target enables: `fhas_types`; an untyped target skips it all.

## Two Layers

There are two type representations, and they serve different purposes:

* **underlying_type** (`misc/types.hpp`) - the low-level width-and-sign data. It records `unsign`, a `read_type` (`bits` or `bytes`), a `precision`, and a `storage_size`. This is what the optimizer reasons about when it needs to know whether an operation loses bits or changes sign.
* **object::type** (`ir/code/types/`) - the high-level type the emitter prints. It wraps an `underlying_type` but adds a `type_kind` (`native`, `primitive`, `object`, `dynamic`), a pointer count, and, for abstract or dynamic types, a nested object or set of possible types.

The split matters because a pass folding arithmetic cares only about the 
`underlying_type` (did this stay 32-bit signed?), while the emitter needs the `object::type` (is this an `int`, a `Struct*`, or a dynamic value?).

## underlying_type

The width primitives live in `luramas::types::native`. They start with widths:
 `t_int8` / `t_uint8` through `t_int64` / `t_uint64` but the set continues in powers of two all the way to `t_int16384` (128, 256, 512, 1024, 2048, 4096, 8192, 16384 bits, each in both signs), 
 plus `t_none` and `t_flag` (a single bit). Each is an `underlying_type` with a fixed sign and storage size.

Lowered native code does arithmetic on values wider than a register, and constant folding has to carry the full width without dropping bits
which is why types use **gmp** / **mpfr** and why a native run sets a large default width (x86 example uses `odefault_bits = 1024`). 

The operations on it are what the passes uses to keep width and sign correct through a rewrite:

* **dominant(other, dom)** / **dominant_t(...)** - combine two types into the one that should win, respecting a sign-dominance preference. This is how `u32 | i32` resolves to a single signedness rather than staying ambiguous.
* **bits()**, **bmin()** / **bmax()** - the width and the value range it can hold.
* **weak_compare** / **compare** - equality at different strictness.
* **diff_bits** / **diff_signess** - do two types differ in width or sign, the two questions a width-changing cast has to answer.

The `signess` enum encodes sign in bit 0 (`sign` = 0, `unsign` = 1), and `read_type` distinguishes a bit-width from a byte-width.

## object::type

The emitter-facing type. Its `type_kind` says what family it is:

* **native** - a concrete machine type, carried in the `native` `tkind` (see the **[Data Model](./data-model.md#tkind-value-kinds)**).
* **primitive** - a primitive reduced to its `underlying_type`.
* **object** - an abstract object, pointing at a nested `object`.
* **dynamic** - a value that can be one of several types, held in `dynamic_types`.

You build one through its `emit` overloads (`emit(tkind)`, `emit(underlying_type)`, `emit(object)`, `emit_dynamic(object)`, `emit_ptrs(n)`), query it with the `is<>()` templates, and it can `clone`, `compare`, `serialize`, and `str()` itself.
The common basic types are pre-made in `ir::types::common`: `i8`, `i16`, `i32`, `i64` and their unsigned forms so passes and the emitter share one space of common types.

## How Types are Inferred

Type reconstruction is the last phase of the pass schedule, run only when `fhas_types` is set. It is a small sequence of passes, each building on the last (see **[Passes -> the schedule](./passes.md#schedule)**):

* **definition_flattening** - collapse a value's definitions so each variable has a single coherent definition to type.
* **definition_inference** - infer each definition's type from how it is produced and used.
* **static_definition_inference** - the static-analysis pass that fixes types that flow across the whole function.
* **set_descriptor_types** - apply the inferred types as the descriptors the emitter reads (gated additionally on `fhas_types`).

The type-directed rewrites that feed this live in `ir/lifter/passes/type.cpp`; the sources are the **Type** pass family in **[Passes -> pass families](./passes.md#pass-families)**.

## What the Emitter Does With Them

Once inference has run, every definition carries an `object::type`, and the emitter prints it - as a variable's declared type, a function parameter type, a return type, or a cast. 
The signature-building in `generate` reads exactly these (`arg_types`, `result_types`) to print typed page-function signatures. 
A target without `fhas_types` gets dynamic, untyped output instead, which is why Lua and Luau (dynamically typed) leave it off and native C++ turns it on. 
See **[Emitters](../Code-Generation/emitters.md)** and **[Customization](./customization.md#capability-flags)**.

## Naming

Types decide *what* a variable is; they do not decide what it is *called*. 
Variable and symbol naming, the readable identifiers in the output, is a separate concern handled at emit time through the **format**'s naming conventions and
 the spell-aware heuristics (backed by **hunspell**). See **[Customization -> formatting](./customization.md#formatting)**.
