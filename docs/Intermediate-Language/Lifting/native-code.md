---
id: native-code
title: Native Code
sidebar_position: 3
---

# Native Code

Bytecode targets hand you a clean `Proto` - the instructions, the constants, the structure, all already laid out. 
Native targets do not. 
A binary is just bytes, control flow is implicit in jumps, and self-modifying code means one address can decode more than one way. 
The **profile** (`luramas::profile`, in `Shared/luramas/`) is how native instruction streams are described before a native lifter ever runs.

This is the native-only front step.
A native lifter (X86) builds a profile, disassembles it through Capstone, then lifts; a bytecode lifter skips all of this. 
If you are only working with bytecode targets, you do not need this page.

## Block Builder

You describe the instruction stream to a **builder::manager**, one instruction at a time, through `emit`:

```cpp
luramas::profile::builder::manager data;

data.emit({0x89, 0xF8});                                                 /* mov eax, edi        */
data.emit({0x6B, 0xC0, 0x07});                                           /* imul eax, eax, 7    */
data.emit({0x74, 0x18}, luramas::profile::inst_kind::jump_to, 1u, true); /* je -> label 1       */
/* ... */
data.emit_label(1u);                                                     /* LABEL 1             */
data.emit({0xC3});                                                       /* ret                 */
```

Each `emit` takes the raw instruction bytes and records an `inst`. Overloads let you say more about what the instruction is:

* **emit(bytes)** - a normal instruction.
* **emit(bytes, entry)** - mark an entry point.
* **emit(bytes, kind, label, conditional)** - a control-flow instruction: an `inst_kind` (`jump_to`, `call_to`, `return_to`, `memset`) targeting a **label id**, optionally conditional.
* **emit(bytes, goto_loc, kind, conditional)** - the same, targeting an address directly.
* **emit_label(id)** - place a label that earlier or later jumps resolve against.

Labels are symbolic: you `emit` a jump to label `1` before or after you `emit_label(1u)`, and the builder resolves the pending references when it finalizes. 
Bytes go into a fixed `inst_bytes` buffer capped at `MAX_INST_LEN` (15, the longest X86 instruction).

## Instructions, discrepancies, and pages

The model is built for the messiness of real native code:

* **inst** - one instruction: its bytes, its `pc` (address in memory), a `real_pc` (its position relative to other instructions), the control-flow `kind` and target `loc`, and flags like `entry`, `conditional`, and `discrepancy`. It knows how to emit its own jumps/calls/returns (`emit_jump`, `emit_call`, `emit_return`, `emit_memset`) and to compare itself byte-for-byte against another.
* **real_inst** - an address that decodes more than one way. `first` is the dominant decode; `discrepancies` holds the alternatives. This is how self-modifying and overlapping code is represented without losing either reading - when you iterate unsafe code, `first` is the one to trust.
* **inst_result** - the finished map for one module: the `real_inst` map keyed by address, plus the label and entry sets.
* **module_id** - native code spans modules (the main image, libraries). Everything is keyed by module so cross-module references stay unambiguous.

## Extracting and analyzing

Once the stream is described, you pull it out and analyze it:

```cpp
boost::unordered_flat_map<profile::module_id, profile::inst_result> mid_res;
data.extract(mid_res);
const auto details = profile::analyze::generate_details(mid_res);
```

* **extract(buffer)** - hand the built instructions out as a `module-id -> inst_result` map. `dump()` prints them; `clear()` resets the builder.
* **analyze::generate_details** - produce the `details` the lifter needs: page starts, the instruction data, and external call locations.
* **analyze::order_of_execution_organized** - the execution order with discrepancies classified (`normal`, `inlaned`, `optional`, `first`), which is exactly what the x86 example walks to feed Capstone one instruction at a time.
* **analyze::generate_pages** - recover page (function) entries from the stream. Pages are the native function model the IR maintains; see **[Passes -> Schedule](../../Intermediate-Representation/passes.md#schedule)**.

## Externals

Calls into known runtime functions are declared as **externals** so the lifter can name them and model their arguments instead of chasing into library code:

```cpp
profile::externals::data<x86_reg> external;
profile::externals::emit(external, func_loc, {x86_reg::X86_REG_RCX, x86_reg::X86_REG_RDX}, {}, "UNK_..." , mid);
```

Each `external` records the function's location, its argument and result registers, and a name. 
The lifter special-cases calls to these; see the `registrar`'s `externals` access in **[Writing a Lifter](writing-lifters.md)**.

## Saving and Loading

A built stream can be saved and reloaded (`profile::fs::save` / `load`), so an expensive trace or profile of a binary does not have to be rebuilt every run.

## Where it feeds

The profile is the input to native disassembly and lifting. 
The x86 example builds a `manager`, extracts it, disassembles each instruction with Capstone, wraps the results as `il::vinst`, and calls `X86::lifter::lift(...)` 
with the details and externals. 
From there it is ordinary IL, and the rest of the pipeline is target-independent. 
See **[Writing a Lifter](writing-lifters.md)** for the lifting half and **[Usage](../../usage.md)** for the full usage.
