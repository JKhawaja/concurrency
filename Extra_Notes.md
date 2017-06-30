# Avoiding Conflicts in Concurrency Structures -- Notes

## Problem

- Technically, our concepts are not free of "blocking"
    + In that, threads can still stall internally and cause dependents to stall forever as well (as dependents are often waiting on a signal from a dependency -- but this is the very definition of a dependency, that the dependent is blocked -- to some degree -- until the dependency completes some operation.)
    + In other words, the topology of a dependency graph *defines* the *degree of blocking* that the program will have.
    + IDEA: implement a form of preemption where a thread waits a maximum amount of time on a dependency before moving ahead, and possibly the thread that has not completed is removed and a new thread assigned to its position.

## Define the Commutativity of Access Types

Dependency defines if the sequential composition of two access procedures is commutative or not.

Independent = Commutative, Dependent = Not-Necessarily-Commutative. 

A single access procedure, such as adding two integers, can be commutative in of itself and set a precedence for how to structure your dependency graph. For example: in a combining tree none of the leaf node threads need to be dependent on each other because they are all just adding two integers and can thus be arbitrarily composed.

A useful technique: define which pairs of Access Types are commutative and whether or not an Access Type is commutative in of itself. This will help determine what *must* be ordered and what mustn't be ordered.

This can be useful when designing posets.

The number of access procedures trying to access the same UI define an n-ary operation on that UI. Addition is a binary operation, but it could very well be that we have two writes and a read that need to be ordered: we define the rules for this 3-ary operation as w1*w2*r != r*w1*w2 != w1*r*w2, etc. or the rules could be w2*r*w1 == w1*r*w1, etc.