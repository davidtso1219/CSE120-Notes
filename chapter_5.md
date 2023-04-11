# Chapter 5

## `fork()` System Call

- The system call used to create a new process
- The return value of the system call is the PID of the child process (the one being created)
- The child process does not start running at `main()` but `fork()`, which is why the following example only prints `hello world` once not twice
- The output of `fork()` is **not deterministic** because two process are in the system, and either of them could be running on the CPU at that point.

### Example

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/cpu-api/p1.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (original process)
        printf("hello, I am parent of %d (pid:%d)\n",
	       rc, (int) getpid());
    }
    return 0;
}
```

```console
$ > ./p1
hello world (pid:29146)
hello, I am parent of 29147 (pid:29146)
hello, I am child (pid:29147)
$ >
```

## `wait()` System Call

- We can use `wait()` to makes the output of `fork()` deterministic
- The **parent** calls `wait()` to delay its execution until the **child** finishes executing

### Example

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/cpu-api/p2.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
        sleep(1);
    } else {
        // parent goes down this path (original process)
        int wc = wait(NULL);
        printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",
	       rc, wc, (int) getpid());
    }
    return 0;
}
```

```
$ > ./p2
hello world (pid:29266)
hello, I am child (pid:29267)
hello, I am parent of 29267 (rc_wait:29267) (pid:29266)
$ >
```

## `exec()` System Call

- This is useful for creating a program different from the calling proram
- However, it does **NOT** create a new process.
- This is what it does:
  1.  It **loads** program and static data from the memory (given the names of the executable and the arguments)
  2.  It **overwrites** the current code segment
  3.  It **re-initializes** the stack, the heap, and other parts of the memory space
- It transforms the currently running program into a different running program

### Example

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/cpu-api/p3.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
        char *myargs[3];
        myargs[0] = strdup("wc");   // program: "wc" (word count)
        myargs[1] = strdup("p3.c"); // argument: file to count
        myargs[2] = NULL;           // marks end of array
        execvp(myargs[0], myargs);  // runs word count
        printf("this shouldn't print out");
    } else {
        // parent goes down this path (original process)
        int wc = wait(NULL);
        printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",
	       rc, wc, (int) getpid());
    }
    return 0;
}
```

```console
$ > ./p3
hello world (pid:29383)
hello, I am child (pid:29384)
      29     107    1030 p3.c
hello, I am parent of 29384 (rc_wait:29384) (pid:29383)
$ >
```

## Shell

- The separation of `fork()` and `exec()` is essential to build a UNIX shell because it lets the shell to run code _after_ `fork()` and _before_ `exec()`
- Shell is simply a **_user program_**
- It shows a prompt, waits for an input for the user, and does the following after user inputs a command
  1. finds the path to the executable
  2. calls `fork()` to create a child process
  3. calls `exec()` to execute the executable
  4. calls `wait()` to wait for the child process to complete
  5. goes back to show a prompt and wait for the user

### Redirection

- The shell accomplishes redirection by doing the following before
  1. calls `fork()` to create a child process
  2. closes the standard output
  3. opens a new file
  4. calls `exec()` to execute the executable
  5. ...

### Example

- Source: [link](https://github.com/remzi-arpacidusseau/ostep-code/blob/master/cpu-api/p4.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <assert.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
    int rc = fork();

    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    }

    else if (rc == 0) {
        // child: redirect standard output to a file
        close(STDOUT_FILENO);
        open("./p4.output", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);

        // now exec "wc"...
        char *myargs[3];
        myargs[0] = strdup("wc");   // program: "wc" (word count)
        myargs[1] = strdup("p4.c"); // argument: file to count
        myargs[2] = NULL;           // marks end of array
        execvp(myargs[0], myargs);  // runs word count
    }

    else {
        // parent goes down this path (original process)
        int wc = wait(NULL);
	    assert(wc >= 0);
    }
    return 0;
}
```

### `pipe()` System Call

- The system call used to connect the output of a process to the input of another one
- The OS will create an inkernel pipe (queue) and connect the output of a process and the input of another one to it

## Other Process Controls

- `kill()` system call: send signal to a process
- `signal()` system call: for a process to catch a signal
- `ctrl+c`: sends **SIGINT** to interrupt a process
- `ctrl+z`: sends **SIGSTP** to pause a process

## Users

- With the signal communication infrastructure, a question arises: _Who can send a signal to a process? and who can not?_
- Modern systems include the concept of **user**
- The user may launch one or many pro- cesses, and exercise full control over them (pause them, kill them, etc.).
- And the user generally only control their own processes
- The OS's job is to parcel the resource to each user
