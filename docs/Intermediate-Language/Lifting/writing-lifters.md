---
id: writing-lifters
title: Writing a Lifter
sidebar_position: 2
---

# Writing a Lifter

Complete guide to writing a semantic lifter, the code that turns one decoded instruction into IL.
Luramas gives lifter writers a small expression DSL that reads like the CPU manual pseudocode, so the handler for example `ADC` looks almost exactly like the Intel description of `ADC`.

> **Keep specific architecture imports isolated to there own disassembler/lifter to avoid polution and conflicts!!!!**

## Mental model

Each handler receives a **registrar** the machine state: registers, flags, the builder, plus the instruction's operands, and describes the effect using ordinary C++ operators. 
Every operator emits IL underneath, so writing `dest = dest + src + carry` emits the adds, the store, and the width casts for you. 
You almost never call `generate_opcode` directly; that is the raw data, and the **build::expr** DSL already utalizes it.

## Two building blocks

* **build** - The emission manager. It knows the current PC, the target `ilang`, and a register allocator. Every `make_*` on it appends IL.
* **build::expr** - One value in the DSL. Wraps a register, an integral/integer, a double, a global, memory, a flag, or a stack slot. Operators on `expr` emit IL and return a new `expr`.

### Example
Construct an `expr` from the builder plus a source:
```cpp
build::expr r  = b.make_reg(reg_id);              /* register value          */
build::expr n  = build::expr(b_ptr, 7);           /* integral/integer        */
build::expr g  = build::expr(b_ptr, "global");    /* global                  */
build::expr t  = b.make_temp(r);                  /* copy of r               */
```

## Operator DSL

`build::expr` overloads nearly every C++ operator. Each one emits the matching IL and returns the result value.

* **Arithmetic** - `+ - * / % & | ^ << >>` between two `expr`, or between an `expr` and a plain integer in either order, so `7 * x` and `x * 7` both work
* **Comparison** - `== != < <= > >=`, and `.cmp(other)` for the compare flag
* **Unary** - `~ ! -` (bitwise not, logical not, negate) and unary `+`
* **Compound assignment** - `= += -= *= /= %= &= |= ^= <<= >>=`, and `++ --`
* **Logical** - `&&` and `||`
* **Indexing** - `expr[offset]` for memory addressing

### Example

Full add-with-carry is one line:
```cpp
void ADC(const registrar &registrar, const std::vector<build::expr> &operands) {
      const auto dest = operands.front();
      const auto src  = operands.back();
      dest = dest + src + FCF;   /* FCF is the carry flag; store, adds, casts all emit here */
}
```

## Registers and flags

Register and flag access is wrapped in macros so handlers read like assembly. 
Under the hood these call `registrar.getr<R>()` / `getf<F>()`, which return a `build::expr` bound to that register or flag.
Assigning to one emits the write; reading one in an expression emits the read. 
The **registrar** is a template on the target's register and flag enums, so `getr<X86_REG_RCX>()` is type-checked against the Capstone register set.

### Example
The x86 lifter defines these in `vm_helpers.hpp` on top of the **registrar**'s typed accessors:

```cpp
#define REG_AL registrar.getr<x86_reg::X86_REG_AL>()
#define REG_AX registrar.getr<x86_reg::X86_REG_AX>()
#define FCF    /* carry-flag accessor */
#define FAF    /* auxiliary-flag accessor */
```

## Control flow in a handler

The DSL provides structured control flow through keyword macros (`keywords.hpp`), so a handler emits `if`/`else` without hand-managing labels and scopes

### Example
```cpp
kif(((REG_AL & 0x0F) + (FAF << 3)) > 1u) {
      REG_AX += 0x106;
      FAF = 1u;
      FCF = 1u;
}
kelse;
{
      FAF = 0u;
      FCF = 0u;
}
kend;
```

`kif` expands to `build->cmp(cond)` followed by `make_scope<et_>(...)`; `kelse` is `make_else()`; `kend` is `close_scope()`. The primitives, if you need them directly:

* **`make_scope<bin_kind>()`** / **`close_scope()`** - open and close a conditional scope
* **make_else()** - begin an else branch
* **make_label(id)** / **make_goto(id)** - label and jump for loops
* **cmp(expr)** - emit the compare the scope tests. `kwhile` / `kwhile_end` wrap label + cmp + scope + goto

## Casts and bit width

Width is a first-class property. Every `expr` carries a size, and the builder inserts casts when widths meet:

* **cast(bits, unsign, precision)** / **cast(underlying_type)** - cast to a width or named type.
* **memread(bits)** - read memory at this address as a width.
* **read(min, max)** / **write(min, max, value)** - read or write the bit range `[min, max]`.

## Calls, memory, stack, pages

High level handlers are also available, they get inlined as IL:

* **make_call(func, args, results)** - a named call.
* **make_lura_built_in(f, args, results)** - a call to a Luramas builtin (the `klura_call` macro wraps this).
* **make_push(expr, id)** / **make_pop(expr, id)** - stack operations.
* **make_page(id)** / **close_page()** / **page_call(...)** / **page_jump(...)** / **make_goto(mid, loc)** - the page model for native code, where functions are pages. `non_direct_*` variants exist for computed targets.
* **make_bitread / make_bitwrite** - explicit bit-field access.

## Handler dispatch

Handlers are registered per mnemonic and share one signature:

```cpp
void MNEMONIC(const registrar &registrar, const std::vector<build::expr> &operands);
```

`vm::parse` routes a decoded instruction to its handler. 
The `registrar` also mains `get_details()` (the original bytes and any SMC discrepancies) and `externals` (declared external functions), so a handler can special-case calls into known runtime functions.

### Example

Rotate-left, start to finish, reflects Intel Software Developer Manual:

```cpp
void ROL(const registrar &registrar, const std::vector<build::expr>; &operands) {
      auto dest  = operands.front();
      auto count = operands.back();
      auto bits  = dest.bits();

      dest = (dest << count) | (dest >> (bits - count));   /* rotate */
      /* flag effects emitted via FCF / FOF as the ISA specifies */
	  return;
}
```

Because each operator emits IL, the handler is both the specification and the implementation. 
That is the design goal: an author writes what the instruction **means**, and the [IR](../../Intermediate-Representation/overview.md) figures out what it **was** without hidden side-effects.

## Bytecode lifters

Bytecode targets use the same `ilang` output but a simpler front: they walk a `Proto` and map each VM opcode to an IL opcode through `il::emitter::generate_opcode<OP>` directly, with no flag modeling.
Structure the target as a **parser** (opcode mapping) plus a **resolver** (upvalue/closure fixups) to allow for Lua inspired native abstraction. 
