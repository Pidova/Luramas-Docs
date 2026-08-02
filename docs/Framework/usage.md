---
id: usage
title: Usage
sidebar_position: 3
---

# Usage

How to actually run a decompilation.
Every target follows the same idea: **lift to IL, bridge to a closure, lift to IR, emit** and the only thing that changes between targets is the lifter. 
This page walks the two ends of the spectrum: a bytecode target, where the front-end is a few lines, and a native target, where you build the instruction stream yourself.

The full sources are under `examples/`.

## Four Stages

Every decompile is the same four calls in a row:

1. **Lift to IL** - a target-specific step that fills an `ilang` buffer.
2. **Bridge** - `closures::gen_closure(il)` turns the buffer into a closure tree. See **[Closures](./Intermediate-Language/closures.md)**.
3. **Lift to IR** - `ir::lift(closure, env_flags)` builds and optimizes the IR. See **[IR Overview](./Intermediate-Representation/overview.md)**.
4. **Emit** - `ir::code::generation::generate(syntax, ir, format)` prints the target language. See **[Emitters](./Code-Generation/emitters.md)**.

Only stage 1 knows the target. Stages 2 to 4 are identical everywhere.

## Example Bytecode Target: Lua 5.3.6

Bytecode is the easy case. The VM hands you a `Proto`, so the front-end is tiny. Compile the source (or load raw bytecode), then run the four stages:

```cpp
std::optional<std::string> luramas::decompile_lua_536(const std::string &code,
								std::shared_ptr<ir::data::format::format> &format,
								const bool bytecode) {

      bool error = false;
      lua_State *ls = nullptr;
      auto proto = compile_script(code, error, ls, bytecode); /* compile to Proto */
      if (error || !proto) {
            lua_close(ls);
            return std::nullopt;
      }

      auto il = il::lifter::lift_proto(proto);                                                           /* lift to IL           */
      auto closure = closures::gen_closure(il);                                                          /* bridge to a closure  */
      const auto ir = ir::lift(closure);                                                                 /* lift to IR           */
      return ir::code::generation::generate(r::code::emitter::syntax::emitter_syntax::lua, ir, format);  /* emit                 */
}
```

Note `ir::lift(closure)` here takes no **environment_flags**: the defaults are correct for a stack VM. A bytecode target rarely needs to configure anything.

## Example Native Target: x86-64

Native is more involved, because there is no `Proto` you need to describe the instruction stream yourself with the **profile** builder, disassemble it, then lift. 
It utalizes [CPU-Tracer](https://github.com/Pidova/CPU-Tracer) to standardize and simplify the disassembly process.
After you generate a graph with [CPU-Tracer](https://github.com/Pidova/CPU-Tracer) it runs it in [CFG-Tools](https://github.com/Pidova/CFG-Tools) to linearize it.
 
### Building the Instruction Stream

Describe the bytes with a `cpu_tracer::blocks::builder::builder`, using labels for control flow (full detail in **[Native Code](./Intermediate-Language/Lifting/native-code.md)**):

```cpp
 profile::externals::data<x86_reg> externals;                                                               /* Externals */
 cpu_tracer::blocks::builder::builder<MAX_LEN, DEFAULT_MODE> b;                                             /* Bytecode builder */
 boost::unordered_flat_set<luramas_address> external_addrs;                                                 /* External addresses */
 boost::unordered_flat_map<luramas_address, boost::unordered_flat_set<luramas_address>> external_addresses; /* External addresses to compile {realpc -> external addr] */

 /* Build assembly */
 {
       /* Build data */
       b.emitd({0xB8, 0x05, 0x00, 0x00, 0x00});                                                                      /* mov eax, 0x5 */
       b.emitd({0xBB, 0x05, 0x00, 0x00, 0x00});                                                                      /* mov ebx, 0x5 */
       b.emitd({0x39, 0xD8});                                                                                        /* cmp eax, ebx */
       const auto i_je_10_rpc = b.emitd({0x74, 0x02}, edges{{0x10, edges_k::next}}, JUMP).first;                     /* je label_10 */
       const auto i_jmp_15_rpc = b.emitd({0xEB, 0x05}, edges{{0x15, edges_k::next}}, JUMP).first;                    /* jmp label_15 */
       const auto label_10 = *b.emit_label(0x10);                                                                    /* label_10: */
       const auto i_call_1e_rpc = b.emitd({0xE8, 0x09, 0x00, 0x00, 0x00}, edges{{0x1E, edges_k::next}}, CALL).first; /* call 1e */
       const auto label_15 = *b.emit_label(0x15);                                                                    /* label_15: */
       b.emitd({0xB8, 0x01, 0x00, 0x00, 0x00});                                                                      /* mov eax, 0x1 */
       b.emitd({0x31, 0xDB});                                                                                        /* xor ebx, ebx */
       b.emitd({0xCD, 0x80});                                                                                        /* int 0x80 */
       const auto label_1E = *b.emit_label(0x1E);                                                                    /* label_1E: */
       external_addresses[b.emitd({0xE8, 0x95, 0x99, 0x92, 0x02}, std::nullopt, CALL).first].insert(0x29299b8);      /* call 29299b8 [EXTERNAL] */
       external_addrs.insert(0x29299b8);                                                                             /* External: 0x29299b8 */
       const auto i_retn_1e_rpc = b.emitd({0xC3}, edges{{0x1E, edges_k::next}}, RETN).first;                         /* ret */
       /* Connect Edges */
       b.connect_edge<edges_k::next>(label_10, i_je_10_rpc);   /* je label_10 -> label_10 */
       b.connect_edge<edges_k::next>(label_15, i_jmp_15_rpc);  /* jmp label_15 -> label_15 */
       b.connect_edge<edges_k::next>(label_1E, i_call_1e_rpc); /* call 1e -> label_1E */
       b.connect_edge<edges_k::next>(label_1E, i_retn_1e_rpc); /* ret -> label_1E */
 }
```

### Disassemble and Lift to IL

Walk execution order, disassemble each instruction with Capstone, and lift the result into an `ilang`:

```cpp
auto buffer = std::make_shared<il::ilang>();
/* ... open Capstone, disassemble each inst in `luramas::profile::analyze::linearize` into pinsts ... */
il::X86::lifter::lift(buffer, pinsts, hw_constants, externals, details);
```

### Configure and Run

Native code has pages(Call/jumpable code regions), memory, and CPU flags, so this is where **environment_flags** gets used:

```cpp
auto closure = closures::gen_closure(buffer);   /* bridge */
closure->flags.fassociated_args = true;

luramas::ir::passes::environment_flags env_flags;
env_flags.fhas_pages       = true;
env_flags.fhas_memory      = true;
env_flags.fhas_types       = true;
env_flags.feliminate_flags = true;
env_flags.fuse_bitwise     = true;
/* ... plus safety, default width/type, and page-call callbacks ... */

return ir::code::generation::generate(ir::code::emitter::syntax::emitter_syntax::cpp, ir::lift(closure, env_flags), format); /* lift and emit */
```

The full x86 example sets a couple dozen flags and the page-call callbacks; see **[Customization](./Intermediate-Representation/customization.md)** for what each one does.

## CLI

More information on CLI usage can be found: [here](../CLI/usage.md)

## Output Language

The last argument to `generate` is the **emitter_syntax**. 
The same optimized IR can be printed as any supported language just by changing it: `cpp`, `lua`, `luau`, `python`, `rust`, etc. 
See **[Emitters](./Code-Generation/emitters.md)** for the full list and **[Customization: formatting](./Intermediate-Representation/customization.md#formatting)** for changing how the output syntax.
