# Chapter 28

## Lock

- Locks are necessarily helpful when we are updating a shared variables across threads
- We can use the lock to execute a critical section as a single atomic instruction
- The process of `lock()` is as the following:
  1. if no other thread holds the lock, the caller thread will acquire the lock and enter the critical section
  2. if other thread holds the lock, the `lock()` call will not return until the lock is free and acquired by the caller thread
- The process after calling `unlock()` is as the following:
  1.  if no other thread is waiting for the lock, the state of lock will be set to **free**
  2.  if there are waiting threads, one of them will be informed of this change and their `lock()` call will return
- We can use just one single lock in a program (a coarse-grained approach), but we can also use a lock for each shared variable (a fine-grained approach) to promote concurrency while maintaining mutual exclusion

### Basic Criteria

1. Mutual Exclusion
2. Fairness (Does each thread contending for the lock get a fair shot at acquiring it once it is free?)
3. Performance (the time overheads added by using the lock)
   1. When a single thread is running and grabs and releases the lock
   2. When multiple threads are contending for the lock on a single CPU

### Different Approaches

#### Approach #1: Controlling Interrupts

- Disable interrupts in a critical section
- Advantages
  1.  Simplicity
- Disadvantages
  1.  This allows a process to perform a privileged operation (turnning on and off interrupts)
  2.  This does not work on multiprocessors (if one thread is running on a CPU and another thread is running on another CPU, they can still both enter the same critical section)
  3.  Turning off interrupts for extended periods of time can lead to interrupts becoming lost.
  4.  **Inefficiency**
- But the OS itself will use interrupt controlling technique to guarantee atomicity inside itself since the OS can trust itself

#### Approach #2: Just Using Loads/Stores

- Use a single variable, say `flag`, to indicate whether a thread is holding a lock
- And the thread will first be looping in a while loop to check if flag is `True`
- Problems

  1. correctness

     - If two threads enter the while loop at the same time because of scheduling, they will both enter the critical section

  2. performance

     - Spin waiting wastes time for the CPU to run the same instruction multiple times with the same result
     - CPU will also needs to perform a lot of useless context switches

#### Approach #3: Test-and-Set Primitive Provided by Hardware

- It is basically a functionality provided by the hardware to atomically **returns the old value while setting a new value simultaneously**
- Test-and-Set means **test (the old value)** and **set (the new value)**
- This type of lock is called **spin-lock** because the lock simply spins
- To work correctly on a single processor, it requires a **preemptive scheduler**, instead of the one that waits for syscall.
- Otherwise, a thread spinning on a CPU will never relinquish it
- Advantages:
  1.  correctness
  2.  simplicity
- Disadvantages:

  1.  fairness: spin locks can't provide any fairness
  2.  performance:

      - On a single CPU, we still wait a lot of CPU cycles on the busy checking for each waiting thread.
      - On multiple CPU's, if the criticsl section is small, spin locks work work pretty well. During a timer interrupt cycle, the lock might be available on the waiting thread and it can grab the lock

##### Pseudo Code

```c
int TestAndSet(int *old_ptr, int new) {
    int old = *old_ptr; // fetch old value at old_ptr
    *old_ptr = new;     // store ’new’ into old_ptr
    return old;         // return the old value
}

typedef struct lock_t {
  int flag;
} lock_t;

void init(lock_t *lock) {
  lock->flag = 0;
}

void unlock(lock_t *lock) {
  lock->flag = 0;
}

void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 0, 1) == 1); // spin wait
}
```

#### Approach #4: Compare-and-Swap Primitive Provided by Hardware

- A more flexible version of test-and-set
- It gives the caller the ability to set the expected value

##### Pseudo Code

```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int original = *ptr;
    if (original == expected)
        *ptr = new;
    return original;
}

void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1); // spin wait
}
```

#### Approach #4: Loaded-Linked & Store-Conditional Primitives Provided by Hardware

- Loaded-Linked is simply loading a value pointed by a pointer
- Store-Condition is to check if no intervening store to the address has taken place
- We can now create a lock by the following steps:
  1. spin-wait until `flag` is set to 0 with loaded-linked
  2. try to set the flag with store-conditional

