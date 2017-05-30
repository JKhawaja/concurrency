# Avoiding Conflicts in Concurrency Structures -- Outline

## Concepts

Dependency graphs will never have multiple edges between the same nodes.

### Signals

+ Should threads have only completion points, or should they be allowed to have "abort" points as well. And does a dependency need to "retry" (perhaps with an exponential backoff) before a dependent is allowed to move ahead?
+ Should aborts be "passed up" like completions are? In which, case if a node has a dependency reply with an abort, then it too must abort and pass the abort up the chain?
+ Threads *must* have defined responses to signals for *each* dependency

SHOULD WE HAVE A *PARTIAL-ABORT* SIGNAL? And if so, how would it work?

### Acidity Sub-Section

https://en.wikipedia.org/wiki/ACID

Threads should be acidic: atomic, consistent, isolated, and durable. Threads can be composable.

https://en.wikipedia.org/wiki/Snapshot_isolation

Snapshot Isolation is essentially the idea of grabbing the value of a piece of data, then checking that it matches the current value right before modifying it. If the value is different than at grab time, the thread will abort the operation.

This is useful as threads may be scheduled in-and-out during their total procedural runtime.

This can also be useful for when a thread will be modifying multiple pieces of data.

### Multiple CDS Systems

Can UIs and a single UI DDAG technically virtualize ALL CDSs across a system, since the UI graph is disjoint anyways, and this would also imply that UIs (since they are essentially just threads) could address CDS sub-graphs for *multiple* CDSs ...

### Freedoms

These are design choices left up to an engineer. But, the concepts can be very easily accommodating to any of these scenarios ...

- *Obstruction Freedom* (least strict)
    + Non-blocking progress guarantee. At any point in time, a single thread is guaranteed to make progress if other threads are suspended. Partially completed operations must be abortable.
- *Lock Freedom*
    + Per-object progress guarantees. It is guaranteed that some operation will always complete successfully in a finite number of steps.
- *Wait Freedom* (most strict)
    + Per-operation progress guarantees. Every operation is guaranteed to complete in a finite number of steps.

## Code

Graphs are global. Threads that are assigned nodes from the graph can cause the nodes to change state and signal that state.

**Extensional Lists vs. Intensional Conditions**
Intensional Conditions would have to be part of a Breadth-First or Depth-First traversal, then every node would be compared against the conditions, and if it passed, would be considered accessible by the UI. 

Since these complete-traversal checks could be computationally expensive to perform every time we want to execute an access procedure, we instead will prefer the creation of extensional lists that are attached to a UI that can be checked.

Intensional Conditions can be used if the CDS is small enough to not become a processing burden.

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

Threads will require to have a starting point and a completion point that can encompass a single modification or a sequence of modifications. A thread can have multiple completion points (similar to generator/async functions in ES6/7).

A completion point allows a thread to mark a node in the DDAG as completed which communicates to its dependents that they can proceed. A thread stalls checking that its dependencies are complete so it can proceed past a completion point.

A thread is allowed to start it's modification if the list of independent dependencies are marked completed.

What happens if we want to use more threads than there are unique independents? This is when we introduce: **Temporal DDAGs**

What happens if we have fewer threads than unique independents? In this scenario, we are assuming that each thread can be assigned to multiple independents, and in this case the thread must have an internal degree of consistency (strict consistency usually). The thread must internally perform modifications in their dependency order. (algorithmically: this just changes the way we construct goRoutines before launching them -- we will focus on making sure that all modifications are pre-ordered within the goroutine beforehand).

The assignment function will assign a single thread per Real Independent and Virtual Independent (which is basically just a thread).

We do not guarantee consistency with our algorithm unless we place read-only threads into a V-DDAG, and then consistency is defined by the ordering of read vs. write threads in the V-DDAG.

### Immutability

UIs are not necessarily immutable, although we could have immutable UIs if we wanted to not allow structure modification access types to change that UI.

An UI can only be immutable if it is strictly-unique i.e. iff it only accesses nodes and edges that no other UI can access.

One of the main problems here is that if a UI covers an part of the structure that another UI addresses, then the other UI could modify the underlying CDS structure and this UI would then technically be changed.

This is why it is better to assign immutability to CDS nodes that way regardless of what UI a node or edges is in, if it is immutable it can never be changed by any access procedure.

### Atomic Modification Sequences composition is Non-Commutative

NOTE: combining trees only work because addition is a commutative operation. In arbitrary V-DDAGs we will need some form of composition for atomic modification sequences. That composition will be Non-Commutative, and will *only* depend on the topology of the DDAG i.e. ordering for nodes that are not dependent on each other will not matter (thus commutative), but if a node is dependent on another node then the dependency operation must occur before the dependent operation (non-commutative).

### Problem

- Technically, our algorithm is not free of "blocking"
    + In that, threads can still stall internally and cause dependents to stall forever as well.
    + We could implement a form of preemption where a thread waits a maximum number of time before moving ahead beyond an unfinished thread, and the thread that has not completed is removed and a new thread assigned to its position. (-- MAKE BETTER)
- *Recursive Threads* (assigned to a single node) for resolving seemingly "cyclic" dependencies in dependency graphs.

### Pass by Reference
- We ALWAYS pass the following by reference ONLY (post-creation):
    + CDS
    + CDS Node
    + CDS Edge
    + Section

## Memory Management

Section will contain: Rust Code (showing custom memory management for DDAG concurrency approach)

- Memory Management techniques for Concurrent Data Structures while this proposed algorithm is in use (??)

The global dependency graph could manage its own memory i.e. check when a virtual nodes life is complete and deallocate the memory for the node and all of its edges.

A CDS object could manage its own memory: everytime a CDS node or edge was removed it could deallocate the memory. -- This could actually be the functionality of Access Types (procedures).

## Conclusions

- In a distributed environment it is likely that the Assignment Function will evolve into something similar to a: Distributed Lock Manager
    * https://en.wikipedia.org/wiki/Distributed_lock_manager

## References

- Concurrent Data Structures
- Concurrencykit.org - Lock-Free Algorithms
- Deadlocks: http://www.javacreed.com/what-is-deadlock-and-how-to-prevent-it/
- Dependency Graph: https://en.wikipedia.org/wiki/Dependency_graph
- Topological Sorting: https://en.wikipedia.org/wiki/Topological_sorting
- Cycle Detection
- Consistency Patterns: http://www.cs.colostate.edu/~cs551/CourseNotes/Consistency/TypesConsistency.html