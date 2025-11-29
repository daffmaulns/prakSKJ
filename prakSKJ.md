* Processes + `fork`/`wait` + zombies + signals
* pthread basics + race conditions
* Mutex patterns
* Semaphores (binary + counting, producer–consumer)
* Deadlocks (conditions + fixes)
* Memory (stack vs heap, leaks, dangling pointers)
* Quick Linux commands relevant to OS practicum

---

# OS Practicum Cheatsheet (Processes, Threads, Sync, Memory)

---

## 1. Processes, `fork()`, Zombies, Signals

### 1.1 Basic `fork() + wait()` pattern (no zombie)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {              // error
        perror("fork");
        exit(1);
    } else if (pid == 0) {      // child
        printf("child here (PID=%d)\n", getpid());
        exit(0);                // important: exit child
    } else {                    // parent
        wait(NULL);             // reap child, avoid zombie
        printf("parent done (PID=%d)\n", getpid());
    }
    return 0;
}
```

**Key notes:**

* `fork()` returns:

  * `< 0` → error
  * `0` → child process
  * `> 0` → parent (value = child PID)
* Zombie appears when child exits but parent never calls `wait()/waitpid()`.

### 1.2 Minimal zombie example + fix

```c
int main(void) {
    if (fork() == 0) {          // child
        printf("child exiting\n");
        return 0;
    }
    sleep(100);                 // parent alive, not calling wait()
    return 0;
}
```

* Child becomes **zombie** right after it exits.
* Remains zombie until parent exits or calls `wait()`.

**Fix (1 line):**

```c
if (fork() == 0) {
    printf("child exiting\n");
    return 0;
}
wait(NULL);                     // reap immediately
```

### 1.3 Process States & Job Control

**Common `ps` STAT letters:**

* `R` – running / runnable
* `S` – sleeping (interruptible)
* `T` – stopped (by job control or signal, e.g., SIGSTOP / SIGTSTP)
* `Z` – zombie (terminated, not yet waited on)

**Shell job control:**

* `cmd` → foreground job
* `cmd &` → background job
* `Ctrl+Z` → sends **SIGTSTP**, process stopped (`T`), can be resumed
* `bg` → resume stopped job in background
* `fg` → bring job to foreground

### 1.4 Important signals (mental map)

* `SIGINT` – interrupt from keyboard (Ctrl+C)
* `SIGTSTP` – interactive stop (Ctrl+Z), can be caught/ignored
* `SIGSTOP` – unconditional stop, cannot be caught/ignored
* `SIGCONT` – continue a stopped process
* `SIGTERM` – polite terminate, process can clean up
* `SIGKILL` – immediate kill, no cleanup, cannot be caught/ignored

---

## 2. pthread Basics & Race Conditions

### 2.1 Creating and joining threads

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void *worker(void *arg) {
    printf("hello from thread\n");
    return NULL;
}

int main(void) {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, worker, NULL);
    pthread_create(&t2, NULL, worker, NULL);

    // wait for both threads to finish
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    return 0;
}
```

**Key points:**

* Thread function signature: `void *func(void *arg)`
* Always `pthread_join()` threads if you need their work before proceeding.

### 2.2 Race condition example (`counter++`)

```c
int counter = 0;

void *worker(void *arg) {
    for (int i = 0; i < 100000; i++) {
        counter++;   // NOT atomic: load, add, store
    }
    return NULL;
}
```

* `counter++` = 3 steps: load → add → store.
* Two threads interleaving → lost updates → final value < expected.

**Race condition = multiple threads access shared data, at least one write, without proper synchronization.**

---

## 3. Mutex: Fixing Races

### 3.1 Basic mutex pattern

```c
#include <pthread.h>

int counter = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *worker(void *arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&lock);   // enter critical section
        counter++;
        pthread_mutex_unlock(&lock); // leave critical section
    }
    return NULL;
}
```

**Rules:**

* Protect all accesses (read/write) to shared data with the **same** mutex.
* Always ensure every `lock` has a matching `unlock` (even on error paths).

### 3.2 Joining + printing safely

```c
pthread_t t1, t2;

pthread_create(&t1, NULL, worker, NULL);
pthread_create(&t2, NULL, worker, NULL);

pthread_join(t1, NULL);
pthread_join(t2, NULL);

printf("counter = %d\n", counter);   // safe: both finished
```

---

## 4. Passing Arguments to Threads (Correct Pattern)

### 4.1 What NOT to do

```c
for (int i = 0; i < 5; i++) {
    pthread_create(&t[i], NULL, run, &i);  // BAD: &i is shared
}
```

* All threads get the same address `&i`.
* By the time they run, `i` has changed → wrong / inconsistent values.

### 4.2 Correct per-thread allocation pattern

```c
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

void *run(void *arg) {
    int id = *(int *)arg;
    free(arg);                  // free after copying
    printf("thread %d\n", id);
    return NULL;
}

int main(void) {
    pthread_t t[5];

    for (int i = 0; i < 5; i++) {
        int *id = malloc(sizeof(int));
        *id = i;
        pthread_create(&t[i], NULL, run, id);
    }

    for (int i = 0; i < 5; i++) {
        pthread_join(t[i], NULL);
    }
    return 0;
}
```

**Idea:** each thread gets its **own heap-allocated copy** of the argument.

---

## 5. Semaphores: Binary, Counting, Resource Limits

### 5.1 Concept snapshot

* **Binary semaphore** (values 0/1) → often used like a mutex (mutual exclusion).
* **Counting semaphore** (0..N) → counts available resources (e.g., N seats, N printers).

**Difference from mutex:**

