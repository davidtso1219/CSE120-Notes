# Chapter 32

## Non-Deadlock Bugs

1.  Atomicity Violation
    - It is usually caused by programmers who assume instructions will be executed atomically, but they aren't
    - The solution could be to simply add locks around to ensure atomicity of critical sections
2.  Order Violation
    - It is usually caused by programmers who assume instructions will be executed in order, but they aren't sometimes
    - The solution could be to simply use conditional variables with locks to preserve the order

## Deadlock Bugs

### Major Reasons

1.  large code space and complex dependencies
2.  encapsulation and the detail is hidden, including the lock

    - For example, `vector` in java

    ```java
    	Vector v1, v2;
    	v1.addAll(v2);
    	v2.addAll(v1);
    ```

### Conditions

1. **Mutual exclusion**: Threads claim exclusive control of resources that they require (e.g., a thread grabs a lock)
2. **Hold-and-wait**: Threads hold resources allocated to them(e.g.,locks that they have already acquired) while waiting for additional re- sources (e.g., locks that they wish to acquire)
3. **No preemption**: Resources (e.g., locks) cannot be forcibly removed from threads that are holding them.
4. **Circular wait**: There exists a circular chain of threads such that each thread holds one or more resources (e.g., locks) that are being re- quested by the next thread in the chain.

### Prevention

1. Circular waiting
   - total ordering, e.g. always wait acquire L1 then L2
   - partial ordering
2. Hold-and-wait
   - Acquire all locks at once
3. No Preemption
   - Avoid waiting on a lock when holding another already
4. Mutual Exclusion
   - Use hardware primitives to simply avoid critical sections

### Avoidance

- We can **avoid** the deadlock via scheduling
- For example, if we have two threads that use the same two locks, we can schedule one thread to run only after the other one has completed

### Detect & Recover

- Run a seperate program to detect the deadlock periodically
- For example, some database systems will run a deadlock detector and show the users
