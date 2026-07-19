---
id: isa
title: Instruction Set Architecture
sidebar_position: 1
---

# Instruction Set Architecture (ISA): Assembly to IL Translation

The **IL** is the architecture-independent instruction set every target lifts into. 
A lifter never emits x86 or Lua opcodes; it emits these, and everything downstream: the IR, every pass, every emitter - only ever sees this. 
That is what lets one pipeline serve unrelated targets: allow a new front-end to produce these opcodes and it inherits the whole optimizer.

This page is the opcode reference. 
It defines the operand notation, the operand sizes, and then every opcode with its meaning and operand layout. 
For how a lifter actually emits them through the DSL, see **[Writing a Lifter](Lifting/writing-lifters.md)**.

# Opcode Table Reference

## Opcode Legend

This legend defines the syntax and notation used for opcodes, hints, and operands within the table.

### Operand Types

* **r??** (Register): Represents a specific CPU or virtual machine register.
* **??** (Kvalue / Constant): A value pulled from the constant pool.
* **??** (Integer / Double): A raw numeric literal value.
* **+/-??** (Jump): Relative instruction offset. When generated, this indicates the number of instructions to skip forward (**+**) or backward (**-**).
* **u??** (Upvalue ID): Unique identifier for a captured upvalue.
* **??** (Global ID): Unique identifier for a global variable.
* **true/false** (Boolean): Explicit boolean literal values.
* **upvalue_kind** (Upvalue Kind): Defines the capture type (see the separate **upvalue_kind** documentation).
* **c??** (Closure ID): Unique identifier for a function closure.
* **??** (Val / Intptr): An integer-sized pointer or generic value holder.
* **??** (Virtual Function): Pointer or identifier for a virtual function.
* **??** (Table Size): Allocated or initial size definition for a table.


### Operand Sizes

The operand size corresponds to the standard C types:

| Notation | Actual Size | Description |
| ------------------  | ------- | --------------------- |
| **sizeof(int8_t)**  | 1 Byte  | 8-bit signed integer  |
| **sizeof(int16_t)** | 2 Bytes | 16-bit signed integer |
| **sizeof(int32_t)** | 4 Bytes | 32-bit signed integer |
| **sizeof(int64_t)** | 8 Bytes | 64-bit signed integer |

