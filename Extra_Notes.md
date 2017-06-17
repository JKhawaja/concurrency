# Avoiding Conflicts in Concurrency Structures -- Notes

## Concepts

### Problem

- Technically, our concepts are not free of "blocking"
    + In that, threads can still stall internally and cause dependents to stall forever as well (as dependents are often waiting on a signal from a dependency -- but this is the very definition of a dependency, that the dependent is blocked -- to some degree -- until the dependency completes some operation.)
    + In other words, the topology of a dependency graph *defines* the *degree of blocking* that the program will have.
    + IDEA: implement a form of preemption where a thread waits a maximum amount of time on a dependency before moving ahead, and possibly the thread that has not completed is removed and a new thread assigned to its position.

## Code

### Acidic Procedures composition is Non-Commutative

NOTE: combining trees only work because addition is a commutative operation.

In arbitrary DGs we will need some form of composition for acidic procedure sequences. That composition will be non-commutative, and will *only* depend on the topology of the DG i.e. the ordering of nodes that are not dependent on each other will not matter (thus commutative), but if a node is dependent on another node then the dependency operation must occur before the dependent operation (non-commutative). 

This means that threads composed of multiple acidic procedures must ensure that acidic operations performed by the dependent thread do not occur before ALL acidic procedures in the dependency thread, that the acidic operation is dependent on, have occured. This implies that dependency relationships between threads can be more fine-grained (due to signaling) than just a hard "wait-till-dependency-completes" relationship.

## References

- Concurrencykit.org - Lock-Free Algorithms
- Dependency Graph: https://en.wikipedia.org/wiki/Dependency_graph
- Consistency Patterns: http://www.cs.colostate.edu/~cs551/CourseNotes/Consistency/TypesConsistency.html