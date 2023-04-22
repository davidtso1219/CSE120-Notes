# Chapter 29

## Concurrent Counters

### Counter with No Lock

```c
typedef struct __counter_t {
  int value;
} counter_t;

void init(counter_t *c) {
  c->value = 0;
}

void increment(counter_t *c) {
  c->value++;
}

void decrement(counter_t *c) {
  c->value--;
}

int get(counter_t *c) {
  return c->value;
}
```

### Unscalable Counter with A Lock

- Locks are acquired and released automatically when we wanna modify its fields

```c
typedef struct __counter_t {
  int value;
  pthread_mutex_t lock;
} counter_t;

void init(counter_t *c) {
  c->value = 0;
  Pthread_mutex_init(&c->lock, NULL);
}

void increment(counter_t *c) {
  Pthread_mutex_lock(&c->lock);
  c->value++;
  Pthread_mutex_unlock(&c->lock);
}

void decrement(counter_t *c) {
  Pthread_mutex_lock(&c->lock);
  c->value--;
  Pthread_mutex_unlock(&c->lock);
}

int get(counter_t *c) {
  Pthread_mutex_lock(&c->lock);
  int rc = c->value;
  Pthread_mutex_unlock(&c->lock);
  return rc;
}
```

### Approximate Counter

- Logical Parts
  - _A local physical counter on each processor_
  - _A global counter_
  - Locks: _one for each local counter_ and _one for the global counter_
- Idea
  - Thread on a core: increment the local counter as it wishes
  - Local counter: adds the current value to the global counter and reset itself
- The frequency of local-to-global transfer is determined by a threshold
  - If the threshold is too small, the counter will behave more like a non-scalable one
  - If the threshold is too large, the counter will be more scalable
  - But the difference between the true value and the real value of the global counter will be bigger
- Implementation

```c
typedef struct __counter_t {
  int               global;         // global count
  pthread_mutex_t   glock;          // global lock
  int               local[NUMCPUS]; // per-CPU count
  pthread_mutex_t   llock[NUMCPUS]; // ... and locks
  int               threshold;      // update frequency
} counter_t;

// init: record threshold, init locks, init values
// of all local counts and global count
void init(counter_t *c, int threshold) {
  c->threshold = threshold;
  c->global = 0;
  pthread_mutex_init(&c->glock, NULL);
  int i;
  for (i=0;i<NUMCPUS;i++){
    c->local[i] = 0;
    pthread_mutex_init(&c->llock[i], NULL);
  }
}

// update: usually, just grab local lock and update
// local amount; once local count has risen ’threshold’,
// grab global lock and transfer local values to it
void update(counter_t *c, int threadID, int amt) {

  int cpu = threadID % NUMCPUS;
  pthread_mutex_lock(&c->llock[cpu]);
  c->local[cpu] += amt;

  if (c->local[cpu] >= c->threshold) {
    // transfer to global (assumes amt>0)
    pthread_mutex_lock(&c->glock);
    c->global += c->local[cpu];
    pthread_mutex_unlock(&c->glock);
    c->local[cpu] = 0;
  }
  pthread_mutex_unlock(&c->llock[cpu]);
}

// get: just return global amount (approximate)
int get(counter_t *c) {
  pthread_mutex_lock(&c->glock);
  int val = c->global;
  pthread_mutex_unlock(&c->glock);
  return val; // only approximate!
}

```

### Approximate Counters Example Timeline

| Time | $L_1$             | $L_2$ | $L_3$ | $L_4$             | $G$ |
| ---- | ----------------- | ----- | ----- | ----------------- | --- |
| 0    | 0                 | 0     | 0     | 0                 | 0   |
| 1    | 0                 | 0     | 1     | 1                 | 0   |
| 2    | 1                 | 0     | 2     | 1                 | 0   |
| 3    | 2                 | 0     | 3     | 1                 | 0   |
| 4    | 3                 | 0     | 3     | 2                 | 0   |
| 5    | 4                 | 1     | 3     | 3                 | 0   |
| 6    | 5 $\rightarrow$ 0 | 1     | 3     | 4                 | 5   |
| 7    | 0                 | 2     | 4     | 5 $\rightarrow$ 0 | 5   |

