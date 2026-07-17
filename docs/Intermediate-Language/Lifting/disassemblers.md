---
id: disassemblers
title: Disassemblers
sidebar_position: 2
---

# Disassemblers

Target disassemblers and how it works.
Every target under **framework/disassembler/** follows the same scheme: an **optable**, an **operand**, and a **disassembly**.

> **Keep specific architecture imports isolated to there own disassembler/lifter to avoid polution and conflicts!!!!**

## What a disassembler is

Decodes bytes into a structured instruction and details which meaning is assigned later, by the lifter. 
Keeping decode and semantics apart is what lets multiple unrelated targets share one downstream pipeline.

## Three fundamental structures

Each target defines the same structure in its own namespace.

* **optable** - The opcode table. Enumerates every opcode with its operand layout, mnemonic, and hint. On register VMs (Lua, Luau, etc) it also encodes each operand's **type**: register, constant index, jump offset, or upvalue. Constant tables use **frozen** where possible so the optable is resolved at compile time.
* **operand** - One decoded operand. Holds the operand kind plus a value union - register id, immediate, jump target, upvalue index, constant-pool index. The union differs per target because operand spaces differ per target.
* **disassembly** - One decoded instruction: address, opcode, mnemonic, hint, operand list, length, and any cross-reference (**ref_addr**) into another instruction.

## How decoding works

Decoding fills a **disassembly** node per instruction and, where the opcode references another location (a jump, a constant), resolves that reference into the node. 
The Lua 5.3.6 operand shows the pattern, a constant operand carries both its pool index and a printable/import string, so downstream stages can name it without re-reading the pool:

```cpp
struct operand {
      op_table::operands operand;
      op_table::type     type;

      union {                       /* exactly one of these is meaningful, per `type` */
            luramas_register  reg;
            std::intptr_t     val;
            std::intptr_t     jmp;
            luramas_address   upvalue;
            luramas_address   table_size;
            std::size_t       k_idx;
      };

      luramas_address ref_addr     = 0u;      /* resolved cross-reference target      */
      std::string     k_value      = "";      /* constant as string; also import name */
      std::uint8_t    k_value_type = LUA_TNIL;
};

struct disassembly {
      luramas_address addr = 0u;
      op_code          op;
      const char     *mnenomic = "";
      const char     *hint     = "";
      std::string     data     = "";
      std::uint8_t    len      = 0u;   /* encoded length, so the decoder can advance */
};
```

## Scaling

The **disassembly** struct carries only what the target has. 
A CISC target like x86 uses Capstone and produces operands; a stack VM like the JVM carries almost nothing, since its operands live implicitly on the stack:

```cpp
struct operand {};                     /* JVM operands are implicit on the stack */

struct disassembly {
      optable::data::instructions op;
      const char           *mnenomic = "";
      const char           *hint     = "";
      std::vector<operand>  operands;
};
```

Both use the same idea into the lifter. That is the point: the decoder does the minimum the target requires, and the IL never learns the target's opcode names.

## Adding a Disassembler

**If you are using an already battle tested disassembler skip this step and just import it.**

1. Create `framework/disassembler/<target>/` with **optable.hpp**, **disassembler.hpp**, **disassembler.cpp**.
2. Enumerate opcodes in the optable - one entry per instruction, with operand layout and mnemonic.
3. Define **operand** and **disassembly** to match the target's operand space, no more fields than it actually has.
4. Decode into those structs. **DO NOT LIFT HERE!**

## Writing Lifters
After disassembling it needs to be lifted to the IL: `framework/il/lifter/langs/<target>/`. 
See **[Writing a Lifter](writing-lifters.md)**.
