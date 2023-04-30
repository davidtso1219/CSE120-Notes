# Chapter 30

## Conditional Variables

- In many cases, a thread wishes to check whether a **condition** is true before continuing its execution
- For example, a parent thread wants to wait until the child thread has finished
- Definition
  - A condition variable is a **queue** that threads can put themselves on when some **condition** is not as desired.
  - When the **condition** is met, we wake one or more of the waiting threads in the queue

### Spin-based Approach

- We could use a shared variable between the parent and the child
- At the same time, we create a while loop in parent thread to wait until the shared variable becomes true
- But this is extremely inefficient and wasteful
- Implementation

```c
volatile int done = 0;

void *child(void *arg) {
  printf("child\n");
  done = 1;
  return NULL;
}

int main(int argc, char *argv[]) {
  printf("parent: begin\n");
  pthread_t c;
  Pthread_create(&c, NULL, child, NULL); // create child
  while (done == 0)
      ; // spin
  printf("parent: end\n");
  return 0;
}
```

### Example in C

- To create a condition variable, we use `pthread_cond_t c`
- To put a thread itself to sleep, we call `wait()`

  - In POSIX, `pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);`
  - It takes a lock as an argument, and it assumes the lock is locked when `wait()` is called
  - The responsiblity of wait is to release the lock and put the calling thread ot sleep atomically
  - Also, when the caller thread wakes up, it must re-acquire the lock before returning to the caller

- When a thread's state has changed and wants to wake a sleeping thread, we call `signal()`

  - In POSIX, `pthread_cond_signal(pthread_cond_t *c);`

- Implementation

```c
int done  = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c  = PTHREAD_COND_INITIALIZER;
void thr_exit() {
  Pthread_mutex_lock(&m);
  done = 1;
  Pthread_cond_signal(&c);
  Pthread_mutex_unlock(&m);
}
void *child(void *arg) {
  printf("child\n");
  thr_exit();
  return NULL;
}
void thr_join() {
  Pthread_mutex_lock(&m);
  while (done == 0)
    Pthread_cond_wait(&c, &m);
  Pthread_mutex_unlock(&m);
}
int main(int argc, char *argv[]) {
  printf("parent: begin\n");
  pthread_t p;
  Pthread_create(&p, NULL, child, NULL);
  thr_join();
  printf("parent: end\n");
  return 0;
}
```

#### Different Scenarios

1. Parent calls `join()` first
   - the parent will acquire the lock
   - the parent will check if `done` is 0, and enter the while loop because it is
   - the parent will call `wait`, release the lock, and sleeps
   - the child will run and call `exit()`
   - the child will acquire the lock and set `done` to 1
   - the child will signal the parent and release the lock
   - the parent will run (returning from `wait()`) and unlock the lock
2. Child runs immediately upon creation
   - the child will run and call `exit()`
   - the child will acquire the lock and set `done` to 1
   - the child will signal no thread and release the lock
   - the parent will run and not enter the while loop because `done` is not 0
   - the parent will unlock and return

## Producer / Consumer Problem (Bounded Buffer Problem)

- Producers generate data items and place them in a buffer
- Consumers grab said items from the buffer and consume them in some way.
- For example, in a multi-threaded webserver, a producer puts HTTP requests into a work queue
- consumer threads take requests out of the work queue and process them
- Intuitive Implementation

```c
int buffer;
int count = 0;

void put(int value) {
  assert(count == 0);
  count = 1;
  buffer = value;
}

int get() {
  assert(count == 1);
  count = 0;
  return buffer;
}
```

```c
void *producer(void* arg) {
  int i;
  int loops = (int) arg;

  for (i = 0; i < loops; i++) {
    put(i);
  }
}

void *consumer(void* arg) {
  while (1) {
    int tmp = get();
    printf("%d\n", tmp);
  }
}
```

### (Broken) Solution #1: use lock to atomically modify shared data

- The main idea is to
  - protect the critical section using lock
  - use conditional variable to signal other threads
- This **will** work for a **single** producer and a **single** consumer
- This will **NOT** work for **multiple** producers or **multiple** consumers
- For example,
  - a consumer first enters the loop and goes to sleep because no data is stored
  - a producer produces a task for the consumer and put it to ready
  - then **another consumer** comes and take the task and finishes it
  - after a context switch to the first consumer, it attempts to `get()` but no task is in the queue, so an assertion triggers
- To be concise, the producer wakes up a consumer for the task it produces and another consumer sneaks in before the awakened consumer

