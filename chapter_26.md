# Chapter 26

## Thread

- A new abstraction within a single running program
- A multi-threaded program has multiple points of execution (multiple PC's)

### Differences between Process & Thread

- They share the _same virtualized memory_ and can _access the same data_
- Therefore, in _context switch_, the address space remains the same
- Also in context switch, we save process state in **Process Control Block (PCB)** for a process, but we thread states in **Thread Control Block (TCB)**
- In multithreaded process, there will be **multiple stacks in one address space**, a stack for each thread

### Why Thread?

1. Parallelism
   - Split up the work to each CPU
2. To avoid blocking program
   - Utilize the CPU while a thread is waiting for a blocking instruction complete, e.g. I/O request

### Example

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/threads-intro/p1.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#include "common.h"
#include "common_threads.h"

void *mythread(void *arg) {
    printf("%s\n", (char *) arg);
    return NULL;
}

int main(int argc, char *argv[]) {

    if (argc != 1) {
      fprintf(stderr, "usage: main\n");
      exit(1);
    }

    pthread_t p1, p2;
    printf("main: begin\n");

    Pthread_create(&p1, NULL, mythread, "A");
    Pthread_create(&p2, NULL, mythread, "B");

    // join waits for the threads to finish
    Pthread_join(p1, NULL);
    Pthread_join(p2, NULL);

    printf("main: end\n");
    return 0;
}
```

## Threads Execution Ordering

- With the example above, we can't assume a particular order that OS scheduler will follow when running the program
- There could be multiple scenarios
  1. creates T1, creates T2, T1 runs, T2 runs, main waits for T1, main waits for T2
  2. creates T1, T1 runs, creates T2, T2 runs, main waits for T1, main waits for T2
  3. creates T1, creates T2, T2 runs, T1 runs, main waits for T1, main waits for T2
  4. creates T1, creates T2, T2 runs, main waits for T1, T1 runs, main waits for T2
  5. ...

## Shared Data

- Because threads in a process share the same address space together, when they are trying to access the shared data, something unexpected might happen due to the uncertainty of scheduling
- For example, ([source](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/threads-intro/t1.c))

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#include "common.h"
#include "common_threads.h"

int max;
volatile int counter = 0; // shared global variable

void *mythread(void *arg) {
    char *letter = arg;
    int i; // stack (private per thread)
    printf("%s: begin [addr of i: %p]\n", letter, &i);
    for (i = 0; i < max; i++) {
      counter = counter + 1; // shared: only one
    }
    printf("%s: done\n", letter);
    return NULL;
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
      fprintf(stderr, "usage: main-first <loopcount>\n");
      exit(1);
    }
    max = atoi(argv[1]);

    pthread_t p1, p2;
    printf("main: begin [counter = %d] [%x]\n", counter, (unsigned int) &counter);
    Pthread_create(&p1, NULL, mythread, "A");
    Pthread_create(&p2, NULL, mythread, "B");

    // join waits for the threads to finish
    Pthread_join(p1, NULL);
    Pthread_join(p2, NULL);
    printf("main: done\n [counter: %d]\n [should: %d]\n", counter, max * 2);
    return 0;
}
```

```console
$ > ./t1
main: begin [counter = 0] [d654048]
A: begin [addr of i: 0x70000351df9c]
B: begin [addr of i: 0x7000035a0f9c]
A: done
B: done
main: done
 [counter: 2362069]
 [should: 4000000]
$ >
```

## Uncontrolled Scheduling

- We will be showing a race condition to demostrate why scheduling can result in unexpected result sometimes
- Considering the following assembly code

```asm
100 mov 0x8049a1c, %eax
105 add $0x1, %eax
108 mov %eax, 0x8049a1c
```

- Let's say we have two threads running the same piece of code, and an interrupt happens after the second instruction of the first thread
- And if the scheduler decides to run the second thread, we will be entering the second thread without the correct global value (0x8049a1c)
- And the result of the global value will end up being added by one only
- Here is the timeline

| OS            | Thread 1                | Thread 2            | PC  | eax    | counter |
| ------------- | ----------------------- | ------------------- | --- | ------ | ------- |
|               | before critical section |                     | 100 | 0      | 50      |
|               | mov 0x8049a1c, %eax     |                     | 105 | **50** | 50      |
|               | add $0x1, %eax          |                     | 108 | **51** | 50      |
| **interrupt** |                         |                     |     |        |         |
| save T1       |                         |                     |     |        |         |
| restore T2    |                         |                     | 100 | 0      | 50      |
|               |                         | mov 0x8049a1c, %eax | 105 | **50** | 50      |
|               |                         | add $0x1, %eax      | 105 | **51** | 50      |
|               |                         | mov %eax, 0x8049a1c | 105 | 51     | **51**  |
| **interrupt** |                         |                     |     |        |         |
| save T2       |                         |                     |     |        |         |
| restore T1    |                         |                     | 108 | **51** | 51      |
|               | add $0x1, %eax          |                     | 113 | 51     | 50      |

## Keywords

1. Thread-local storage: the stack of the relevant thread
2. `pthread_join(thread)`: the function to have main thread wait for a thread to finish
3. race condition: the results depend on the timing execution of the code
4. critical section: a scenario when multiple threads of execution enter the critical section and both attempt to update a shared data
5. synchronization primitives: the instructions that couldn't be interrupted (**needs more research**)
