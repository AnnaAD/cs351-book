## Lecture 1: Threads and Processes

A **distributed system** is defined as a system wherein processes or components of that system communicate via message passing.

However, in order to program such a system, we immediately run into inefficiencies if we write single-threaded programs (as you probably have been doing up until this point).

### Why Multi-Thread?

Consider the following example. Let’s say we have three satellites and a command center on earth. The satellites are constantly performing measurements of their current distance to the moon. We want our command center on earth to poll the three satellites for their reading list, average the readings, and then report a list of the three averages, one per satellite. 

If we write this serially, our program might look something like this:

| CODE\_SECTION |
| :---- |
| for s in satellites: send\_msg(to: s, “update?”) res := wait\_response() sum := 0 for record in res.records: sum \+= record append(sum/len(res.records)) |
| CODE\_SECTION\_END |

But consider, what happens if one of our three satellites is down? wait\_response() may block forever\! Also consider, are we wasting time? The answer is– probably yes\! Imagine that receiving a response can take anywhere from 1-6 seconds, and calculating the average takes about 1 second. Ideally, we would like to send all our requests out at the same time. 

For example, if one satellite responds in 1 second, one responds in 4 seconds, and one responds in 2 seconds, how long would it take serially? 1+1 \+ 4 \+ 1+2+1=10 seconds\!

But consider if we sent out all requests at the same time? Well, after 1 second we receive satellite\_1’s response, and calculate the average in additional 1 second. Right when we are done averaging we get satellite\_3’s response, and one second of averaging. And then finally, we receive satellite\_2’s response and average that one. How long is our process now? 1+1+1+1 (idle)+1 \= 5 seconds\!

Clearly, in a distributed system where communication time is non negligible, it is important we write code in a way that it can be run simultaneously. Ideally, we would want three small programs running at once– one that sends messages and averages the response for each of the three satellites. This small program– can be called a **thread.** 

### 

### Threads and Processes

What is a thread? In order to answer this question completely, we must remember from CS210, what is a **process?** 

*A **process** is a running program. The **operating system** manages processes. The operating system allows many running processes to seamlessly share a fixed set of hardware devices. The operating system **interrupts** running processes and interleaves these running processes so that each process has a fair turn, running on the CPU. Deciding which processes run on the CPU is known as **scheduling.***

*The operating system also facilitates the **isolation of memory** between processes. The operating system manages the **page table,** a data structure through which each memory lookup goes through. Processes have instructions with memory addresses. The addresses that the process uses are **virtual addresses.** These virtual addresses are then translated via the page table to **physical memory addresses** to lookup in RAM.*

So what is a **thread?** A process may have multiple threads (a process has a single thread by default). These threads are **scheduled independently** by the operating system. Two or more threads of the same process may run simultaneously or be arbitrarily interleaved. The decision of what thread to run at what time is up to the operating system. The operating system notes when threads are waiting for resources, like a response over the network, and can spend CPU time progressing on a different thread\!

Another key feature of threads is that threads of a process share the same **virtual address space**. This means that two threads of the same process have access to the same memory, the same data, the same variables. Shared memory is a big advantage– we don’t need to worry about message passing or the like in order to communicate between threads. However, unfettered access to shared memory can cause problems, like **data races.** 

### Goroutines

So how do we program a multi-threaded program? Golang, fortunately, is a language that was designed with concurrency as a main priority. The keyword `go` in golang creates a new **goroutine** to execute a provided function call**.** 

**Goroutines** are similar, but not directly analogous to threads. Goroutines are the unit of concurrency in the go programming language. Like threads, they are scheduled independently meaning that they can run simultaneously or in an interleaved fashion when one goroutine is blocked. Also like threads, two goroutines of the same go process share memory, they can share any data that is in-scope for both function calls.

Goroutines differ from threads in how they are scheduled and managed. Instead of the operating system scheduling goroutines, the **go runtime** is responsible for scheduling goroutines and schedules them on top of OS threads. Operating system scheduling is costly– to run OS code we need to perform a **context switch**, saving all information and interrupting the running process. However, the go runtime runs entirely in userspace, meaning that its scheduling decisions are much more efficient. The go runtime is free to switch go routines from running on top of one OS thread to another without a context switch. Because of this feature, **goroutines are lightweight compared to threads**, it is easy to have 100K goroutines without seeing memory problems or performance issues.

### Concurrency Control Primitives

When two or more concurrent threads access the same data, and one of those threads performs a write, we have a **data race**.

Consider the two following threads:

| CODE\_SECTION |
| :---- |
| **Thread 1:** sum \= 2 \+ 4 \+ 6 print(“avg \=”, sum / 3\)  **Thread 2:** sum \= 7 \+ 3 print(“avg \=”, sum / 2\) |
| CODE\_SECTION\_END |

There are two shared variables, `sum` and `avg`, and perform reads and writes to both variables in the two threads. What might happen?

| CODE\_SECTION |
| :---- |
| 4 5 |
| CODE\_SECTION\_END |

But what happens in the multi-threaded setting?

Well, it is possible we set sum \= 12, then run thread 2, set sum \= 10, print avg \= 5, then finally print avg \= 10/3\!

| CODE\_SECTION |
| :---- |
| 5 3.3333 |
| CODE\_SECTION\_END |

So how do we avoid unwanted data races? Well, there may be certain sections of code where we want **concurrency control.** Concurrency control involves limiting simultaneous execution of specific sections of critical code.

### Locks

One tool for concurrency control is known as a **lock** or a **mutex** (standing for mutual exclusion). 

A lock allows for only **one thread** to acquire the lock and progress past a blocking `lock` statement. Any other threads that call `lock` on that lock will block until the holding thread releases the lock.When a thread holding the lock calls `unlock` exactly one other thread may succeed in acquiring the lock and unblocking to progress.

| CODE\_SECTION |
| :---- |
| **Thread 1: lock.Lock()** sum \= 2 \+ 4 \+ 6 avg \= sum / 3 print(avg) **lock.Unlock()**  **Thread 2: lock.Lock()** sum \= 7 \+ 3 avg \= sum / 2 print(avg) **lock.Unlock()** |
| **CODe\_SECTION\_END** |

With this scheme we may now see:

| CODE\_SECTION |
| :---- |
| 4 5 |
| CODE\_SECTION\_END |

or we may also observe:

| CODE\_SECTION |
| :---- |
| 5 4 |
| CODE\_SECTION\_END |

As it is possible that either thread may execute first, succeeding in acquiring the lock. However, it is not possible for the two threads to simultaneously execute the section in between the acquire/release statements.

### Channels

In golang another useful concurrency control primitive is a **channel.** A channel allows for two goroutines to pass and receive messages or data. Using a channel often **eliminates a shared variable** (thus eliminating a data race), and instead requires goroutines to send partial results or computations to some accumulator goroutine via the channel.

`chan <-` is the ***send*** channel operator. This will block if the channel is full (by default the buffer size of the channel is 0 and this will block until someone reads from the channel).

`<- chan`  is the ***receive*** channel operator. This will block if there is nothing to read from the channel.

`make(chan T)` creates an unbuffered channel (size 0, will block writing until someone reads from channel).

`make(chan T, N)` creates a buffered channel of size N. N messages may be written to the channel without blocking. Once full, writers will block until space in the channel buffer opens up.
