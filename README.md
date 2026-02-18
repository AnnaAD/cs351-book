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


## Synchronization and Fault Tolerance

As we started to see with the inspection of MapReduce, when we have many servers and many messages being sent, failures become the expectation instead of the exception. Thus, we want to design systems that behave correctly even when failures are present. This desirable property is known as **fault tolerance.**

But, what is “correct” behavior? In a typical program, we take for granted the **ordering** of events that occurred. But in a distributed system, as we rely on faulty messages passing between machines that do not have a shared clock, it can become difficult, or impossible, to be precisely sure of the ordering of events within our system.

In this section we will also start to inspect how distributed systems are designed to continue to operate even when failures are present. Furthermore, we will study the problems of **synchronization** in distributed systems and use **timestamps** in order to reliably order causally related events in a system when real-time fails us.

- [chapter 6](/chapters/lec6.md)
