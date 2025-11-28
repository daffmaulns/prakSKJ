Here’s a **single `.md` cheatsheet** you can copy–paste and then resize/layout into A3 however you want.

I’ve included:

* **Wireshark (1 segment worth)**
* **All the OS stuff you actually need**: `fork`, zombies, signals, threads, mutex, semaphores, deadlock, stack/heap, leaks.

You can delete sections you don’t need, but this is the “max power” version.

---

````md
# OS & Network Practicum Cheatsheet

---

## 0. Wireshark – What to Click, What to Filter

### 0.1 General Workflow

1. **Set display filter:**
   - `http` – all HTTP
   - `http.request` / `http.response`
   - `dns` – all DNS
   - `udp` / `tcp` – by transport protocol
   - `ip.addr == X.X.X.X` – all traffic involving host X

2. **Use the 3 panes:**
   - **Top** – packet list (time, src, dst, protocol, info)
   - **Middle** – details (Ethernet, IP, TCP/UDP, HTTP/DNS)
   - **Bottom** – raw hex (for UDP header hex questions)

3. **Right-click helpers:**
   - `Follow TCP Stream` – see full HTTP/TCP conversation
   - `Follow UDP Stream` – see related DNS/UDP packets

---

### 0.2 DNS

**What DNS does:**
- Resolves **names → IPs** (A/AAAA records)
- Also handles **NS** (authoritative name server), **CNAME**, **PTR** (reverse lookup)
- Usually **UDP 53**, sometimes TCP 53

**Where to look (filter: `dns`):**

In DNS packet (middle pane → DNS):

- **Transaction ID** – matches query with response
- **Flags** – query vs response, recursion desired/available
- **Questions section:**
  - Name (e.g., `www.stanford.edu`)
  - Type (A, NS, PTR, etc.)
- **Answer/Authority/Additional:**
  - A: IPv4 address
  - AAAA: IPv6
  - CNAME: canonical name
  - NS: nameserver
  - PTR: reverse DNS

**Typical `nslookup` patterns:**
```sh
nslookup -type=A domain.com      # expect A in Answers
nslookup -type=NS domain.com     # expect NS in Authority/Answers
nslookup -type=PTR X.X.X.X       # expect PTR to some name
````

---

### 0.3 HTTP (over TCP)

**Role:**

* Application protocol over TCP (port 80/8080 in labs).

**HTTP Request (filter: `http.request`):**

* Request line: `GET /path/file.html HTTP/1.1` or `POST /...`
* Headers:

  * `Host: example.com`
  * `User-Agent: ...`
  * `If-Modified-Since: ...` (caching)
* Source IP = client, Dest IP = server

**HTTP Response (filter: `http.response`):**

* Status line: `HTTP/1.1 200 OK`, `304 Not Modified`, `404 Not Found`, etc.
* Headers:

  * `Content-Length:`
  * `Last-Modified:` (paired with `If-Modified-Since`)
  * `Location:` (for redirects: 3xx)

**Caching pattern (Assg. 2 logic):**

* First GET – no `If-Modified-Since`, server returns full file (`200 OK`)
* Later GET – has `If-Modified-Since`, server may respond `304 Not Modified` (no body)

---

### 0.4 TCP & RTT

**TCP basics:**

* Reliable, ordered, connection-oriented
* Uses **sequence** and **acknowledgment** numbers

**3-way handshake (filter: `tcp.flags.syn == 1`):**

1. SYN (client → server):

   * `SYN` flag set
   * `ACK` flag = 0
2. SYN/ACK (server → client):

   * `SYN` + `ACK` flags set
3. ACK (client → server):

   * `ACK` flag set, no SYN

**Simple RTT idea:**

* RTT ≈ time between:

  * client sending data segment
  * server sending ACK that acknowledges that segment
* Can also use `tcp.analysis.ack_rtt` field shown in Wireshark.

---

### 0.5 UDP & Header

**UDP header (always 8 bytes):**

1. Source Port (2 bytes)
2. Destination Port (2 bytes)
3. Length (2 bytes) = header (8) + payload
4. Checksum (2 bytes)

**Length sanity check (IPv4):**

* From IP header:

  * `IP Total Length` – includes IP header + UDP header + payload
  * `IP Header Length` – e.g. 20 bytes
* So:

  * `UDP Length = IP Total Length – IP Header Length`
  * `UDP Payload = UDP Length – 8`

---

## 1. Processes, fork(), Zombies, Signals

### 1.1 Basic fork() + wait() pattern (NO zombie)

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
        printf("child here\n");
        exit(0);
    } else {                    // parent
        wait(NULL);             // reap child, avoid zombie
        printf("parent done\n");
    }
    return 0;
}
```

