C is a low-level, compiled, statically-typed language with manual memory management and no runtime safety net. It maps closely to the hardware: you manage memory yourself, pointers are everywhere, and the standard library is small. 

> Focus is C99/C11 with `gcc`/`clang` on Linux. Compile with warnings on (`-Wall -Wextra`) and symbols (`-g`) so your binaries debug cleanly in GDB/Ghidra.
## Contents

- [Toolchain and Anatomy](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#toolchain-and-anatomy)
    - [Compilation](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#compilation)
    - [Program Structure](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#program-structure)
    - [Common Includes](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#common-includes)
- [Language Fundamentals](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#language-fundamentals)
    - [Comments and Literals](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#comments-and-literals)
    - [Variables and Typing](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#variables-and-typing)
    - [Operators](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#operators)
    - [Increment and Decrement (i++ vs ++i)](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#increment-and-decrement-i-vs-i)
    - [lvalue vs rvalue](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#lvalue-vs-rvalue)
    - [Computing and Comparisons](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#computing-and-comparisons)
- [Control Flow](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#control-flow)
    - [Switch](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#switch)
    - [Loops](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#loops)
- [Functions and Scope](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#functions-and-scope)
    - [Functions](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#functions)
    - [Command Line Arguments](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#command-line-arguments)
    - [Scope and Lifetime](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#scope-and-lifetime)
    - [Storage Classes and Qualifiers](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#storage-classes-and-qualifiers)
- [Pointers, Arrays, and Memory](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#pointers-arrays-and-memory)
    - [Pointers](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#pointers)
    - [Arrays](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#arrays)
    - [Strings (char arrays)](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#strings-char-arrays)
    - [Function Pointers](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#function-pointers)
    - [Dynamic Memory (heap)](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#dynamic-memory-heap)
- [Composite Types](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#composite-types)
    - [Structs](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#structs)
    - [Struct Padding and Alignment](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#struct-padding-and-alignment)
    - [Unions](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#unions)
    - [Enums](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#enums)
- [Error Handling and I/O](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#error-handling-and-io)
    - [Errors and Return Codes](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#errors-and-return-codes)
    - [File I/O](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#file-io)
- [Preprocessor and Build](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#preprocessor-and-build)
    - [The Preprocessor](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#the-preprocessor)
    - [Header Files (multi-file projects)](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#header-files-multi-file-projects)
- [Low-Level and Offensive](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#low-level-and-offensive)
    - [Integer Pitfalls](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#integer-pitfalls)
    - [Endianness and Byte Packing](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#endianness-and-byte-packing)
    - [Format String Vulnerabilities](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#format-string-vulnerabilities)
    - [Inspecting Compiled Binaries](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#inspecting-compiled-binaries)
    - [Memory and RE Notes](https://claude.ai/chat/73e52315-3eaf-4902-8c9b-843830a4c89d#memory-and-re-notes)

---

## Toolchain and Anatomy

### Compilation

```bash
gcc -Wall -Wextra -g prog.c -o prog   # warnings + debug symbols
gcc -g -O0 prog.c -o prog             # no optimization, cleanest for RE
gcc -m32 prog.c -o prog               # 32-bit build (early exploit-dev)
gcc -static prog.c -o prog            # static link, no shared libs
gcc -c prog.c                         # compile to prog.o, no link
./prog                                # run
```

### Program Structure

```c
/* smallest complete program */
#include <stdio.h>             // declarations for printf
int main(void) {              // entry point; execution starts here
    printf("Hello, world\n"); // \n = newline
    return 0;                 // 0 = success to the OS, nonzero = error
}
// build & run:  gcc hello.c -o hello && ./hello

/* anatomy */
// #include   preprocessor line, runs before compilation
// int        main's return type (the exit code)
// (void)     takes no arguments
// { ... }    a block; statements run top to bottom
// ;          ends every statement (not blocks, not # directives)
```

### Common Includes

```c
#include <stdio.h>      // printf, scanf, FILE, fopen
#include <stdlib.h>     // malloc, free, atoi, exit, rand
#include <string.h>     // strcpy, strlen, strcmp, memcpy, memset
#include <unistd.h>     // read, write, close, getpid (POSIX)
#include <stdint.h>     // fixed-width types: uint8_t, int32_t, uint64_t
#include <stdbool.h>    // bool, true, false (C99)
```

## Language Fundamentals

### Comments and Literals

```c
/* comments */
// single-line (C99+)
/* multi-line */

/* integer literals */
42        // decimal
0x2A      // hex (= 42)
052       // octal, leading 0 (= 42); watch accidental leading zeros
0b101010  // binary (GCC/Clang ext, = 42)
42U  42L  42UL  42LL      // unsigned, long, unsigned long, long long

/* float / char / string literals */
3.14   3.14f   1e3        // double, float, 1000.0
'A'                       // char (single quotes), 65 in ASCII
"text"                    // string (double quotes), hidden '\0' at the end

/* escape sequences (inside '' or "") */
'\n' newline   '\t' tab   '\0' null   '\\' backslash
'\'' quote   '\"' dquote  '\r' return  '\x41' hex byte (= 'A')
```

### Variables and Typing

```c
/* static typing: every variable has a fixed declared type */
char    c   = 'A';         // 1 byte
int     count = 10;        // usually 4 bytes
long    big  = 100000L;    // 8 bytes on 64-bit Linux
float   pi   = 3.14f;      // 4-byte float
double  precise = 3.14159; // 8-byte float
bool    active = true;     // needs <stdbool.h>

/* fixed-width (<stdint.h>): use when size matters (RE, protocols, exploits) */
uint8_t  byte_val  = 0xFF; // exactly 1 byte, unsigned
int32_t  signed_32 = -1;   // exactly 4 bytes, signed
uint64_t addr      = 0x401136;

/* const: cannot change after init */
const int MAX = 100;

/* casts: explicit, C does not protect you */
int    x = (int)3.99;      // 3 (truncates)
double d = (double)10 / 3; // 3.333 (cast before divide or you get int division)

/* printf format specifiers */
printf("%s id:%d ptr:%p hex:%#x\n", name, id, (void*)p, 0xdead);
// %d int  %u unsigned  %ld long  %f double  %c char  %s string
// %p pointer  %x hex  %#x hex w/ 0x  %zu size_t  %% literal percent
```

### Operators

```c
/* arithmetic */
+  -  *  /  %                 // % = modulo (remainder), ints only

/* assignment (compound = do op, then assign) */
=  +=  -=  *=  /=  %=         // x += 3  is  x = x + 3
&=  |=  ^=  <<=  >>=

/* increment / decrement */
++  --                       // see i++ vs ++i below

/* comparison (result is 0 or 1) */
==  !=  <  >  <=  >=

/* logical (short-circuit) */
&&  ||  !                    // && stops if left false; || stops if left true

/* bitwise */
&  |  ^  ~  <<  >>           // AND OR XOR NOT shl shr

/* other */
?:          // ternary:  cond ? if_true : if_false
sizeof      // bytes of a type/object:  sizeof(int), sizeof x
&  *        // address-of, dereference
.  ->       // struct member (value, pointer)
[]          // subscript:  arr[i] == *(arr + i)
(type)      // cast

/* precedence, high to low (the trip-ups) */
// ()  []  .  ->
// !  ~  ++  --  (type)  *  &  sizeof
// *  /  %   then  +  -
// <<  >>   then  <  <=  >  >=   then  ==  !=
// &   then  ^   then  |
// &&  then  ||   then  ?:   then  =  +=
// when unsure, PARENTHESIZE:  `a & b == c`  means  `a & (b == c)`
```

### Increment and Decrement (i++ vs ++i)

```c
/* same change (+1), different returned value */
int i = 5;  int a = i++;   // POST: use old, then +1.   a=5, i=6
int j = 5;  int b = ++j;   // PRE:  +1, then use new.    b=6, j=6
// on their own line, i++; and ++i; are identical

/* UB: two writes to one variable with no sequence point */
int oops = i++ + i++;      // undefined
a[i] = i++;                // undefined
// one side effect per variable per statement
```

### lvalue vs rvalue

```c
// lvalue: has an address; can sit LEFT of =, works with &
// rvalue: a temporary value, no address; right side only
int x = 10;    // x lvalue, 10 rvalue
x = 20;        // ok
int *p = &x;   // ok, x has an address

/* these fail: left side has no address */
// 10 = x;        // literal
// (x + 1) = 5;   // computed temporary
// &x = p;        // computed address
// "lvalue required..." = you assigned to something with no address.
// lvalues: variables, arr[i], *p.   rvalues: literals, arithmetic, most returns.
```

### Computing and Comparisons

```c
/* integer division truncates, no automatic float */
int    result = 10 / 3;     // 3 (not 3.33)
int    rem    = 10 % 3;     // 1
double real   = 10.0 / 3;   // 3.33 (one operand must be float)

/* truthiness: 0 is false, ANY nonzero is true */
if (ptr) { }                // true when ptr is non-NULL

/* bitwise flags (everywhere in low-level work) */
uint8_t flags = 0;
flags |= (1 << 2);          // set bit 2
flags &= ~(1 << 2);         // clear bit 2
if (flags & (1 << 2)) { }   // test bit 2
```

## Control Flow

### Switch

```c
switch (opt) {
    case 1:
        puts("one");
        break;              // no break = fall through into the next case
    case 2:
    case 3:                 // stacked: 2 or 3 do the same
        puts("two or three");
        break;
    default:
        puts("other");
}
// value must be int/char/enum; labels are constants.
// forgetting break is the #1 switch bug; comment intentional fallthrough.
```

### Loops

```c
/* for: the workhorse */
for (int i = 0; i < 5; i++) printf("Attempt %d\n", i);

/* while */
int n = 0;
while (n < 3) { puts("check"); n++; }

/* do-while: body runs at least once */
int v;
do { v = get_next(); } while (v != 0);

/* no foreach: iterate with an index + known length */
char *targets[] = {"10.0.0.1", "10.0.0.2"};
int len = sizeof(targets) / sizeof(targets[0]);   // array-length idiom
for (int i = 0; i < len; i++) printf("Ping %s\n", targets[i]);
// break exits the loop; continue skips to the next iteration.
```

## Functions and Scope

### Functions

```c
int pwn_check(char *target, int port);   // prototype (often in a .h)

int pwn_check(char *target, int port) {  // definition
    if (strcmp(target, "127.0.0.1") == 0) return -1;  // skip localhost
    printf("Scanning %s:%d\n", target, port);
    return 0;
}

void banner(void) { puts("[*] starting"); }  // void = no return / no args
// C has NO default args and NO overloading. One name, one signature.
```

### Command Line Arguments

```c
// argc = count, argv = array of strings. argv[0] is the program name.
int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <target_ip>\n", argv[0]);
        return 1;
    }
    char *target = argv[1];
    int   port   = (argc > 2) ? atoi(argv[2]) : 80;  // atoi: string -> int
    printf("Scanning %s:%d\n", target, port);
    return 0;
}
```

### Scope and Lifetime

```c
// scope = where a name is visible. lifetime = how long its memory lives.
int g = 1;                 // global: file scope, whole-program lifetime

void f(void) {
    int local = 2;         // block scope: dies when f returns
    {
        int inner = 3;     // visible only in these braces
        int local = 99;    // shadows the outer 'local' here
        printf("%d\n", local);   // 99
    }
    // inner gone; local is 2 again
}
// locals live on the stack and vanish at block exit.
// a static local keeps its value across calls (see Storage Classes).
// shadowing (inner name hides outer) is legal and a common bug.
int *bad(void) { int t = 5; return &t; }   // returns a dangling pointer, bug
```

### Storage Classes and Qualifiers

```c
/* static (file scope): internal linkage, invisible to other .c files */
static int secret = 0;

/* static (local): persists across calls, NOT reset each time */
int counter(void) { static int n = 0; return ++n; }   // 1, 2, 3, ...

/* extern: declare a global defined in another .c file */
extern int g_config;

/* const: cannot change through this name (not armor) */
/* volatile: never cache in a register, reread memory every access */
volatile uint32_t *reg = (uint32_t*)0x40021000;   // MMIO / hardware registers
*reg = 1;   // actually hits hardware, not optimized away

/* register: hint to keep in a CPU register (mostly ignored today) */
```

## Pointers, Arrays, and Memory

### Pointers

```c
// a pointer holds an ADDRESS. T *p; declares a pointer to T.
// &x = address-of    *p = value-at (dereference)
int  x = 5;
int *p = &x;        // p holds x's address
printf("%d\n", *p); // 5
*p = 42;            // write through p; x is now 42

/* pass by reference: C copies by default, pass a pointer to modify the original */
void increment(int *np) { (*np)++; }
int a = 5;  increment(&a);   // a = 6

/* NULL = points to nothing; dereferencing it segfaults */
int *q = NULL;
if (q) *q = 1;      // always check before deref

/* pointer arithmetic moves in units of the pointed-to TYPE, not bytes */
int arr[] = {10, 20, 30};
int *ap = arr;              // points at arr[0]
printf("%d\n", *(ap + 1));  // 20

void *raw = &x;    // void* = generic pointer; cast before use
int  *ip  = (int*)raw;
char **argv;       // pointer to pointer (argv, and common in RE)
```

### Arrays

```c
// fixed size, set at declaration. NO bounds checking: overrun = UB.
int primes[] = {2, 3, 5, 7, 11};   // size inferred (5)
int nums[5]  = {0};                // all zero
int first    = primes[0];

/* arrays decay to a pointer when passed; the length does NOT travel with them */
void print_all(int *arr, int len) {
    for (int i = 0; i < len; i++) printf("%d\n", arr[i]);
}

int grid[2][3] = {{1,2,3}, {4,5,6}};   // 2D
int val = grid[1][2];                  // 6

// length idiom works on a REAL array, not a decayed pointer:
int count = sizeof(primes) / sizeof(primes[0]);
```

### Strings (char arrays)

```c
// a C string is a char array ending in '\0'. There is no string type.
char  name[] = "jell0man";   // 9 bytes: 8 chars + '\0'
char *lit    = "readonly";   // points at a literal, do NOT modify
size_t slen  = strlen(name); // 8 (excludes '\0')

/* <string.h> essentials */
strcpy(dst, src);               // UNSAFE: no bounds check
strncpy(dst, src, sizeof(dst)); // bounded, but may not null-terminate
strcat(dst, src);               // UNSAFE
strcmp(a, b);                   // 0 if equal
memcpy(dst, src, len);          // raw byte copy, no '\0' logic
memset(buf, 0, sizeof(buf));    // zero a buffer

/* the classic bug: fixed buffer + unbounded copy = stack smash (OSED) */
void vuln(char *input) {
    char buf[64];
    strcpy(buf, input);   // input > 63 chars overruns the stack
}
/* safe: bound it and terminate yourself */
void safe(char *input) {
    char buf[64];
    strncpy(buf, input, sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0';
}
```

### Function Pointers

```c
// holds the ADDRESS of code. core to callbacks, dispatch tables, and (your lane)
// GOT/PLT entries and C++ vtables.
int add(int a, int b) { return a + b; }
int (*fp)(int, int) = add;   // "fp: pointer to fn(int,int) returning int"
int r = fp(2, 3);            // call through it (== (*fp)(2,3)) -> 5

typedef int (*binop_t)(int, int);   // readable alias

/* dispatch table (interpreters, syscall tables, plugins) */
int sub(int a, int b) { return a - b; }
binop_t ops[] = {add, sub};
int y = ops[1](10, 4);       // 6

/* callback (qsort takes a comparator) */
int cmp(const void *a, const void *b) { return *(int*)a - *(int*)b; }
qsort(arr, n, sizeof(int), cmp);

// RE tie-in: an indirect CALL (call rax, call [rip+off]) is a function pointer
// firing. overwrite one (GOT, struct callback, vtable slot) and you redirect
// control flow. that is the primitive behind many exploits.
```

### Dynamic Memory (heap)

```c
// stack memory is automatic; heap memory you manage by hand.
// every malloc needs a free. leak / double-free / use-after-free = corruption.
int *buf = malloc(10 * sizeof(int));
if (!buf) { perror("malloc"); return 1; }   // malloc returns NULL on failure
buf[0] = 42;                                 // use like an array
free(buf);                                   // release
buf = NULL;                                  // avoid a dangling pointer

int *z = calloc(10, sizeof(int));     // like malloc but zeroed
buf = realloc(buf, 20 * sizeof(int)); // resize

// rules: malloc/calloc <-> free (1:1). never free twice, use after free, or
// free the stack.
```

## Composite Types

### Structs

```c
struct target {
    char ip[16];
    int  port;
    bool is_up;
};

/* instantiate + access with . */
struct target t = {.ip = "10.10.10.5", .port = 443, .is_up = true};
printf("%s:%d\n", t.ip, t.port);

/* pointer to struct uses -> (shorthand for (*p).field) */
struct target *tp = &t;
tp->port = 8080;

/* typedef drops the 'struct' keyword */
typedef struct { char ip[16]; int port; } host_t;
host_t h = {"192.168.1.1", 22};

/* pass a POINTER to avoid copying and to modify the original */
void set_port(host_t *host, int port) { host->port = port; }
```

### Struct Padding and Alignment

```c
// the compiler inserts gaps so each field lands on its natural alignment.
// sizeof(struct) is usually LARGER than the sum of its fields.
struct s {
    char a;   // offset 0
              // 3 bytes padding
    int  b;   // offset 4 (int wants 4-byte alignment)
    char c;   // offset 8
              // 3 bytes trailing padding
};            // sizeof == 12, not 6

// reorder fields largest-to-smallest to shrink structs.
// in RE, wrong offsets = wrong field reads: always account for padding.
#pragma pack(push, 1)
struct wire { uint8_t op; uint32_t len; };   // sizeof == 5, no padding
#pragma pack(pop)

#include <stddef.h>
size_t off = offsetof(struct s, b);   // 4; handy for matching decompiler output
```

### Unions

```c
// all members overlay the SAME memory; size = largest member.
// writing one clobbers the others. used for type punning and parsing raw bytes.
union pun {
    uint32_t u32;
    uint8_t  bytes[4];
    float    f;
};
union pun p;
p.u32 = 0x41424344;
printf("%c%c\n", p.bytes[0], p.bytes[3]);   // byte view depends on endianness

/* tagged union: read a header, branch on a type field */
struct packet {
    uint8_t type;
    union {
        struct { uint32_t addr; }    a;
        struct { char     msg[16]; } b;
    } body;
};
```

### Enums

```c
enum state { DOWN, UP, UNKNOWN };   // auto: DOWN=0, UP=1, UNKNOWN=2
enum state st = UP;
enum proto { TCP = 6, UDP = 17 };   // explicit values
typedef enum { DEPOSIT, WITHDRAWAL } txn_t;   // typedef form
txn_t tx = DEPOSIT;
```

## Error Handling and I/O

### Errors and Return Codes

```c
// no exceptions. signal failure via return values + errno.
// convention: 0 / valid pointer on success, nonzero / NULL on failure.
FILE *f = fopen("targets.txt", "r");
if (!f) {
    fprintf(stderr, "fopen: %s\n", strerror(errno));   // errno -> text
    return 1;
}
fclose(f);

int fd = open("/etc/shadow", O_RDONLY);   // needs <fcntl.h>
if (fd < 0) perror("open");               // prints "open: <errno text>"
exit(1);                                  // 0 success, nonzero error
```

### File I/O

```c
/* read line by line (fgets is bounded, safe) */
FILE *f = fopen("targets.txt", "r");
if (f) {
    char line[256];
    while (fgets(line, sizeof(line), f)) printf("target: %s", line);
    fclose(f);
}

/* write ("w" truncate, "a" append, "r" read) */
FILE *out = fopen("loot.txt", "w");
if (out) { fprintf(out, "pwned %s\n", "DC01"); fclose(out); }

/* raw byte read (POSIX, syscall-level) */
int fd = open("file.bin", O_RDONLY);   // <unistd.h>, <fcntl.h>
char buf[512];
ssize_t nread = read(fd, buf, sizeof(buf));   // bytes read, -1 on error
close(fd);
```

## Preprocessor and Build

### The Preprocessor

```c
// runs BEFORE compilation: text substitution, no type checking.
// every directive starts with # and takes NO semicolon:
//   #include #define #undef #if #ifdef #ifndef #elif #else #endif
//   #pragma #error #line

/* includes */
#include <stdio.h>     // <...> = system headers
#include "myheader.h"  // "..." = current dir first

/* macros */
#define MAX 100              // object-like (constant)
#define SQ(x) ((x) * (x))    // function-like: ALWAYS parenthesize args
int y = SQ(3 + 1);           // (3+1)*(3+1) = 16
#undef MAX                   // remove a definition
#define STR(x)   #x          // # stringizes:    STR(hi)      -> "hi"
#define CAT(a,b) a##b        // ## token-pastes: CAT(foo,bar) -> foobar

/* conditional compilation */
#ifdef DEBUG
    printf("[dbg] %d\n", x);
#endif
#if defined(__linux__)
    // linux-only
#elif defined(_WIN32)
    // windows-only
#endif

/* include guard (or just #pragma once at the top of the header) */
#ifndef MYHEADER_H
#define MYHEADER_H
// declarations
#endif
```

### Header Files (multi-file projects)

```c
/* scanner.h: the interface (declarations) */
#ifndef SCANNER_H
#define SCANNER_H
int scan_port(char *ip, int port);   // prototype only
#endif

/* scanner.c: the implementation */
#include "scanner.h"
int scan_port(char *ip, int port) { /* ... */ return 0; }

/* main.c: uses it */
#include "scanner.h"
int main(void) { return scan_port("10.0.0.1", 80); }

// build: compile each .c, then link
//   gcc -c scanner.c -o scanner.o
//   gcc -c main.c    -o main.o
//   gcc scanner.o main.o -o scanner
```

## Low-Level and Offensive

### Integer Pitfalls

```c
// integer bugs are a vuln class of their own; the hardware wraps silently.

/* overflow / wraparound (unsigned wraps mod 2^N; signed overflow is UB) */
uint8_t u = 255; u++;       // 0
int total = a + b;          // can go negative and poison size checks

/* signed/unsigned compare: the signed operand converts to unsigned */
int i = -1; unsigned m = 1;
if (i < m) { }              // FALSE: -1 becomes 0xFFFFFFFF > 1

/* truncation: wider -> narrower drops high bits */
uint32_t wide   = 0x1FF;
uint8_t  narrow = wide;     // 0xFF (top bits gone)

/* the classic shape: signed check passes, huge unsigned length used */
if (len < BUFSZ) memcpy(dst, src, len);   // negative/huge len -> overflow
```

### Endianness and Byte Packing

```c
// x86/x86-64 and ARM (default) are little-endian: least significant byte at the
// lowest address. network byte order is big-endian.
uint32_t v = 0x11223344;
uint8_t *b = (uint8_t*)&v;   // b[0]=0x44 b[1]=0x33 b[2]=0x22 b[3]=0x11

// why you care: an address written into a payload goes in reversed.
// return addr 0x401136 -> bytes 36 11 40 00   (Python: struct.pack("<I", 0x401136))
// byte-order mistakes are the #1 reason a paper-correct overflow fails to land.

uint32_t net = htonl(v);   // host -> network (big-endian), <arpa/inet.h>
uint16_t pt  = htons(443);
```

### Format String Vulnerabilities

```c
printf(user_input);        // VULN: "%x %x %s" walks/leaks the stack
printf("%s", user_input);  // SAFE: data is an argument, not the format

// %x / %p  leak stack memory (infoleak, defeat ASLR)
// %n       WRITES bytes-printed-so-far to a pointed-at address = arbitrary write
// rule: the format string is a constant YOU control, never attacker input.
```

### Inspecting Compiled Binaries

```bash
file ./prog                 # type, arch, stripped?, PIE?
checksec --file=./prog      # RELRO, canary, NX, PIE (which mitigations)
nm ./prog                   # symbol table (empty if stripped)
strings -n 6 ./prog         # printable strings (creds, paths, format strings)
objdump -d -M intel ./prog  # disassemble .text, Intel syntax
readelf -a ./prog           # ELF headers, sections, dynamic symbols
ltrace ./prog               # trace library calls
strace ./prog               # trace syscalls
gdb -q ./prog               # debug (pair with GEF or pwndbg)
```

### Memory and RE Notes

```c
// stack: locals, args, return addresses. grows DOWN. freed on return.
//        overrunning a stack buffer can overwrite the saved return address:
//        classic stack smashing (OSED).
// heap:  malloc/free. grows UP. you manage it. use-after-free, double-free,
//        and heap overflow are their own exploit classes.
// .text  code (r-x)   .data  init globals   .bss  uninit globals (zeroed)
//
// Ghidra's decompiler output approximates C: reading C fluently = reading
// decompiled output fluently. compile with `gcc -g -O0`, open in Ghidra, and
// match your source to the decompiler view.
```