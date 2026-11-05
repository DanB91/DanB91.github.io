---
title: "A proposal for a 6502-specific modern programming language"
date: 2026-11-05
draft: false
---
I've been working on a project for the Commodore 64 Ultimate.  For this project, I’ve been writing in 6502 assembly, which has been fun. But I sometimes do miss having things like if-statements, for-loops and structs.  Which got me thinking… I’m not programming on the C64U itself.  I am programming on my Macbook Pro.  Why couldn’t we use a higher-level language?

Of course, the obvious answer here is just use C or something like Prog8. And there are more than a few C compilers for the 6502 and some try to squeeze as much performance out of the 6502 as they can (shout out to OSCAR64). But, in my opinion, no matter how much you optimize your compiler, C just isn't a great match for the 1-workhorse-register, 256-byte-stack, 8-bit CPU. 

So what if of instead of taking a high-level language and trying to make it fit on a limited system, you take the limited system and make a high-level language from that?

**Disclaimer: I don't plan on writing a compiler for this hypothetical language. I am not interested in writing and maintaining a compiler in my free time. I'll just be the “idea guy” here.  I also have never designed a language or written a compiler either, so if you see something that doesn’t make sense… it probably doesn’t (and please say so in the comments).**

**Having said that, if you are interested in writing such a compiler I am definitely open to collaborating in some way!**

### Goals

For this language I’d like to have:

- Control flow structures like if-else statements and for loops
- Type-safety
- Procedures/functions with ability to specify input/output registers
- Explicit control of 6502 registers
- Automatic detection of register clobbers
- Compile error if trying to use clobbered register right after calling such a procedure
- Pointer and array types that map cleanly to 6502’s limited indirect addressing modes
- Structs
- Lexically scoped constants/variables
- Multiple encoding options for strings and characters including PETSCII, ASCII, and C64 screen codes.
- Automatic global and zero-page memory allocation.
- Manual global and zero-page memory allocation.
- Explicit control of the memory map.
- Compile errors for memory segments that overlap but not supposed to
- Compiler errors for data that overflows its predefined memory segment
- Easily translatable to assembly in my head.
- Strong guarantees for what assembly is outputted.  That is, no non-deterministic compiler optimizations.
- Tailored for the 6502

I want to emphasize the last 2 points, which I believe would set this language apart from other modern systems programming languages. 

### Strong guarantees for assembly output

Most (all?) modern languages do not guarantee what exact assembly is outputted.  The reason for this is that the compiler is free to optimize the assembly as it sees fit. And, the reason for this is that modern CPUs have on the order of 1000 instructions, have pipelines, memory caches, branch predictors, multithreading, etc. You’d have to consider all of these crazy, complicated things to have optimal assembly.  The 6502 has none of that.

It is for this reason that I think we can have a high level language where you can intuitively know the assembly being outputted.  Of course, the tradeoff is that the compiler cannot optimize the assembly.  But with a language tailored for the 6502, I feel that is a tradeoff worth making. Especially, since the resulting assembly would be pretty optimal already (though, not perfect).

### Tailored for the 6502

Most, if not all, systems programming languages want to target different CPUs.  But for this language, the goal is to solely support the 6502.  In theory, it could easily be extended for the 65C02 as well.  Thus, I am free to make any decisions that make sense for the 6502, but not for any other CPU.

### Choosing a high-level language structure

First we need to start somewhere.  We need some sort of basic language structure to build from. Luckily there are a seemingly infinite amount of new programming languages these days, so it’d be best to just steal one.

I've also been on and off working on translating BoksOS from Zig to Odin. One of the things about Odin I like is its very simple Pascal-based structure, so I feel this is a perfect language to start from.

I do want to note that Zig or JAI (when it comes out) would also be great bases as well. In fact, unlike Odin, they can run arbitrary code at compile time, which I feel many people would want for this language. I mostly choose Odin here because it's fresh in my mind, there are some features like `bit_set` I like, and, well, I enjoy writing in it more.

