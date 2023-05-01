# Chapter 8

## Multi-level Feedback Queue (MLFQ)

- The basic idea is to have a priority queue of queues, where jobs will be placed to different queues based on their priority levels
- Here are the two basic rules for MLFQ
  - (1) if priority(A) > priority(B), A runs
  - (2) if priority(A) == priority(B), A & B runs in RR
- The priority of a job will vary based on its observed behaviors, for example,
  1.  if a job repeatedly yields the CPU to wait an input from the keyboard, then MLFQ will keep its priority high
  2.  if a job uses the CPU intensely for a long period of time, then MLFQ will keep its priority low

### Attempt #1: How to Change Priority

- We have to decide how we are going to change the priority level of the job over some time
- The basic idea is to decide a new priority based on a job's behavior
- These are the rules for our first attempt at a priority adjustment algorithm
  - (3) When a job enters the system, it is placed at the highest priority level
  - (4) If a job uses the entire time slice while running, its priority level decreases. Otherwise, it increases.
- Problems
  - if there are too many _interactive_ jobs in the system, none of them will be penalzied and they will consume all CPU time
  - a user could write their program in the way that the job gives up the CPU right before the time slice ends, so they would never be penalized and could consume most of the CPU time
  - a program may change its behavior from CPU-intense to non CPU-intense, but there is no way to increase a job's priority level

### Attempt #2: The Priority Boost

- We have to increase a job's priority level at some time
- The basic idea is to boost a job's priority level once in a while, so every job can have a new chance
- These are the rule for our second attempt at a priority adjustment algorithm
  - (5) After some time period S, move all the jobs in the system to the topmost queue
- Problems
  - if S is set too high, long-running jobs could starve
  - if S is set too low, interactive jobs may not get a proper share of the CPU

### Attempt #3: Better Accounting

- We have to account gaming of our scheduler
- The basic idea is to check the overall usage of the CPU time of a job during a time slice regardless how many times it gives up the CPU
- These are the rule for our second attempt at a priority adjustment algorithm
  - (4) Once a job uses up its time slice, its priority level decreaases (regardless if it gives up CPU in the middle)

## Tuning MLFQ

- A big question with MLFQ is how to parameterize a scheduler
- For example
  - how many queues
  - what priority level for a queue
  - how often a job should be boosted
- It is not an easy task to find the most optimal values for each of the question
- So there are many research on different approaches
- For example,
  - Most MLFQ variants allow various length for time slices across different queues
  - Some MLFQ will use a configuration table to configure its behavior while some will use some mathematical formula to adjust the priorities