```c
int loops; // must initialize somewhere...
cond_t  cond;
mutex_t mutex;

void *producer(void *arg) {
  int i;
  for (i = 0; i < loops; i++) {
    Pthread_mutex_lock(&mutex);
    if (count == 1)
        Pthread_cond_wait(&cond, &mutex);
    put(i);
    Pthread_cond_signal(&cond);
    Pthread_mutex_unlock(&mutex);
  }
}

void *consumer(void *arg) {
  int i;
  for (i = 0; i < loops; i++) {
    Pthread_mutex_lock(&mutex);
    if (count == 0)
        Pthread_cond_wait(&cond, &mutex);
    int tmp = get();
    Pthread_cond_signal(&cond);
    Pthread_mutex_unlock(&mutex);
    printf("%d\n", tmp);
  }
}
```

### (Broken) Solution #2: use while instead of if to check after awakened

- The main idea is to solve the problem mentioned above
- Once a thread is awakened by the signal, it should go back to check again whether it condition is still met using a while loop
- However, this still will **NOT** work because we don't know which kind of thread will be awaked after `signal()`
- For example,
  - two consumers enter the loop separtely and both go to sleep because no data is stored
  - a producer produces a task for awake a consumer and sleeps because the data is full
  - the awakened consumer finishes the task and awake the next thread using the conditional variable
  - however, the next sleeping thread to be awake could be the other consumer
  - if the other consumer is awakened, it checks the data and goes to sleep immediately because no data is stored
  - all threads are now sleeping
- To be concise, the consumer should but not necessarily wake up the producer

```c
int loops; // must initialize somewhere...
cond_t  cond;
mutex_t mutex;

void *producer(void *arg) {
  int i;
  for (i = 0; i < loops; i++) {
    Pthread_mutex_lock(&mutex);
    while (count == 1)
        Pthread_cond_wait(&cond, &mutex);
    put(i);
    Pthread_cond_signal(&cond);
    Pthread_mutex_unlock(&mutex);
  }
}

void *consumer(void *arg) {
  int i;
  for (i = 0; i < loops; i++) {
    Pthread_mutex_lock(&mutex);
    while (count == 0)
        Pthread_cond_wait(&cond, &mutex);
    int tmp = get();
    Pthread_cond_signal(&cond);
    Pthread_mutex_unlock(&mutex);
    printf("%d\n", tmp);
  }
}
```

### Solution #3: use different conditional variables

- Use one for producers and one for consumers

```c
int buffer[MAX];
int fill_ptr = 0;
int user_ptr = 0;
int count    = 0;

void put(int value) {
  buffer[fill_ptr] = value;
  fill_ptr = (fill_ptr + 1) % MAX;
  count++;
}

int get() {
  int tmp = buffer[use_ptr];
  use_ptr = (use_ptr + 1) % MAX;
  count--;
  return tmp;
}
```

```c
int loops; // must initialize somewhere...
cond_t  empty, fill;
mutex_t mutex;

void *producer(void *arg) {
  int i;
  for (i = 0; i < loops; i++) {
    Pthread_mutex_lock(&mutex);
    while (count == MAX)
        Pthread_cond_wait(&empty, &mutex);
    put(i);
    Pthread_cond_signal(&fill);
    Pthread_mutex_unlock(&mutex);
  }
}

void *consumer(void *arg) {
  int i;
  for (i = 0; i < loops; i++) {
    Pthread_mutex_lock(&mutex);
    while (count == 0)
        Pthread_cond_wait(&fill, &mutex);
    int tmp = get();
    Pthread_cond_signal(&empty);
    Pthread_mutex_unlock(&mutex);
    printf("%d\n", tmp);
  }
}
```

## Convering Condition

- Sometimes waking which producer or consumer could be a problem, too.
- For example,
  - If we have no free space to allocate, and two consumer threads are waiting for more memory to allocate
  - They need 100, 20 units of space, respectively
  - If a producer produces 50 units of space, but wake up only the first consumer thread that needs 100
  - Then it wouldn't be the best to choose which thread to wake up
- One possible solution is to replace `signal()` with `broadcast()` and wake up all consumer threads
- But the performance might not be good because many of them will be ready and put to sleep very soon

```c
// how many bytes of the heap are free?
int bytesLeft = MAX_HEAP_SIZE;

// need lock and condition too
cond_t c;
mutex_t m;

void* allocate(int size) {
  Pthread_mutex_lock(&m);
  while (bytesLeft < size)
  Pthread_cond_wait(&c, &m);
  void *ptr = ...; // get mem from heap
  bytesLeft -= size;
  Pthread_mutex_unlock(&m);
  return ptr;
}

void free(void *ptr, int size) {
  Pthread_mutex_lock(&m);
  bytesLeft += size;
  Pthread_cond_signal(&c); // whom to signal??
  Pthread_mutex_unlock(&m);
}
```
