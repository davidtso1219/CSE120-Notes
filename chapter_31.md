# Chapter 30

## Semaphore

- A semaphore is an object with an integer value that we can manipulate with two routines
  1.  `sem_wait()`
      - either returns right away
      - or decrements the value of the semaphore, pauses the caller, and waits for a post call
  2.  `sem_post()`
      - increments the value of the semaphore, and wake up a waiting thread if any
- If the value of the semaphore is negative, which is the number of waiting threads

### Initialization

- Because the value in a semaphore determines everything, initializing the value becomes extremely important
- Ex.

```c
#include <semaphore.h>

sem_t s;
sem_init(&s, 0, 1);
```

## Binary Semaphore (Lock)

- The main idea is to use the value of the semaphore to denote whether the semaphore is held or not
- And if it is held, it will be the number of waiting threads
- Timeline if two threads are using a semaphore

| Val | Thread 0                | State | Thread 1           | State |
| --- | ----------------------- | ----- | ------------------ | ----- |
| 1   |                         | Run   |                    | Ready |
| 1   | call `sem_wait`         | Run   |                    | Ready |
| 0   | `sem_wait` returns      | Run   |                    | Ready |
| 0   | enters crit-sect        | Run   |                    | Ready |
| 0   | interrupt; switch to T1 | Ready |                    | Run   |
| 0   |                         | Ready | call `sem_wait`    | Run   |
| -1  |                         | Ready | decrement Val      | Run   |
| -1  |                         | Ready | (Val < 0): sleep   | Sleep |
| -1  |                         | Run   | switch to T0       | Sleep |
| -1  | leaves crit-sect        | Run   |                    | Sleep |
| -1  | call `sem_post`         | Run   |                    | Sleep |
| 0   | increment Val           | Run   |                    | Sleep |
| 0   | wake T1                 | Run   |                    | Ready |
| 0   | `sem_post` returns      | Run   |                    | Ready |
| 0   | interrupt; switch to T1 | Ready |                    | Run   |
| 0   |                         | Ready | `sem_wait` returns | Run   |
| 0   |                         | Ready | enteres crit-sect  | Run   |
| 0   |                         | Ready | call `sem_post`    | Run   |
| 1   |                         | Ready | `sem_post` returns | Run   |

## The Producer / Consumer Problem

- The problem we went through in chapter 30

```c
int buffer[MAX];
int fill     = 0;
int use      = 0;

void put(int value) {
  buffer[fill] = value;
  fill = (fill + 1) % MAX;
  count++;
}

int get() {
  int tmp = buffer[use];
  use = (use + 1) % MAX;
  return tmp;
}
```

### (Broken) Solution #1: Use Two Semaphores

- The main idea is
  - To use two semaphores to wake and sleep producers and consumers separately
- This will work if MAX == 1
- But if MAX is larger than 1, there could be a race condition
- For example,
  - A producer enters `put` and fill an entry in a buffer, and an interrupt occurs before fill is updated
  - Another producer enters the `put` and **overwrite** the entry

```c
sem_t empty;
sem_t full;

void* producer(void* arg) {
	int i;

	for (i = 0; i < loops; i++) {
		sem_wait(&empty);
		put(i);
		sem_post(&full);
	}
}

void* consumer(void* arg) {
	int tmp = 0;
	while (tmp != -1) {
		sem_wait(&full);
		tmp = get();
		sem_post(&empty);
	}
}

int main(int argc, char* argv[]) {
	sem_init(&empty, 0, MAX);
	sem_init(&full, 0, 0);
}
```

### Solution: Use Mutual Exclusion

- Put the mutual exclusion around the `put` and `get`
- Make sure not to put lock around the semaphores or it will cause deadlocks

```c
void* producer(void* arg) {
	int i;

	for (i = 0; i < loops; i++) {
		sem_wait(&empty);
		sem_wait(&mutex);
		put(i);
		sem_post(&mutex);
		sem_post(&full);
	}
}

void* consumer(void* arg) {
	int tmp = 0;
	while (tmp != -1) {
		sem_wait(&full);
		sem_wait(&mutex);
		tmp = get();
		sem_post(&mutex);
		sem_post(&empty);
	}
}

int main(int argc, char* argv[]) {
	sem_init(&empty, 0, MAX);
	sem_init(&full, 0, 0);
}
```

## Reader-Writer Lock

- The problem is to synchronize the read and write operations on a data structure
- One sample solution:
  - Create a write lock to ensure only one writer is updating the data structure at a time
  - Create a read lock to ensure only one reader is reading the data structure at a time
  - Acquire the write lock in the read operation to make sure no updates occur when readers are reading
- The disadvantages are
  - The readers might starve writers (a writer will have to wait until all readers have finished)
  - More complicated solutions might not be worth it (performance worse than simply using a single lock everywhere)

```c
typedef struct _rwlock_t {
  sem_t lock;      // binary semaphore (basic lock)
  sem_t writelock; // allow ONE writer/MANY readers
  int   readers;   // #readers in critical section
  rwlock_t;
void rwlock_init(rwlock_t *rw) {
  rw->readers = 0;
  sem_init(&rw->lock, 0, 1);
  sem_init(&rw->writelock, 0, 1);
}
void rwlock_acquire_readlock(rwlock_t *rw) {
  sem_wait(&rw->lock);
  rw->readers++;
  if (rw->readers == 1) // first reader gets writelock
    sem_wait(&rw->writelock);
  sem_post(&rw->lock);
}
void rwlock_release_readlock(rwlock_t *rw) {
  sem_wait(&rw->lock);
  rw->readers--;
  if (rw->readers == 0) // last reader lets it go
    sem_post(&rw->writelock);
  sem_post(&rw->lock);
}
void rwlock_acquire_writelock(rwlock_t *rw) {
  sem_wait(&rw->writelock);
}
void rwlock_release_writelock(rwlock_t *rw) {
  sem_post(&rw->writelock);
}
```

## Dining Philosophers

- With 5 philosophers sitting at a round table, between them there are 5 forks.
- They all either think or eat, but if they want to eat, they have to wait until the forks on both sides are available
- A simple tuition is to wait for the one on the left and the one on the right as the following

```c

void* philosopher(void* args) {
	while (1) {
		think();
		get_forks(p);
		eat();
		put_forks(p);
	}
}

void get_forks(int p) {
	sem_wait(&forks[left(p)]);
	sem_wait(&forks[right(p)]);
}

void put_forks(int p) {
	sem_post(&forks[left(p)]);
	sem_post(&forks[right(p)]);
}
```

- However, this will not work because of a possible deadlock where all of them are waiting for the one on their right
- The simple solution is to break the dependency in at least one of them, e.g. the last one acquires the one on their right first

## Thread Throttling

- This is another use case for semaphores: when there are **too many threads** trying to do something at once
- We can use a semaphore to limit the number of threads to a _threshold_
- For example
  - some programs are memory-instensive, and if too many memory-intensive programs are running, we might run out of physical memory
  - the machine will have to swap data between memory and disks (**thrashing**)
- With a simple semaphore, we initialize the maximum number of threads we limit to enter memory-intensive regions
- Therefore, the later one will naturally have to wait until the queue moves forward
