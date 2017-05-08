# Avoiding Conflicts in Concurrency Structures -- Outline

## Concepts

Dependency graphs will never have multiple edges between the same nodes.

https://en.wikipedia.org/wiki/Snapshot_isolation

Snapshot Isolation is essentially the idea of grabbing the value of a piece of data, then checking that it matches the current value right before modifying it. If the value is different than at grab time, the thread will abort the operation.

This is useful as threads may be scheduled in-and-out during their total procedural runtime.

This can also be useful for when a thread will be modifying multiple pieces of data.

https://en.wikipedia.org/wiki/ACID
https://en.wikipedia.org/wiki/Concurrency_control#Database_transaction_and_the_ACID_rules

### Freedoms

These are design choices left up to the user. But, the concepts can be very easily accommodating to any of these scenarios ...

- *Obstruction Freedom* (least strict)
    + Non-blocking progress guarantee. At any point in time, a single thread is guaranteed to make progress if other threads are suspended. Partially completed operations must be abortable.
    + Should threads have only completion points, or should they be allowed to have "abort" points as well. And does a dependency need to "retry" (perhaps with an exponential backoff) before a dependent is allowed to move ahead?
    + Should aborts be "passed up" like completions are? In which, case if a node has a dependency reply with an abort, then it too must abort and pass the abort up the chain?
    + Three signal classes: complete, aborted, abort-retry
- *Lock Freedom*
    + Per-object progress guarantees. It is guaranteed that some operation will always complete successfully in a finite number of steps.
- *Wait Freedom* (most strict)
    + Per-operation progress guarantees. Every operation is guaranteed to complete in a finite number of steps.

## Algorithm

Graphs are global. Threads assigned to nodes in the graph can mark the nodes as either incomplete, complete, aborted, or abort-retry.

SHOULD WE HAVE A *PARTIAL-ABORT* SIGNAL? And if so, how would it work?

goRoutine (Access procedure) Constructor Function: configures goroutines (access procedures) with appropriate information at launch.

Section will contain: Go Code Example (using goRoutine access to a shared data structure)

- Unordered (unique) Set of Unique Independents (must not be greater than the number of concurrent processes accessing the structure)
    + perform *cycle detection* on independents graph
- Unordered (unique) Set of dependencies
- Every thread is assigned:
    + a Unique-Independent (and its list of dependencies)
    + multiple virtual dependency complexes (temporal dependency complexes)
- Assignment Function
    + The Assignment function needs to also be aware of which unique independents have a thread or a virtual DDAG (Dependency-DAG) already assigned to them. It needs to be able to (re-)assign threads to available unique independents. This will be useful for when a thread is e.g. a processing request on a server, and will have a finite lifespan. (In other words, since virtual nodes are basically just threads, the assignment function will need to keep track of thread orderings per real node.)
    + Can we run into an issue with the Assignment Function being blocked from updating its assignment list for a unique independent (??)
    + Assignments can also be classified: e.g. a thread can be assigned to a unique independent but it is only allowed to read from the data structure, whereas another thread can be assigned that is allowed to modify the data, etc.

The total change to the structure can be considered completed if every node in the DDAG is marked completed.

Threads will require to have a starting point and a completion point that can encompass a single modification or a sequence of modifications. A thread can have multiple completion points (similar to generator/async functions in ES6/7). A completion point allows a thread to mark a node in the DDAG as completed which communicates to its dependents that they can proceed. A thread stalls checking that its dependencies are complete so it can proceed past a completion point.

A thread is allowed to start it's modification if the list of independent dependencies are marked completed.

What happens if we want to use more threads than there are unique independents? This is when we introduce: **Temporal DDAGs**

What happens if we have fewer threads than unique independents? In this scenario, we are assuming that each thread can be assigned to multiple independents, and in this case the thread must have an internal degree of consistency (strict consistency usually). The thread must internally perform modifications in their dependency order. (algorithmically: this just changes the way we construct goRoutines before launching them -- we will focus on making sure that all modifications are pre-ordered within the goroutine beforehand).

The assignment function will assign a single thread per Real Independent and Virtual Independent (which is basically just a thread).

We do not guarantee consistency with our algorithm unless we place read-only threads into a V-DDAG, and then consistency is defined by the ordering of read vs. write threads in the V-DDAG.

### Atomic Modification Sequences composition is Non-Commutative

NOTE: combining trees only work because addition is a commutative operation. In arbitrary V-DDAGs we will need some form of composition for atomic modification sequences. That composition will be Non-Commutative, and will *only* depend on the topology of the DDAG i.e. ordering for nodes that are not dependent on each other will not matter (thus commutative), but if a node is dependent on another node then the dependency operation must occur before the dependent operation (non-commutative).

### Problem

- Technically, our algorithm is not free of "deadlocking"
    + In that, threads can still stall internally and cause dependents to stall forever as well.
    + We could implement a form of preemption where a thread waits a maximum number of time before moving ahead beyond an unfinished thread, and the thread that has not completed is removed and a new thread assigned to its position. (-- MAKE BETTER)
- WHEN A THREAD IS MARKED IN A COMPLETED STATE -- WHEN CAN IT RESUME INTO THE NEXT INCOMPLETED STATE? ALL OF ITS DEPENDENTS NEED TO BE AWARE THAT IT COMPLETED! AND SHOULDN'T THEY COMPLETE BEFORE THE THREAD RUNS THE NEXT ATOMIC OPERATION?
    + this might be why we would use wait groups, and reuse a wait group. For example, all of a nodes dependencies in a wait group will let the node know when it can proceed (when everything in the waitgroup is marked done).
    + However, the question arises: should a dependency add its dependents to a waitgroup that should complete before moving on to its next atomic operation?
    + The answer is: IF YOU WANT IT TO (??), however, this creates the pattern of dual dependency (CIRCULAR DEPENDENCIES)``
    + This can be resolved by a: *Recursive Thread* (assigned to a single node)

## Memory Management

Section will contain: Rust Code (showing custom memory management for DDAG concurrency approach)

- Memory Management techniques for Concurrent Data Structures while this proposed algorithm is in use (??)

## Conclusions

- Performance (how does this effect linear speedup)
- In a distributed environment it is likely that the Assignment Function will evolve into something similar to a: Distributed Lock Manager
    * https://en.wikipedia.org/wiki/Distributed_lock_manager

## References

- Concurrent Data Structures
- Concurrencykit.org - Lock-Free Algorithms
- Deadlocks: http://www.javacreed.com/what-is-deadlock-and-how-to-prevent-it/
- Dependency Graph: https://en.wikipedia.org/wiki/Dependency_graph
- Topological Sorting: https://en.wikipedia.org/wiki/Topological_sorting
- Cycle Detection