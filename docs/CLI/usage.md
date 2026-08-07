---
id: usage
title: Usage
sidebar_position: 1
---

# CLI

The example `main.cpp` uses [CLI11](https://github.com/CLIUtils/CLI11) as its front end, so a build can be pointed at a file and a target:

```
Luramas -i <input> [-t <target>] [-b]
```

Only `-i` is required. Everything else has a default.

## Options

* `-i`, `--input <path>` - input file. Required. Treated as source unless `-b` is passed.
* `-t`, `--target <id>` - target architecture / VM version. Defaults to `x86`.
* `-b`, `--bytecode` - treat the input as bytecode rather than source. Off by default.
* `-test <directory>` - run the test suite from this root test directory. The directory must exist.
* `-all-tests` - run the test suite across **all** supported target versions instead of just the compiled-in one. Off by default.
* `-h`, `--help`, `?` - print the help message (including the accepted targets) and exit.

## Targets

### Compiled
* `X86` - x86 machine code

### Lua
* `Lua-V51` - Lua 5.1.xx VM bytecode/source
* `Lua-V52` - Lua 5.2.xx VM bytecode/source
* `Lua-V53` - Lua 5.3.xx VM bytecode/source
* `Lua-V54` - Lua 5.4.xx VM bytecode/source
* `Lua-V55` - Lua 5.5.xx VM bytecode/source

### LuaU
* `LuaU-V1` - Luau VM bytecode/source version 1
* `LuaU-V2` - Luau VM bytecode/source version 2
* `LuaU-V3` - Luau VM bytecode/source version 3
* `LuaU-V4` - Luau VM bytecode/source version 4
* `LuaU-V5` - Luau VM bytecode/source version 5
* `LuaU-V6` - Luau VM bytecode/source version 6
* `LuaU-V7` - Luau VM bytecode/source version 7
* `LuaU-V8` - Luau VM bytecode/source version 8
* `LuaU-V9` - Luau VM bytecode/source version 9
* `LuaU-V10` - Luau VM bytecode/source version 10
* `LuaU-V11` - Luau VM bytecode/source version 11
* `LuaU-V12` - Luau VM bytecode/source version 12

`-t` is validated against the full list of targets Luramas knows about, not against the ones your binary actually contains. 
Which targets get compiled in is decided at compile time by the `LURAMAS_TARGET_*` macros see **[Supported Targets](../supported-targets)**.

### Example Lua 5.3.xx

Decompile a Lua 5.3.xx source file:

```
Luramas -i script.lua -t Lua-V53
```

Output is written to stdout.

### Exit codes

* `0` - success
* `1` - the input file could not be opened or read

Argument parsing errors are handled by CLI11 and exit with its own status before any work starts.

## Tests

`-test` points the runner at a test root; it walks the per-version script directories underneath it and decompiles every script against the matching target.
 Extensions are per language (`.lua` for the Lua targets, `.luau` for Luau, etc).

```
Luramas -test C:/Luramas/tests
```

By default only versions compiled into the binary are used. `-all-tests` widens that to every supported version:

```
Luramas -test ./tests -all-tests
```

## See also

* **[Building](../framework/building)** - Building Luramas
* **[Program metadata](./program-metadata.md)** - What the decompiler needs for an input program
