---
id: building
title: Building Luramas
sidebar_position: 2
---

## How to build Luramas

This page covers everything needed to compile Luramas from source: the required toolchain, the third-party dependencies and how they are resolved, 
and the build commands with the compile-time flags that select which targets end up in the binary. 
**The first build only needs a C++20 compiler, CMake, and Conan.**

## Toolchain

Luramas is C++20, built with CMake, and use Conan package manager. 

* **Compiler** - Any C++20 compiler. MSVC needs the WinRT/UWP default libs stripped, which the top-level **CMakeLists.txt** already does through **/NODEFAULTLIB:vccorlib**.
* **CMake** - 3.16 or newer.
* **Conan** - Resolves dependencies through **conan_provider.cmake**, marked in as **CMAKE_PROJECT_TOP_LEVEL_INCLUDES**, so the first configure pulls everything.

## Dependencies

All third-party libraries are declared in **conanfile.txt** and fetched automatically; only the VM sources under **3rdparty/** (Lua, Luau) are vendored.

* **boost 1.86.0** - Containers everywhere. **unordered_flat_map** / **unordered_flat_set** are the default types across the framework.
* **capstone 5.0.1** - Native disassembly backend for the x86-64 lifter.
* **gmp 6.3.0** and **mpfr 4.2.1** - Arbitrary-precision integer and float math, so constant folding never silently loses bits on wide values.
* **frozen 1.2.0** - Compile-time constant maps for opcode tables.
* **rapidjson cci.20230929** - Serialization and formatting profiles.
* **hunspell 1.7.2** - Spell-aware heuristics for symbol and variable naming.

## Building
```bash
cmake -S . -B build
cmake --build build --config Release
```


## Macros


## Non-Debug Flags

These flags are used for production and development builds to strip out unused architectures, define specific version, or enable features.

### Targets

* **LURAMAS_TARGET_X86** - Enabled x86-64 assembly lifters and disassemblers
* **LURAMAS_TARGET_LUA** - Enables Lua lifters and disassemblers
	* **LURAMAS_TARGET_VERSION_536** - Targets Lua version 5.3.6 bytecode
* **LURAMAS_TARGET_LUAU** - Enables Luau lifters and disassemblers
	* **LURAMAS_TARGET_VERSION_6** - Targets LuaU Version 6 bytecode.

### Other Flags

* **LURAMAS_ECC** - Enables internal error correction checks make sure things have no obvious structural errors (e.g. ifs with no ends)
* **LURAMAS_PROFILE** - Enable time checks on passes


## Debug Flags

These flags should typically be reserved for local development and debugging environments. 

* **DEBUG** - Basic debuggable options like assertions, debug prints, etc
* **LURAMAS_DEBUG_SSA_STEPS** - Prints each step of SSA to debug
* **LURAMAS_DEBUG_NO_DUMP** - Overrides flaggable debugging to disallow dumps
* **LURAMAS_DEBUG_ERROR_CLIPBOARD** - Mostly user implemented, copies dumped data to clipboard see LURAMAS_TARGET_BIGDATA flag
* **LURAMAS_TARGET_BIGDATA** - If data is big it dumps it all into LURMAS_GLOBAL_PDUMP buffer