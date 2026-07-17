---
id: emitters
title: Emitters
sidebar_position: 1
---

# Emitters

The last stage!!!
Once the IR is optimized it is still `ir_stat` not text.
The emitter walks that tree and prints a target high-level language. 
Because everything upstream is language-independent, adding an output language is a self-contained job: 
write the language-specific pieces, add one case to the dispatch, and every target can now decompile to it.

## Api Call

A whole emit is a single call:

```cpp
std::string generate(const emitter::syntax::emitter_syntax syn,
                     const ir_stat::space &code,
                     const std::shared_ptr<ir::data::format::format> &format,
                     const bool use_annotations = true);
```

* `syn` - picks the language
* `code` - is the optimized IR from  `lift` 
* `format` - transforms the text (indentation, spacing, and naming see **[Customization -> formatting](../Intermediate-Representation/customization.md#formatting)**).
* `use_annotations` - emits any comments

It returns IR representation as a string in given syntax.

## emitter_syntax

The languages the dispatch knows about, from `generation_syntax.hpp`:

```cpp
enum class emitter_syntax : std::uint8_t {
      nothing, python, rust, ...
};
```

`emitter_syntax_str(syn)` maps each to its versioned name (`Cpp-23`, `Lua-5.4`, `Python-3.11.0`, etc). 
This enum is the **emitter_syntax dispatch** the rest of the framework refers to: `generate` switches on it to select signatures and per-language behavior.

## How an emitter is structured

Every language lives under `framework/ir/code/generation/lang/<language>/` and is assembled from small, single-purpose emitters. 

* **langkeywords.hpp** - the language's vocabulary as macros: operators, parentheses, indexing, keywords, maps `arith_add` to `"+"`, `pow` to `"std::pow", etc. 
* **support.hpp** - the language's capability declarations. For example `supported_arith_assignment` is the set of ops the language can write as compound assignment (`+=`, `<<=`, ...); leave it empty if every assignment form is supported. The generator consults this to decide whether to fold `a = a + b` into `a += b`.
* **generate/emit.hpp** - the include hub that pulls the language's units together.
* **generate/statement/** - one emitter per statement shape it handles (`assignment`, `arith`, `comment`, ...).
* **generate/expression/** - one emitter per expression shape (`arith`, `call`, `logical`, `table`, `ternary`, `unary`, `memory`, ...).
* **generate/constant/** - the structural constructs (`branch`, `for`, `loop`, `return`, `pages`, ...).
* **generate/line/** - line-level concerns: semicolons, line breaks, indentation.

Each emitter is a small function that appends to a `std::string &buffer`, pulling its punctuation from **langkeywords.hpp** and its spacing from the **format**. 
It never inspects raw bytes or IL; it only ever sees IR.

## Common Generator

Most IR emit almost identically across languages, so shared logic lives under **generation/common/generate/** and the per-language files only override what actually differs. 
A language that has no ternary simply does not provide one; a language that spells `and` as `&&` says so in its keywords and inherits the rest. 
The registry of which languages exist is **generation/common/generate/supported.hpp**, which includes each language's `common.hpp`.

## Adding an output language

1. Create `framework/ir/code/generation/lang/<language>/` with `langkeywords.hpp`, `support.hpp`, `common.hpp`, and a `generate/` tree.
2. Fill **langkeywords.hpp** with the language's operators and punctuation; this is the bulk of the work.
3. Declare support in **support.hpp** (which assignment forms it supports, and so on).
4. Provide only the statement/expression/constant emitters that differ from the common layer; inherit the rest.
5. Add the language to the **emitter_syntax** enum and add its case to the dispatch in `generate`.
6. Register it in **supported.hpp**.

Nothing in the IR, the passes, or any lifter changes. 
The emitter is the only thing that knows the target language exists, exactly as the lifter is the only thing that knows the source architecture exists. 
See **[Contributing -> adding a target](../contributing.md#adding-a-target)**
