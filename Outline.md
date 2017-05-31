# Avoiding Conflicts in Concurrency Structures -- Outline

## Concepts

Dependency graphs will never have multiple edges between the same nodes.

### Signals and Signal Responses

+ Should threads be required to have defined responses for each Signal response type? I.e. each thread would have a 'Started', 'Completed', 'Aborted', etc. signal response.
    * Example Response: abort chains/trees -- once a node aborts an operation, then all dependent nodes abort their operations, and the dependent nodes dependents do the same, etc.

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

###Extensional Lists vs. Intensional Conditions

Intensional Conditions: would have to be part of a Breadth-First or Depth-First traversal, where every node and edge in the CDS would be compared against the defined condition, and if it passed, would be considered accessible by the UI.

Since these complete-traversal checks could be computationally expensive to perform every time we want to execute an access procedure, we instead will prefer the creation of extensional lists that are attached to a UI that can be used.

Intensional Conditions can be used if the CDS is small enough to not become a processing burden.

### Assignment Process

goRoutine Construction: configures goroutines (access procedures) with appropriate information at launch.

- Number of UIs (best performance: must not be greater than the number of concurrent processes accessing the structure)

What happens if we want to use more threads than there are unique independents? This is when we introduce: Temporal Nodes.

What happens if we have fewer threads than UIs? In this scenario, we are assuming that each thread can be assigned multiple UIs, and in this case the thread must have an internal degree of consistency defined. The thread must internally perform operations in their dependency order. (algorithmically: this just changes the way we construct goRoutines before launching them -- we will focus on making sure that all modifications are pre-ordered within the goroutine beforehand).

The assignment function will assign a single (V)UI to each thread.

### Uses of Signals

A thread can have multiple completion points i.e. 'completed' signals (similar to generator/async functions in ES6/7). 

IDEA: Iterators can be used for thread signal responses, i.e. we are expecting so many 'Completed' signals, and an iterator object could keep track of how many 'Completed' signals we have received from a certain dependency.

A completed signal can tell a thread to proceed or not with its next operation. A thread can stall while checking that its dependencies have signaled complete so it can proceed into a starting point.

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

### Pass by Reference
s
- We ALWAYS pass the following by reference ONLY (post-creation):
    + CDS
    + CDS Node
    + CDS Edge
    + Section

## Memory Management

Section will contain: Rust Code (showing custom memory management for DDAG concurrency approach)

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