**Key points:**

* child path = `pid == 0`
* parent path = `pid > 0`
* `wait(NULL);` in parent → prevents zombie.

---

### 1.2 Zombie quick notes

* Child exits, parent **does not** call `wait()` / `waitpid()`.
* Kernel keeps child entry in process table → state `Z` (zombie).
* Zombie disappears when:

  * parent calls `wait()` OR
  * parent exits (then `init` or equivalent reaps child).
* Fix: parent must call `wait(NULL);` / `waitpid(pid, ...)`.

---

### 1.3 Foreground / Background / Job control

* `sleep 100` → foreground job.
* `sleep 100 &` → background job.
* `Ctrl+Z`:

  * sends **SIGTSTP** (terminal stop)
  * process paused, `ps` STAT = `T`
* `bg` → resume in background (sends SIGCONT).
* `fg` → bring to foreground.
* `Ctrl+C`:

  * sends **SIGINT** (interrupt)
  * normally terminates foreground program.

---

### 1.4 Common signals and meaning

* **SIGINT** – interrupt from keyboard (`Ctrl+C`), can be caught.
* **SIGTSTP** – terminal stop (`Ctrl+Z`), can be caught/ignored.
* **SIGSTOP** – unconditional stop (cannot be caught/ignored).
* **SIGCONT** – resume a stopped process.
* **SIGTERM** – polite request to terminate, can run cleanup handlers.
* **SIGKILL** – forced kill, **no** cleanup, cannot be caught/ignored.

---

## 2. pthread Basics & Race Conditions