##### Pseudo Code

```c
int LoadedLinked(int* ptr) {
  return *ptr;
}

int StoreConditional(int* ptr, int value) {
  if (no update to ptr since load linked to ptr) {
    *ptr = value;
    return 1;
  }
  else {
    return 0;
  }
}

void lock(lock_t *lock) {
    while (LoadedLinked(&lock->flag, 0, 1) == 1); // spin wait

    if (StoreCondition(&lock->flag, 1) == 1) return; // if we can successfully set flag to 1, return
    // otherwise, do it all over again
}
```

#### Approach #5: Fetch-and-Add Primitive Provided by Hardware

- Fetch-and-Add is simply adding one to the value the address is pointing to
- We can not create a lock by the following steps:
  1. fetch a ticket for the thread itself
  2. spin-wait until the turn is equal to my ticket
- Advantage:
  - fairness: all threas will eventually enter their critical sections

##### Pseudo Code

```c

int FetchAndAdd(*ptr) {
  int old = *ptr;
  *ptr = old + 1;
  return old;
}

void lock(lock_t *lock) {
  int my_turn = FetchAndAdd(&lock->ticket);
  while (lock->turn !== my_turn);
}

typedef struct lock_t {
  int ticket;
  int turn;
} lock_t;

void init(lock_t *lock) {
  lock->ticket = 0;
  lock->turn = 0;
}

void unlock(lock_t *lock) {
  lock->turn += 1;
}
```

#### OS Optimization #1: Just Yield

- We know that if the condition of the while loop is not true, it will never be true for the rest of the time slice (timespan between interrupts)
- Therefore, we make the thread give up the CPU immediately and go back to **ready** state by calling `yield()`
- However, we are still wasting resources on context switches
- Ex.

```c
void lock() {
  while (condition) yield();
}
```

#### OS Optimization #2: Use A Queue And Use **Sleep** State

- Other than using the tickent and turn, we can use a queue to keep track of the sequence of the threads to enhance fairnesss
- And we use two locks, guard and flag, to do works separately, **putting threads to sleep** and **acquiring the real lock**
- We also reduce a lot of unneccessary spinning for the real lock, because the guard lock is released fast
- We use `park()` and `unpark()` to put a thread to sleep and wake a thread separately
- A little problem is

  - if a new thread is being added to queue and right before being parked, and if a switch happens to be triggered and the thread that holds the lock is awakened, then the waking thread might release the lock and potentitally the new thread will never be awakened again if no thread is calling `lock()`
  - we can use a system call called `setpark()`
  - by calling this, a thread can indicate it is about to park. If it then happens to be interrupted and another thread calls unpark before park is actually called, the subsequent park returns immediately instead of sleeping.
  - Ex.

  ```c
    queue_add(m->q, gettid());
    setpark();
    m->guard = 0;
  ```

- Ex.

```c
typedef struct __lock_t {
  int flag;
  int guard;
  queue_t *q;
} lock_t;

void lock_init(lock_t *m) {
  m->flag = 0;
  m->guard = 0;
  queue_init(m->q);
}

void lock(lock_t *m) {
  while (TestAndSet(&m->guard, 1) == 1); // acquire guard lock by spinning

  if (m->flag == 0) {
    m->flag = 1; // acquire lock
    m->guard = 0; // release guard lock to keep putting other thread to sleep
  }
  else {
    queue_add(m->q, gettid()); // put the current thread to sleep
    m->guard = 0;
    park()
  }
}

void unlock(lock_t *m) {
  while (TestAndSet(&m->guard, 1) == 1); // try to acqure guard lock by spinning
  if (queue_empty(m->q))
    m->flag = 0; // let go of the lock because no body is waiting
  else
    unpark(queue_remove(m->q)); // hold the lock and release the next thread

  m->guard = 0; // release guard lock to keep putting other thread to sleep
}
```

## Keywords

1. spin-waiting: a thread waits and endly checks to acquire a lock that is already held
2. preemptive scheduler: a scheduler that uses a timer to interrupt and regain control