## Concurrent Linked Lists

- We can utilize the basic approach again
  1.  acquire the lock at the beginning of any operation
  2.  release the lock at the end of any operation
- However, not every single line of code in an operation is in the actual critical section
- Implementation

```c
// basic node structure
typedef struct __node_t {
  int key;
  struct __node_t *next;
} node_t;

// basic list structure (one used per list)
typedef struct __list_t {
  node_t *head;
  pthread_mutex_t lock;
} list_t;

void List_Init(list_t *L) {
  L->head = NULL;
  pthread_mutex_init(&L->lock, NULL);
}

int List_Insert(list_t *L, int key) {
  pthread_mutex_lock(&L->lock);
  node_t *new = malloc(sizeof(node_t));

  if (new == NULL) {
      perror("malloc");
      pthread_mutex_unlock(&L->lock);
      return -1; // fail
  }

  new->key  = key;
  new->next = L->head;
  L->head   = new;
  pthread_mutex_unlock(&L->lock);
  return 0; // success
}

int List_Lookup(list_t *L, int key) {
  pthread_mutex_lock(&L->lock);
  node_t *curr = L->head;

  while (curr) {
    if (curr->key == key) {
      pthread_mutex_unlock(&L->lock);
      return 0; // success
    }
    curr = curr->next;
  }

  pthread_mutex_unlock(&L->lock);
  return -1; // failure
}

```

### Scaling

- One technique is called hand-over-hand locking
- Instead of having a single lock for the entire list, we add a lock for each node or each group of numerous nodes
- When traversing the list, the code first grabs the next node or group's lock, and then releases the current node or group's lock

## Concurrent Queues

- We use two locks for both head and tail to create atomic critical section for `enqueue` and `dequeue` operations
- Implementation

```c
typedef struct __node_t {
  int value;
  struct __node_t *next;
} node_t;

typedef struct __queue_t {
  node_t *head;
  node_t *tail;
  pthread_mutex_t head_lock, tail_lock;
} queue_t;

void Queue_Init(queue_t *q) {
  node_t *tmp = malloc(sizeof(node_t));
  tmp->next = NULL;
  q->head = q->tail = tmp;
  pthread_mutex_init(&q->head_lock, NULL);
  pthread_mutex_init(&q->tail_lock, NULL);
}

void Queue_Enqueue(queue_t *q, int value) {
  node_t *tmp = malloc(sizeof(node_t));
  assert(tmp != NULL);
  tmp->value = value;
  tmp->next  = NULL;
  pthread_mutex_lock(&q->tail_lock);
  q->tail->next = tmp;
  q->tail = tmp;
  pthread_mutex_unlock(&q->tail_lock);
}

int Queue_Dequeue(queue_t *q, int *value) {
  pthread_mutex_lock(&q->head_lock);
  node_t *tmp = q->head;
  node_t *new_head = tmp->next;
  if (new_head == NULL) {
      pthread_mutex_unlock(&q->head_lock);
      return -1; // queue was empty
  }
  *value = new_head->value;
  q->head = new_head;
  pthread_mutex_unlock(&q->head_lock);
  free(tmp);
  return 0;
}
```

## Concurrent Hash Table

- We will be discussing the hash table using bucket lists
- And instead of using a single block for the entire table, we simply use the concurrent lists for each of the busket
- Doing so enables concurrent operations to take places for each shared bucket list
- Implementation

```c
#define BUCKETS (101) 2
typedef struct __hash_t {
  list_t lists[BUCKETS];
} hash_t;

void Hash_Init(hash_t *H) {
  int i;
  for (i = 0; i < BUCKETS; i++)
      List_Init(&H->lists[i]);
}

int Hash_Insert(hash_t *H, int key) {
  return List_Insert(&H->lists[key % BUCKETS], key);
}

int Hash_Lookup(hash_t *H, int key) {
  return List_Lookup(&H->lists[key % BUCKETS], key);
}

```

## Keywords

1. thread safe: a thread-safe data structure is usable by threads like adding locks
2. perfect scaling: multiple threads run on a single processor as fast as single threads run on multiple processors
