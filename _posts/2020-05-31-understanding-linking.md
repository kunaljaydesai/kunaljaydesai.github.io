---
layout: post
title: Understanding Linking
excerpt_separator:  <!--more-->
---
You may have wondered at some point: how does a program of multiple files access variables or functions defined in other files? That is the goal of the linker. The linker aims to take multiple pieces of code and combine them into one that can be read and understood by the computer. This has allowed us to decompose large software projects into smaller sets of libraries or modules.
<!--more-->
In C, we usually use a compiler which preprocesses source code, compiles it, assembles it, and then links it for us. The first step in this is preprocessing source code.
Let’s look at an example for a simple program.

<script src="https://gist.github.com/kunaljaydesai/730d8adf14d074c0f2ae497658dec790.js"></script>

Preprocessing source code will process all the macros like AGE and evaluate the conditionals to output something like the following:

<script src="https://gist.github.com/kunaljaydesai/8d9a8f6d5994f0d55a9f2a87bcec9ddc.js"></script>

You can try this by running `gcc -E main.c -o main.i` and looking at the output.
Once the code has been preprocessed, it is given to the compiler which takes each file and turns the source code into assembly language. To see this, we can run `gcc -S main.c -o main.s` and peek at the output. If we run this on our original program, we will get something that looks like this:

<script src="https://gist.github.com/kunaljaydesai/1ed57c89a288184db82ada2db01f2ad5.js"></script>

You will notice that there is a call to `_printf` and `_ask_question` but the definition is nowhere to be found in the assembly. How does the computer know what to do when these functions are invoked? That is the job of the linker.

The next step is to translate the assembly code into machine instructions. This can be accomplished by running the assembler using `as` or `gcc -f main.c -o main.o`. If we want to inspect the output of the object file, we can use a tool called objdump which shows us a human-readable version of the file. It will look like the following (`objdump -d main.o`):

<script src="https://gist.github.com/kunaljaydesai/8eb2060d0595d5d412e0429c84c15341.js"></script>

Looking at the symbol table, you’ll see:

<script src="https://gist.github.com/kunaljaydesai/8780e87e563b11e948a59e848fc4c4ad.js"></script>

You’ll notice that the functions are still undefined. Let’s try to run the linker on this object file and see what happens. Running `ld -o age main.o` results in:
```
Undefined symbols for architecture x86_64:  
    "_ask_question", referenced from:
        _main in main.o
    "_printf", referenced from:
        _main in main.o
ld: symbol(s) not found for architecture x86_64
```
As expected, the linker can’t find the definitions of these functions yet. Once the assembler has done its magic, it is finally time to figure out where these functions are defined. Let’s define a separate file which defines the `ask_question` function.

<script src="https://gist.github.com/kunaljaydesai/84a9e26941cde8558190b1a00b07cbf9.js"></script>

We should assemble this file as well and get its relocatable object file. We can do this by running `gcc -c helper.c -o helper.o`. Notice that this file also doesn’t have `printf` defined. Let’s run the linker this time with the relocatable object file that contains the definition of `ask_question`. We can run `ld -o age main.o helper.o` which outputs:
```
Undefined symbols for architecture x86_64:
  "_printf", referenced from:
      _main in main.o
      _ask_question in helper.o
ld: symbol(s) not found for architecture x86_64
```
Progress! We were able to resolve the `ask_question` symbol this time. This is a process known as static linking, you can read more about it in this article. However, we are still unable to find `printf` anywhere. Functions like `printf` are in shared modules and are dynamically linked. We will discuss this in a future <a href="/2020/06/02/static-linking.html">post</a>.