* Mutex is usually “owned” by the thread that locked it (only owner should unlock).
* Semaphore can be `sem_post()`-ed by any thread (often used for signaling between threads).

### 5.2 Example: 3 printers (resource pool)

Only 3 threads “printing” at once:

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

#define NUM_THREADS 10

sem_t printers;  // counting semaphore for printers

void *print_job(void *arg) {
    sem_wait(&printers);        // acquire one printer

    printf("Thread %ld printing...\n", (long)pthread_self());
    // simulate printing
    sleep(1);

    sem_post(&printers);        // release printer
    return NULL;
}

int main(void) {
    pthread_t t[NUM_THREADS];

    sem_init(&printers, 0, 3);  // 3 printers available

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&t[i], NULL, print_job, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(t[i], NULL);
    }

    sem_destroy(&printers);
    return 0;
}
```

### 5.3 Producer–Consumer (bounded buffer) pattern (concept)

We use:

* `sem_t empty` – how many empty slots left (init = buffer size)
* `sem_t full` – how many filled slots (init = 0)
* `pthread_mutex_t mutex` – protects the actual buffer

**Producer:**

1. `sem_wait(&empty);` → wait for empty slot
2. `pthread_mutex_lock(&mutex);`
3. put item in buffer
4. `pthread_mutex_unlock(&mutex);`
5. `sem_post(&full);` → signal one more filled slot

**Consumer:**

1. `sem_wait(&full);` → wait for item
2. `pthread_mutex_lock(&mutex);`
3. remove item from buffer
4. `pthread_mutex_unlock(&mutex);`
5. `sem_post(&empty);` → signal one more empty slot

You don’t necessarily need full code, just this flow.

---

## 6. Deadlock: Conditions & Fixes

### 6.1 Coffman conditions (deadlock possible when ALL hold)

1. **Mutual exclusion** – resource held by at most one process at a time
2. **Hold and wait** – holding one resource while waiting for others
3. **No preemption** – resources can’t be forcibly taken
4. **Circular wait** – cycle of threads each waiting for the next

Break **any one** of these → no deadlock.

### 6.2 Classic deadlock example with two mutexes

```c
pthread_mutex_t A = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t B = PTHREAD_MUTEX_INITIALIZER;

void *T1(void *arg) {
    pthread_mutex_lock(&A);
    sleep(1);
    pthread_mutex_lock(&B);
    // ...
    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}

void *T2(void *arg) {
    pthread_mutex_lock(&B);
    sleep(1);
    pthread_mutex_lock(&A);
    // ...
    pthread_mutex_unlock(&A);
    pthread_mutex_unlock(&B);
    return NULL;
}
```

* T1 holds A, waits for B
* T2 holds B, waits for A
* **Circular wait** → deadlock

### 6.3 Fix by global lock ordering

Enforce same order (e.g., always lock A → B):

```c
void *T2(void *arg) {
    pthread_mutex_lock(&A);  // changed to match T1
    sleep(1);
    pthread_mutex_lock(&B);
    // ...
    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}
```

Now both T1 and T2 lock A first, then B → no cycle.

### 6.4 Other deadlock-handling strategies (short)

* Use `pthread_mutex_trylock()` + timeout/backoff instead of waiting forever.
* Request all needed resources at once (avoid hold-and-wait).
* Deadlock detection & recovery: periodically check for cycles, then kill/rollback a thread.

---

## 7. Memory: Stack vs Heap, Leaks, Dangling Pointers

### 7.1 Stack vs heap basics

```c
int  a = 5;                        // stack (local variable)
int  b[1000];                      // stack (local array)
int *p = malloc(sizeof(int)*1000); // heap (data), p itself on stack
```

**Stack:**

* Local variables, return addresses
* Automatically managed (function call/return)
* Limited size → large local arrays can cause **stack overflow**
* Typically grows **downward** (toward lower addresses)

**Heap:**

* Dynamic allocation: `malloc`, `calloc`, `realloc`, `free`
* Good for large or long-lived data
* Grows **upward** (toward higher addresses) on many systems

### 7.2 Dangling pointer / use-after-free

```c
int *p = malloc(sizeof(int));
*p = 10;
free(p);      // memory returned to heap allocator
*p = 20;      // use-after-free: UNDEFINED behavior
```

* `p` still points to the old address, but that memory may be reused.
* May “work” sometimes, but can randomly crash or corrupt other data.

**Safer pattern:**

```c
free(p);
p = NULL;     // avoid accidental reuse
```

### 7.3 Memory leaks (simple patterns)

a) **Leak:**

```c
int *p = malloc(100);
return 0;          // never freed -> leak
```

b) **No leak:**

```c
int *p = malloc(100);
free(p);
return 0;
```

c) **Leak:**

```c
int *p = malloc(100);
p = malloc(200);   // original 100 bytes lost (no pointer)
free(p);          // frees only the 200 bytes
```

---

## 8. Quick Linux Commands (OS Context)

Useful for interpreting behaviour during practicum:

```sh
ps aux            # list processes
ps -el            # detailed view, STAT column (R, S, T, Z)
top               # live CPU/memory by process
jobs              # list shell jobs
fg %1             # bring job 1 to foreground
bg %1             # resume job 1 in background

kill -STOP PID    # send SIGSTOP (hard stop)
kill -CONT PID    # send SIGCONT (resume)
kill -TERM PID    # polite terminate
kill -KILL PID    # force kill

strace ./a.out    # trace system calls of a.out

cat /proc/$$/maps # show memory layout of current shell (Linux)
```

LOCALLY ROOTED
GLOBALLY RESPECTED
