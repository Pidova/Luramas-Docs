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

### Building the Instruction Stream

Describe the bytes with a `profile::builder::manager`, using labels for control flow (full detail in **[Native Code](./Intermediate-Language/Lifting/native-code.md)**):

```cpp
luramas::profile::builder::manager data;
data.emit({0x89, 0xF8});                                                 /* mov eax, edi   */
data.emit({0x74, 0x18}, luramas::profile::inst_kind::jump_to, 1u, true); /* je Label 1     */
data.emit_label(1u);                                                     /* Label 1        */
data.emit({0xC3});                                                       /* ret            */

boost::unordered_flat_map<profile::module_id, profile::inst_result> mid_res;
data.extract(mid_res);
const auto details = profile::analyze::generate_details(mid_res);
```

### Disassemble and Lift to IL

Walk execution order, disassemble each instruction with Capstone, and lift the result into an `ilang`:

```cpp
auto buffer = std::make_shared<il::ilang>();
/* ... open Capstone, disassemble each inst in order_of_execution_organized into pinsts ... */
il::X86::lifter::lift(pinsts, buffer, details, external, il::X86::lifter::bit_mode::x32);
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

The example `main.cpp` wraps these behind a small CLI, so a build can be pointed at a file and a target:

```
Luramas -i <input> -t <target> [-b]
```

* `-i` is the input file
* `-t` selects the target (`x86`, `lua-536`, etc) 
* `-b` treats the input as bytecode rather than source. Which target actually compiles into the binary is decided at build time by the `LURAMAS_TARGET_*` flags - see **[Building](./building.md#macros)**.

`-help` : Shows you supported architectures and describes usage.

## Output Language

The last argument to `generate` is the **emitter_syntax**. 
The same optimized IR can be printed as any supported language just by changing it: `cpp`, `lua`, `luau`, `python`, `rust`, etc. 
See **[Emitters](./Code-Generation/emitters.md)** for the full list and **[Customization: formatting](./Intermediate-Representation/customization.md#formatting)** for changing how the output syntax.
