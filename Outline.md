# Avoiding Conflicts in Concurrency Structures -- Outline

## Introduction

- Concurrency Structures: Definition
    + Combining Trees
        * as well as the fact that only leaf nodes are assigned a thread i.e. the number of leaf nodes equals the number of threads
- Conflicts
    + redudant writes
    + out-of-order modifications
    + blocking (can still happen)
    + deadlocking (can never happen in our system as defined)
        * deadlock: A deadlock is a state where two, or more, threads are blocked waiting for the other blocked waiting thread (or threads) to finish and thus none of the threads will ever complete
- Locking, Fine-grained Locking, Linearization (and other proposed solutions)
- An important thing we will find: the actual structure of the data structure is not what matters for concurrency.

## Dependency DAGs (DDAGs)

The total change to the structure can be considered completed if every node in the DDAG is marked completed.

A thread is allowed to start it's modification if the list of independent dependencies are marked completed.

What happens if we want to use more threads than there are unique independents? This is when we introduce: **Virtual DDAGs**

Sidepoint: every node (both real and virtual independents) can be represented by 4 numbers (real dependencies, virtual dependencies, real dependents, virtual dependents).

- Real Nodes: {n, m, l, 0}
- Real Boundary Nodes: {n, m, 0, 0}
- Virtual Nodes: {0, n, m, l}
- Virtual Boundary Nodes: {0, 0, n, m}

A boundary node is a node without dependencies and can thus always proceed infinitely often post completion points.

Threads with access to boundary nodes can modify the structure of assigned independent at those nodes.

- Modifying the structure itself: e.g. adding or deleting unique independents, modifying an independents dependencies list, or just modifying which part of the structure a unique independent is referring too.
    + this is easy when we have e.g. a 2-Regular Tree with only 3 nodes, then we can create three independents: A (the root), B (the left sub-tree), and C (the right sub-tree), now adding or deleting nodes to the tree is handled exclusively by B OR C (this is also simple for n-Regular Trees in general).
    + Invariants: in the above example we assume that the root nodes for the B and C sub-trees will not dissapear. So, when defining unique independents as sub-structures it can be helpful to be able to note if any of the structure is "immutable". Or even more precisely, which parts of the structure are invariant under a modification or class of modifications.
    + the question becomes: how do we generalize this to arbitrary structures?
        * Boundary Nodes
        * perhaps it is outside the scope of this paper ...
- Dynamic Dependency Relationships
    + What if dependencies were not just spatial but also temporal? Like, what if a particuliar write on one independent needed to happen before a different write on another independent, but that this was not necessarily true for all writes for these independents?
    + If threads are essentially just atomic operations, then we can create a virtual dependency complex that connects the two unique independents to the final write thread and then that final write thread is virtually dependent on the initial write thread.
        * virtual dependencies are thread dependencies only, and not independent dependencies i.e. they are purely temporal dependencies on the same spatial location.
        * If the spatial location is multiple independents, then that V-DDAG is called a *Temporary Spatial Dependency Complex*
    - this will require that garbage collecting virtual dependency complexes will become essential
    - every thread can be assigned to multiple virtual dependency complexes

### Operation Priorities as a Partial Ordering for V-DDAGs

V-DDAGs can be ordered by *operation priority* i.e. if `read`s have higher priority than `write`s, or vice versa.

This can be used to provide a specific degree of consistency for reads on the structure (as to a specific degree for the gurantee of a read having the latest data).

## Algorithm

Section wil contain: Go Code Example (using goRoutine access to a shared data structure)

- Unordered (unique) Set of Unique Independents (must not be greater than the number of concurrent processes accessing the structure)
- Unordered (unique) Set of dependencies
- Every thread is assigned:
    + a Unique-Independent (and its list of dependencies)
    + multiple virtual dependency complexes (temporal dependency complexes)
- Assignment Function
    + The Assignment function needs to also be aware of which unique independents have a thread or a virtual DDAG (Dependency-DAG) already assigned to them. It needs to be able to (re-)assign threads to available unique independents. This will be useful for when a thread is e.g. a processing request on a server, and will have a finite lifespan. (In other words, since virtual nodes are basically just threads, the assignment function will need to keep track of thread orderings per real node.)
    + Can we run into an issue with the Assignment Function being blocked from updating its assignment list for a unique independent (??)
    + Assignments can also be classified: e.g. a thread can be assigned to a unique independent but it is only allowed to read from the data structure, whereas another thread can be assigned that is allowed to modify the data, etc.

Threads will require to have a starting point and a completion point that can encompass a single modification or a sequence of modifications. A thread can have multiple completion points (similiar to generator/async functions in ES6/7). A completion point allows a thread to mark a node in the DDAG as completed which communicates to its dependents that they can proceed. A thread stalls checking that its dependencies are complete so it can proceed past a completion point.

What happens if we have fewer thread than unique independents? In this scenario, we are assuming that each thread can be assigned to multiple independents, and in this case the thread must have an internal degree of consistency (strict concistency usually). The thread must internally perform modifications in their dependency order.

The assignment function will assign a single thread per Real Independent and Virtual Independent (which is basically just a thread).

We do not gurantee consistency with our algorithm unless we place read-only threads into a V-DDAG, and then consistency is defined by the ordering of read vs. write threads in the V-DDAG.

### Atomic Modification Sequences composition is Non-Commutative

NOTE: combining trees only work because addition is a commutative operation. In arbitrary V-DDAGs we will need some form of composition for atomic modification sequences. That composition will be Non-Commutative, and will *only* depend on the topology of the DDAG i.e. ordering for nodes that are not dependent on each other will not matter (thus commutative), but if a node is dependent on another node then the dependency operation must occur before the dependent operation (non-commutative).

### Freedoms

How does this algorithm do in terms of:

- *Obstruction Freedom* (least strict)
    + Non-blocking progress guarantee. At any point in time, a single thread is guaranteed to make progress if other threads are suspended. Partially completed operations must be abortable.
- *Lock Freedom*
    + Per-object progress guarantees. It is guaranteed that some operation will always complete successfully in a finite number of steps.
- *Wait Freedom* (most strict)
    + Per-operation progress guarantees. Every operation is guaranteed to complete in a finite number of steps.

### Problem

- Technically, our algorithm is not free of "deadlocking"
    + In that, threads can still stall internally and cause dependents to stall forever as well.
    + We could implement a form of preemption where a thread waits a maximum number of time before moving ahead beyond an unfinished thread, and the thread that has not completed is removed and a new thread assigned to its position. (-- MAKE BETTER)

## Formal Verification

## Memory Management

Section will contain: Rust Code (showing custom memory management for DDAG concurrency approach)

- Memory Management techniques for Concurrent Data Structures while this proposed algorithm is in use (??)

## Conclusions

- Performance (how does this effect linear speedup)
- In a distributed environment it is likely that the Assignment Function will evolve into something similiar to a: Distributed Lock Manager
    * https://en.wikipedia.org/wiki/Distributed_lock_manager

## References

- Concurrent Data Structures
- Concurrencykit.org - Lock-Free Algorithms
- Deadlocks: http://www.javacreed.com/what-is-deadlock-and-how-to-prevent-it/