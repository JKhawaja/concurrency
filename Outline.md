# Avoiding Conflicts in Concurrency Structures -- Outline

## Concepts

### Problem

- Technically, our concepts are not free of "blocking"
    + In that, threads can still stall internally and cause dependents to stall forever as well (as dependents are often waiting on a signal from a dependency -- but this is the very definition of a dependency, that the dependent is blocked -- to some degree -- until the dependency completes some operation.)
    + IDEA: implement a form of preemption where a thread waits a maximum amount of time on a dependency before moving ahead, and possibly the thread that has not completed is removed and a new thread assigned to its position.

## Code

### Assignment Function

goRoutine Construction: configures goroutines (access procedures) with appropriate information at launch.

- Number of UIs (best performance: must not be greater than the number of concurrent processes accessing the structure)

What happens if we want to use more threads than there are unique independents? This is when we introduce: Temporal Nodes.

What happens if we have fewer threads than UIs? In this scenario, we are assuming that each thread can be assigned multiple UIs, and in this case the thread must have an internal degree of consistency defined. The thread must internally perform operations in their dependency order. (algorithmically: this just changes the way we construct goRoutines before launching them -- we will focus on making sure that all modifications are pre-ordered within the goroutine beforehand).

The assignment function will assign a single (V)UI to each thread.

- In a distributed environment it is likely that the Assignment Function will evolve into something similar to a: Distributed Lock Manager
    * https://en.wikipedia.org/wiki/Distributed_lock_manager

### Uses of Signals

A thread can have multiple completion points i.e. 'completed' signals (similar to generator/async functions in ES6/7). 

IDEA: Iterators can be used for thread signal responses, i.e. we are expecting so many 'Completed' signals, and an iterator object could keep track of how many 'Completed' signals we have received from a certain dependency.

A completed signal can tell a thread to proceed or not with its next operation. A thread can stall while checking that its dependencies have signaled complete so it can proceed into a starting point.

### Immutability

UIs are not necessarily immutable, although we could have immutable UIs if we wanted to not allow structure modification access types to change that UI.

An UI can only be immutable if it is strictly-unique i.e. iff it only accesses nodes and edges that no other UI can access.

One of the main problems here is that if a UI covers a part of the structure that another UI addresses, then the other UI could modify the underlying CDS structure and this UI would then technically be changed.

This is why it is better to assign immutability to CDS nodes that way regardless of what UI a node or edges is in, if it is immutable it can never be changed by any access procedure.

### Acidic Procedures composition is Non-Commutative

NOTE: combining trees only work because addition is a commutative operation.

In arbitrary DGs we will need some form of composition for acidic procedure sequences. That composition will be non-commutative, and will *only* depend on the topology of the DG i.e. the ordering of nodes that are not dependent on each other will not matter (thus commutative), but if a node is dependent on another node then the dependency operation must occur before the dependent operation (non-commutative). 

This means that threads composed of multiple acidic procedures must ensure that acidic operations performed by the dependent thread do not occur before ALL acidic procedures in the dependency thread, that the acidic operation is dependent on, have occured. This implies that dependency relationships between threads can be more fine-grained (due to signaling) than just a hard "wait-till-dependency-completes" relationship.

### Pass by Reference

- We ALWAYS pass the following by reference ONLY (post-creation):
    + CDS
    + CDS Node
    + CDS Edge
    + Section

## Memory Management

Section will contain: Rust Code (showing custom memory management for DDAG concurrency approach)

The global dependency graph could manage its own memory i.e. check when a virtual nodes life is complete and deallocate the memory for the node and all of its edges.

A CDS object could manage its own memory: everytime a CDS node or edge was removed it could deallocate the memory. -- This could actually be the functionality of Access Types (procedures).

## References

- Concurrencykit.org - Lock-Free Algorithms
- Dependency Graph: https://en.wikipedia.org/wiki/Dependency_graph
- Consistency Patterns: http://www.cs.colostate.edu/~cs551/CourseNotes/Consistency/TypesConsistency.html