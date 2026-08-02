---
id: supported-targets
title: Supported Targets
sidebar_position: 8
---

A *target* is an architecture or VM bytecode version that Luramas can lift and disassemble. 
Targets are selected at compile time with the `LURAMAS_TARGET_*` macros; the binary only contains the target lifters it was compiled with and then chosen at run time with the CLI's `-t` flag.

## Targets

* **`LURAMAS_TARGET_X86`** - enables the x86-64 assembly lifters and disassemblers.
* **`LURAMAS_TARGET_LUA`** - enables the Lua lifters and disassemblers. Pair with a version macro:
	* **`LURAMAS_TARGET_VERSION_536`** - Lua 5.3.6 bytecode.
* **`LURAMAS_TARGET_LUAU`** - Enables the Luau lifters and disassemblers. Pair with a version macro:
	* **`LURAMAS_TARGET_VERSION_6`** - Luau version 6 bytecode.
	* **`LURAMAS_TARGET_VERSION_12`** - Luau version 12 bytecode.

Some targets dont have verisions but the cases it does one of them need to be enable on the supporting target.
**THESE MACROS NEED TO BE DEFINED TO ENABLE THE TARGET IN LURAMAS** 

## Macro to CLI Target

The `-t` flag takes a short target ID rather than a macro name:

* `X86` - `LURAMAS_TARGET_X86`
* `Lua-V53` - `LURAMAS_TARGET_LUA` + `LURAMAS_TARGET_VERSION_536`
* `LuaU-V6` - `LURAMAS_TARGET_LUAU` + `LURAMAS_TARGET_VERSION_6`
* `LuaU-V12` - `LURAMAS_TARGET_LUAU` + `LURAMAS_TARGET_VERSION_12`

> Note. that `-t` accepts any ID in the list above regardless of how the binary was built. Passing a target that wasn't compiled in parses fine and then fails at run time see **[Usage](../Framework/usage)**.

