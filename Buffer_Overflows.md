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
