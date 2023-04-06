# Chapter 2

- Code Reference: https://github.com/remzi-arpacidusseau/ostep-code/tree/master/intro

## Virtualization

- The OS takes a **physical resource** (processor, memory, or a disk) and transforms it into a virtual form of itself. Thus, we sometimes refer to the operating system as a virtual machine
- And hence, the OS is sometimes called as **resource manager**

### Virtualizing CPU

- Turns a single CPU (or a small set of them) into a **seemingly infinite number** of CPUs
- This allows many programs to seemingly run simultaneously
- We will need to solve some problems, e.g. if two programs run simultaneously, which one should take over the CPU and run?
  - This is why we need an OS _policy_
- Ex. [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/intro/cpu.c)

### Virtualizing Memory

- Turns a physical memory into a number of **seemingly same private memory** for each process
- This allows many programs to seemingly share the same part of memory, but each of them is assigned a private part of the physical memory by the OS
- Ex. [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/intro/mem.c)

## Concurrency

- This refers to a host of problems that arise and must be addressed simultaneously (concurrently) in the same program
- Ex. [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/intro/threads.c)

## Persistence

- This refers to the storage of values that will not go away even if the power is off, e.g. files, software
  - For example, **hard drive** is a common repository for long-lived information
- Unlike CPU and memory, the OS doesn't create a private and virtualized disk for each application.
- It assumes users will want to share information on the disk
- Ex. [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/intro/io.c)

## Keywords

1. Memory
   - An array of bytes; to **read** memory, we need an **address** to access the data; to **write** memory, we need to specify new data to write into the given address
2. Thread
   - A function running within the same memory space as other functions, with more than one of them active at a time
3. System Calls
   - The API that OS provides for other applications running on the OS to utilize the resources
   - Or we can call them **standard libraries**
4. PID (process identifier)
   - An unique identifier for each running process
5. File System
   - The software in the OS that usually manages the disk
6. Isolation
   - Isolating processes from one another is the key to protection of processes
7. Batch Processing
   - Setting a number of jobs and running them in a _batch_

## Notes

- Instructions, i.e. the program is in memory, too, so memory is accessed on each instruction fetch
