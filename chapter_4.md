# Chapter 4

## Process

- Process is an instance of a running program
- Program itself is lifeless, and it just sits in the disk
- To understand what constitutes a process, we have to understand its machine state

### Machine State

1. memory: instructions are in memory, and programs tend to read and write data from and to memory
2. registers: many programs explicitly read and update the registers
3. I/O information: information about the I/O storage devices, such as list of files a process currently opens

### Process API

1. Create: methods to create new process
2. Destroy: methods or interfaces to destroy processes as a user wishes
3. Wait: wait for a process to stop running
4. Miscellaneous Control: many other methods to control processes other than create, wait, destroy
   - For example, to suspend (stop it for a while) a process, or to resume it
5. State: methods or interfaces to get information about the state

### Process States

1. Running: the process is running on the processor
2. Ready: the process is ready to run, but the OS chose not to run
3. Blocked: the process has performed operations that make it not ready to run (i.e. waiting for the operation to complete)
   - For example, waiting an I/O operation to finish
   - **_The OS must also track blocked processes, so that when the waiting is over, the OS should make sure to wait the correct process and ready it to run again_**

#### Examples

1. CPU only

| Time | Process_0 | Process_1 | Notes              |
| ---- | --------- | --------- | ------------------ |
| 1    | Running   | Ready     |                    |
| 2    | Running   | Ready     |                    |
| 3    | Running   | Ready     |                    |
| 4    | Running   | Ready     | Process_0 now done |
| 5    | -         | Running   |                    |
| 6    | -         | Running   |                    |
| 7    | -         | Running   |                    |
| 8    | -         | Running   | Process_1 now done |

2. CPU and I/O

| Time | Process_0 | Process_1 | Notes                                   |
| ---- | --------- | --------- | --------------------------------------- |
| 1    | Running   | Ready     |                                         |
| 2    | Running   | Ready     |                                         |
| 3    | Running   | Ready     | Process_0 initiates I/O                 |
| 4    | Blocked   | Running   | Process_0 is blocked, so Process_1 runs |
| 5    | Blocked   | Running   |                                         |
| 6    | Blocked   | Running   |                                         |
| 7    | Ready     | Running   | Process_0 I/O done                      |
| 8    | Ready     | Running   | Process_1 now done                      |
| 9    | Running   | -         |                                         |
| 10   | Running   | -         | Process_0 now done                      |

### Data Structure

```c
// the registers xv6 will save and restore
// to stop and subsequently restart a process
struct context {
  int eip;
  int esp;
  int ebx;
  int ecx;
  int edx;
  int esi;
  int edi;
  int ebp;
};

// the different states a process can be in
enum proc_state { UNUSED, EMBRYO, SLEEPING,
                  RUNNABLE, RUNNING, ZOMBIE };

// the information xv6 tracks about each process
// including its register context and state
struct proc {
	char *mem;                  // Start of process memory
	uint sz;                    // Size of process memory
	char *kstack;               // Bottom of kernel stack
                              // for this process
	enum proc_state state;      // Process state
	int pid;                    // Process ID
	struct proc *parent;        // Parent process
	void *chan;                 // If !zero, sleeping on chan
	int killed;                 // If !zero, has been killed
	struct file *ofile[NOFILE]; // Open files
	struct inode *cwd;          // Current directory
	struct context context;     // Switch here to run process
	struct trapframe *tf;       // Trap frame for the
                              // current interrupt
 };
```

## How OS Starts A Process

1. Load the program and any static data to the memory in executable format from the I/O storage devices such as disks or SSDs
   - In early stages, the OS load code and static data **eagerly**, i.e. load everything at once
   - In modern stages, the OS load code and static data **lazily**, i.e. load anything only when needed
2. Allocate memory for program's **run-time stack**
3. Allocate memory for program's **heap**
4. Some other initialization tasks (such as I/O related setup)
   - For example, in UNIX systems, the OS will by default set up three **file descriptors**, standard intput, output, error
5. Jump to the entry point of the process, namely `main()`
6. Transfer the control of the CPU to the new process

## Keywords

1. Policy
   - an algorithm that makes decisions within the OS
2. Program Counter (PC) or Instruction Pointer (IP)
   - an pointer for the instruction to run
3. Stack Pointer and Frame Pointer
   - pointers for the stack of function parameters, local variables, and return addresses
4. Zombie State
