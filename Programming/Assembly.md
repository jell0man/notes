Assembly is a low-level language that maps almost directly to a CPU's machine instructions, using human-readable mnemonics (like `mov`, `push`, `call`) instead of raw binary opcodes. It give precise, hardware-level control over registers, memory, and execution flow.

> This will focus mainly on the Intel x86 64-bit assembly language (also known as x86_64 and AMD64), using Intel syntax (`Netwide Assembler (NASM)`. Might expand later to include ARM etc...
# Architecture
Most modern computers are built on Von Neumann Architecture. Essentially, CPU, memory, and I/O.
## Speed
Speed is largely determined by distance from the CPU and clock speed. Further distance = slower.

| Component   | Speed                             | Size                |
| ----------- | --------------------------------- | ------------------- |
| `Registers` | Fastest                           | Bytes               |
| `L1 Cache`  | Fastest, other than Registers     | Kilobytes           |
| `L2 Cache`  | Very fast                         | Megabytes           |
| `L3 Cache`  | Fast, but slower than the above   | Megabytes           |
| `RAM`       | Much slower than all of the above | Gigabytes-Terabytes |
| `Storage`   | Slowest                           | Terabytes and more  |
## Memory
Cache - Inside CPU, faster than RAM, limited in size
RAM - larger than cache memory, slower

RAM has four main segments:

|Segment|Description|
|---|---|
|`Stack`|Has a Last-in First-out (LIFO) design and is fixed in size. Data in it can only be accessed in a specific order by push-ing and pop-ing data.|
|`Heap`|Has a hierarchical design and is therefore much larger and more versatile in storing data, as data can be stored and retrieved in any order. However, this makes the heap slower than the Stack.|
|`Data`|Has two parts: `Data`, which is used to hold variables, and `.bss`, which is used to hold unassigned variables (i.e., buffer memory for later allocation).|
|`Text`|Main assembly instructions are loaded into this segment to be fetched and executed by the CPU.|
This segmentation applies to RAM as well as each application (each app is allocated virtual memory when run).
## CPU
The Central Processing Unit (CPU) is the main processing unit of a computer.
	`Control Unit (CU)` : in charge of moving data
	`Arithmetic/Logic Unit (ALU)` : performs various arithmetic as requested by program through assembly instructions

The manner in which a CPU processes its instructions depends on its `Instruction Set Architecture (ISA)`. RISC, CISC, etc...

Clock speed : 
	Every tick of the clock runs a clock cycle that processes a basic instruction.
	`Hertz` : The frequency the cycles occur in cycles per second.
	Multi-core processor = multiple cycles at the same time.

`Instruction Cycle`: The cycle the CPU takes to process a single machine instruction.

| **Instruction** | **Description**                                                                                                                           |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `1. Fetch`      | Takes the next instruction's address from the `Instruction Address Register` (IAR), which tells it where the next instruction is located. |
| `2. Decode`     | Takes the instruction from the IAR, and decodes it from binary to see what is required to be executed.                                    |
| `3. Execute`    | Fetch instruction operands from register/memory, and process the instruction in the `ALU` or `CU`.                                        |
| `4. Store`      | Store the new value in the destination operand.                                                                                           |
> All of the stages in the instruction cycle are carried out by the Control Unit, except when arithmetic instructions need to be executed "add, sub, ..etc", which are executed by the ALU.

Each Instruction Cycle takes multiple clock cycles to finish, then increments to the next instruction.
### Processor Specific
It is important to understand that each processor has its own set of instructions and corresponding machine code.

Example:
	64-bit x86 architecture
		`4883C001` as `add rax, 1`
	ARM processor
		`4883C001` as `biceq r8, r0, r8, asr #6`

Why?
	Instruction Set Architectures (ISA) differ.

Also, a single ISA may have several syntax interpretations for the same assembly code. 
	`add rax, 1` - Intel syntax
	`addb $0x1,%rax` - AT&T syntax.

> So, each processor type has its Instruction Set Architectures, and each architecture can be further represented in several syntax formats

## Instruction Set Architectures
An `Instruction Set Architecture` (`ISA`) specifies the syntax and semantics of the assembly language on each architecture. Consists of the following:

|Component|Description|Example|
|---|---|---|
|`Instructions`|The instruction to be processed in the `opcode operand_list` format. There are usually 1,2, or 3 comma-separated operands.|`add rax, 1`, `mov rsp, rax`, `push rax`|
|`Registers`|Used to store operands, addresses, or instructions temporarily.|`rax`, `rsp`, `rip`|
|`Memory Addresses`|The address in which data or instructions are stored. May point to memory or registers.|`0xffffffffaa8a25ff`, `0x44d0`, `$rax`|
|`Data Types`|The type of stored data.|`byte`, `word`, `double word`|
`Complex Instruction Set Computer` (`CISC`) - Used in `Intel` and `AMD` processors in most computers and servers.
`Reduced Instruction Set Computer` (`RISC`) - Used in `ARM` and `Apple` processors, in most smartphones, and some modern laptops.

| Area                             | CISC                                                        | RISC                                                |
| -------------------------------- | ----------------------------------------------------------- | --------------------------------------------------- |
| `Complexity`                     | Favors complex instructions                                 | Favors simple instructions                          |
| `Length of instructions`         | Longer instructions - Variable length 'multiples of 8-bits' | Shorter instructions - Fixed length '32-bit/64-bit' |
| `Total instructions per program` | Fewer total instructions - Shorter code                     | More total instructions - Longer code               |
| `Optimization`                   | Relies on hardware optimization (in CPU)                    | Relies on software optimization (in Assembly)       |
| `Instruction Execution Time`     | Variable - Multiple clock cycles                            | Fixed - One clock cycle                             |
| `Instructions supported by CPU`  | Many instructions (~1500)                                   | Fewer instructions (~200)                           |
| `Power Consumption`              | High                                                        | Very low                                            |
| `Examples`                       | Intel, AMD                                                  | ARM, Apple                                          |
## Registers, Addresses, and Data Types
### Registers
Each CPU core has a set of registers. Fastest component, but limited in size -- They can only hold a few bytes of data at a time.

Many registers in x86. Here are the basics:

|**Data Registers**|**Pointer Registers**|
|---|---|
|`rax`|`rbp`|
|`rbx`|`rsp`|
|`rcx`|`rip`|
|`rdx`||
|`r8`||
|`r9`||
|`r10`|
`Data Registers` : storing instructions / syscall arguments.
`Pointer Registers` : store specific important address pointers.
### Sub-Registers
Each `64-bit` register can be further divided into smaller sub-registers containing lower bits, like 8-bit, 2 bytes, 4 bytes, etc...

Sub-registers can be accessed as:

| Size in bits | Size in bytes | Name                                   | Example |
| ------------ | ------------- | -------------------------------------- | ------- |
| `16-bit`     | `2 bytes`     | the base name                          | `ax`    |
| `8-bit`      | `1 bytes`     | base name and/or ends with `l`         | `al`    |
| `32-bit`     | `4 bytes`     | base name + starts with the `e` prefix | `eax`   |
| `64-bit`     | `8 bytes`     | base name + starts with the `r` prefix | `rax`   |
Another example set could be bx, bl, ebx, rbx.
### Memory Addresses
The following are names of sub-registers in x86_64 architecture.

|Description|64-bit Register|32-bit Register|16-bit Register|8-bit Register|
|---|---|---|---|---|
|**Data/Arguments Registers**|||||
|Syscall Number/Return value|`rax`|`eax`|`ax`|`al`|
|Callee Saved|`rbx`|`ebx`|`bx`|`bl`|
|1st arg - Destination operand|`rdi`|`edi`|`di`|`dil`|
|2nd arg - Source operand|`rsi`|`esi`|`si`|`sil`|
|3rd arg|`rdx`|`edx`|`dx`|`dl`|
|4th arg - Loop counter|`rcx`|`ecx`|`cx`|`cl`|
|5th arg|`r8`|`r8d`|`r8w`|`r8b`|
|6th arg|`r9`|`r9d`|`r9w`|`r9b`|
|**Pointer Registers**|||||
|Base Stack Pointer|`rbp`|`ebp`|`bp`|`bpl`|
|Current/Top Stack Pointer|`rsp`|`esp`|`sp`|`spl`|
|Instruction Pointer 'call only'|`rip`|`eip`|`ip`|`ipl`|
### Address Endianness
Address endianness is the order of its bytes in which they are stored and retrieved.

`Little-Endian` : little-end byte of the address is filled/retrieved first right-to-left.
`Big-Endian` : Big byte is filled/retrieved first left-to-right.

Example:
	`0x0011223344556677` needs to be stored in memory.
	Little-endian, it looks like `0x7766554433221100` when stored. When retrieved it would return the original value.

> We will use little-endian byte order going forward. IE: Our bytes will be stored right-to-left.
### Data Types
The following are the most common data types we will be using with instructions:

|Component|Length|Example|
|---|---|---|
|`byte`|8 bits|`0xab`|
|`word`|16 bits - 2 bytes|`0xabcd`|
|`double word (dword)`|32 bits - 4 bytes|`0xabcdef12`|
|`quad word (qword)`|64 bits - 8 bytes|`0xabcdef1234567890`|
Instructions and data types or vars must be of the same size. The following table shows appropriate data type for each sub-register.

|Sub-register|Data Type|
|---|---|
|`al`|`byte`|
|`ax`|`word`|
|`eax`|`dword`|
|`rax`|`qword`|

# Assembling & Debugging
## Assembly File Structure
This section covers the basics of how assembly files are structured.

>**NOTE:** In assembly code,`;` is a comment.

Hello World!
```nasm
; LABELS        INSTRUCTIONS   OPERANDS
; ---------------------------------------------------------
                global         _start            ; <- DIRECTIVES

; ============================== SECTIONS ==================
                section        .data
message:        db             "Hello World!"

                section        .text
_start:
                mov            rax, 1
                mov            rdi, 1
                mov            rsi, message
                mov            rdx, 18
                syscall

                mov            rax, 60
                mov            rdi, 0
                syscall
```
> The tabs aren't really necessary, more of a visual thing to help understand.

| Section         | Description                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------- |
| `global _start` | This is a `directive` that directs the code to start executing at the `_start` label defined below. |
| `section .data` | This is the `data` section, which should contain all of the variables.                              |
| `section .text` | This is the `text` section containing all of the code to be executed.                               |
Both `.data` and `.text` refer to data and text memory segments.
### Directives
Assembly is line-based. File is processes line-by-line. `global _start` instructs the machine to process instructions after `_start` label. 
### Variables
`.data` section holds our variables to make it easier for us to define variables and reuse them. When program is run, vars are loaded into memory in `data` segment.

Defining variables:
	`db` - list of bytes
	`dw` - list of words
	`dd` - list of digits
	etc...

Examples:

| Instruction                         | Description                                    |
| ----------------------------------- | ---------------------------------------------- |
| `db 0x0a`                           | Defines the byte `0x0a`, which is a new line.  |
| `message db 0x41, 0x42, 0x43, 0x0a` | Defines the label `message => ABC\n`.          |
| `message db "Hello World!", 0x0a`   | Defines the label `message => Hello World!\n`. |


We can use `equ` with `$` to eval an expression, ie: length.
```nasm
section .data
    message db "Hello World!", 0x0a
    length  equ $-message
```

What is happening?
	`equ` defines a constant at assemble time
	`$` = current position
	`message` = a label marking where the string starts (starting address)
	`$-message` subtracts the string's starting address from the current address, giving you the number of bytes in between, which is the length of the string.
### Code
`.text` section holds all the assembly instructions and loads them into the `text` memory segment. 

Default convention is to have the `_start` label at the beginning of the `.text` section. Starts the main code that will be executed as the program runs.
## Assembling & Disassembling
We can use `nasm` to assemble files, and link it using `ld` to utilize various OS features and libraries.
### Assembling & Linking
First put code in file. `.s` and `.asm` extensions are both fine.

helloWorld.s
```nasm
global _start

section .data
    message db "Hello HTB Academy!"
    length equ $-message

section .text
_start:
    mov rax, 1
    mov rdi, 1
    mov rsi, message
    mov rdx, length
    syscall

    mov rax, 60
    mov rdi, 0
    syscall
```

Assemble
```bash
# Assembling
nasm -f elf64 helloWorld.s
	# elf = executable and linkable format
	# -f elf64 assembles in 64-bit 
	# -f elf assembles in 32-bit

# Output
helloWorld.o  # object file -- NOT executable yet
```
> The `helloWorld.o` file that is output that was assembled into machine code. Its not executable yet. Needs to be linked first.

Linking
```bash
# Linking
ld [-m elf_i386] -o helloWorld helloWorld.o
	# -m elf_i386 = 32-bit

# Executing
./helloWorld
	# Hello HTB Academy!
```

A simple script to assemble and link all at once
```bash
#!/bin/bash

fileName="${1%%.*}" # remove .s extension

nasm -f elf64 ${fileName}".s"
ld ${fileName}".o" -o ${fileName}
[ "$2" == "-g" ] && gdb -q ${fileName} || ./${fileName}


# Usage
chmod +x ./assembler.sh
./assembler.sh <file>.s [-g]  # -g auto-opens it in GDB
```
### Disassembling
To disassemble a file, we need `odjdump`. It dumps machine code from a file and interprets the assembly instruction of each hex code.

```bash
# Dissasemble
objdump -M intel -d helloWorld [--no-show-raw-insn --no-addresses]
	# -M : specify instructions in intel syntax
	# -d : dissable what file?
	# --no-show-raw-insn --no-addresses : only show assembly code

# Examine strings and .data
objdump -sj .data helloWorld

# Example stdout
helloWorld:     file format elf64-x86-64

Disassembly of section .text:

0000000000401000 <_start>:
  401000:   b8 01 00 00 00          mov    eax,0x1
  401005:   bf 01 00 00 00          mov    edi,0x1
  40100a:   48 be 00 20 40 00 00    movabs rsi,0x402000
  401011:   00 00 00
  401014:   ba 12 00 00 00          mov    edx,0x12
  401019:   0f 05                   syscall
  40101b:   b8 3c 00 00 00          mov    eax,0x3c
  401020:   bf 00 00 00 00          mov    edi,0x0
  401025:   0f 05                   syscall
```
> **NOTE:** `-d` only disassembles `.text`. To dump strings, use `-s`, and `-j .data` to examine `.data` section
## GNU Debugger (GDB)
Debugging is a term used to find and remove issues from code. We perform debugging by setting breakpoints and seeing how our program acts on each.

With Assembly, we set breakpoints in the memory location in which machine code is loaded.

`GDB` = GNU debugger. There are other debuggers, including:
- Linux: Radare, Hopper 
- Windows: Immunity Debugger, WinGDB 
- Cross-platform: IDA Pro, EDB

Basic Commands
```bash
# Install
sudo apt-get update && sudo apt-get install gdb

# Install GEF (GDB Enhanced Features) Plugin 
wget -O ~/.gdbinit-gef.py -q https://gef.blah.cat/py
echo source ~/.gdbinit-gef.py >> ~/.gdbinit

# Using GDB
gdb -q ./<file>

# Info
info functions   # Check what functions are defined in binary
info variables   # View all available variables within program

# Disassemble
disas <function>
```
>Disassembly shows each instruction's memory address. You'll often see addresses like `0x00000000004xxxxx` instead of raw ones like `0xffffffffaa8a25ff`. This is **`$rip`-relative addressing** in **PIE (Position-Independent Executables)**: addresses are relative to the instruction pointer (`$rip`), not raw memory. PIE can be disabled, but addresses become static and predictable, making the binary easier to exploit.
### Debugging with GDB (GEF)
Summarized by four steps: **Break → Examine → Step → Modify**.

| Step    | What it does                                              |
| ------- | --------------------------------------------------------- |
| Break   | Stop execution at a point of interest                     |
| Examine | Inspect registers / memory at that point                  |
| Step    | Move through the program one instruction at a time        |
| Modify  | Change register/memory values to see how execution reacts |
Break + examine + step + modify lets you see exactly how a program handles input  without editing and recompiling. Modifying registers/memory live is the foundation for testing conditions and developing exploits from those failures.

#### Break
Stop execution at an address or function.
```bash
b _start          # break at a function
b *_start+10      # break at an offset  (* = break at the instruction at this addr)
b *0x40100a       # break at a raw address
r                 # run (starts program from the beginning)
c                 # continue to next breakpoint (use this, NOT r, once running)

info breakpoint   # lists all breakpoints. Can disable / enable / delete them
```
>A breakpoint stops before the instruction at that address runs (so a register shows the value going into that instruction, not the result)

#### Examine
Format: `x/FMT ADDRESS` where FMT = **Count + Format + Size**.

| Part   | Options                                               |
| ------ | ----------------------------------------------------- |
| Count  | number of repeats (`2`, `4`, `10`)                    |
| Format | `x` hex, `s` string, `i` instruction                  |
| Size   | `b` byte, `h` halfword, `w` word, `g` giant (8 bytes) |
```bash
x/4ig $rip        # next 4 instructions ($rip = addr of next instruction)
x/s 0x402000      # examine memory as a string  -> "Hello HTB Academy!"
x/wx 0x401000     # examine as hex word -> 0x000001b8 (little-endian: b8 01 00 00)

registers             # GEF: dump all register values
info registers rax    # print ONE register's value (narrower than `registers`)
p $rax                # same idea; `p`/print also evaluates expressions
```
>If you omit Size or Format, it reuses the last one you used.

#### Step
Current instruction is marked with `→` and has **not** run yet.

```bash
si          # step ONE assembly instruction (stepi)
si 3        # step 3 instructions
s           # step (step) — runs to next line / exits current function;
            #   breaks INTO called functions
n           # next — like s but steps OVER (skips) function calls
ni          # nexti — like si but steps over function calls
<Enter>     # repeat the last command
```

Quick mental model:
	`si` / `ni` = instruction level. `s` / `n` = line level.
	`si`/`s` step **into** calls. `ni`/`n` step **over** calls.
	Running `s` on a finished `_start` runs to program exit (prints output, exits normally).

#### Modify
Change register/memory values at a breakpoint to test behavior without recompiling.
```bash
# GEF patch — write to memory
help patch                              # syntax reference
patch string 0x402000 "Patched!\x0a"    # \x0a = newline

# set — modify a register
set $rdx=0x9
```

Worked example (change the printed string):
```bash
break *0x401019                         # break at the write syscall
r
patch string 0x402000 "Patched!\x0a"    # overwrite the .data string
set $rdx=0x9                            # fix length: rdx holds the write count
c                                       # -> prints "Patched!"
```

> **NOTE:** Patching only overwrites up to your new string's length; leftover bytes of the old string remain unless you also fix the length. The write syscall printed `0x12` bytes, so `rdx` had to be set to the new length (`0x9`) to avoid trailing garbage.

# Basic Instructions
# Control Instructions
# Functions
# Shellcoding
# Cheatsheet
