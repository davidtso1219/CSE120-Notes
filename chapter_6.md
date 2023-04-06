# Chapter 6

## Challenges of CPU Virtualization

1. performance: how to achieve virtualization without adding excessive overhead to the system
2. control: how to run process efficiently without losing control over the CPU\
   - Without control, a process can take over the machine easily or access information it should not be allowed to

## Limited Direct Execution

- We have a technique called **limited direct execution** to make a program run as fast as possible
- Direct execution means to run the program directly on the CPU
- Limited means to restrict the privilege of user programs to protect our resource

## Direct Execution Protocol

| OS                                  | Program                   |
| ----------------------------------- | ------------------------- |
| Create an entry in the process list |                           |
| Allocate memory for program         |                           |
| Load program into memory            |                           |
| Set up stack with _argc/argv_       |                           |
| Clear registers                     |                           |
| Execute **call main()**             |                           |
|                                     | Run main()                |
|                                     | Executre return in mani() |
| Free memory of process              |                           |
| Remove process from process list    |                           |

## Challenges of Direct Execution Protocol

1. How can we make sure the program doesn't do anything we don't want it to do
2. How can we take over the CPU and switch to another process

## Our Approach to Challenge #1: Restricted Operations

- We introduce the **modes**, _kernel mode_ and _åuser mode_
  - In user mode, we restrict what the program can do
  - In kernel mode, we allow program to do whatever it wants, including privileged operations

### System Calls

- The API that kernel carefully provides to user program to perform privileged operations
- In order to execute a system call, the procedure would be like
  1.  user program executes a special **trap instruction**
  2.  the instruction jumps into kernel and raise the privilege level to kernel mode
  3.  the system performs privileged operations (if allowed)
  4.  the OS calls a special **return-from-trap instruction** to user program when it finishes and reduces the privilege level to user mode

### Trap Table

- In a system call, the trap gets the code from a map (trap table) mapping from a number (system-call number) to a set of instructions (trap handlers)
  - We can't let call processes (user programs) decide what to run in system calls.
  - Otherwise, they will have the privileged level to run the code that they specify
- Trap table is set up at boot time, when the kernel does it in the kernel mode

## Limited Direct Execution Protocol

- With the idea of our approach to challenge #1, we create the **Limited Direct Execution Protocol**

| OS @ boot time            | Hardware                                           |
| ------------------------- | -------------------------------------------------- |
| **Initialize trap table** |                                                    |
|                           | Save addresses of syscall handlers (trap handlers) |

| OS @ main                           | Hardware                       | Program                 |
| ----------------------------------- | ------------------------------ | ----------------------- |
| Create an entry in the process list |                                |                         |
| Allocate memory for program         |                                |                         |
| Load program into memory            |                                |                         |
| Set up stack with _argc/argv_       |                                |                         |
| Fill kernal stack with _reg/PC_     |                                |                         |
| **return-from-trap**                |                                |                         |
|                                     | restore regs from kernel stack |                         |
|                                     | move to user mode              |                         |
|                                     | jump to main                   |                         |
|                                     |                                | Run main()              |
|                                     |                                | ...                     |
|                                     |                                | Call syscall            |
|                                     |                                | **trap** into OS        |
|                                     | save regs to kernel stack      |                         |
|                                     | move to kernal mode            |                         |
|                                     | jump to trap handler           |                         |
| Handle trap                         |                                |                         |
| _Do work of syscall_                |                                |                         |
| **return-from-trap**                |                                |                         |
|                                     | restore regs from kernel stack |                         |
|                                     | move to user mode              |                         |
|                                     | jump to PC after trap          |                         |
|                                     |                                | ...                     |
|                                     |                                | return from main        |
|                                     |                                | **trap** (via _exit()_) |
| Free memory of process              |                                |                         |
| Remove process from process list    |                                |                         |

## Our Approaches to Challenge #2: Switching between Processes

- The challenge is that when the user program has the control over CPU, the OS can't do much until it regains the control

1. Approach #1: (cooperative) wait for system calls
   - The OS trusts the process and wait the process to call a system call to give the CPU control back to the OS
   - Or the OS waits until the process to make some illegal moves to create an exceptional event and takes over the control
   - However, there might be some cases, where the process never makes system calls or illegal moves for too long.
     - For example,a process ends up in an infinite loop. (We can only reboot the machine to regain the control)
2. Approaches #2: (non-cooperative) the OS takes control
   - The OS set a timer interrupt to interrupt the running process every many milliseconds to regain the control
   - The timer interrupt is set using a timer device, so it is actually the hardware stops the user program

### Scheduler

- A part of the OS that will make the decision to whether continue running the current process or switch to a new one

### Context Switch

- The OS saves the register values for the current process and restore register values for the new process

## Limited Direct Execution Protocol (Timer Interrupt)

- With the idea of our approaches to challege #1 and #2, we create a revised version of Limited Direct Execution Protocol

| OS @ boot time            | Hardware                                                |
| ------------------------- | ------------------------------------------------------- |
| **Initialize trap table** |                                                         |
|                           | Save addresses of syscall handlers and interrupt hadler |
| **start interrupt timer** |                                                         |
|                           | start timer                                             |
|                           | interrupt CPU in X ms                                   |

| OS @ main                      | Hardware                                 | Program   |
| ------------------------------ | ---------------------------------------- | --------- |
|                                |                                          | Process A |
|                                |                                          | ...       |
|                                | save regs(A) to kernel_stack(A)          |           |
|                                | move to kernal mode                      |           |
|                                | jump to trap handler (interrupt handler) |           |
| Handle trap                    |                                          |           |
| Call _switch()_ routine        |                                          |           |
| - save regs(A) -> proc_t(A)    |                                          |           |
| - restore regs(B) -> proc_t(B) |                                          |           |
| - switch to kernal_stack(B)    |                                          |           |
| **return-from-trap**           |                                          |           |
|                                | restore regs(B) from kernel_stack(B)     |           |
|                                | move to user mode                        |           |
|                                | jump to B's PC after trap                |           |
|                                |                                          | Process B |
|                                |                                          | ...       |
