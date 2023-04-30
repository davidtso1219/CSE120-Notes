# Chapter 7

## Workload Assumptions

- The assumptions we make about processes, or jobs

1. Each job runs for the same amount of time.
2. All jobs arrive at the same time.
3. Once started, each job runs to completion.
4. All jobs only use the CPU (i.e., they perform no I/O) 5. The run-time of each job is known.

## Scheduling Metrics

- The metric that we can use to compare different scheduling policies
- For example, we can use **turnaround time** to measure the performance of the policy
- Another example is the **Jain's Fairness Index**, which is a fairness metric

## Example Policies

### Example Policy #1: FIFO

- Advantages: simple, easy to implement, and good given our assmuptions about our workload
- Disadvantages:
  - If we have a long job that comes first to the queue,
  - every job behind it will have to wait for it to complete
  - This results in a large average turnaround time of all jobs

### Example Policy #2: Shortest Job First (SJF)

- It runs the shortest job first when multiple jobs join the queue at the same time
- Advantages: solves the downside described in FIFO
- Disadvantages:
  - If we have two short jobs that come shortly after the long job starts
  - the short jobs will still have to wait for it to complete
  - This also results in a large average turnaround time of all jobs

### Example Policy #3: Shortest Time-to-Completion First (STCF)

- To address the disadvantage of previous example, we have to relax our third assumption
- In this policy, we preempt the first long job and switch to other small ones.
- So the general idea would be **any time a new job enters the system, the STCF scheduler will find the job with the least time to complete its execution**
- Advantages: solves the downside described in SJF
- Disadvantages: our response time would be terrible because a job will have to wait for another job to entirely complete its execution just to start

### Example Policy #4: Round Robin (RR)

- The idea is to run a job for a **time slice** instead of running until completion
- For example,
  - If three jobs arrive at the same time, they all need 10 seconds to finish.
  - We run 2 second on the first one, then the second one, and then the third one.
  - We loop through this cycle for five times
  - The response time of each job would be approximately 2 second
- Advantage: decrease the response time significantly
- Disadvantage:
  - increase the turnaround due to the context switches
  - unable to handle I/O requests sufficiently

### Example Policy #5: Break Down A Job

- The idea is to break down a job into different sub-jobs like I/O sub-job, CPU sub-job, etc.
- And we can run other job on CPU when a job enters a I/O sub-job
- Advantage: run different type of jobs more efficiently
- Disadvantage: we still haven't relaxed our last assumption: the scheduler know the length of each job

## Keywords

- schedule policies (disciplines): high-level policies that an OS scheduler employs
- workload: the running processes in the system
- job: process
- turnaround time: the duration from the moment a job arrives to the moment a job completes
- response time: the duration from the moment a job arrives to the moment a job starts to run at the first time
