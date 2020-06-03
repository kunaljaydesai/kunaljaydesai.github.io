---
layout: post
title: Deep Dive Into Static Linking
excerpt_separator:  <!--more-->
---
In a <a href="/2020/05/31/understanding-linking.html">previous article</a>, we gave an overview of what the linker does. In this article, we will dive deeper into how static linking works.
<!--more-->

<b>What does static linking mean?</b>

Static linking takes in a set of relocatable object files and creates an executable object file which can be used to run your program. When a program is compiled, it compiles all the files separately and ignores undefined symbols (initialized but undefined functions or variables). The static linker is responsible for taking all the relocatable object files from the compiler output and links the undefined symbols to their definitions as well as relocating the sections in memory (we will talk about what this means below).

In the previous <a href="/2020/05/31/understanding-linking.html">article</a>, we saw that the `ask_question` function signature was initialized, but not defined. The static linker to the relocatable object files of `main.c` and `helper.c` and resolves the symbols so that the program can be run. This is known as the symbol resolution step of the linker.
Relocating the sections in memory is the other job of the linker. The compiler assigns sections in memory for the sections in the object file assuming that it starts at the memory address 0. However, in a multi-file program, not all the sections loaded into memory can start at memory address 0. The linker has to relocate the sections so that it can be properly loaded into one program’s area in memory.

<b>What is a relocatable object file?</b>

A standard format in Linux for a relocatable object file is called ELF (Executable and Linkable Format). The format of this file is:

1. ELF Header — this contains information about the word size (32 vs. 64) and byte ordering (little endian vs. big endian) of the system that generated the file. It also includes the offset into the section header table.
2. Sections — the sections include the text segment (where the machine code is stored), the symbol table (where information about functions and global variables is stored), and the data section (which stores initialized global variables).
3. Section Header Table — this contains information about the size and offsets of the sections described above. These entries are of a fixed size.


We won’t go into each of the sections in a lot of depth, but we will talk about the symbol table more deeply.

<b>What is the symbol table?</b>

The symbol table has information about all the symbols that are referenced by and defined by this relocatable object file. There are three types of symbols:

1. Global symbols that are defined by this file and can be referenced by other modules.
2. Global symbols that are referenced by this module but defined in another module.
3. Local symbols are referenced and defined by this module and can’t be accessed by other modules. To be clear, local non-static function variables are not included in the symbol table since they are addressed at runtime.


The symbol table is essentially a list of symbols where each symbol contains the following information:
1. Name — pointer to the string of the variable name.
2. Value — address of this symbol.
3. Size — size of the object that is being pointed to.
4. Type — data or function.
5. Binding — local or global.
6. Section — ABS (symbols that should not relocate), UND (referenced in this object file but not defined here), COMMON (uninitialized data).


One thing to note that for COMMON entries, the value field is actually the alignment requirement and the size is the minimum size.

Once the symbol table is defined for all of the files, the linker can now resolve the symbols. One thing you may wonder is, if we are given multiple of the same symbols (same function name, same variables), how does the linker resolve them?

To understand how the linker resolves them, we need to understand strong symbols and weak symbols. Strong symbols are global defined functions or defined variables (`int x = 1;`). Weak symbols are global undefined symbols (`int x;`). To resolve symbols, we use the following rules:
1. Multiple strong symbols aren’t allowed
2. If there is a strong symbol and multiple weak symbols, always prefer the strong symbol
3. If there are no strong symbols and multiple weak symbols, choose any of the week symbols

One interesting thing to note is that these symbols don’t take variable type, function return type, or function parameters into consideration when resolving symbols. This means that we need to be careful when defining symbols of different types (int, double, float, etc…) with the same name. In C++ and Java, however, they do take things like function parameters into account to allow you to have multiple symbols with the same identifier.

<b>Static Linking Libraries</b>

It can be really tedious to compile and link every relocatable object file that you use in a program. Additionally, it may waste a lot of space in memory to store unused functions and variables in object files (particularly from external libraries). An easier and more efficient way to do this is to convert a module of relocatable object files into a single file which contains only the relevant objects referenced by the program. This is essentially what a static library is and what static linking allows you to do. A static library allows developers to modularize their code and only include referenced object modules in their program memory.

<b>Relocation</b>

Once all the symbols of a program are resolved to their definitions, we need to take the relocatable object file sections and arrange them into a program’s memory. When you compile source code, it compiles each file separately and marks unresolved symbols as undefined. As a reminder, the symbols that are defined are referenced based on a memory address starting at 0 for that file. However, when you aggregate a bunch of relocatable object files, you need to adjust the memory address for each of these references as well as the section offsets (.text, .data, .bss, etc…) since they will get larger.

The way relocation works is as follows. Every time a reference to a symbol doesn’t have the correct runtime memory address yet, then a relocation entry is created. A relocation entry contains information like:
1. Offset — offset for this section in memory that needs to have an updated address.
2. Symbol — the name of the symbol that this should be pointing to.
3. Type — generally this is whether to use PC relative addressing or absolute address.

When you have a relocation entry that needs to be assigned using PC (program counter) relative addressing, you take the symbol’s address in memory and calculate the delta between the address of the symbol and the PC at the time of execution. This delta is used in the assembly to move to the correct next instruction. When you have a relocation entry that needs to be assigned using absolute references, you just put in the actual address of that symbol.

<b>Conclusion</b>

There are two steps in static linking. The first is to resolve the symbols and the second is to aggregate and relocate them into memory addresses that the operating system can use. One disadvantage with static libraries is that code from libraries like the standard C library may be copied into memory for multiple different programs. Memory is scarce so this is can be a huge waste. Additionally, if a library needs to be updated, then the entire program needs to be compiled again. Dynamic linking through the use of shared libraries solves these issues. A future post about dynamic linking will address its inner workings.

<b>Citations</b>

Randal E. “Computer Systems: a Programmer’s Perspective.” Computer Systems: a Programmer’s Perspective, by David R. O’Hallaron, Pearson, 2019, pp. 655–691.