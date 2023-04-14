# Chapter 27

## Thread Creation

- We are going to create a thread using `pthread_create()`
- Important! **NEVER return a pointer referring to something created on the thread stack**
- Its signature looks like this:

```c
#include <pthread.h>

int pthread_create(
  pthread_t                 *thread,
  const pthread_attr_t      *attr,
  void                      *(*start_routine)(void*),
  void                      *arg
);
```

### `pthread_create()` arguments

1. `pthread_t *thread` is a pointer to a structure of type, and the caller of `pthread_create()` will use this pointer to interact with the thread instance
2. `const pthread_attr_t attr` is used to specify any attributes this thread might have such as stack size, scheduling priority, etc.
3. `void *(*start_routine)(void*)` is a function pointer to tell the thread which function to start the thread with.
4. `void *arg` the argument that needed to be passed into the **start_routine**

### More on `start_routine`

- If the `start_routine` takes different types of arguments, we need to check the type of the `arg` that we pass to the `pthread_create`
- If the `start_routine` returns different type, we should also change the return type in the signature of `pthread_create`
- For example, if this routine requires an interger argument

```c
int pthread_create(..., // first two args are the same
                   void *(*start_routine)(int),
                   int arg
);
```

- For another example, if this routine takes a void pointer as an argument, and returns an integer

```c
int pthread_create(..., // first two args are the same
                   int (*start_routine)(void *),
                   void *arg
);
```

### Example

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/threads-api/thread_create.c)

```c
#include <assert.h>
#include <stdio.h>
#include <pthread.h>

typedef struct {
    int a;
    int b;
} myarg_t;

void *mythread(void *arg) {
    myarg_t *args = (myarg_t *) arg;
    printf("%d %d\n", args->a, args->b);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    myarg_t args = { 10, 20 };

    int rc = pthread_create(&p, NULL, mythread, &args);
    assert(rc == 0);
    (void) pthread_join(p, NULL);
    printf("done\n");
    return 0;
}
```

## Thread Completion

- In order to wait for a thread to complete, we use `pthread_join()`
- Important! **NEVER return a pointer referring to something created on the thread stack**
- Its signature looks like this:

```c
int pthread_join(pthread_t thread, void **value_ptr);
```

### `pthread_join()` arguments

1. `pthread_t thread`: this specifies which thread to wait for
2. `void **value_ptr`: this is the return value from the thread

### Examples

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/threads-api/thread_create_with_return_args.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include "common_threads.h"

typedef struct {
    int a;
    int b;
} myarg_t;

typedef struct {
    int x;
    int y;
} myret_t;

void *mythread(void *arg) {
    myarg_t *args = (myarg_t *) arg;
    printf("args %d %d\n", args->a, args->b);
    myret_t *rvals = malloc(sizeof(myret_t));
    assert(rvals != NULL);
    rvals->x = 1;
    rvals->y = 2;
    return (void *) rvals;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    myret_t *rvals;
    myarg_t args = { 10, 20 };
    Pthread_create(&p, NULL, mythread, &args);
    Pthread_join(p, (void **) &rvals);
    printf("returned %d %d\n", rvals->x, rvals->y);
    free(rvals);
    return 0;
}
```

- If we are simply passing **a single value**, we don't have to package it up as an argument
- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/threads-api/thread_create_simple_args.c)

```c
#include <stdio.h>
#include <pthread.h>
#include "common_threads.h"

void *mythread(void *arg) {
    long long int value = (long long int) arg;
    printf("%lld\n", value);
    return (void *) (value + 1);
}

int main(int argc, char *argv[]) {
    pthread_t p;
    long long int rvalue;
    Pthread_create(&p, NULL, mythread, (void *) 100);
    Pthread_join(p, (void **) &rvalue);
    printf("returned %lld\n", rvalue);
    return 0;
}
```

## Lock

- Locks are used to provide mutual exclusion to critical sections of a thread
- We should create a lock following these steps:
  1. Initialize a lock
  2. Wait for the lock to be released and acquire the lock
  3. Enter a critical section
  4. Release the lock
  5. Destroy the lock

### Initialize A Lock

- There are two ways to initialize a lock

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```

```c
int rc = pthread_mutex_init(&lock, NULL);
assert(rc == 0); // always check success!
```

### Acquire And Release A Lock

```c
pthread_mutex_t lock;
pthread_mutex_lock(&lock);
x = x + 1; // or whatever your critical section is
pthread_mutex_unlock(&lock);
```

### Destroy A Lock

- At the end of the program, we could release the lock by calling `pthread_mutex_destroy()`

### Other Lock Interaction

- `pthread_mutex_trylock(pthread_mutex_t *mutex);`
  - returns a failure if the lock is already held
- `pthread_mutex_timedlock(pthread_mutex_t *mutex, struct timespec *abs_timeout);`
  - returns after a timeout or after requiring the lock, whichever happens first

## Conditional Variables

- This will be covered more in chapter 30
- Conditional Variables are used to signal between threads if one thread needs to pause and continue after some actions of the other thread
- The timeline looks like this
  1. Thread A calls `pthread_cond_wait` to wait for some signal from Thread B
  2. Thread B calls `pthread_cond_signal`
- These operations need to be put in a pair of `pthread_mutex_lock` and `pthread_mutex_unlock`
- Notes: `pthread_cond_wait` releases the lock when putting the caller to sleep and reacquire at the beginning of the wait sequence