## Optable
|Opcode               |Description                                                                    |Operands                                                                                                                         |
|---------------------|-------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
|OP_NOP               |Nothing                                                                        |None                                                                                                                             |
|OP_LOADBOOL          |Loads a boolean to the destination register                                    |Dest(Register), Source(Boolean)                                                                                                  |
|OP_LOADINT           |Loads an integer to the destination register                                   |Dest(Register), Source(Integer)                                                                                                  |
|OP_LOADNONE          |Loads a None to the destination register                                       |Dest(Register)                                                                                                                   |
|OP_LOADKVAL          |Loads a kvalue to the destination register                                     |Dest(Register), Source(Kvalue)                                                                                                   |
|OP_LOADGLOBAL        |Loads a global to the destination register                                     |Dest(Register), Source(GlobalID)                                                                                                 |
|OP_GETTABUPVALUE     |Loads a kvalue from structure to the destination register                      |Dest(Register), Source(Kvalue)                                                                                                   |
|OP_SETGLOBAL         |Set global                                                                     |Source(Register), Dest(GlobalID)                                                                                                 |
|OP_MOVE              |Move                                                                           |Dest(Register), Source(Register)                                                                                                 |
|OP_ADD               |Arith/Bitwise (+)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_SUB               |Arith/Bitwise (-)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_MUL               |Arith/Bitwise (*)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_DIV               |Arith/Bitwise (/)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_MOD               |Arith/Bitwise (%)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_POW               |Arith/Bitwise (^)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_AND               |Arith/Bitwise (&)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_XOR               |Arith/Bitwise (~)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_SHL               |Arith/Bitwise (`<<`)                                                             |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_SHR               |Arith/Bitwise (`>>`)                                                             |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_IDIV              |Arith/Bitwise (//)                                                             |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_OR                |Arith/Bitwise (&#124;)                                                              |Dest(Register), Source(Register), Value(Register)                                                                                |
|OP_ADDK              |Arith/Bitwise Kvalue (+)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_SUBK              |Arith/Bitwise Kvalue (-)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_MULK              |Arith/Bitwise Kvalue (*)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_DIVK              |Arith/Bitwise Kvalue (/)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_MODK              |Arith/Bitwise Kvalue (%)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_POWK              |Arith/Bitwise Kvalue (^)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_ANDK              |Arith/Bitwise Kvalue (&)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_XORK              |Arith/Bitwise Kvalue (~)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_SHLK              |Arith/Bitwise Kvalue (`<<`)                                                      |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_SHRK              |Arith/Bitwise Kvalue (`>>`)                                                      |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_IDIVK             |Arith/Bitwise Kvalue (//)                                                      |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_ORK               |Arith/Bitwise Kvalue (&#124;)                                                       |Dest(Register), Source(Register), Value(Kvalue)                                                                                  |
|OP_ADDN              |Arith/Bitwise Integer (+)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_SUBN              |Arith/Bitwise Integer (-)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_MULN              |Arith/Bitwise Integer (*)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_DIVN              |Arith/Bitwise Integer (/)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_MODN              |Arith/Bitwise Integer (%)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_POWN              |Arith/Bitwise Integer (^)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_ANDN              |Arith/Bitwise Integer (&)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_XORN              |Arith/Bitwise Integer (~)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_SHLN              |Arith/Bitwise Integer (`<<`)                                                     |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_SHRN              |Arith/Bitwise Integer (`>>`)                                                     |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_IDIVN             |Arith/Bitwise Integer (//)                                                     |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_ORN               |Arith/Bitwise Integer (&#124;)                                                      |Dest(Register), Source(Register), Value(Integer)                                                                                 |
|OP_LEN               |Unary (#)                                                                      |Dest(Register), Source(Register)                                                                                                 |
|OP_NOT               |Unary (not)                                                                    |Dest(Register), Source(Register)                                                                                                 |
|OP_MINUS             |Unary (-)                                                                      |Dest(Register), Source(Register)                                                                                                 |
|OP_BITNOT            |Unary (~)                                                                      |Dest(Register), Source(Register)                                                                                                 |
|OP_PLUS              |Unary (+)                                                                      |Dest(Register), Source(Register)                                                                                                 |
|OP_REF               |Unary (&)                                                                      |Dest(Register), Source(Register)                                                                                                 |
|OP_CCALL             |C function and closure call                                                    |Dest(Register), Arguement count(Val), Return count(Val)                                                                          |
|OP_VCALL             |Virtual function call (clears virtual function arg stack once executed)        |Dest(Register), Virtual function(ID), Arg count(Val), Return count(Val)                                                          |
|OP_VPUSH             |Pushes arguement to virtual function stack                                     |Dest(Register)                                                                                                                   |
|OP_SELF              |Loads function in table to a register                                          |Dest(Register), Source(Register), Kvalue(Kvalue)                                                                                 |
|OP_RETURN            |Return from a function                                                         |Start register(Register), Return count(Val)                                                                                      |
|OP_CONCAT            |String concatation                                                             |Dest(Register), Start register(Val), End register(Val)                                                                           |
|OP_JUMP              |Jump to a address                                                              |Target(Jump)                                                                                                                     |
|OP_CMP               |Writes two compare registers to a cmp flag                                     |Compare(Register), Compare(Register)                                                                                             |
|OP_CMPK              |Writes compare register and a kvalue to cmp flag                               |Compare(Register), Compare(Kvalue)                                                                                               |
|OP_CMPN              |Writes compare register and an integer to cmp flag                             |Compare(Register), Compare(Val)                                                                                                  |
|OP_CMPB              |Writes compare register and a boolean to cmp flag                              |Compare(Register), Compare(Boolean)                                                                                              |
|OP_CMPNONE           |Writes compare register and a none object to cmp flag                          |Compare(Register)                                                                                                                |
|OP_CMPS              |Writes one compare register to cmp flag                                        |Compare(Register)                                                                                                                |
|OP_CMPSK             |Writes one compare kvalue to cmp flag                                          |Compare(Kvalue)                                                                                                                  |
|OP_CMPSN             |Writes one compare integer to cmp flag                                         |Compare(Val)                                                                                                                     |
|OP_CMPSNONE          |Writes one compare none object to cmp flag                                     |None                                                                                                                             |
|OP_JUMPIF            |Jump if cmp flag                                                               |Jump address(jump)                                                                                                               |
|OP_JUMPIFNOT         |Jump if not cmp flag                                                           |Jump address(jump)                                                                                                               |
|OP_JUMPIFEQUAL       |Jump if == comparative to cmp flag                                             |Jump address(jump)                                                                                                               |
|OP_JUMPIFNOTEQUAL    |Jump if != comparative to cmp flag                                             |Jump address(jump)                                                                                                               |
|OP_JUMPIFLESS        |Jump if `<` comparative to cmp flag                                              |Jump address(jump)                                                                                                               |
|OP_JUMPIFLESSEQUAL   |Jump if `<=` comparative to cmp flag                                             |Jump address(jump)                                                                                                               |
|OP_JUMPIFGREATER     |Jump if `>` comparative to cmp flag                                              |Jump address(jump)                                                                                                               |
|OP_JUMPIFGREATEREQUAL|Jump if `>=` comparative to cmp flag                                             |Jump address(jump)                                                                                                               |
|OP_SETIF             |Set true or false if cmp flag                                                  |Dest(Register)                                                                                                                   |
|OP_SETIFNOT          |Set true or false if not cmp flag                                              |Dest(Register)                                                                                                                   |
|OP_SETIFEQUAL        |Set true or false if == comparative to cmp flag                                |Dest(Register)                                                                                                                   |
|OP_SETIFNOTEQUAL     |Set true or false if != comparative to cmp flag                                |Dest(Register)                                                                                                                   |
|OP_SETIFLESS         |Set true or false if `<` comparative to cmp flag                                 |Dest(Register)                                                                                                                   |
|OP_SETIFLESSEQUAL    |Set true or false if `<=` comparative to cmp flag                                |Dest(Register)                                                                                                                   |
|OP_SETIFGREATER      |Set true or false if `>` comparative to cmp flag                                 |Dest(Register)                                                                                                                   |
|OP_SETIFGREATEREQUAL |Set true or false if `>=` comparative to cmp flag                                |Dest(Register)                                                                                                                   |
|OP_SETUPVALUE        |Set Upvalue                                                                    |Source(Register), Upvalue ID(UpvalueID)                                                                                          |
|OP_GETUPVALUE        |Get Upvalue                                                                    |Dest(Register), Upvalue ID(UpvalueID)                                                                                            |
|OP_DESTROYUPVALUES   |Destroy all upvalues with target                                               |None                                                                                                                             |
|OP_DESTROYUPVALUESA  |Destroy upvalues with target                                                   |Start(Register)                                                                                                                  |
|OP_ADDUPVALUE        |Sets upvalue for prev closure instruction (Follows closure instruction)        |Type(UpvalueKind), Source(Register)                                                                                              |
|OP_INIT              |Inits                                                                          |None                                                                                                                             |
|OP_BITCAST           |Cast register to n amounts of bits                                             |Dest(Register), Source(Register), Bits(Val u16), precision(Val u8), Unsigned(Boolean)                                            |
|OP_GETVARIADIC       |Gets variadics                                                                 |Dest(Register), Amount(Val)                                                                                                      |
|OP_SETTABLE          |Set table                                                                      |Source(Register), Table(Register), Index(Register)                                                                               |
|OP_GETTABLE          |Get table                                                                      |Dest(Register), Table(Register), Index(Register)                                                                                 |
|OP_SETTABLEN         |Set table                                                                      |Source(Register), Table(Register), Index(Val)                                                                                    |
|OP_GETTABLEN         |Get table                                                                      |Dest(Register), Table(Register), Index(Val)                                                                                      |
|OP_SETTABLEK         |Set table                                                                      |Source(Register), Table(Register), Index(Kvalue)                                                                                 |
|OP_GETTABLEK         |Get table                                                                      |Dest(Register), Table(Register), Index(Kvalue)                                                                                   |
|OP_INITFORLOOPN      |Inits numeric for loop                                                         |Dest(Register), Maximum/target value register(Register), Incrementation value register(Register), Loop iteration location(Jump)  |
|OP_INITFORLOOPG      |Inits generic for loop                                                         |Start value register(Register)(+3), Variable count (Value), Loop iteration location(Jump)                                        |
|OP_INITFORLOOPSPECIAL|Inits abstract special generic for loop                                        |Dest/Start(Register), End(Register), Step(Register), Variables(Registers), Loop iteration location(Jump)                         |
|OP_NEWCLOSURE        |Creates new closure                                                            |Dest(Register), Closure ID(ClosureID)                                                                                            |
|OP_REFCLOSURE        |Gets closure from kvalue                                                       |Dest(Register), Closure(Kvalue)                                                                                                  |
|OP_NEWTABLE          |Creates new table                                                              |Dest(Register), Exact table size(Table_Size), Exact array size(Table_Size)                                                       |
|OP_REFTABLE          |Gets table from kvalue                                                         |Dest(Register), Table(Kvalue) (Exact size, node)                                                                                 |
|OP_NEWTABLEA         |Creates new table                                                              |Dest(Register), Approximate table size(Table_Size), Approximate array size(Table_Size)                                           |
|OP_REFTABLEA         |Gets table from kvalue                                                         |Dest(Register), Table(Kvalue) (Approximate size, node)                                                                           |
|OP_SETLIST           |Appends elements in a table                                                    |Dest table(Register), Start register(Register), Exact table size(Val), Index(Val)                                                |
|OP_FORLOOPG          |For loop generic                                                               |Start register(Register), Loop variable count(Val), Jump back(Jump)                                                              |
|OP_FORLOOPN          |For loop numeric                                                               |Start value register(Register), Maximum/target value register(Register), Incrementation value register(Register), Jump back(Jump)|
|OP_POPTOP            |Pops register from top of the stack                                            |None                                                                                                                             |
|OP_POPARG            |Adds argument to poparg flag to get popped when next OP_CALL instruction is hit|Ignore Register(Register)                                                                                                        |
|OP_MEMSET            |Sets memory                                                                    |Target(Register), Source(Register), Bits(Val u16)                                                                                |
|OP_MEMREAD           |Reads memory                                                                   |Dest(Register), Source(Register), Bits(Val u16)                                                                                  |
|OP_SETFLAG           |Appends flag to flag stack                                                     |Flags ID ENUM(Val)                                                                                                               |
|OP_SALLOC            |Stack allocates n bytes                                                        |Dest(Register), Bytes(Val)                                                                                                       |
|OP_GETSTACK          |Gets stack pointer                                                             |Dest(Register), ID(Val)                                                                                                          |
|OP_SETSTACK          |Set stack pointer                                                              |Dest(Register), ID(Val)                                                                                                          |
|OP_STACKPUSH         |Pushes register contents to the given stack                                    |Stack ID(VAL), Source(Register)                                                                                                  |
|OP_STACKPOP          |Pops register contents from the given stack                                    |Stack ID(VAL), Source(Register)                                                                                                  |
|OP_POPTOPSTACK       |Pops top from given stack                                                      |Stack pointer(Register)                                                                                                          |
|OP_CLOGIC_AND        |Performs condition logic if sources are truthy puts result in dest             |Dest(Register), Source(Register), Source(Register)                                                                               |
|OP_CLOGIC_OR         |Performs condition logic if either sources are truthy puts result in dest      |Dest(Register), Source(Register), Source(Register)                                                                               |
|OP_PEND              |Psuedo-instruction (Pending analysis)                                          |Userdata(Register), * Userdata(Val), * Userdata(Val), * Userdata(Val)                                                            |
|OP_MARK              |Psuedo-instruction (Marks spot)                                                |None                                                                                                                             |
|OP_MOBJ_CAST         |Casts register to object from object map                                       |Dest(Register), Source(Register), Index(Val)                                                                                     |
|OP_NCTOR_MOBJ        |See if source is object from object map if not construct                       |Dest(Register), Source(Register), Index(Val)                                                                                     |
|OP_SCALL             |Call to symbol table                                                           |ID(Val)                                                                                                                          |
|OP_FLAGSET           |Set flag with source                                                           |ID(Val), Source(Reg)                                                                                                             |
|OP_FLAGREAD          |Read flag to dest                                                              |Dest(Register), Source(Val)                                                                                                      |
|OP_CREATE_STACK      |Creates new stack (overrides current)                                          |Dest(Register)                                                                                                                   |
|OP_PRETURN           |Return from page function                                                      |Valid Flag, Jump target (Register), Jump page ID (Integral)                                                                      |
|OP_STARTPAGEFUNC     |Starts page function, not effected by isolate                                  |ID(Val)                                                                                                                          |
|OP_ENDPAGEFUNC       |Ends page function, not effected by isolate                                    |ID(Val)                                                                                                                          |
|OP_PCALL             |Call to page function                                                          |ID(Val), Reg(Register), Entry(Val)                                                                                               |
|OP_PJUMP             |Jump to page                                                                   |ID(Val)                                                                                                                          |
|OP_SEGREGATE         |Next instruction any native flags set will be offset                           |OFFSET(Val)                                                                                                                      |
|OP_COMBINE           |Psuedo-instruction (Flag to combine data from previous instruction to next)    |None                                                                                                                             |
|OP_FLAGJUMP          |If flag (ID) is true jump to page                                              |ID(Val), ID(Val)                                                                                                                 |
|OP_ICALL             |If register(reg) == ID(Val) call                                               |Reg(Register), ID(Val)                                                                                                           |
|OP_TAG_START         |Set tag                                                                        |Name(Kvalue)                                                                                                                     |
|OP_TAG_KV            |Set next tag key value pair                                                    |Key(Kvalue), Value(Kvalue)                                                                                                       |
|OP_TAG_END           |End current tag                                                                |None                                                                                                                             |
|OP_METADATA          |Meta data, user handled instruction                                            |Name(Kvalue), Data(Kvalue)                                                                                                       |
|OP_ENTRY_POINT       |Entry point                                                                    |None                                                                                                                             |
|OP_COMMAND           |Internal command refer to command table                                        |Command (Kvalue), Arg count (Val)                                                                                                |
|OP_BITREAD           |Read bits (Index starts at 0)[MIN, MAX] and interpret result                   |Dest(Register), Source(Register), Min(Register), Max(Register), Cast_bits(Val u16), Unsigned(Boolean)                            |
|OP_BITWRITE          |Writes bits (Index starts at 0)[MIN, MAX] to the first source                  |Dest(Register), Lvalue(Register), Source(Register), Min(Register), Max(Register), Cast_bits(Val u16), Unsigned(Boolean)          |
|OP_BITWRITEA         |Writes bits assign (Index starts at 0)[MIN, MAX] to the Dest buffer            |Dest(Register), Source(Register), Min(Register), Max(Register)                                                                   |
|OP_ANNOTATE_PREV     |Implement annotation to previous instruction                                   |Str (Kvalue)                                                                                                                     |
|OP_AMT               |Amount                                                                         |None                                                                                                                             |
