# <a name="main"></a> AMX Assembly

3rd June 2018

AMX version: 3.x

AMX file version: > 7

This document is an *unofficial* documentation of the assembly of the Abstract Machine eXecutor. 
The contents of this document may not be accurate and are subject to changes. 
Not all the terms defined in the [Terminology section](#terminology) are defined in the official manual but have been introduced for the purpose
of this document as supporting terms to help the reader.

# <a name="index"></a> Index
* [Prerequisites](#prerequisites)
* [Terminology](#terminology)
  * [Abstract Machine eXecutor (AMX)](#term-amx)
  * [AMX Assembly](#term-amx-assembly)
  * [AMX Binary](#term-amx-binary)
  * [AMX Instance](#term-amx-instance)
  * [Host Program](#term-host-program)
  * [Extension Module](#term-extension-module)
* [Basic Concepts](#basic-concepts)
  * [cell concept](#concept-cell)
  * [Memory Addresses](#concept-memory-addresses)
  * [Stack & Heap](#concept-stack-heap)
  * [Instructions](#concept-instructions)
  * [Stored program concept and von Neumann architecture](#concept-spc-vna)
  * [Registers](#concept-registers)
* [File and memory layout](#file-memory-layout)
* [Instruction Set](#instruction-set)
  * [Pawn Inline Assembly Syntax](#pawn-inline-assembly)
  * [Instruction Table](#instruction-table)
  * [Load/Store Instructions](#loadstore-instructions)
  * [Indexing Instructions](#indexing-instructions)
  * [Arithmetic Instructions](#arithmetic-instructions)
  * [Logical Instructions](#logical-instructions)
  * [Relational Instructions](#relational-instructions)
  * [Stack Manipulation Instructions](#stack-manipulation-instructions)
  * [Heap Instructions](#heap-instructions)
  * [Control Register Manipulation Instructions](#control-register-manipulation-instructions)
  * [Control Flow Instructions](#control-flow-instructions)
  * [Switch-case Instructions](#switch-case-instructions)
* [Call stack and calling convention](#call-stack-and-call-convention)
  * [Local variables and function frame](#local-variables-and-function-frame)
  * [Call convention](#call-convention)
  * [Methods of passing arguments](#methods-passing-arguments)
  * [System Requests](#system-requests)
* [Multi-dimensional arrays](#multi-dimensional-arrays)
  
# <a name="prerequisites"></a> Prerequisites
- Pawn Scripting Language
- [Binary Number System](https://en.wikipedia.org/wiki/Binary_number)
- [Hexadecimal Number System](https://en.wikipedia.org/wiki/Hexadecimal)
- Memory Addresses (understanding [pointers](https://en.wikipedia.org/wiki/Pointer_(computer_programming)) is a plus)
- [Stack (as an abstract data type)](https://en.wikipedia.org/wiki/Stack_(abstract_data_type))

# <a name="terminology"></a> Terminology
 <a name="term-amx"></a>**Abstract Machine eXecutor (AMX)**: AMX is an [abstract machine](https://en.wikipedia.org/wiki/Abstract_machine) (or a [virtual machine](https://en.wikipedia.org/wiki/Virtual_machine)). 
 
 <a name="term-amx-assembly"></a>**AMX Assembly**: The [assembly language](https://en.wikipedia.org/wiki/Assembly_language) for AMX.
 
 <a name="term-amx-binary"></a>**AMX Binary**: AMX [binary](https://en.wikipedia.org/wiki/Executable) is the executable file (usually have the `.amx` extension) that is produced after compiling. *AMX program can refer to AMX binary in some contexts.*
 
 <a name="term-amx-instance"></a>**AMX Instance**: Every AMX binary [loaded](https://en.wikipedia.org/wiki/Loader_(computing)) exists independently. The loaded program is known as AMX Instance. *AMX program can refer to AMX instance in some contexts.*
 
 <a name="term-host-program"></a>**Host Program**: The program which embeds the AMX is known as the host program.
 In the case of [San Andreas Multiplayer (SA-MP)](http://sa-mp.com/), the SA-MP server is the host program.
 
 <a name="term-extension-module"></a>**Extension Module**: An extension module is an independent module linked (usually dynamically linked) to the host program which
 provides additional native functions. This is commonly known as a plugin.
 
# <a name="basic-concepts"></a> Basic Concepts

* ##  <a name="concept-cell"></a> cell concept

  Pawn is a typeless language: there are no data types. All the data is stored as raw data in a cell or in a collection of cells. 
  
  ```pawn
  new a = 5, b = 'A';
  ```

  The above code creates two separate cells each identified by `a` and `b`. The cell `a` will contain the value 5 and the cell `b` will contain the value 65: the ASCII equivalent of the character 'A'.
  
  To compensate for the lack of data types, Pawn provides the ability to tag cells. 
  These tags are mere compile time helpers and are not directly<sup>[[1]](#note-1)</sup> present in the AMX binary.
  
  ```pawn
  new Float:x = 5.0, Float:y = 10.0, Float:z;
  z = x + y;
  ```
  
  The above code creates two tagged cells identified by `x`, `y` and `z`. They are initialized with the value `5.0`, `10.0` and `0.0` (default initialization to binary zero) respectively in the correct [binary representation of those floating point numbers](https://en.wikipedia.org/wiki/Floating-point_arithmetic#Internal_representation).
  The compiler remembers the tags of the variables it encounters and ensures that the correct code is generated wherever they are used: the compiler emits the correct code to add two floating point numbers `x` and `y` as it processes the second line.
  Remember that integer addition and floating point addition are carried out differently. The compiler uses the tag associated with the cells to figure out what kind of addition operation has to be carried out.
  
  The information about the tag of the variables is lost after compilation. The produced AMX binary will contain instructions to carry out floating point addition on the data stored in `x` and `y` and store the result in `z`. 
  During execution, the floating point addition takes place without caring whether the tag of `x` and `y` in the script were originally `Float`. It is the duty of the compiler to place the correct code (`floatadd` in this case) in the binary based on the information it has while compiling.

  <a name="note-1"></a> [[1]](#note-1) *The Pawn language provides the `tagof` operator which returns the compile-time id associated with a tag that can be stored. 
  The compiler also creates a list of publicly accessible tags in the tag table in the AMX binary.*

* ## <a name="concept-memory-addresses"></a> Memory Addresses
  The AMX follows a flat byte-addressable memory model. This would imply that the memory in AMX can be thought of as a linear collection
  ([flat memory](https://en.wikipedia.org/wiki/Flat_memory_model)) of bytes with each byte having a unique address ([byte addressable](https://en.wikipedia.org/wiki/Byte_addressing)).
  
  ```pawn
  new a[5];
  ```
  
  The above code creates an array of five cells (default initialized to zero) identified by `a`. These cells are stored contiguously, i.e. they are stored one after another.
  Assuming the size of cells to be 4 bytes, if the address of the location where `a[0]` is stored is `1000`, the address of the location where `a[1]` is stored would be `1004`.
  
  ```
  data:    a[0] | a[1] | a[2] | a[3] | a[4]
  address: 1000   1004   1008   1012   1016
  ```
  
  *Required*: absolute addresses<sup>[reference required]</sup>, [base addresses](https://en.wikipedia.org/wiki/Base_address) and [relative addresses](https://en.wikipedia.org/wiki/Offset_(computer_science))
  
  *See also*: [Array data structure, Element identifier and addressing formulas](https://en.wikipedia.org/wiki/Array_data_structure#Element_identifier_and_addressing_formulas)
  
  *See also*: [Endianness](https://en.wikipedia.org/wiki/Endianness)
    
* ## <a name="concept-stack-heap"></a> Stack and heap
  Every AMX instance consists of an *internal* [program stack](https://en.wikipedia.org/wiki/Call_stack) (also known as the call stack) and [program heap](https://en.wikipedia.org/wiki/Memory_management#HEAP).
  These are regions of memory in your AMX program which are specialized for a set of tasks.
  
  The stack is responsible for:
  * storing local variables
  * storing function arguments
  * storing function call information (such as the number of arguments)
  * providing temporary storage
  
  The heap is responsible for:
  * providing memory for dynamic allocation
    * storing default arguments which are taken as reference/array
    * storing constants and expressions which are not lvalue and are passed as a variable argument
  
  ###
  
  ```pawn
  f(argc, ...) { }
  main () 
  {
      new a, b, c[100];
      f(1, 25);
  }
  ```
  
  In the above snippet, the variables `a`, `b`, `c` and the constant `1` are allocated space in the stack and the constant `25` is allocated space in the heap.
  
* ## <a name="concept-instructions"></a> Instructions
  
  [Instructions](https://en.wikipedia.org/wiki/Instruction_set_architecture#Instructions) are the discrete units of execution. Most of the instructions carry out simple tasks such as basic arithmetic. The higher constructs such as loops, conditional statements, etc. can be reduced to a series of simple instructions. In fact, all code in its actual binary form is just a collection of instructions. 
  
  For example, the instructions expressed in assembly which pushes and pops a value to and from the call stack could be:
  
  ```
  push 100
  pop
  ```
  
  The binary code generated by an [assembler](https://en.wikipedia.org/wiki/Assembly_language#Assembler) for the above code could be:
  
  `push-opcode | 100 | pop-opcode`
   
  `0x0013 0x0064 0x0014` or maybe `0000 0000 0000 0000 0000 0000 0001 0011 0000 0000 0000 0000 0000 0000 0110 0100 0000 0000 0000 0000 0000 0000 0001 0100`
  
  Yes, it's just a series of numbers/bits which the CPU knows how to read. This representation is known as [machine code](https://en.wikipedia.org/wiki/Machine_code).
 
* ## <a name="concept-spc-vna"></a> Stored program concept and von Neumann architecture

  The idea that both code and data reside in memory (could be on separate memories) is known as the [stored program concept](https://en.wikipedia.org/wiki/Stored-program_computer). 
  The idea that both code and data reside in the **same memory** is a principle of the [von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture).
  
  The AMX uses a common memory to store both the code and data. This would mean that all your code is in the same memory as your data.
  This also implies that every instruction in memory has an address just like any data.

* ## <a name="concept-registers"></a>Registers
  It is important to know that your processor and memory are at different places.
  
  ```
  |-------|                  |------------------|
  |  CPU  |  =============== |      MEMORY      |
  |-------|        BUS       |------------------|
  ```

  The memory can store data but CPU is the one that operates on the data. For every operation that happens in the CPU, the processor has to bring the operands from the memory and store them temporarily inside the CPU. These temporary memory locations are known as [registers](https://en.wikipedia.org/wiki/Processor_register). The registers can hold a very small amount of data (in the range of few bytes, ex: 2 bytes, 4 bytes, 8 bytes, etc).
  
  Suppose the processor had to add two variables `x` and `y` and store the result in variable `z`, a typical processor would do the following:
  1. bring the value of `x` from the memory (i.e. read from `x`'s address) and store it in a register, say `R1`
  2. bring the value of `y` from the memory (i.e. read from `y`'s address) and store it in a register, say `R2`
  3. add the contents of `R1` and `R2` and store it in some register, say `R3`
  4. store the contents of `R3` at in `z` (write to `z`'s address)

  A processor consists of different [types of registers](https://en.wikipedia.org/wiki/Processor_register#Categories_of_registers) serving different purposes. Some registers are specialized for a specific task and some can be used for any purpose.
  The registers used in the aforementioned example belongs to the class of general purpose registers, i.e. they can be used for anything.
  
  The AMX's register set consists of the following registers:
  * **Primary Register (PRI)**: general purpose register (frequently used as an [accumulator register](https://en.wikipedia.org/wiki/Accumulator_(computing)))
  * **Alternate Register (ALT)**: general purpose register (frequently used as an address register)
  * **Code Segment Register (COD)**: absolute address to the start of the code segment in memory
  * **Data Segment Register (DAT)**: absolute address to the start of the data segment in memory
  * **Current Instruction Pointer (CIP)**: address (relative to the COD register) of the next instruction to be executed
  * **Stack Top Register (STP)**: address (relative to the DAT register) to the top of the stack
  * **Stack Index Register (STK)**: address (relative to the DAT register) to the current location on the stack
  * **Frame Pointer Register (FRM)**: address (relative to the DAT register) of the start of the current function's frame in the stack (*explained later*)
  * **Heap Pointer (HEA)**: address (relative to the DAT register) to the top of the heap
  
  The addresses we deal with while writing assembly code are usually not absolute addresses; they are relative to one of the segment registers or the frame pointer register.
  The addresses of global variables, literals, etc. are relative to DAT register and the addresses of the functions or instructions or labels are relative to the COD register.
  This means that the absolute address of the variable is the relative address plus the value of the DAT register and similar is the case with code addresses except that you add the value contained in the COD register. 
  
  The addresses of local variables are relative to the frame pointer register. This is a bit complex and will be dealt with later.
  
  *Henceforth, when we talk about code addresses and data addresses, it is implied that the addresses are relative to the COD register and the DAT register respectively.* 

# <a name="file-memory-layout"></a> File and memory layout
There exists a format in which executable binaries are stored in files. All non-debug AMX binary files are organized as follows:

```
START OF BINARY FILE
    | ---------------------- |
              PREFIX            
    | ---------------------- |
               CODE
    | ---------------------- |
               DATA
    | ---------------------- |
END OF BINARY FILE
 ```
 
 * **Prefix**: contains essential information about the program
 * **Code**: contains the code
 * **Data**: contains the data such as global variables (including arrays), literals
 
 *Note: The prefix section may be padded to [align](https://en.wikipedia.org/wiki/Data_structure_alignment) the code and data sections based on the compiler options used.*
 
 *Note: The binary image of scripts compiled with debug enabled will contain the symbolic debug information appended at the end. Refer to the Pawn Implementer Guide for more details.*
 
 The in-memory structure of the program is slightly different. 
 The AMX builds the stack and the heap from the information contained in the prefix.
 
```
LOW ADDRESS (0)
    | ---------------------- |
              PREFIX            
    | ---------------------- |
               CODE
    | ---------------------- |
               DATA
    | ---------------------- |
               HEAP
               | |
               | |
            FREE SPACE
               | |
               | |
              STACK
    | ---------------------- |
HIGH ADDRESS
```
    
The heap and stack share a common memory region. They start from opposite ends of the region and grow in opposite directions. The heap grows towards the higher address and the stack grows towards the lower address.
Since they share the same region of memory, they could potentially overwrite each other (this is the situation when the heap pointer and the stack pointer collide). When this happens, the AMX aborts with an error message as shown below:

```
Script[gamemodes/TEST.amx]: Run time error 3: "Stack/heap collision (insufficient stack size)"
```

Based on the information the compiler has, it estimates and indirectly sets the heap-stack size in the prefix section of the AMX binary. You may, however, provide your own estimate of the required size of the heap-stack region using
the `#pragma dynamic [estimated number of cells]` directive.

### Fields of the prefix section

| Type          | Size          | Description                                                                               |
| ------------- | ------------- | ----------------------------------------------------------------------------------------- |
| size          | 4             | size of the memory image; excluding the stack and heap                                    |
| magic         | 2             | indicates the format and cell size                                                        |
| file version  | 1             | the format version; (8 for SA-MP)                                                         |
| amx version   | 1             | required minimum version of the abstract machine                                          |
| flags         | 2             |                                                                                           |
| defsize       | 2             | size of a record in the tables                                                            |
| cod           | 4             | offset to the start of the code section                                                   |
| dat           | 4             | offset to the start of the data section                                                   |
| hea           | 4             | initial value of the heap (marks the end of the data section)                             |
| stp           | 4             | stack top value (indicates the total memory requirement)                                  |
| cip           | 4             | address of the `main` function; `-1` if it does not exist                                 |
| publics       | 4             | offset to the start of the public function table                                          |
| natives       | 4             | offset to the start of the native function table                                          |
| libraries     | 4             | offset to the start of the libraries table                                                |
| pubvars       | 4             | offset to the start of the public variables table                                         |
| tags          | 4             | offset to the start of the tags table                                                     |
| name table    | 4             | offset to the start of name table                                                         |
| overlays      | 4             | offset to the start of the overlays table                                                 |
| *publics*     | variable      | list of public functions                                                                  |
| *natives*     | variable      | list of native functions                                                                  |
| *libraries*   | variable      | list of libraries                                                                         |
| *pubvars*     | variable      | list of public variables                                                                  |
| *tags*        | variable      | list of public tags                                                                       |
| *overlays*    | variable      | list of overlays                                                                          |
| *name table*  | variable      | list of symbol names                                                                      |

*Note: For detailed information about how information is encoded in the magic, version and flags fields; refer to the Pawn Implementer Guide.*

*Note: All the multi-byte fields in the prefix are stored in the [little endian format](https://en.wikipedia.org/wiki/Endianness) irrespective of the platform on
which the AMX binary was produced.*

It can be seen that the prefix consists of a fixed part that is followed by a series of tables. 
Every record in the tables except the name table consists of two fields as shown in the following structure:

| Field                 | Size          | Description                                                   |
| --------------------- | ------------- | ------------------------------------------------------------- |
| *variable*            | cell size     | *variable*                                                    |
| name string offset    | 4 bytes       | offset to a string in the name table from the start of prefix |

The size of each record in the tables (except the name table) is given by the `defsize` field in the prefix: `defsize` = 4 + cell size.

An index number starting from zero is assigned to every record in the tables. The first record of each table is assigned the index zero
and subsequent records' index number is one plus the index of the record preceding it.

**Name Table**:
The records in the tables (other than the name table) contain a pointer to a string. 
These strings are stored in the name table as null-terminated C strings.
The size of records in this table is the size of the string it holds.

**Public Function Table**:
A record is created for every public function that is defined in the script.
The records contain the address of the public function (address of the first instruction of the function) and the offset to
the name of the corresponding public function.

The index of a record in this table is the index of the corresponding public function.

**Native Function Table**:
A record is created for every native function that is called in the script. 
The first field of the record is set to zero by the compiler in the binary file but the host program initializes this field to the correct value when the binary is loaded.
The second field contains an offset to the name of the native function in the name table.

The index of a record in this table is the index of the corresponding native function.

**Library Table**:
Pawn language provides a pragma directive `#pragma directive [library name]` to inform the compiler that some of the native functions called require an extension module.
A record is created in the library table for libraries whose native functions have been called in the script.
The purpose is to inform the host program that the AMX binary depends on an extension module.

The first field of the records are used internally and are set to zero in the AMX binary.
The other field contains an offset from the start of prefix to the name of the library in the name table.

**Public Variable Table**:
A record is created for every public variable that is declared in the script.
The record contains the address of the public variable and the offset to the name of the public variable.

The index of a record in this table is the index of the public variable.

**Tag Table**:
The tags which are used with the `sleep` or the `exit` statement and the tags which are used with the `tagof` operator are considered useful.
A record is created for the tags considered useful.
The first field contains the tag id number and the second field stores an offset to the name of the tag.

# <a name="instruction-set"></a> Instruction Set

<a name="pawn-inline-assembly"></a> Most of the AMX assembly instructions as accepted by the Pawn compiler can be expressed in the following format:

```
mnemonic[.prefix][.register] operand

SHL.C.pri 3
ZERO.alt
ADD.C 100
LIDX
```

The mnemonic gives an idea of what the instruction does, the optional prefix indicates the type of operand the instruction takes and the optional suffix
tells which register the instruction mainly works on.

**List of prefixes**:
 * .C = constant
 * .S = stack
 * .I = indirection
 * .B = variant of the one without B<sup>[needs better description]</sup>
 * .ADR = address
 
**List of suffixes**:
 * .pri = primary register
 * .alt = alternate register
 
Every instruction in its binary form requires a cell to store the opcode and an additional cell for every operand.
A vast majority of instructions have implied registers as operands. 
This reduces the number of explicit operands needed which in turn decreases the size of the code segment and improves the performance.

<a name="instruction-table"></a> *Reading the table:*
 1. *[address] refers to the value stored at the location `DAT + address`*
 2. *the operators used in the sematics column perform the same operation as they do in Pawn*

| opcode | mnemonic   | operand   | semantics                                    | 
| ------ | ---------- | --------- | -------------------------------------------- |
| 1      | LOAD.pri   | address   | PRI = [address]                              |
| 2      | LOAD.alt   | address   | ALT = [address]                              |         
| 3      | LOAD.S.pri | offset    | PRI = [FRM + offset]                         |
| 4      | LOAD.S.alt | offset    | ALT = [FRM + offset]                         |
| 5      | LREF.pri   | address   | PRI = [[address]]                            |
| 6      | LREF.alt   | address   | ALT = [[address]]                            |
| 7      | LREF.S.pri | offset    | PRI = [[FRM + offset]]                       |
| 8      | LREF.S.alt | offset    | ALT = [[FRM + offset]]                       |
| 9      | LOAD.I     |           | PRI = [PRI]                                  |
| 10     | LODB.I     | number    | PRI = "number" of bytes from [PRI] (read 1/2/4 bytes) |
| 11     | CONST.pri  | value     | PRI = value                                  |
| 12     | CONST.alt  | value     | ALT = value                                  |
| 13     | ADDR.pri   | offset    | PRI = FRM + offset                           |
| 14     | ADDR.alt   | offset    | ALT = FRM + offset                           |
| 15     | STOR.pri   | address   | [address] = PRI                              |
| 16     | STOR.alt   | address   | [address] = ALT                              |
| 17     | STOR.S.pri | offset    | [FRM + offset] = PRI                         |
| 18     | STOR.S.alt | offset    | [FRM + offset] = ALT                         |
| 19     | SREF.pri   | address   | [[address]] = PRI                            |
| 20     | SREF.alt   | address   | [[address]] = ALT                            |
| 21     | SREF.S.pri | offset    | [[FRM + offset]] = PRI                       |
| 22     | SREF.S.alt | offset    | [[FRM + offset]] = ALT                       |
| 23     | STOR.I     |           | [ALT] = PRI (full cell)                      |
| 24     | STRB.I     | number    | number of bytes at [ALT] = PRI (store 1/2/4 bytes) |
| 25     | LIDX       |           | PRI = [ ALT + (PRI x cell size) ]            |
| 26     | LIDX.B     | shift     | PRI = [ ALT + (PRI << shift) ]               |
| 27     | IDXADDR    |           | PRI = ALT + (PRI x cell size) (calculate indexed address) |
| 28     | IDXADDR.B  | shift     | PRI = ALT + (PRI << shift) (calculate indexed address) |
| 29     | ALIGN.pri  | number    | Little Endian: PRI ^= cell size - number     |
| 30     | ALIGN.alt  | number    | Little Endian: ALT ^= cell size - number     |
| 31     | LCTRL      | index     | PRI = value contained in the selected register; 1=COD, 1=DAT, 2=HEA,3=STP, 4=STK, 5=FRM, 6=CIP |
| 32     | SCTRL      | index     | selected register = PRI; 2=HEA, 4=STK, 5=FRM, 6=CIP |
| 33     | MOVE.pri   |           | PRI = ALT                                    |
| 34     | MOVE.alt   |           | ALT = PRI                                    |
| 35     | XCHG       |           | Exchange contents of PRI and ALT             |
| 36     | PUSH.pri   |           | STK = STK - cell size, [STK] = PRI           |
| 37     | PUSH.alt   |           | STK = STK - cell size, [STK] = ALT           |
| 38     | PICK       | offset    | PRI = [STK + offset]                         |
| 39     | PUSH.C     | value     | STK = STK - cell size, [STK] = value         |  
| 40     | PUSH       | address   | STK = STK - cell size, [STK] = [address]     |
| 41     | PUSH.S     | offset    | STK = STK - cell size, [STK] = [FRM + offset] |
| 42     | POP.pri    |           | PRI = [STK] and STK = STK + cell size        |     
| 43     | POP.alt    |           | ALT = [STK] and STK = STK + cell size        |
| 44     | STACK      | value     | ALT = STK and STK = STK + value              |
| 45     | HEAP       | value     | ALT = HEA and HEA = HEA + value              |
| 46     | PROC       | value     | STK = STK - cell size, [STK] = FRM, FRM = STK |
| 47     | RET        |           | FRM = [STK], STK = STK + cell size, CIP = [STK], STK = STK + cell size |
| 48     | RETN       |           | FRM = [STK], STK = STK + cell size, CIP = [STK], STK = STK + [STK] + cell size |
| 49     | CALL       | offset    | STK = STK − cell size, [STK] = CIP, CIP = CIP + oﬀset |
| 50     | CALL.pri   |           | STK = STK − cell size, [STK] = CIP, CIP = PRI |
| 51     | JUMP       | oﬀset     | CIP = CIP + oﬀset                            |
| 52     | JREL       | oﬀset     | obsolete                                     |
| 53     | JZER       | oﬀset     | if PRI == 0 then CIP = CIP + oﬀset           |
| 54     | JNZ        | oﬀset     | if PRI != 0 then CIP = CIP + oﬀset           |
| 55     | JEQ        | oﬀset     | if PRI == ALT then CIP = CIP + oﬀset         |
| 56     | JNEQ       | oﬀset     | if PRI != ALT then CIP = CIP + oﬀset         |
| 57     | JLESS      | oﬀset     | if PRI < ALT (unsigned) then CIP = CIP + oﬀset | 
| 58     | JLEQ       | oﬀset     | if PRI <= ALT (unsigned)  then CIP = CIP + oﬀset |
| 59     | JGRTR      | oﬀset     | if PRI > ALT (unsigned) then CIP = CIP + oﬀset |
| 60     | JGEQ       | oﬀset     | if PRI >= ALT (unsigned) then CIP = CIP + oﬀset |
| 61     | JSLESS     | oﬀset     | if PRI < ALT (signed) then CIP = CIP + oﬀset  |
| 62     | JSLEQ      | oﬀset     | if PRI <= ALT (signed) then CIP = CIP + oﬀset | 
| 63     | JSGRTR     | oﬀset     | if PRI > ALT (signed) then CIP = CIP + oﬀset |
| 64     | JSGEQ      | oﬀset     | if PRI >= ALT (signed) then CIP = CIP + oﬀset |
| 65     | SHL        |           | PRI = PRI << ALT                             |
| 66     | SHR        |           | PRI = PRI >> ALT (without sign extension)    |
| 67     | SSHR       |           | PRI = PRI >> ALT (with sign extension)       |
| 68     | SHL.C.pri  | value     | PRI = PRI << value                           |
| 69     | SHL.C.alt  | value     | ALT = ALT << value                           |
| 70     | SHR.C.pri  | value     | PRI = PRI >> value                           |
| 71     | SHR.C.alt  | value     | ALT = ALT >> value                           |
| 72     | SMUL       |           | PRI = PRI * ALT (signed multiply)            |
| 73     | SDIV       |           | PRI = PRI / ALT (signed divide), ALT = PRI mod ALT |
| 74     | SDIV.alt   |           | PRI = ALT / PRI (signed divide), ALT = ALT mod PRI |
| 75     | UMUL       |           | PRI = PRI * ALT (unsigned multiply)          |
| 76     | UDIV       |           | PRI = PRI / ALT (unsigned divide), ALT = PRI mod ALT |
| 77     | UDIV.alt   |           | PRI = ALT / PRI (unsigned divide), ALT = ALT mod PRI |
| 78     | ADD        |           | PRI = PRI + ALT                              |
| 79     | SUB        |           | PRI = PRI - ALT                              |
| 80     | SUB.alt    |           | PRI = ALT - PRI                              |
| 81     | AND        |           | PRI = PRI & ALT                              |
| 82     | OR         |           | PRI = PRI | ALT                              |
| 83     | XOR        |           | PRI = PRI ^ ALT                              |
| 84     | NOT        |           | PRI = !PRI                                   |
| 85     | NEG        |           | PRI = -PRI                                   |
| 86     | INVERT     |           | PRI = ~PRI                                   |
| 87     | ADD.C      | value     | PRI = PRI + value                            |
| 88     | SMUL.C     | value     | PRI = PRI * value                            |
| 89     | ZERO.pri   |           | PRI = 0                                      |
| 90     | ZERO.alt   |           | ALT = 0                                      |
| 91     | ZERO       | address   | [address] = 0                                |
| 92     | ZERO.S     | offset    | [FRM + offset] = 0                           |
| 93     | SIGN.pri   |           | sign extent the byte in PRI to a cell        |
| 94     | SIGN.alt   |           | sign extent the byte in ALT to a cell        |
| 95     | EQ         |           | PRI = PRI == ALT ? 1 : 0                     |
| 96     | NEQ        |           | PRI = PRI != ALT ? 1 : 0                     |
| 97     | LESS       |           | PRI = PRI < ALT ? 1 : 0 (unsigned)           |     
| 98     | LEQ        |           | PRI = PRI <= ALT ? 1 : 0 (unsigned)          |
| 99     | GRTR       |           | PRI = PRI > ALT ? 1 : 0 (unsigned)           |
| 100    | GEQ        |           | PRI = PRI >= ALT ? 1 : 0 (unsigned)          |
| 101    | SLESS      |           | PRI < ALT ? 1 : 0 (signed)                   |
| 102    | SLEQ       |           | PRI = PRI <= ALT ? 1 : 0 (signed)            |
| 103    | SGRTR      |           | PRI = PRI > ALT ? 1 : 0 (signed)             |
| 104    | SGEQ       |           | PRI = PRI >= ALT ? 1 : 0 (signed)            |
| 105    | EQ.C.pri   | value     | PRI = PRI == value ? 1 : 0                   |
| 106    | EQ.C.alt   | value     | PRI = ALT == value ? 1 : 0                   |
| 107    | INC.pri    |           | PRI = PRI + 1                                |
| 108    | INC.alt    |           | ALT = ALT + 1                                |
| 109    | INC        |           | [address] = [address] + 1                    |
| 110    | INC.S      | offset    | [FRM + offset] = [FRM + offset] + 1          |
| 111    | INC.I      |           | [PRI] = [PRI] + 1                            |
| 112    | DEC.pri    |           | PRI = PRI - 1                                |
| 113    | DEC.alt    |           | ALT = ALT - 1                                |
| 114    | DEC        | address   | [address] = [address] - 1                    |
| 115    | DEC.S      |           | [FRM + offset] = [FRM + offset] - 1          |
| 116    | DEC.I      |           | [PRI] = [PRI] - 1                            |
| 117    | MOVS       | number    | Copy memory from [PRI] to [ALT]. The parameter speciﬁes the number of bytes. The blocks should not overlap. |
| 118    | CMPS       | number    | Compare memory blocks at [PRI] and [ALT]. The parameter speciﬁes the number of bytes. The blocks should notoverlap. |
| 119    | FILL       | number    | Fill memory at [ALT] with value in [PRI]. The parameter specifes the number of bytes, which must be a multiple of the cell size. |
| 120    | HALT       | 0         | Abort execution (exit value in PRI), parameters other than 0 have a special meaning. |
| 121    | BOUNDS     | value     | Abort execution if PRI > value or if PRI < 0 |
| 122    | SYSREQ.pri |           | call system service, service number in PRI   |
| 123    | SYSREQ.C   | address   | call system service                          |
| 128    | JUMP.pri   |           | CIP = PRI
| 129    | SWITCH     | offset    | Compare PRI to the values in the case table (whose address is passed as an oﬀset from CIP) and jump to the associated the address in the matching record. |
| 130    | CASETBL    | ...       | A variable number of case records follows this opcode, where each record takes two cells. |
| 131    | SWAP.pri   |           | [STK] = PRI and PRI = [STK]                  |
| 132    | SWAP.alt   |           | [STK] = ALT and ALT = [STK]                  |
| 133    | PUSH.ADR   | offset    | STK = STK - cell size, [STK] = FRM + offset  |
| 134    | NOP        |           | no operation; used for code alignment        |

### <a name="loadstore-instructions"></a> Load/store instructions:

When a global variable is used as an operand, the Pawn compiler substitutes the correct address of the variable.
The first load instruction in the code below effectively becomes `#emit LOAD.pri 1288` if the address of `some_global` was 1288 bytes.
Note that the address substituted by the compiler is actually the offset from the start of the data segment; hence, the address is relative to the 
DAT register.

```pawn
new some_global = 10, another_global = 25;

main()
{
    #emit LOAD.pri some_global // load the value of 'some_global' into the primary register
    #emit LOAD.alt another_global  // load the value of 'another_global' into the alternate register
}
```

When a local variable (but not arrays) are used as an operand, the Pawn compiler substitutes the correct offset from the start of the function frame. *(explained later)*

```pawn
main()
{
    static s_local = 20; // static local variables are stored in the data segment

    new some_local = 10, another_local = 25; // local variables are stored in the stack
    #emit LOAD.S.pri some_local // load the value of 'some_local' into the primary register
    #emit LOAD.S.alt another_local  // load the value of 'another_local' into the alternate register
  
    #emit LOAD.pri s_local // note that static local variables behave like global variables
}
```

Given that the compiler substitutes the correct offsets for global variables, `CONST.pri/CONST.alt` can be used to obtain the address of those variables.

```pawn
new some_global;
main()
{
    #emit CONST.pri 10 // put 10 in to the alternate register
    #emit CONST.alt 50 // put 50 in to the alternate register

    #emit CONST.pri some_global // store the address of 'some_global' in the primary register
    #emit CONST.alt some_global // store the address of 'some_global' in the alternate register
}
```

```pawn
new some_global;
main()
{
    new some_local;
    #emit ZERO.pri // store zero in the primary register
    #emit STOR.pri some_global // set 'some_global' to the value stored in primary register (zero in this case)
  
    #emit CONST.alt 125
    #emit STOR.S.alt some_local // set 'some_local' to the value stored in the alternate register (125 in this case)
}
```

### <a name="indexing-instructions"></a> Indexing instructions:

```pawn
new global_arr[10];
main ()
{
    #emit CONST.alt global_arr // load the address of 'global_arr' (address of the first element of 'global_arr') into the alternate register
    #emit CONST.pri 2 // set the index of the elment of 'global_arr' that we are interested in
    #emit LIDX // the primary register now has the value stored at 'global_arr[2]`
  
    #emit CONST.alt global_arr // load the address of 'global_arr' (address of the first element) into the alternate register
    #emit CONST.pri 2 // set the index of the elment of 'global_arr' that we are interested in
    #emit IDXADDR // the primary register now has the address of 'global_arr[2]`
}
```

### <a name="arithmetic-instructions"></a> Arithmetic instructions:

```pawn
main ()
{
    #emit CONST.pri 4
    #emit CONST.alt 5
  
    #emit SMUL // the primary register now has 20 (SMUL does signed multiplication; UMUL does unsigned multiplication)
    #emit ADD // adds 5 to the 20 (the primary register was storing 20 because of the previous instruction)
    #emit ADD.C 10 // adds 10 to the primary register; now the primary register holds 35
    
    #emit SUB.alt // subtract 35 from 5; the primary register now has -30
    #emit SMUL.C 2 // multiply -30 by 2; this gives -60
}
```

### <a name="logical-instructions"></a> Logical instructions:

```pawn
main ()
{
    #emit CONST.pri 5 // .. 0000 0101
    #emit CONST.alt 3 // ... 0000 0011
    #emit AND // primary register will now contain ... 0000 0001
    #emit XOR // primary register will now contain ... 0000 0110
  
    #emit INVERT // take one's complement of the value stored in the primary register
    #emit NEG // take two's complement of the value stored in the primary register (essentially negation)
}
```

### Relational instructions:

```pawn
main ()
{
    #emit CONST.pri 5
    #emit CONST.alt 8
    #emit EQ // set primary register to 1 if 5 == 8, otherwise zero
    #emit LESS // set primary register to 1 if 0 < 8, otherwise zero
}
```

### <a name="stack-manipulation-instructions"></a> Stack manipulation instructions:

The local variables are stored in the stack. If a local variable is used as an operand, the compiler substitutes the offset of the local variable
from the start of the current function frame in the stack. A detailed explanation will be provided in a later section but for now, we assume that
using `CONST.pri some_local` gives some offset which when added to the base address stored in FRM register gives the actual address.

```pawn
main ()
{
    new some_local = 25;
    #emit ADDR.alt some_local // computes the address of 'some_local'
    #emit CONST.pri 100
    #emit STOR.I // store 100 in 'some_local'
  
    #emit CONST.pri 100
    #emit STOR.S.pri some_local //a better way to achieve what the above code did
  
    #emit CONST.pri some_local // mysteriously equivalent to CONST.pri -4 (explained later)
}
```

The stack can be used to store temporary data.

```pawn
main ()
{
    #emit PUSH.C 100 // pushes the value 100 on to the call stack
    #emit POP.pri // pops a value from the stack and stores the result in the primary register
  
    #emit PUSH.pri // push the value of the primary register
    #emit PUSH.alt // push the value of the alternate register
    #emit POP.pri
    #emit POP.alt // the last 4 instructions effectively swap the contents of the primary register and the alternate register
  
    #emit XCHG // a better way to swap the contents of the primary register and the alternate register
}
```

The local arrays are on the stack. Hence, `ADDR.alt/ADDR.pri` must be used to obtain the full address before using the indexing instructions.

```pawn
main ()
{
    new local_array[10];
    #emit ADDR.alt local_array // loads the value stored in 'local_array' which is the address of the array
    #emit CONST.pri 5
    #emit LIDX // effectively stores the value of 'local_array[5]' in the primary register
}
```

### <a name="heap-instructions"></a> Heap instructions:
The heap pointer (the HEA register) points to the top of the heap. By moving the heap pointer ahead by `x` bytes, we effectively
reserve `x` bytes on the heap.

```pawn
main ()
{
     #emit HEAP 16 // make room for 4 cells (assuming cells are 4 bytes)
     // note that the HEAP instruction had also set the alternate register to the start of our reserved memory
     // ALT = HEA, HEA += 16
   
     #emit CONST.pri 50
     #emit STOR.I // effectively store the value 50 in the first cell of our reserved area of the heap
    
     #emit HEAP -16 // let's return the reserved memory back
}
```

### <a name="control-register-manipulation-instructions"></a> Control register manipulation:
The contents of registers (other than primary and alternate) can be directly read and modified using the `LCTRL` and `SCTRL` instruction respectively.

| mnemonic | operand | description |
| -------- | ------- | ----------- |
| LCTRL    | index   | PRI = value contained in the selected register; 0=COD, 1=DAT, 2=HEA, 3=STP, 4=STK, 5=FRM, 6=CIP |
| SCTRL    | index   | selected register = PRI; 2=HEA, 4=STK, 5=FRM, 6=CIP |

```pawn
main ()
{
     new cod, dat;
     #emit LCTRL 0 // store the value of the COD segment register in the primary register
     #emit STOR.S.pri cod
   
     #emit LCTRL 1 // store the value of the DAT segment register in the primary register
     #emit STOR.S.pri cod
   
     printf("%d %d", cod, dat);
}
```

When a function name is used as an operand, the compiler substitutes it with the address of the function. The address substituted is an 
offset from the start of the code segment which is essentially an address relative to the COD register.

```pawn
f()
{
    print("f() was called.");
}
main ()
{
   #emit PUSH.C 0 // number of bytes taken up by arguments (explained later)
   #emit LCTRL 6 // get the value of CIP which is the address of the next instruction (ADD.C in our case)
   #emit ADD.C 28 // compute the address of the instruction to be executed after 'f' returns (explained later)
   #emit PUSH.pri // push the return address so that 'f' knows where to return to (explained later)
   #emit CONST.pri f // store the address of the function 'f' in pri
   #emit SCTRL 6 // set the current instruction pointer to the value stored in the primary register
   
   // the function 'f' executes
   // after the function 'f' returns, the next instruction (NOP in our case) will begin to execute
   
   #emit NOP // an instruction which does literally nothing 
}
```

### <a name="control-flow-instructions"></a> Control flow instructions:

```pawn
main()
{  
    #emit JUMP check // jump to the check label

    not_equal:
       printf("1 is not equal to 2");
       return 0;

    equal:
       print("1 is equal to 2");
       return 0;

    check:
       #emit CONST.pri 1
       #emit CONST.alt 2
       #emit JEQ equal // if the value of the primary register is equal to that of the alternate register, then jump to 'equal'
       #emit JUMP not_equal // if we ended up here, it implies that the values of the two registers were not equal; hence, jump to 'not_equal'
}
```

### <a name="switch-case-instructions"></a> Switch case instructions:

A switch-case block is implemented using a [case table](https://en.wikipedia.org/wiki/Branch_table). A case table is merely a list of
tuples consisting of a case number and its corresponding jump address. For a given case value, the AMX searches the case table and jumps
to the correct jump address. The case table records are arranged in ascending order of the case numbers. This allows the AMX to perform a 
binary search on the case table but the current version of AMX does not.

The beginning of a switch-case block is marked by the `SWITCH` instruction. It takes an offset relative to the CIP register to the case table (which is also present in the code segment) as an operand.
The case table formally begins with a `CASETBL` instruction which is followed by a series of `CASE` records each taking two parameters. The first case record has special meaning associated with its two parameters: 
the first parameter is the number of cases in the case table and the second parameter is the offset relative to the CIP register of the default case. 
If a default case isn't provided, the second parameter contains the offset relative to the CIP register of the instruction coming after the switch-case block.
The rest of the `CASE` records contain the case value and its corresponding jump offset relative to the CIP register of its record. 

```pawn
switch(expression)
{
    case 2: {}
    case 4: {}
    case 3: {}
    case 7: {}
    case 5: {}
}
 ```   

The compiler adds the instructions to evaluate the given `expression` whose result is stored in the primary register.
A `SWITCH` instruction immediately follows which causes the AMX to search the case table for the case value matching the
the value stored in the primary register.
If a match is found, the execution jumps to the address pointed by the matching record; otherwise, it jumps to the default address.

```assembly
; the compiler makes it easier to read assembly output by using labels instead of real offsets/addresses
; every label begins with the prefix "l."

switch 0 ; note that the zero is a label here

l.2 ; label 2
   jump 1 ; jump to label 1
l.3
   jump 1
l.4
   jump 1
l.5
   jump 1
l.6
   jump 1
  
l.0
  casetbl 
  case 5 1 ; number of records, default jump address (label 1 in this case)
   case 2 2 ; real first record
   case 3 4 ; case value: 3, jump label: 4
   case 4 3 ; the compiler writes correct addresses in place of the labels in the actual binary
   case 5 6 ; 
   case 7 5 ; the last record
  
 l.1
   ; rest of the code
 ```   

# <a name="call-stack-and-call-convention"></a> Call stack and call convention

*We assume that a cell is 4 bytes for this section.*

### <a name="local-variables-and-function-frame"></a> Local variables and function frames

For every local variable declared, a room is made for it on the stack by simply pushing zero (or the initialization value provided) on to the stack.

```pawn
f()
{
    new local1,     // PUSH.C 0
        local2 = 5; // PUSH.C 5
  
    local1 = local2 + 50; // directly access the 'local1' and 'local2' in the stack
}
```

The compiler adds `PUSH.C 0` and `PUSH.C 5` instructions in response to the declarations of `local1` and `local2` respectively.
All accesses and modifications to the variables will directly access and modify the appropriate cells in the stack. 
However, using the address of the locals to read or write to local variables is not possible since the address can be known only during run-time;
for example, think of a local variable of a recursive function.

The solution is to abandon the use of addresses and instead use offsets from a particular reference position in the stack.
The reference point used for assigning offsets is the position of the top of the stack just after the function is called.
This reference point is known as the beginning of the current function's frame.
The corresponding address of that position is known as the frame address of the current function.

Just after a function is called, the frame of the function is empty; the top of the stack relative to the frame is zero, i.e: there's nothing.
Note that as the stack grows downwards, the address of the recently pushed item will be lesser than that of the item preceding it. 
This implies the offsets of the items that are pushed on to the stack are negative. The offsets of `local1` and `local2` will be `-4` and `-8` respectively.

```pawn
f()
{
    // function's frame is empty at this point

    // add local variables to the function's frame
    new local1, 
        // PUSH.C 0
  
        local2[100], 
        // STACK -400 (assuming cells of 4 bytes)
        // ADDR.alt -404
        // ZERO.pri
        // FILL 400

        local3 = 32;
        // PUSH.C 32
}
```

When a local array is declared, a room is made for the array by moving stack pointer down by the number of bytes required to store the array. The array
is then filled with zeros or with the initializer list provided. In the above snippet, the offset for `local1`, `local2` and `local3` are 
`-4`, `-404` and `-408` respectively.

```pawn
f()
{
    new local1;
    // PUSH.C 0
    // local1 = -4
  
    for(new i = 0; i < 10; i++) 
    // PUSH.C 0 
    // i = -8
  
    {
      new local2, 
      // PUSH.C 0
      // local2 = -12
    
          local3[10];
          // STACK -40
          // ADDR.alt -52
          // ZERO.pri
          // FILL 40
        
      // STACK 44 (removes 'local2' and 'local3')
    }
    // STACK 4 (removes 'i')
}
```

The storage duration of local variables is restricted to the code block in which they were declared. Therefore, the local variables
must somehow be removed from the stack when they go out of scope. Notice that all the variables that were declared in the scope (and its subscope) 
will be present one after another in the stack. Therefore, a single `STACK` instruction to move the pointer up by the number of bytes of local symbols that were
declared in current scope would remove the local symbols from the stack.

### <a name="call-convention"></a> Call convention

The arguments are pushed on to the stack by the caller. Since they are already on the stack when the callee is called (which sets the FRM register),
the offsets of the arguments relative to the callee's frame are positive. However, they do not start from 0; instead, the first argument is
at offset 12 and the second is at 16 and so on. The first three cells above the current function's frame contain information about the function call itself.

If the call stack is explored after a function call, it will have the following structure:

```
HIGH ADDRESS
.                      .                                 <= frame of caller 
.                      .
.                      .
16                     argument 2
12                     argument 1
8                      (number of arguments)
4                      (return address)
0                      (frame address of the caller)     
-4                     callee local variable 1           <= frame of callee begins from this position
-8                     callee local variable 2
.                      .
.                      .
.                      .
LOW ADDRESS
```

The series of steps taken to make a function call in detail are:
1. push the arguments (last argument pushed first)
2. push the number of arguments (in terms of size in bytes)
3. push the return address
4. set CIP to point to the beginning of the callee
5. save the value of the FRM register (caller's frame address) on the stack
6. set the FRM to STK (callee's frame address)
7. execute callee
8. restore the stack to the condition it was just after the function was called (i.e. remove local variables)
9. store the return value in the primary register
10. pop the frame address of the caller and set FRM register (without altering the return value in the primary register)
11. pop the return address and set CIP register (without altering the return value in the primary register)
12. remove arguments off the stack (without altering the return value in the primary register)

Steps 1 and 2 have to be done manually. Note that the arguments are pushed in reverse order, i.e. the last argument is pushed first.
Steps 3 and 4 are both done together by the `CALL` instruction (and its `CALL.pri` variant).
Steps 5 and 6 are both carried out together by the `PROC` instruction.
Step 8 is carried out using the `STACK` instruction.
Step 10 and 11 are done together by `RET` instruction.
The `RETN` instruction does steps 10, 11 and 12 together. 

If the `RET` instruction is used, the responsibility of cleaning up the stack is upon the caller.
The `STACK`, `RET`, `RETN` instructions do not alter the contents of the primary register. 
Hence, the return value is not affected.

```pawn
new stk, stp, tmp; // globals are not involved in the stack
f(arg)
{
    // compiler automatically adds a PROC instruction at the beginning of every function

    new x = 200;
  
    #emit LCTRL 3
    #emit STOR.pri stp
  
    #emit LCTRL 4
    #emit STOR.pri stk
  
    printf("STP: %d STK: %d\n", stp, stk);
  
    // prints the contents of the stack from top to bottom (lower address to higher address)
    while(stk != stp) {
        #emit LOAD.pri stk
        #emit LOAD.I
        #emit STOR.pri tmp
        printf("%d", tmp);
        stk += 4;
    }
    
    // compiler uses whatever information it has and adds instructions to correct the stack before returning
    // if items have been pushed manually using #emit, the compiler won't be aware of it
    return 1234;
}
main () {
    new a = 1;
    f(101);
    #emit STOR.S.pri a // store the value stored in the primary register (return value) in 'a'
    printf("\nPRI: %d", a);
} 
```

```
STP: 16500 STK: 16464

200    ; local variable 'x'
16488  ; frame address of 'main'
308    ; return address
4      ; 4 bytes of arguments were passed
101    ; argument 'arg' of function 'f' 
1      ; local variable 'a'
0      ; unused frame address (there's no caller for main)
0      ; return to address 0 which has a 'halt 0' instruction
0      ; main does not take any arguments

PRI: 1234
```

### <a name="methods-passing-arguments"></a> Methods of passing arguments

**Pass by value**: 
The arguments passed by value have copies of the actual parameter pushed on to the stack.
Since an independent copy exists on the stack, any modification done by the callee on the argument won't affect the actual parameter.

```pawn
f(arg1, arg2)
{
    #emit CONST.pri 10
    #emit STOR.S.pri arg1  // this does not affect 'x' in 'main' since it modifies the copy of 'x' on the stack not 'x' itself
}
main ()
{
    new x = 150;
  
    // f(x, 10);  
    #emit PUSH.C 10 // push the 
    #emit PUSH.S x  // effectively pushes the value contained in 'x' on to the stack
    #emit PUSH.C 8  // two arguments were pushed; hence, the total size is 8 bytes
    #emit CALL f
}
```

**Pass by reference**:
The address of the actual parameter is passed. The callee uses the address to read and modify the argument.
Therefore, any change made by the callee to the argument will affect the actual parameter.

```pawn
new global;
f(&arg1, &arg2)
{
    #emit CONST.pri 10
    #emit SREF.S.pri arg // 'arg' has address and the data has to be written to that address => [[FRM + arg]] = PRI
}
main ()
{
    new x = 150;
    // f(x, global);
    #emit PUSH.C global // push the address of 'global'
    #emit PUSH.adr x // push address of 'x', i.e: FRM + x
    #emit PUSH.C 8
    #emit CALL f  
}
```

**Passing arrays**:
Arrays are passed as reference, i.e: the address of the array is passed. This avoids the costly array copying but makes modifications to the argument transparent.

```pawn
f(arr1[], arr2[])
{
    #emit CONST.pri 2
    #emit LOAD.S.alt arr1 // loads the address of the array 'arg1' points to in to the alternate register
    #emit IDXADDR // calculate the address of arr1[2]
    #emit MOVE.alt // move the address calculated by IDXADDR in to the alternate register
    #emit CONST.pri 100
    #emit STOR.I // store 100 at arr1[2]
}
g(arg[])
{
    new x[100];
    // f(x, arg);
    #emit PUSH.S arg // 'arg' stores the address of the array; hence, PUSH.S which pushes the value stored at 'arg' (pushes [FRM + arg])
    #emit PUSH.adr x // 'x' exists locally and directly points to the array; hence, we use PUSH.adr (pushes 'FRM + x')
    #emit PUSH.C 8
    #emit CALL f
}
```

The Pawn syntax allows passing an array from a particular index. This can be achieved by passing the address of the element at that index.

```pawn
f(arr[]) { }
main()
{
    new x[100];
    // f(x[2]);
    #emit ADDR.alt x // load the address of 'x' (FRM + x) in to the alternate register
    #emit CONST.pri 2 // index 2
    #emit IDXADDR // compute the address of x[2]
    #emit PUSH.pri // push the address of x[2]
    #emit PUSH.C 4
    #emit CALL f
}
```

**Passing variable arguments**:
The parameters passed as a variable argument are passed by reference. If a constant or an expression which is not an lvalue has to be passed,
the result must be temporarily stored in the heap and its address must be passed.

```pawn
f(arg, ...)
{
    new argc;
    #emit LOAD.S.pri 8 // offset 8 has the number of arguments in bytes
    #emit CONST.alt 4
    #emit UDIV         // primary register has the number of arguments that were passed
    #emit ADD.C -1     // subtract 1 because we are not interested in 'arg'
    #emit STOR.S.pri argc // argc now has the number of arguments
  
    // go through the arguments list in reverse order
    while(argc--) {
      new offset = 16 + argc*4;
      #emit LOAD.S.pri offset
      #emit LOAD.I // primary register has the address of the argument
    
      // do what you want
    }  
}
main ()
{
    new x;
    //f(10, x, 25);
    #emit HEAP 4 // allocate space on the heap for 25
    #emit CONST.pri 25
    #emit STOR.I // store 25 in the allocated space
    #emit PUSH.alt // push the address of the allocated cell
    #emit PUSH.adr x
    #emit PUSH.C 10 // pass by value
    #emit PUSH.C 12
    #emit CALL f
    #emit HEAP -4 // free the allocated space  
}
```

### <a name="system-requests"></a> System Requests

The procedure for calling native functions is almost the same as that for any other function except that `SYSREQ.pri/SYSREQ.C` instruction is used in place of `CALL/CALL.pri` and the stack has to be cleaned up by the caller. 
The compiler creates an entry in the native function table for every native function that is used in the script. The records of the native functional table are assigned an index in order starting with zero for the first record. 
This index is used as an operand for `SYSREQ.pri/SYSREQ.C` instructions.

If the name of the native function is provided as an operand, the compiler substitutes the correct native function index.

```pawn
native random(); // returns a random number
native printf(const frmt[], ...); // prints a formatted string
main ()
{
    static const str[] = "Random number: %d";

    //printf(str, random());
    
    #emit PUSH.C 0 // zero arguments
    #emit SYSREQ.C random
    #emit STACK 4 // must manually clean the stack (only one item was pushed on to the stack; hence, we move STK by 4)
    // random value was returned in the primary register

    // prepare to push the random value
    // note that 'printf' takes values as variable arguments: we need to push addresses
    #emit HEAP 4 // make space for the random value in the heap
    #emit STOR.I // store the value of the primary register in the newly allocated space
    
    #emit PUSH.alt // push the address of the random value
    #emit PUSH.C str // 'str' is static local string; hence, it is present in the data segment
    #emit PUSH.C 8
    #emit SYSREQ.C printf
    #emit STACK 12 // total of three items were pushed; hence, pop 3 cells
    
    #emit HEAP -4
}
```

# <a name="multi-dimensional-arrays"></a> Multi-dimensional arrays

Two-dimensional arrays consist of an indirection table in addition to the array data. 
The indirection table is a one-dimensional array of offsets to the sub-arrays.
For a two dimensional 4 by 3 array, the structure would look like:

```
address of [0] | address of [1] | address of [2] | address of [3]
      |                |                |                |
      |                |                |                ([3][0] [3][1] [3][2])
      |                |                ([2][0] [2][1] [2][2])             
      |                ([1][0] [1][1] [1][2])
      ([0][0] [0][1] [0][2])
```

The indirection table consists of 4 offsets pointing to their corresponding 3-element sub-array. 
These offsets are relative to the address of the cell from which they are read, i.e. to obtain
the address of [2][3], the value stored at [2] must be added to the address of [2].

```pawn
new arr[5][5];

#emit CONST.alt arr // load the address of the indirection table
#emit CONST.pri 2 // set the sub-array index to access (major dimension index)
#emit IDXADDR // address of the cell in the indirection table storing offset to sub-array [2] is in the primary register now

#emit MOVE.alt // keep a copy of the address
#emit LOAD.I // load the offset into the primary register

// currently ALT contains the address of the cell storing the offset of the sub-array [2]
// currently PRI contains the offset to the sub-array [2]
// the offset stored is relative to the address of the cell it was read from; hence, we add PRI and ALT together
#emit ADD

// PRI now has the address of the sub-array [2]
// the sub-array [2] can now be treated as a one-dimensional array
#emit MOVE.alt // copy the address of the sub-array [2]
#emit CONST.pri 4 // index of the minor dimension
#emit LIDX // loads [2][4] in to the primary register
```

Every N-dimensional (N != 1) contains an indirection table with offsets pointing to the (N - 1)-dimensional sub-arrays. To access an 
element, the indirection tables must be recursively traversed until a one-dimensional array is obtained
```pawn
new arr[5][5][5];

#emit CONST.pri arr

// PRI contains the address of the indirection table for the three-dimensional array
#emit MOVE.alt
#emit CONST.pri 3
#emit IDXADDR 
#emit MOVE.alt
#emit LOAD.I
#emit ADD

// PRI contains the address of the indirection table of the two-dimensional arr[3] sub-array
#emit MOVE.alt
#emit CONST.pri 2
#emit IDXADDR
#emit MOVE.alt
#emit LOAD.I
#emit ADD

// PRI contains the address of the one-dimensional arr[3][2] sub-array
#emit MOVE.alt
#emit CONST.pri 4
#emit LIDX

// PRI contains the value stored at arr[3][2][4]
```