Even though people reading this may not know Odin, I am not going to go too deep into it.  If you’re familiar with any systems programming language like C or C++, you should be able to follow along.  Though, I highly encourage you to read the [Odin overview](https://odin-lang.org/docs/overview/).

### What to keep and what to change from Odin

I feel we can keep many of Odin’s features and semantics unchanged.  This includes niche features like the `using` keyword and the `bit_set` . Though, there are several things that I’d drop because the 6502 doesn’t natively support them.  Some of these things are:

- Most pointer semantics except for the cases I go over later in the **Pointers** section. Basic things like `string` and `^MyStruct` wouldn’t be supported. When I first started programming for the 6502, I was surprised by how limited the native support for indirect addressing is.
- Any semantics that deal with floating point.
- Array programming semantics
- Zero initialization as default

### This post is incomplete

I had originally planned to flush out each and every idea I had for this language. I got pretty far with this, but this post was taking way longer to write that I’d like. As such, there are several pieces to this language I haven’t explicitly defined.  I typically wouldn’t release an unfinished post like this. But, in this case, I feel confident what’s here should give a pretty decent idea of what I am going for.

### Hello World

Without further ado, here is a heavily commented example C64 program for printing “HELLO WORLD!” to the screen:

```odin
package main

//The .Program segment type specifies that any object within this segment is stored into the resulting executable or ROM file
PROGRAM_SEGMENT :: Segment{.Program, 0x800..<0x2000}
KERNAL_SEGMENT :: Segment{.External, 0xE000..<=0xFFFF}
RAM_SEGMENT :: Segment{.RAM, 0x2..<0x100, 0x2000..<0x8000}

with_memory_map(PROGRAM_SEGMENT, RAM_SEGMENT, KERNAL_SEGMENT)
{

//Required BASIC stub which tells C64 BASIC to run the procedure at address 0x810 (in this case, main)
//It would automatically allocated at 0x800 since that's where the program
//segment starts
_ = []byte{0, 0xE, 0x8, 0xA, 0x0, 0x9E}
_ = p" (2064)" //p"" strings are PETSCII strings
_ = []byte{0,0,0}

//Even though this is declared before main, it'd be layed out in memory
//after main because there is no room in between the BASIC stub and main()
//If we had specifed a memory address at, say 0x810, you should get a compile error
//since the procedures would overlap
print :: proc(str: [^]petscii, str_len: byte) {
    
    CHROUT :: proc(A) @ 0xFFD2 //0xFFD2 is within the KERNAL_SEGMENT, which is External. 
                                                    //External means don't actually put anything at this address. Just reference it
                                                     //If there was a procedure body here, that would be a compiler error since you can't
                                                     //put any data in a segment marked External
    
    //A and Y here really refer to the A and Y registers
    for Y = 0; Y < str_len; Y++ {
        A = str[Y] //This corresponds to the indirect-indexed addressing mode. 
                             //Like in 6502 assembly, Y must be the index,
                             //else you'd get a compile error
        CHROUT(A)
    }
}

//Here we manually specify main to be layed out at 0x810
main :: proc() @ 0x810  {
  //this is allocated in the PROGRAM_SEGMENT since it's static initialzed data
  @(static)
    hello := p"HELLO WORLD!" 
    
    //len(hello) is compile-time known, so can be treated like a constant
    print(&hello, len(hello) via A)
    
    //hang forever at end of program
    for {}
}

}

```

This would translate to this 6502 assembly:

```asm

CHROUT = $FFD2
HELLO_LEN = hello_end - hello ;outputted from len(hello)

;PROGRAM_SEGMENT
.org $800

;BASIC stub
.db 0, $E, $8, $A, 0, $9E
.tx " (2064) "
.db 0, 0, 0

.org $810
main:
  lda #(<hello)
  sta str
  lda #(>hello)
  sta str+1
  
  lda #HELLO_LEN
  sta str_len
  
  jsr print
    
    
    ;hang forever at end of program
hang:
    jmp hang
    

print: 
  ldy #0

loop:
    cpy str_len
    beq exit
    lda (str),y
    jsr CHROUT
    iny
    jmp loop
exit:
    
    rts
    
hello: .tx "HELLO WORLD!"
hello_end:

.org 2
str: .res 2 ;Since str is a [^]petscii, it must be allocated on the zero page.
                        ;Pointers are 2 bytes big
                         

.org $2000            
str_len: .res 1 ;This is allocated the first free address
                        

```

As you can see, there's a lot to explain just in this small program. I'll break it down by concept

### Primitive Types

There are a handful of types and type-related features that would be supported in this language:

- `byte` — 8-bit value.  Can be signed or unsigned.  Can have value from -128 to 255. Signedness only matters when doing comparisons.
- `petscii` — Same as byte, but meant for a single PETSCII character.
- `ascii` — Same as byte but meant for single ASCII character.
- `word` — 16-bit value. Same as `[2]byte` .  Taking the address of any in-memory variable (e.g. `&my_var`) can implicitly convert to a `word` .
- `A`, `X`, `Y`, `S` , `P` — These correspond to the 6502 registers. Registers are both a type and a variable. When defining aliases, the register names are used as a type.

Notice how there aren't any signed and unsigned types. Since the 6502 can only do addition and subtraction 2’s complement style, the only time we need to care about signedness is when we are doing comparisons.

### Where in memory do global variables live?

So, this was trickier to answer than I expected…  

In a modern systems programming language on a modern computer, there are actually 2 types of global variables:

- Uninitialized globals (e.g.: `num: byte`) — These in a zeroed-out portion of memory known strangely as the `bss` segment. This is a segment of memory that is not included with the program executable and is zeroed out when the program starts.
- Initialized globals (e.g. `num: byte = 5`) — These go in something more aptly called the `data` segment.  This segment of memory is included with the program executable.  So, the more memory you need for initialized globals, the larger the program executable becomes.  There is also the `rodata` segment which is the same as `data` , but read-only.

For the rest of this post I will refer to what lives in the `data` segment as **program memory** and what lives in `bss` as **working RAM**.

The crucial thing here is that the distinction between these 2 types of globals are typically invisible to the programmer.  As far as the programmer is concerned `num: byte = 5` is functionally equivalent to declaring `num: byte` and then doing `num = 5` in some initialization procedure. 

For 6502 programs, this works for programs that are loaded and ran from RAM (e.g. programs distributed on floppy disk or tape). But this becomes an issue for programs that are distributed ROM cartridges (e.g. NES games, C64 cartridges).  All the sudden, all the variables that live in the program memory are now read-only! 

This means that for `num: byte = 5` , `num` would be immutable, but if `num` is declared as `num: byte`, `num` would be mutable.  This feels bad.

But honestly, I don’t see a better solution than to just to live with this, because if you’re working on a ROM-resident program, I feel this is just a weird quirk you can get used to quickly, and anything else I can think of (e.g. forcing a `readonly` keyword on every declaration, or having a special assignment operator like `::=`) just feels like it complicates the language too much for the benefits they give.

With respect to uninitialized globals, you probably don’t want to require that they are zeroed since that is pretty expensive on the 6502.  Especially if you’re on a system that is using banked memory. Uninitialized globals on the 6502 should actually be uninitialized.

So to finally answer this question:

- Uninitialized globals live in uninitialized working RAM. They can start with any value and are always mutable.
- Initialized globals live in program memory.
    - If program memory is RAM, then the initialized global is mutable, as expected.
    - If program memory is ROM, then the initialized global is immutable, enforced by the compiler.

### Where in memory do procedure-local variables and parameters live?

In a modern systems programming language, local variables and procedure parameters live on the stack. The problem the 6502, though, is its stack sucks.  It’s hardwired to be 256 bytes big and the stack is always located at 0x100 to 0x1FF.  It is for this reason that I feel its best for local variables and parameters to actually live in global memory, rather than on the stack. More specifically, they should either live in program memory, or in working RAM. There are a few consequences to this:

- The compiler should not allow recursion.
- The compiler should not allow circular dependent procedures. E.g. `f()` can call `g()` , and `g()` can call `f()`

I’d happily give up recursion (which I rarely use anyway) in order to not have to worry about overflowing a tiny stack.

As I have not written a compiler, I am not 100% sure, but I am pretty sure both of these cases can be detected and disallowed by the compiler.

However, for those who *really* want recursion or circular dependent procedures, perhaps you can have something like like `recursable_proc` that will allocate all local variables on the stack and push any used registers onto the stack.  But, for the rest of post, I will assume all local variables are allocated in global memory.

Like for global variables, all uninitialized local variables (e.g. `num: byte`) will live in working RAM.  But, unlike global variables, initialized variables will live in working RAM, as well.  I expand a bit more on this in the **Variable assignment and initialization** section.

We’d also keep Odin’s `@(static)` attribute. Any local initialized variable with this attribute lives in program memory. We will cover procedures in detail later on.

### Constants

Constants in this language function the same as Odin’s.  They are compile-time known values.  They are declared with `::` .

```odin
MAX_ENTIIES :: 256
STRING_CONSTANT :: a"Hello"
BYTE_CONSTANT : byte : 42
```

### Registers and aliases

In this language, you get explicit control of the 6502’s 3 semi-general-purpose 8-bit registers `A`, `X`, `Y`  and 2 special-purpose registers `S` for the stack pointer and `P`  for the processor flags. They can be used as types or variables. If used as a type in a variable or parameter, the name of that variable or parameter is an alias to that register. `S` can only be used for assignments to and from `X`. `P` can only used be used for the built-in `push()` and `pull()` procedures which push certain registers to the stack.

```odin
A = MEANING_OF_LIFE_ETC //same as 'lda #MEANING_OF_LIFE_ETC'
X = A                   //same as the 'tax' instruction
my_alias: A             //a compile-time known alias for the A register
my_alias = X            //same as 'txa' instruction
S = X                   //same as 'txs' instruction
X = S                   //same as 'tsx' instruction
push(P)                 //same as 'php' instruction
```

It’s important to note that assignments to a register or alias must correspond to a single 6502 instruction.  This means:

- `A` can be assigned to `X` (`txa` ), `Y` (`tya`), a numerical constant in the range of `-128` to `255` , or a  byte-sized variable that lives in memory (`lda`)
- `X` and `Y` can be assigned to `A` (`tax` , `tay`),  a numerical constant in the range of `-128` to `255` , or a byte-sized variable that lives in memory (`ldx` , `ldy`).  But, `X` and `Y` cannot be assigned to each other.
- `X` can also be assigned to `S` (`tsx`)
- `S` can only be assigned to `X` (`txs`)

For convenience, aliases can also be initialized to a valid source value for that register given the rules above. Examples:

```odin
alias_1: A = 5 //same as lda #5
alias_2: X = alias_1 //same as tax
alias_2 = Y //illegal,  cannot directly assign X to Y
```

### The `via` keyword

On the 6502, with the exception of the `inc` and `dec` , it is not possible to modify memory without loading and storing to and from either the `A`, `X`, or `Y` registers. So you can’t do something like this:

```odin
my_variable = 42
```

You'd need to something like

```odin
A = 42
my_variable = A
```

Not terrible, but it would be convenient to do this on one line which is why I came up with the `via` keyword. So, instead of doing the above, you should be able to write: 

```odin
my_variable = 42 via A
```

The `via` keyword is also a useful concept to have with regard to parameter passing for procedures. I go over this when I cover procedures later on. 

The `via` keyword must also be used when assigning to arrays.  More on that when we go over arrays.

One thing to consider is that you could potentially make `A` the default “via register” so that something like `my_variable = 42` would be valid, implicitly using the `A` register.  You’d have to explicitly specify `X` and `Y` if you wanted to use those.  Though, if you want to extend support to the 65C02, having `A` be implicit could get confusing for say `my_variable = 0`. Since for something like this, you would want to use a `stz` instruction, not a `sta` instruction.

### Variable assignment and initialization

For variables declared in global scope, either outside of a procedure or marked with the `@(static)` attribute,  if it is not initialized, it is allocated in working RAM.  If it is initialized, it is allocated  and the initialization value stored in program memory.  If a global variable is initialized and the program memory is ROM, then that variable is read-only (i.e. it is similar to a constant, but it is actually baked into the program executable, unlike compile-time constants).  

For variables local to procedures, they are always stored in working RAM.  If a local variable is initialized to a value, it is semantically equivalent to declaring that variable and then assigning it immediately after that.  This means, for example, if you want to initialize a variable to some number, you have to use the `via` keyword in the initialization. Examples:

```odin
//variables in global scope
//these wouldn't require the via keyword since
//they would be stored in program memory and included in 
//with the executable
my_global_variable := 20
my_global_array := [?]byte{1,2,3,4}

//variables in local scope
//these would require the via keyword
//since they are initialized at runtime
my_local_variable := 20 via A
my_local_array := [?]byte{1,2,3,4} via A
```

which would compile to

```asm
lda #20
sta my_local_variable

lda #1
sta my_local_array + 0
lda #2
sta my_local_array + 1
lda #3
sta my_local_array + 2
lda #4
sta my_local_array + 3

my_local_variable: .resb 1
my_local_array: .resb 4

my_global_variable: .byte 20
my_global_array:    .byte 1, 2, 3, 4
```

### Arrays

Arrays in this language would be value types with a length known at compile time.  Like in Odin `[?]` can be used for array literals where the length of the array is dictated by the number elements in the literals. 

Here are some examples of array declarations:

```odin
my_32_byte_array: [32]byte
global_array := [?]byte{1,2,3,5,6} //global_array has type [6]byte
local_array := [?]byte{1,2,3,5,6} via A
```

Due to the limitations of the 6502, as a general rule, arrays in this language can only be indexed by constants.  However, arrays with 1-byte elements (e.g. `[32]byte`) are special in that they can be indexed by the X or Y registers, plus an optional constant, which corresponds to the “absolute,x” and “absolute,y” address modes. Or, if the arrays live in the zero page, the “zero-page,x” and “zero-page,y” address modes. When using the X or Y registers, they can only represent values 0-255, since X and Y are only 1-byte large. Examples:

```odin
array: [512]byte
A = 42
X = 255      //highest value for X
array[X] = A //same as "sta array, X" 
array[256+X] = A //same as "sta array+256, X"

MyStruct :: struct {
    a, b: byte
}
struct_array: [512]MyStruct
struct_array[255].a = A //totally fine since indexed by constant.
                        //same as "sta (struct_array * 2) + 255
struct_array[X].a = A   //illegal. X cannot be used as index where array element size != 1 byte
```

The length of an array can be gotten with the `len()` builtin function.  It is always known at compile time, so the compiler can know if the length can fit in a register or not. Example:

```odin
small_array: [32]byte
large_array: [512]byte

X = len(small_array) //fine, 32 fits in 1 byte
X = len(large_array) //compile error, since 512 needs 2 bytes
CONSTANT :: len(large_array) //always fine
```

Arrays would also be initialized with scalar values using the `via` keyword.  There would be 2 ways to do this.  you can do something like:

```odin
array: [4]byte = 0 via A
```

This will generate 5 instructions  like so:

```asm
lda #0
sta array + 0
sta array + 1
sta array + 2
sta array + 3
```

This is useful for smaller arrays.  But the bigger your arrays are, the more instructions you’d generate.  In some cases this is actually okay since this is more performant than using a loop.  But if you care more about program size, you’d want the compiler to generate a loop instead. So the language should also support something like this:

```
array: [32]byte = 0 via A, X
```

Rather than generating 33 instructions, it would generate:

```asm
    lda #0
    ldx #0
loop:
    sta array, X
    inx
    cpx #32
    bne loop
```

Since X can only go up to 255, we’d need to do a bit more for arrays larger than 256.  Example:

```
array: [512]byte = 0 via A, X
```

would generate

```asm
    lda #0
    ldx #0
loop1:
    sta array + 0, X
    inx
    cpx #0
    bne loop1

loop2:
    sta array + 256, X
    inx
    cpx #0
    bne loop2
```

### Strings

This language would handle strings differently than Odin. In this language, strings would be arrays of bytes, like in C.  But, since ASCII wasn't a universal encoding on 6502 machines (e.g. PETSCII), you must specify the encoding.  The following encodings are supported:

- `petscii`  — Example usage: `my_string := p"HELLO"`
- **TODO: put in rest of encodings**

### Pointers

The 6502 has a concept of pointers (the 6502 refers to them as “indirect addressing”), but they have severe limitations, including how they must live on the zero page.  For this reason, any pointer declared in this language is automatically allocated in the zero page.  

This language should reflect the different 6502 indirect addressing modes through various pointer types.  The following pointer types can be supported in this language:

- `[^]byte` — A pointer to an unknown number of bytes. Also called a “multipointer”. This corresponds to indirect-indexed addressing mode of the 6502. As such it can only be indexed by `Y`.
- `[^]petscii` and `[^]ascii` - Same as `[^]byte` but meant for pointers to strings.
- `[?]^byte` — An array of byte pointers. This corresponds to the uncommonly-used indexed-indirect addressing mode.
    - It can only be indexed by `X`.
    - In Odin, the usage would be something like `array[X]^` , but in this language, the `^`  should be omitted (i.e. `array[X]` ) , since this would be a single operation on the 6502.
    - In theory you'd have versions for `petscii` and `ascii`, but not sure how useful that is
- `[^]T` or `[?]^T`  — where `T` is user-defined type which is 1 byte in size. Same semantics as `[^]byte` and `[?]^byte` above.
- `proc(…) → !` — A pointer to a procedure that doesn't return. Corresponds the indirect addressing mode which is typically used for jump tables. Only the `jmp` instruction supports this, so the pointed-to procedure cannot return. Can support any argument like a normal procedure though.

### Procedures

Procedures (aka functions, aka subroutines) are also structured the same as Odin’s.  Here is an example:

```odin
checksum :: proc(data: [^]byte, data_len: byte) -> A {
  sum: A = 0
    for Y in Y..<data_len {
        sum += data[Y]
    }
    return sum
}
```

Assuming the same segments as the hello world program, this translates to the assembly:

```asm
checksum:
    lda #0
    
    ldy #0
loop:
    cpy data_len
    beq exit
    clc
    adc (data),y
    iny
    jmp loop 
exit:

    rts
    
org 2
data: .res 2

org $2000
data_len .res 1

```

### Procedure Parameters

Parameters in this language are mostly the same as Odin’s with some changes.

- A parameter can be a register.
- A parameter can be a value that lives in working RAM
- A parameter can be a “reference”.

First, lets discuss register parameters.  Only `A` , `X` , and `Y` are supported registers.  You can specify them as an alias, or just the register name itself, like so:

```odin
array: [32]byte

//here we have A specified by register name and X by the alias "index" 
set_array :: proc(A, index: X) {
    array[index] = A
}

calling_proc :: proc() {
    A = 5
    X = 20
    set_array(A, X)
}
```

which compiles to

```asm
set_array:
    sta array, X
    rts

calling_proc:
    lda #5
    ldx #20
    jsr set_array
    
    rts

org $2000
array: .resb 32
```

You can also use number literals for register parameters as well.  They would  implicitly be loaded into the corresponding registers.  Example:

```odin
array: [32]byte

//here we have A specified by register name and X by the alias "index" 
set_array :: proc(A, index: X) {
    array[index] = A
}

calling_proc :: proc() {
    set_array(5, 20)
}
```

which compiles to

```asm
set_array:
    sta array, X
    rts

calling_proc:
    lda #5
    ldx #20
    jsr set_array
    
    rts

org $2000
array: .resb 32
```

Parameters can also be normal variables that live in working RAM.  If a value in memory is passed in you must specify the `via` clause like a regular variable.  Example:

```odin
//now takes a multipointer which lives in working ram in the zero page,
//since that's the only place pointers can live.
//Also notice index is now Y since multipointers can only be indexed by Y
set_array :: proc(A, index: Y, ptr: [^]byte) {
    ptr[index] = A
}

calling_proc :: proc() {
    array: [32]byte //remember, this lives in working RAM, not the stack
    A = 5
    set_array(A, 20, &array via X)
}
```

which compiles to

```asm
set_array:
    sta (ptr), Y
    rts

calling_proc:
    lda #5
    
    ldx #<calling_proc_array
    stx ptr
    ldx #>calling_proc_array
    stx ptr + 1
    
    ldy #20
    jsr set_array
    
    rts

org $2000
calling_proc_array: .resb 32

org $2
ptr: .resb 2
```

Now in some cases, it would be nice if `set_array` could *reference* `array` directly rather than using expensive and limited pointers. Also, remember that pointers must live in the zero page. So your zero page will fill up pretty quickly if you use pointer parameters everywhere!

For this issue we can use parameter **references**. You can declare a reference by prepending an `&` to the type. So we can rewrite the above example:

```odin
set_array :: proc(A, index: Y, array: &[?]byte) {
    array[index] = A
}

calling_proc :: proc() {
    array: [32]byte
    A = 5
    set_array(A, 20, array)
}
```

which compiles to

```asm
set_array:
    sta calling_proc_array,Y
    rts

calling_proc:
    lda #5
    ldy #20
    jsr set_array
    
    rts

org $2000
calling_proc_array: .resb 32

```

That’s much better!  Also notice we can specify the array with `[?]` size, so we don’t have to care what size it is, like a multipointer. References are basically the compile-time counterpart to pointers.  The way it works is that a new version of the procedure is generated for every combination of input addresses.  Because of this, you’d need to be careful of your usage.  Else, you program can easily get too big.  For example:

```odin
set_array :: proc(A, index: Y, array: &[?]byte) {
    array[index] = A
}

calling_proc :: proc() {
    array1, array2: [32]byte
    A = 5
    
    
    set_array(A, 20, array1)
    set_array(A, 21, array1)
    
    set_array(A, 22, array2)
}
```

compiles to

```asm
set_array1:
    sta calling_proc_array1,Y
    rts

set_array2:
    sta calling_proc_array2,Y
    rts

calling_proc:
    lda #5
    
    ldy #20
    jsr set_array1
    
    ldy #21
    jsr set_array1
    
    ldy #22
    jsr set_array2
    
    
    rts

org $2000
calling_proc_array1: .resb 32
calling_proc_array2: .resb 32
```

References are also a good solution to limited support for pointers in this language.  It would, for example, be the only way to pass in a struct by reference. Example:

```odin
MAX_ENTITIES :: 256

GameState :: struct {
    entity_x: [MAX_ENTITIES]byte,
    entity_y: [MAX_ENTITIES]byte,
    entity_vx: [MAX_ENTITIES]byte,
    entity_vy: [MAX_ENTITIES]byte,
    
}

move_entity :: proc(entity: X, gs: &GameState) {
        
        //we will get into how arithmetic works with the via keyword later
        gs.entity_x[entity] += gs.entity_vx[entity] via A
        gs.entity_y[entity] += gs.entity_vy[entity] via A
}
```

compiles to

```asm
move_entity:
    lda gs + 256*0, X     ;entity_x
    clc
    adc gs + 256*2, X     ;entity_vx
    sta gs + 256*0, X     ;entity_x
    
    lda gs + 256*1, X     ;entity_y
    clc
    adc gs + 256*3, X     ;entity_vy
    sta gs + 256*1, X     ;entity_y
    
    rts
    
    
org $2000    
gs: .resb 256 * 4
```

### Procedure Return Values

Procedure return values are basically the same as Odin’s (including multiple return values), but with the addition of being able to return registers/aliases as well.  It is important to note that for non-register return types (e.g. `byte`), the return value needs to live in working RAM.  This means that for non-register return types:

- They need to be specified with the `via` keyword, like parameters.
- The calling procedure needs to copy the return value into the destination variable.

The last point can be a pain point if you’re returning a value many bytes in size.  In that case, use a reference instead of a return value.

Example:

```odin
max_byte :: proc(a, b: byte) -> byte {
    A = a
    
    if A < b {
        return b via A
    }
    
    return a via A
    
}

calling_proc :: proc(a, b: byte) {
    //....

    c := max_byte(a via A, b via A) via A
    
    //....
}
```

compiles to

```asm
max_byte:
    lda max_byte__a
    cmp max_byte__b
    bcs a_is_bigger_or_equal
    
    lda max_byte__b
    sta max_byte__return
    rts
    
a_is_bigger_or_equal:
    lda max_byte__a
    sta max_byte__return
    rts

calling_proc:
    ;....
    
    lda calling_proc__a
    sta max_byte__a
    lda calling_proc__b
    sta max_byte__b
    
    jsr max_byte
    lda max_byte__return
    sta calling_proc__c
    
    ;....
    
    rts

org $2000

calling_proc__a:      .resb 1
calling_proc__b:      .resb 1
calling_proc__c:      .resb 1

max_byte__a:      .resb 1
max_byte__b:      .resb 1
max_byte__return: .resb 1

    
```

As stated before, return values can be registers.  So `max_byte` can be rewritten as:

```odin
max_byte :: proc(a, b: byte) -> A {
    A = a
    
    if A < b {
        return b via A
    }
    
    return A
    
}

calling_proc :: proc(a, b: byte) {
    //....

    A = max_byte(a via A, b via A)
    
    //....
}
```

which compiles to

```asm
max_byte:
    lda max_byte__a
    cmp max_byte__b
    bcs a_is_bigger_or_equal
    
    lda max_byte__b
    rts
    
a_is_bigger_or_equal:
    rts

calling_proc:

    ;....
    
    lda calling_proc__a
    sta max_byte__a
    lda calling_proc__b
    sta max_byte__b
    
    jsr max_byte
    
    ;....
    
    rts

org $2000

calling_proc__a:      .resb 1
calling_proc__b:      .resb 1

max_byte__a:      .resb 1
max_byte__b:      .resb 1
```

Notice that there is no double copying of the return value unlike in the `-> byte` version.  So rule of thumb is that it is probably best to use registers as return values if you can.

### If statements

If statements are mostly the same as Odin’s with some caveats. Basic comparisons are allowed between registers and memory or constants, which correspond to the `cmp`, `cpx` and `cpy` instructions combined with a branch instruction like `beq`. 

```odin
if A == 42 {
   //...
}

some_byte := 124

//...

if A < some_byte {
   //...
}
```

In theory, we could allow memory-to-memory or memory-to-constant comparisons using the `via` keyword, but I feel that can get too messy when you start to incorporate logic operators (e.g. `&&` ), so for the purposes of this post, I will assume that is disallowed.  Register-to-register comparisons (e.g. `A < X`) are also disallowed, since there are no corresponding 6502 instructions for them.

It’s important to note that, by default, the branch instruction generated corresponds to the **opposite** condition. So for:

```odin
if A == 42 {
    //...
}
```

a `bne` (”branch if NOT equal”) would be generated like so:

```asm
    cmp #42
    bne outside_if_block
    
    ;...
    
outside_if_block:

```

Adding an `else` block like

```odin
if A == 42 {
    //inside if block
} else {
  //inside else block
}
```

would generate

```asm
    cmp #42
    bne else_block
    
    ;inside if block
    
    jmp outside_if_block
    
else_block:
    
    ;inside else block
    
outside_if_block:
```

It’s important to consider an annoying quirk of 6502’s branch instructions: they can only jump at an offset of -128 to 127 bytes from the branch instruction. Which means that if your if block has more than 127 bytes worth of instructions, you have generate the if-else statement above a bit differently.  You’d actually change the condition to match the comparison operation so now the `bne` changes to `beq` and then we add a `jmp` instruction.  Unlike the branch instructions, the `jmp` instruction takes a full 16-bit address.  This is what the assembly would look like:

```asm
    cmp #42
    beq if_block    ;here we actuall
    jmp else_block
    
if_block:
    
    ;inside if block
    ;more than 127 bytes of instructions here
    
    jmp outside_if_block
    
else_block:
    
    ;inside else block
    
outside_if_block:

```

We could have some sort of keyword like `#short if` that ensures the if-block is less than 127 bytes, else there would be a compile error.

This language would also support standard short-circuit boolean logic operators `&&` and `||` . Example assuming the if block is small (i.e. < 127 bytes):

```odin
if (A >= 42 && A < 100) || X == 0 {
    //inside if block
} else {
  //inside else block
}
```

compiles to

```asm
    cmp #42
    bcc or_test
    cmp #100
    bcs or_test
    jmp if_block
    
or_test:
    cpx #0
    bne else_block
if_block:
    
    ;inside if block
    
    jmp outside_if_block
    
else_block:
    
    ;inside else block
    
outside_if_block:
```

Many 6502 instructions other than the compare instructions, affect the status flags.  In fact, there are many cases where you to branch based on these other instructions.  A common one is checking if a certain value in memory is zero like so:

```asm
    lda my_var
    beq is_zero

    ;my_var != 0 here

is_zero:
```

The `beq` instruction actually tests if the `Z` flag of the processor status flags is set.  If its set, then branch.  `Z` is set if we are comparing a register to a value and they are equal (e.g. `cmp #42` will set `Z` if `A` is equal to 42).  `Z` is also set if a load instruction (e.g. `lda` ) loads a value that is equal to 0 into a register, which is illustrated in the example above. We should be able to express the above example in this language like so:

```odin
A = my_var
if \Z {
    //executed if A is 0
}

//Or could be rewritten as

A = my_var; if \Z {
    //executed if A is 0
}
```

As you can see, you can specify a flag by using the backslash followed by the flag name.  You can also reference by flag number rather than name.  E.g. `\Z`, which is bit 1 of the processor flags, can also be written as `\P1` like so:

```odin
A = my_var; if \P1 {
    //executed if A is 0
}
```

### What’s left

As I said in the intro, there are several things that I wanted to detail here, but just didn’t finish, since this post was taking up too much of my time.  These are, but not limited to:

- Loops
- Structs
- Arithmetic, including multi-byte arithmetic
- Segments and `with_memory_map`
- The compiler’s static memory allocator
- Register clobber detection
- References for local variables
- Context variable
- Slices
- Explicit zero page variables
- Built-in functions
- Probably many other things

### Other Architectures

One thing I’d like to emphasize, is that while this language is designed specifically for the 6502, there’s no reason why you couldn’t have a similar language for any other 8-bit CPU, like the Z80, or even 16-bit CPUs like the 68000.  I feel once you hit 32-bit machines that start having things like virtual memory and caches, the value of having such a language goes down dramatically.

### NES Hello World Example

This example is taken from. https://github.com/thomaslantern/nes-hello-world/blob/main/helloWorld.asm.  The following program will essentially compile into this hello world program.  One thing to point out is that the assembly explicitly embeds an iNES header and makes sure the character ROM data is an explicit byte offset in the resulting `.nes` file. In this language, such things should be offloaded to the compiler.  For example, the compiler should support a built-in `NESCharacterROM` segment type.

```odin

RAM_SIZE :: kb(2)

RAM_SEGMENT :: Segment{.WorkingRAM, 0..<RAM_SIZE}
MMIO_SEGMENT :: Segment{.NonAllocatableRAM, 0x800..<0x4020}
PROGRAM_SEGMENT :: Segment{.ProgramMemoryROM, 0xC000..<0x10000}

with_memory_map(MMIO_SEGMENT, PROGRAM_SEGMENT, RAM_SEGMENT) {

PPURegisters :: struct {
    ctrl: byte,
    mask: byte,
    status: byte,
    oam_addr: byte
    oam_data: byte,
    scroll: byte,
    addr: byte,
    data: byte,
}

oam_dma: byte @ 0x4014
ppu: PPURegisters @ 0x2000
apu_enable: byte @ 0x4015
apu_dmc: byte @ 0x4010
apu_frame_counter: byte @ 0x4017

ram: [RAM_SIZE]byte @ 0x0

nmi_handler :: interrupt_proc {
        oam_dma = 2 via A // Transfer sprite data from from $0200 to DMA
}

irq_handler :: interrupt_proc {
    //does nothing
}

startgame :: proc() {
    sei()
    cld()
    
    //setup stack
    X = 0xFF
    S = X
    X++ //wraps around to 0
    
    ppu.ctrl = X
    ppu.mask = X
    apu_enable = X
    apu_dmc = X
    apu_frame_counter = 0x40 via A
    
    A = 0
    
    //Wait for bit 7 of the PPU status flags to be set
    //which indicates V-Blank
    for ;!\P7; bit(ppu.status) {}
    
    OAM_PAGE :: hi(&oam)
    
    //Initialize RAM
    for X in 0..<256 {
        //We need to do this page-by-page since the X register can only go from
        //0 - 255
        #unroll for page in 0..<8 {
            when page != OAM_PAGE {
                ram[X + (page * 0x100)] = A
            } else {
                A = 0xFF
                ram[X + (page * 0x100)] = A  //hide sprites
                A = 0
            }
                
        }
    }
    
    for ;!\P7; bit(ppu.status) {}
    
    A = ppu.status
    ppu.addr = 0x3F via X
    ppu.addr = 0 via X
    
    for A, X in initial_palette {
        ppu.data = A
    }
    
    A = OAM_PAGE
    oam_dma = A
    
    for A, X in hello {
        oam[X] = A
    }
    
    oam_dma = OAM_PAGE via A
    
    ppu.ctrl = 0b10010000 via A
    
    ppu.mask = 0b00011110 via A
    
    for {}
    
}

//Remember this is allocated in working RAM since its uninitalized
//This will get allocated at 0x200 since it is not marked as a zero-page variable
//And 0x200 is the first free address in working RAM (remember, 0x100-0x1FF is the stack)
@(align=0x100)
oam: [256]byte

//But, these are allocated in ROM since they are initialized. 
//They are also read only since they are in ROM.
initial_palette := [?]byte{0x1F, 0x21, 0x33, 0x30}

hello := [?]byte {
    0x6c, 0x00, 0x00, 0x3d, // H
    0x6c, 0x01, 0x00, 0x46, // E
    0x6c, 0x02, 0x00, 0x4f, // L
    0x6c, 0x02, 0x00, 0x58, // L
    0x6c, 0x03, 0x00, 0x61, // O

    0x75, 0x04, 0x00, 0x3d, // W
    0x75, 0x03, 0x00, 0x46, // O
    0x75, 0x05, 0x00, 0x4f, // R
    0x75, 0x02, 0x00, 0x58, // L
    0x75, 0x06, 0x00, 0x62, // D
    0x75, 0x07, 0x00, 0x6b, // !
    }

}

interrupt_vectors := [?]word{&nmi_handler, &startgame, &irq_handler}

CHR_ROM :: Segment{.NESCharacterROM}

with_memory_map(CHR_ROM) {
  _ := [?]byte{
     0b11000011,    // H (00)
     0b11000011,
     0b11000011,
     0b11111111,
     0b11111111,
     0b11000011,
     0b11000011,
     0b11000011,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b11111111,    // E (01)
     0b11111111,
     0b11000000,
     0b11111100,
     0b11111100,
     0b11000000,
     0b11111111,
     0b11111111,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b11000000,    // L (02)
     0b11000000,
     0b11000000,
     0b11000000,
     0b11000000,
     0b11000000,
     0b11111111,
     0b11111111,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b01111110,    // O (03)
     0b11100111,
     0b11000011,
     0b11000011,
     0b11000011,
     0b11000011,
     0b11100111,
     0b01111110,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b11000011,    // W (04)
     0b11000011,
     0b11000011,
     0b11000011,
     0b11011011,
     0b11011011,
     0b11100111,
     0b01000010,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b01111110,    // R (05)
     0b11100111,
     0b11000011,
     0b11000011,
     0b11111100,
     0b11001100,
     0b11000110,
     0b11000011,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b11110000,    // D (06)
     0b11001110,
     0b11000010,
     0b11000011,
     0b11000011,
     0b11000010,
     0b11001110,
     0b11110000,
     0, 0, 0, 0, 0, 0, 0, 0,

     0b00011000,    // ! (07)
     0b00011000,
     0b00011000,
     0b00011000,
     0b00011000,
     0b00000000,
     0b00011000,
     0b00011000,
     0, 0, 0, 0, 0, 0, 0, 0,

  }
}

```

would compile to essentially the same assembly as in the Github.  But to prevent possible future dead links, this is it:

```asm
    ; Hello, World!
    ; by Thomas Wesley Scott, 2023
    ; Code starts at $C000.
    ; org jump to $BFFO for header info

    org $BFF0        ; This command just tells
                ; compiler to start at $BFF0
                ; (where the header starts)

    ; This is a run of the mill NES cartridge header.
    ; Without this, your code won't work!
    ; For now, always include this header exactly like
    ; this.
    db "NES",$1a        ; Default NES header info
    db $1            ; Number of PRG-ROM pages
    db $1            ; Number of CHR-ROM pages
    db %00000000        ; Mapper and other info.
    db %00000000        ; More mapper/other info.
    db 0            ; Number of Ram pages
    db 0,0,0,0,0,0,0    ; Unused 7 bytes

; The nmihandler is basically the code that runs
; every time the screen refreshes, which happens
; about 60 times a second.
nmihandler:
    lda #$02     ; Transfer sprite data from
    sta $4014    ; from $0200 to DMA
    rti        ; return from interrupt

; The irqhandler is the code that runs every
; time some NES hardware interrupts your code.
; I've used the "sei" command to disable every
; interrupt (except the screen/NMI above), so
; this can just be left mostly blank.
irqhandler:
    rti

; startgame is code that runs everytime the game 
; is turned on or reset. We're basically starting
; by "zeroing out" any data that lingered here
; after a reset. This also gives us some time 
; for the NES to load up properly before it 
; can accept/use graphics.

startgame:
    sei            ; Disable interrupts
    cld            ; Clear decimal mode

    ldx #$ff    
    txs            ; Set-up stack
    inx            ; x is now 0
    stx $2000        ; Disable/reset graphic options 
    stx $2001        ; Make sure screen is off
    stx $4015        ; Disable sound
    stx $4010        ; Disable DMC (sound samples)
    lda #$40
    sta $4017        ; Disable sound IRQ
    lda #0    
waitvblank:
    bit $2002        ; check PPU Status to see if
    bpl waitvblank        ; vblank has occurred.
    lda #0
clearmemory:            ; Clear all memory info
    sta $0000,x         ; from $0000-$07FF
    sta $0100,x
    sta $0300,x
    sta $0400,x
    sta $0500,x
    sta $0600,x
    sta $0700,x
    lda #$FF
    sta $0200,x        ; Load $FF into $0200-$02FF 
    lda #$00        ; to hide sprites 
    inx            ; x goes to 1, 2... 255
    cpx #$00        ; loop ends after 256 times,
    bne clearmemory     ; clearing all memory
        

waitvblank2:
    bit $2002        ; Check PPU Status one more time
    bpl waitvblank2        ; before we start loading in graphics

    lda $2002        ; Read PPU status to reset high-low latch
    ldx #$3F        ; Load high byte of $3F00 into $2006
    stx $2006
    ldx #$00        ; Load low byte of $3F00 into $2006
    stx $2006
copypalloop:            ; Start storing palette info
    lda initial_palette,x    
    sta $2007
    inx
    cpx #4            ; loop 4 times
    beq copypalloop

    lda #$02        ; Store sprite info 
    sta $4014        ; into OAM DMA

; Loop to load sprites onto screen
    LDX #$00
spriteload:
    lda hello,x        ; Loads one of four values into $0200,x:
    sta $0200,x         ; x-value, tile #, flip options, y-value
    inx
    cpx #$2C        ; Loop 44 times (11 tiles with 4 attributes each)
    bne spriteload

    lda #%10010000        ; Enable NMI on vblank, and use
    sta $2000        ; $1000 as background tile address

    lda #%00011110        ; Turn on sprites, background,
    sta $2001        ; and clipping for both

; Necessary loop to keep program running
forever:
    jmp forever

; This is the only palette for this tutorial.
; Each byte is one colour.
initial_palette:
    db $1F,$21,$33,$30

; This is the data for our sprite placement.
; The first byte on each line is the y-coordinate.
; The second byte is the tile # in memory.
; The third byte is for flipping tiles.
; The fourth byte is the x-coordinate.

hello:
    db $6c, $00, $00, $3d ; H
    db $6c, $01, $00, $46 ; E
    db $6c, $02, $00, $4f ; L
    db $6c, $02, $00, $58 ; L
    db $6c, $03, $00, $61 ; O

    db $75, $04, $00, $3d ; W
    db $75, $03, $00, $46 ; O
    db $75, $05, $00, $4f ; R
    db $75, $02, $00, $58 ; L
    db $75, $06, $00, $62 ; D
    db $75, $07, $00, $6b ; !

; This is the footer for our program.
; It is where we define (in order):
; - our nmihandler (what we do during vblank)
; - our reset (here it's called startgame)
; - our irqhandler (for handling interrupts)
; We have disabled our irqhandler, but all
; three are always included in our programs.
    org $FFFA
    dw nmihandler
    dw startgame
    dw irqhandler

; After your footer, you include your
; tile data, which gets stored in CHR-ROM.
; You can include a file containing this
; data, so the code is a bit cleaner,
; but I've included it here so you can see
; the actual letters for yourself.

chr_rom_start:

    db %11000011    ; H (00)
    db %11000011
    db %11000011
    db %11111111
    db %11111111
    db %11000011
    db %11000011
    db %11000011
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %11111111    ; E (01)
    db %11111111
    db %11000000
    db %11111100
    db %11111100
    db %11000000
    db %11111111
    db %11111111
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %11000000    ; L (02)
    db %11000000
    db %11000000
    db %11000000
    db %11000000
    db %11000000
    db %11111111
    db %11111111
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %01111110    ; O (03)
    db %11100111
    db %11000011
    db %11000011
    db %11000011
    db %11000011
    db %11100111
    db %01111110
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %11000011    ; W (04)
    db %11000011
    db %11000011
    db %11000011
    db %11011011
    db %11011011
    db %11100111
    db %01000010
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %01111110    ; R (05)
    db %11100111
    db %11000011
    db %11000011
    db %11111100
    db %11001100
    db %11000110
    db %11000011
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %11110000    ; D (06)
    db %11001110
    db %11000010
    db %11000011
    db %11000011
    db %11000010
    db %11001110
    db %11110000
    db $00, $00, $00, $00, $00, $00, $00, $00

    db %00011000    ; ! (07)
    db %00011000
    db %00011000
    db %00011000
    db %00011000
    db %00000000
    db %00011000
    db %00011000
    db $00, $00, $00, $00, $00, $00, $00, $00

; Lastly, if we have less than 512 tiles
; we need to "pad" our rom to give it
; the correct file size.
; If you ever have issues with your .nes
; file, this might be it! (Best to leave
; this line of code in just to be safe,
; since it guarantees the correct size of
; ROM data.)

chr_rom_end:
    ds 8192-(chr_rom_end-chr_rom_start)

```

### Commodore 64 Move Sprite example

This example is a program where the user can move a hot air balloon sprite around using the WSAD keys. This is taken from <https://github.com/digitsensitive/c64/blob/main/src/examples/moveSprite.asm>.  The output assembly is slightly different from the original program.

```odin
package main

//The .Program value specifies that any object within this segment is stored into the resulting executable or ROM file
PROGRAM_SEGMENT :: Segment{.Program, 0x800..<0x2000}
KERNAL_SEGMENT :: Segment{.External, 0xE000..<=0xFFFF}
VIDEO_MEMORY_SEGMENT :: Segment{.External, 0x340..<=0x800}
VIC_MMIO_SEGMENT :: Segment{.External, 0xD000..<=0xD02F}
RAM_SEGMENT :: Segment{.RAM, 0x2..<0x100, 0x2000..<0x8000}

SpritePosition :: struct {x, y: byte}

Colors :: enum byte {
    BLACK = 0,                      
    WHITE,                      
    RED,                          
    CYAN,                        
    PURPLE,                   
    GREEN,                     
    BLUE,                        
    YELLOW,                   
    ORANGE,                  
    BROWN,                    
    LIGHT_RED,             
    GRAY_ONE,              
    DARK_GREY,            
    GRAY_TWO,             
    MEDIUM_GREY,       
    LIGHT_GREEN,        
    LIGHT_BLUE,            
    GRAY_THREE,          
    LIGHT_GREY
}
    

with_memory_map(PROGRAM_SEGMENT, RAM_SEGMENT, KERNAL_SEGMENT, VIDEO_MEMORY_SEGMENT,VIC_MMIO_SEGMENT)
{

//Required BASIC stub which tells C64 BASIC to run the procedure at address 0x810 (in this case, main)
_ = []byte{0, 0xE, 0x8, 0xA, 0x0, 0x9E}
_ = p" (2064)" //p"" strings are PETSCII strings
_ = []byte{0,0,0}

clear_screen :: proc() @ 0xE544
scan_keyboard :: proc() @ 0xFF9F
get_in :: proc() -> A @ 0xFFE4

sprite_pointers: [8]byte @ 0x07f8
sprite_positions: [8]SpritePosition @ 0xD000
sprite_msb_x: byte @ 0xD010
sprite_enable: byte @ 0xD015
sprite_colors: [8]Colors @ 0xD027
sprite0_video_memory: [64]byte @ 0x340

main :: proc() @ 0x810 {
    
    clear_screen()
    
    sprite_pointers[0] = 13 via A      //use block 13 for sprite 0
    sprite_enable = 0b0000_0001 via A  //enable sprite 0
    sprite_colors[0] = .CYAN via A
    
    //Since this variable is initialized and is static,
    //it will live in the next free spot in program memory.
    @(static)
    hot_air_balloon_sprite_data := [?]byte{
        0x00, 0x7F, 0x00,
        0x01, 0xFF, 0xC0,
        0x03, 0xFF, 0xE0,
        0x03, 0xE3, 0xE0,
        0x07, 0xD9, 0xF0,
        0x07, 0xDF, 0xF0,
        0x07, 0xD9, 0xF0,
        0x03, 0xE7, 0xE0,
        0x03, 0xFF, 0xE0,
        0x03, 0xFF, 0xE0,
        0x02, 0xFF, 0xA0,
        0x01, 0x7F, 0x40,
        0x01, 0x3E, 0x40,
        0x00, 0x9C, 0x80,
        0x00, 0x9C, 0x80,
        0x00, 0x49, 0x00,
        0x00, 0x49, 0x00,
        0x00, 0x3E, 0x00,
        0x00, 0x3E, 0x00,
        0x00, 0x3E, 0x00,
        0x00, 0x1C, 0x00,
    }
        
    //Copy sprite data into video memory
    for A, X in hot_air_balloon_sprite_data {
        sprite0_video_memory[X] = A
    }
    
    sprite_msb_x = 0 via A
    
    //compile-time-known reference to sprite_positions[0]
    //we can do this because we know the address of sprite_positions[0] at compile time
    spr_pos := ref sprite_positions[0]
    
    
    spr_pos = {100, 70} via A
    
    //main loop
    for {
        key: A
        
        scan_keyboard()
        key = get_in()
        
        switch key {
            case 87:      //W - Up
                spr_pos.y--
            case 83:      //S - down
                spr_pos.y++
            case 65:      //A - left
                spr_pos.x--
            case 68:      //D - right
                spr_pos.x++
        }
        
    }
    
}

}
```

compiles to

```asm
; ==========================================================
; COMMODORE 64 - Examples in 6502 Assembly language
; © Digitsensitive; digit.sensitivee@gmail.com
; How to move a sprite
; ==========================================================

; ----------------------------------------------------------
; Default screen memory (1000 bytes)
; $0400-$07FF, 1024-2047
; ----------------------------------------------------------
SCREEN_MEMORY                       = $0400 ; 1024
UNUSED_16_BYTES                     = $07e8 ; 2024
SPRITE_POINTERS                     = $07f8 ; 2040

; Sprite horizontal and vertical position registers
SP0X                                = $d000 ; 53248
SP0Y                                = $d001 ; 53249
SP1X                                = $d002 ; 53250
SP1Y                                = $d003 ; 53251
SP2X                                = $d004 ; 53252
SP2Y                                = $d005 ; 53253
SP3X                                = $d006 ; 53254
SP3Y                                = $d007 ; 53255
SP4X                                = $d008 ; 53256
SP4Y                                = $d009 ; 53257
SP5X                                = $d00A ; 53258
SP5Y                                = $d00B ; 53259
SP6X                                = $d00C ; 53260
SP6Y                                = $d00D ; 53261
SP7X                                = $d00E ; 53262
SP7Y                                = $d00F ; 53263

; Most significant bits of sprites 0-7 horizontal positions
MSIGX                               = $d010 ; 53264
; Sprite enable register
SPRITE_ENABLE_REGISTER              = $d015 ; 53269

; Multicolor registers
SPRITE_MULTICOLOR_REGISTERS         = $d01c ; 53276

; Sprite Multicolor
SPMC0                               = $d025 ; 53285
SPMC1                               = $d026 ; 53286

; Sprite color registers
SPRITE_0_COLOR_REGISTER             = $d027 ; 53287
SPRITE_1_COLOR_REGISTER             = $d028 ; 53288
SPRITE_2_COLOR_REGISTER             = $d029 ; 53289
SPRITE_3_COLOR_REGISTER             = $d02a ; 53290
SPRITE_4_COLOR_REGISTER             = $d02b ; 53291
SPRITE_5_COLOR_REGISTER             = $d02c ; 53292
SPRITE_6_COLOR_REGISTER             = $d02d ; 53293
SPRITE_7_COLOR_REGISTER             = $d02e ; 53294

; ----------------------------------------------------------
; Labels
; ----------------------------------------------------------

; each sprite is 3 bytes wide with 21 lines = 63 bytes required for shape
; but it actually uses 64 bytes
; the value @ the pointer times 64 equals
; the starting location of the sprite shape data table
spr0ShapeData = $0340                           ; = 13 * 64 = 832

*=$800
     .byte $00                                 
        .byte $0E, $08                            
        .byte $0A, $00                            
        .byte $9E                                 
        .text " (2064)"                           
        .byte $00, $00, $00                 
                                                
                *=$810                       

; ----------------------------------------------------------
; Init and create sprite
; ----------------------------------------------------------

init            jsr $e544                       ; clear the screen
                            
                lda #%00001101                  ; use block 13 for sprite 0
                sta SPRITE_POINTERS
                lda #%00000001                  ; load accumulator with value 1
                sta SPRITE_ENABLE_REGISTER      ; enable sprite 0
                lda #CYAN                       ; use cyan for sprite 0
                sta SPRITE_0_COLOR_REGISTER     ; load cyan to color register 0

build           lda spr0,x                      ; get byte from sprite0+x
                sta spr0ShapeData,x             ; store byte at spr0ShapeData+x
                inx
                cpx #63
                bne build

                ; set position
                lda #%00000000                  ; restrict horizontal position to 0-255
                sta MSIGX

                ; starting sprite location
                lda #100
                sta SP0X
                lda #70
                sta SP0Y

; ----------------------------------------------------------
; Main Loop
; ----------------------------------------------------------

                ; use of the kernal jump table
loop            jsr SCNKEY                      ; jump to scan keyboard
                jsr GETIN                       ; jump to get a character

; ----------------------------------------------------------
; Evaluate Input
; ----------------------------------------------------------

input           cmp #87                         ; W - up
                beq up
                cmp #83                         ; S - down
                beq down
                cmp #65                         ; A - left
                beq left
                cmp #68                         ; D - right
                beq right
                jmp switch_exit

up              dec SP0Y
                jmp switch_exit

down            inc SP0Y
                jmp switch_exit

left            dec SP0X
                jmp switch_exit

right           inc SP0X
   
                
switch_exit
                                jmp loop
                                

; ----------------------------------------------------------
; Data
; ----------------------------------------------------------

spr0            .byte $00, $7F, $00
                .byte $01, $FF, $C0
                .byte $03, $FF, $E0
                .byte $03, $E3, $E0
                .byte $07, $D9, $F0
                .byte $07, $DF, $F0
                .byte $07, $D9, $F0
                .byte $03, $E7, $E0
                .byte $03, $FF, $E0
                .byte $03, $FF, $E0
                .byte $02, $FF, $A0
                .byte $01, $7F, $40
                .byte $01, $3E, $40
                .byte $00, $9C, $80
                .byte $00, $9C, $80
                .byte $00, $49, $00
                .byte $00, $49, $00
                .byte $00, $3E, $00
                .byte $00, $3E, $00
                .byte $00, $3E, $00
                .byte $00, $1C, $00

```

### Closing Thoughts

I hope you can see how such a language could be useful.  If there is just one person out there who could make this language (or similar) a reality, I’d consider this post a smashing success.

Please let me know what you think in the comments below or reach out to me at <dan@boksos.com>.  Thanks for reading!