### 2.1 Creating threads and joining

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void *worker(void *arg) {
    // do stuff
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

**Why `pthread_join` is needed:**

* Without it, main may exit before threads finish.
* With it, main waits until each thread is done.

---

### 2.2 Race condition example (increment)

```c
int counter = 0;

void *worker(void *arg) {
    for (int i = 0; i < 100000; i++) {
        counter++;   // NOT atomic: (load, add, store)
    }
    return NULL;
}
```

**Why race condition:**

* `counter++` = 3 operations:

  1. load current value
  2. add 1
  3. store result
* Two threads can interleave these steps and lose increments.

---

## 3. Mutex for Mutual Exclusion

### 3.1 Fixing the race with pthread_mutex_t

```c
#include <pthread.h>
#include <stdio.h>

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

int main(void) {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, worker, NULL);
    pthread_create(&t2, NULL, worker, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("counter = %d\n", counter);
    return 0;
}
```

**Key ideas:**

* Critical section: code that accesses shared state (`counter`).
* Mutex ensures only **one** thread is in that region.
* Always unlock in all code paths (avoid deadlock).

---

## 4. Passing Arguments to Threads Safely

### 4.1 Buggy pattern (do NOT use)

```c
for (int i = 0; i < 5; i++) {
    pthread_create(&t[i], NULL, run, &i); // &i reused
}
```

* All threads receive the same address `&i`.
* By the time thread runs, `i` may have changed.
* Leads to wrong/identical IDs across threads.

---

### 4.2 Correct pattern: allocate per-thread argument

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void *run(void *arg) {
    int id = *(int *)arg;
    free(arg);                  // free allocated memory
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

**Pattern:**
“Allocate arg on heap → pass pointer → copy inside thread → free.”

---

## 5. Semaphores & Limited Resources

### 5.1 Binary semaphore vs mutex (concept)

* **Similarity:**

  * Binary semaphore (0/1) can act like a mutex → ensures mutual exclusion.
* **Difference (common explanation):**

  * Mutex:

    * Usually owned by the thread that locked it.
    * Only owner should unlock.
  * Semaphore:

    * Any thread can `sem_post`.
    * Often used for **signaling** and resource counting, not just mutual exclusion.

---

### 5.2 Example: 3 printers (resource pool)

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_THREADS 10

sem_t printers;  // counts available printers

void *print_job(void *arg) {
    sem_wait(&printers);        // acquire one printer

    // ---- critical section: printing ----
    printf("Thread %ld printing...\n", (long)arg);
    // ------------------------------------

    sem_post(&printers);        // release printer
    return NULL;
}

int main(void) {
    pthread_t t[NUM_THREADS];

    sem_init(&printers, 0, 3);  // 3 printers available

    for (long i = 0; i < NUM_THREADS; i++) {
        pthread_create(&t[i], NULL, print_job, (void *)i);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(t[i], NULL);
    }
    sem_destroy(&printers);
    return 0;
}
```

**Concept:**
`sem_init(&printers, 0, 3);` → at most 3 threads inside “printing” at once.

---

### 5.3 Producer–Consumer (bounded buffer) sketch

**Semaphores:**

* `sem_t empty;` – counts empty slots (init = buffer size)
* `sem_t full;` – counts full slots (init = 0)

**Mutex:**

* `pthread_mutex_t mutex;` – protects the shared buffer itself

**Producer steps:**

1. `sem_wait(&empty);`      // wait for an empty slot
2. `pthread_mutex_lock(&mutex);`
3. Put item into buffer
4. `pthread_mutex_unlock(&mutex);`
5. `sem_post(&full);`       // one more full slot

**Consumer steps:**

1. `sem_wait(&full);`       // wait for an available item
2. `pthread_mutex_lock(&mutex);`
3. Remove item from buffer
4. `pthread_mutex_unlock(&mutex);`
5. `sem_post(&empty);`      // one more empty slot

---

## 6. Deadlock: Conditions & Fix

### 6.1 Coffman conditions (all must hold for deadlock)

1. **Mutual exclusion** – resource held by at most one process/thread.
2. **Hold and wait** – holding at least one resource and waiting for others.
3. **No preemption** – cannot forcibly take resources away.
4. **Circular wait** – a cycle exists where each process waits for another.

Break ANY one of these → no deadlock.

---

### 6.2 Typical deadlock example

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

**What happens:**

* T1 locks A, then waits for B.
* T2 locks B, then waits for A.
* Circular wait → deadlock if both reach that point.

---

### 6.3 Fix by global lock ordering (break circular wait)

```c
void *T2(void *arg) {
    pthread_mutex_lock(&A);  // enforce same order: lock A first
    sleep(1);
    pthread_mutex_lock(&B);
    // ...
    pthread_mutex_unlock(&B);
    pthread_mutex_unlock(&A);
    return NULL;
}
```

**Rule:**
Define a global order (e.g. always lock A before B).
All threads follow this order → no circular wait.

---

### 6.4 Other deadlock-avoidance ideas

* Request all required resources at once (avoid hold-and-wait).
* Use `pthread_mutex_trylock()` with timeouts → back off if cannot get lock.
* Deadlock detection + recovery:

  * system checks for cycles, then kills/rolls back one thread.

---

## 7. Memory: Stack vs Heap, Dangling Pointers, Leaks

### 7.1 Stack vs heap

```c
int  a = 5;                        // stack
int  b[1000];                      // stack (local array)
int *p = malloc(sizeof(int)*1000); // heap (p on stack, data on heap)
```

**Stack:**

* Automatic storage: local variables, return addresses, call frames
* Limited size; large arrays risk **stack overflow**
* Typically grows **downward** (towards lower addresses)

**Heap:**

* Dynamic storage: `malloc`, `calloc`, `realloc`, `free`
* Good for large or long-lived objects
* Typically grows **upward**

---

### 7.2 Dangling pointer / use-after-free

```c
int *p = malloc(sizeof(int));
*p = 10;
free(p);      // p now dangling
*p = 20;      // use-after-free: UNDEFINED behavior
```

**What’s wrong:**

* `p` still points to memory that has been freed.
* Memory can be reused by allocator; writing to it can corrupt other data or crash.

**Safer pattern:**

```c
free(p);
p = NULL;     // reduce risk; deref NULL → obvious error, not silent corruption
```

---

### 7.3 Leak or no leak?

**(a)**

```c
int *p = malloc(100);
return 0;
```

* **Leak**: allocated memory is never freed.

**(b)**

```c
int *p = malloc(100);
free(p);
return 0;
```

* **No leak**: allocation matched with `free`.

**(c)**

```c
int *p = malloc(100);
p = malloc(200);
free(p);
```

* **Leak**:

  * First `malloc(100)` is lost when `p` is overwritten.
  * Only the 200-byte block is freed.

---

## 8. Quick Linux Commands (OS Context)

* `ps aux`, `ps -el` – see processes, their PIDs, states.
* `top` / `htop` – live CPU/memory usage + processes.
* `jobs` – list background/paused jobs.
* `fg %n` / `bg %n` – job control for job `n`.
* `kill -STOP pid`, `kill -CONT pid` – stop / continue processes.
* `kill -TERM pid`, `kill -KILL pid` – polite vs forced termination.
* `strace ./a.out` – show system calls used by program.
* `cat /proc/<PID>/maps` – show memory layout (text, heap, stack, libs).

```

---

If you want, next step we can **prune or compress** this for your actual A3 layout (e.g., bolding only the stuff you truly forget, shrinking commentary, etc.), but this `.md` has everything you said you needed to not get wrecked.
```

