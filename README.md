# CS351 "Book"

This is a collection of detailed lecture notes for BU's CS351.

# Processes, Communication, Parallelism

In order to design a distributed system, we need a mechanism for **communication over the network**. Unfortunately, communication is not instantaneous which lends itself to the need to **concurrently** perform computation *while* we wait for that communication to occur. For beginner programmers, this can be a big shift away from the types of programs we have written thus far. Thus, we will dive into how to write a **multi-threaded process** within a distributed system– a key building block in an efficient distributed system.

Then, we will see how distributed systems lend themselves to highly parallel computation. We will study how distributed systems are designed with parallelism in mind via **sharding.** Finally, we will also study a foundational, real-world, highly parallel distributed computing framework– **MapReduce**.

- [chapter 1](/chapters/lec1.md)
- [chapter 2](/chapters/lec2.md)
- [chapter 3](/chapters/lec3.md)
- [chapter 4](/chapters/lec4.md)
- [chapter 5](/chapters/lec5.md)


# Synchronization and Fault Tolerance

As we started to see with the inspection of MapReduce, when we have many servers and many messages being sent, failures become the expectation instead of the exception. Thus, we want to design systems that behave correctly even when failures are present. This desirable property is known as **fault tolerance.**

But, what is “correct” behavior? In a typical program, we take for granted the **ordering** of events that occurred. But in a distributed system, as we rely on faulty messages passing between machines that do not have a shared clock, it can become difficult, or impossible, to be precisely sure of the ordering of events within our system.

In this section we will also start to inspect how distributed systems are designed to continue to operate even when failures are present. Furthermore, we will study the problems of **synchronization** in distributed systems and use **timestamps** in order to reliably order causally related events in a system when real-time fails us.

- [chapter 6](/chapters/lec6.md)
- [chapter 7](/chapters/lec7.md)
- [chapter 8](/chapters/lec8.md)

## Replication and Consensus

One method to achieve fault tolerance is to have multiple **replicas** of a dataset or service at any one time. This way, if a single server fails, we can redirect clients to a working replica server instead. However, ensuring that all replicas stay up to date can prove difficult especially as messages are lost, or network **partitions** form. 

In this section we discuss replication and some simple schemes for achieving a working replication protocol such as **primary/backup replication**. We will then move to discuss **distributed consensus protocols**, where we do not rely on any outside authority servers, and instead can achieve consensus strictly through **quorum** (voting) based protocols.


- [chapter 9](/chapters/lec9.md)
- [chapter 10](/chapters/lec10.md)
- [chapter 11](/chapters/lec11.md)
- [chapter 12](/chapters/lec12.md)

## Consistency

As we have seen in the previous sections, when we introduce replication into a distributed system, or data store, it is possible that clients may experience executions that would not exist on a single, local data store. For example, reads may return stale values when we switch to read from a different, out-dated, replica server. 


A consistency model, for example, answers the question: is it okay that our system sometimes returns old values to clients? Or do all reads need to reflect the latest writes? Maybe we can return stale data, as long as we reflect the latest write that the client themselves has performed. 

As distributed system engineers we get to decide what consistency guarantees our system provides and there is an inherent tradeoff: systems with stronger consistency models are more difficult to design to be performant and available, but systems with weaker consistency models can be difficult for clients to have correct behavior for their applications.

In this section we will learn about different consistency models and understand how different real-world systems are designed to provide consistency.

- [chapter 13](/chapters/lec13.md)
- [chapter 14](/chapters/lec14.md)



# Other Excellent Resources

[https://book.mixu.net/distsys/index.html](https://book.mixu.net/distsys/index.html)   
[https://lamport.azurewebsites.net/pubs/time-clocks.pdf](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)   
[https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/](https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/)   
[https://jepsen.io/consistency/models](https://jepsen.io/consistency/models)   
[http://nil.csail.mit.edu/6.824/2022/schedule.html](http://nil.csail.mit.edu/6.824/2022/schedule.html) 

[http://nil.csail.mit.edu/6.824/2022/notes/l-cr.txt](http://nil.csail.mit.edu/6.824/2022/notes/l-cr.txt)   
[http://nil.csail.mit.edu/6.824/2022/papers/vm-ft.pdf](http://nil.csail.mit.edu/6.824/2022/papers/vm-ft.pdf)   
[https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf](https://www.cs.cornell.edu/home/rvr/papers/OSDI04.pdf) 

[https://www.distributed-systems.net/index.php/books/ds4/](https://www.distributed-systems.net/index.php/books/ds4/)