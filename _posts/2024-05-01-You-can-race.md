---
layout: post
title: "You Can Race, But You Can't Hide : On the efficient  detection of determinacy races in Cilk Programs"
date: 2024-05-01
---
### Anurag Choubey
---

### Full bibliographic citation for research paper<br>
Mingdong Feng and Charles E. Leiserson. 1997. Efficient detection of determinacy races in Cilk programs. In Proceedings of the ninth annual ACM symposium on Parallel algorithms and architectures (SPAA '97). Association for Computing Machinery, New York, NY, USA, 1–11. https://doi.org/10.1145/258492.258493

---

### Description

The paper referenced by the citation above provides a time efficient algorithm for the detection of determinacy races in programs that are written in the Cilk programming language. The authors also present a debugging tool called the Nondeterminator. In this blogpost we shall discuss the motivation behind the paper as well as the algorithm in the paper. There is also a discussion of a section that extends the algorithm to support programs that support atomic accumulation of results within the program. We have skipped the correctness argument for the algorithm as well as the experimental results in this blogpost in order to keep it brief and not too technical. The curious reader may take a look at it in the original paper.

Note: All images and mathematical formulae / notations have been taken from the original paper.

---

## 1. Introduction and Motivation

A multithreaded deterministic program is defined as the one which produces the same behavior regardless of how the thread scheduling takes place. In this context, a determinacy race occurs if a thread is updating a memory location while another thread accesses it. Due to their non-repeatability, bugs arising out of race conditions are hard to debug using the normal techniques like breakpointing.
Due to this limitation the authors come up with an SP bags algorithm that detects potential races in Cilk programs. This algorithm is used in the Nondeterminator debugging tool earlier mentioned. This algorithm is inspired by Tarjan's least common ancestors algorithm.

### Some Terms and Concepts

Let us introduce some constructs that we shall use later.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images//fig_1.png" width="400" height="400">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 1: Program</p>
</div>
<br>

Consider the code in Figure 1. The eyeballing of a potential race condition here is left to the reader. We introduce the concept of a spawn tree as seen in Figure 2. This spawn tree has as children of main() the two subprocedures that run in parallel.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images//fig_2.png" width="400" height="200">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 2: Spawn Tree</p>
</div>
<br>

We also represent the code in Figure 1 as a directed acyclic graph as seen in Figure 3. The vertices are the spawn/sync constructs and the edges represent the Cilk threads. It is important to note here that the paper represents the Cilk threads as __"maximal sequence of instructions that do not contain any parallel control constructs."__ What this means is that the labels marked 'e1', 'e2' etc. in Fig 1 are threads, since they are instructions in the code that have nothing to do with spawning and syncing.
Such a construction of the programs allows us to represent the code as shown in Figure 3.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images//fig_3.png" width="450" height="300">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 3: Parallel Control Flow DAG</p>
</div>
<br>

We shall now proceed to understand the SP Bags algorithm.

### 2. The SP Bags Algorithm

The SP bags algorithm uses the disjoint set data structure, which is aso employed by Tarjan's LCA algorithm. For a description of both see the links below.

* https://cp-algorithms.com/data_structures/disjoint_set_union.html
* https://cp-algorithms.com/graph/lca_tarjan.html

#### Description

- The SP bags algorithm is a serial algorithm. It executes the parallel Cilk program serially.
- It maintains two shadow spaces known as reader and writer spaces. Every procedure executing in a Cilk program is assigned a unique ID.
- If a procedure writes to a memory location x, then the UID of that procedure is stored in the location writer(x) (location of the writer space).
- For reads too the same step is followed.
- Shadow locations are modified as the program executes.

#### Bags

- Two bags of procedure ID's are maintained for every procedure in the program call stack.
- The S bag (SF) stores the ID's of all the completed children (and their descendants) of F (current procedure) that precede the current executing thread. It also stores ID of F itself.
- The P bag (PF) contais ID's of the completed children (and their descendants) that operate in parallel with the current executing thread.
- The S and P bags are represented as a disjoint set data structure.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images//fig_8.png" width="350" height="480">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 5: SP Bags Algorithm</p>
</div>
<br>

- The algorithm takes an action whenever it encounters the read, write, spawn, sync and return keywords in the program.

#### Explanation

As the algorithm executes, the contents of the S and P bags are updated each time a spawn, sync and return keyword is encountered.
- A spawn (say procedure F) creates a new singleton set containing with just F is added to the disjoint set data structure.
- When a subprocedure F' returns to its parent F, the contents of SF' are emptied into PF. The paper says that is because any subprocedure of F' can execute in parallel with the current thread.
- When a sync keyword is encountered, the contents of PF are emptied into SF. This is because every thing prior to the sync statement logically precedes everything after the sync statement. 
- So when does a determincacy race occur?
  - When F writes to a location x, discovers that previous reader or writer of x belongs to a P bag. This implies parallel operations.
  - When F reads a location x, only to find out that the previous writer is in a P bag.

#### Confused Yet??

- We found the wordings in the original paper to be quite wanting. The updates of the S bag and P bags were extremly confusing. We attempt to answer some questions that we asked ourselves.
- Why should a subprocedure empty it's S bag into the parent's P bag upon a return? Well because remember the SP bags algorithm runs the Cilk program serially. So a DAG like that in Figure 3 is squished to a single long line of serial procedures and constructs. Look at Figure 3. While being executed inside of SP bags algorithm, F1 shall precede e1. The only way to represent parallelism in such a serial execution is to empty the contents of the S bag of F1 into the P bag of F. What that action says is "If F1 and e1 were to execute in parallel then the subprocedures of F1 would have been in parallel execution with F, hence contents of F1's S bag go in P bag of current thread".
- Why does a sync cause a procedure to empty its P bag into its S bag? Again take the example of Fig 3. All procedures of F after e2 logically precede e3. All the children of F1 and F2 and their spawned grandchildren now are wrapped up.

### 3. Support For Atomic Accumulation

The Cilk language supports atomic accumulation of results that are returned from spawned procedures. For example, if the operators involved in accumulation are *= and *= or += and -= then the accumulations cannot be seen as races, since the operations in the pairs are commutative to each other.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images//fig_13.png" width="230" height="250">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 8: Atomic Accumulation</p>
</div>
<br>

In the above figure, the updates to x are commutative as thread scheduling will not affect the final result of x. This may be viewed as a legal determincacy race. We now see how the SP Bags algorithm can be modified to accomodate legal races. There are two changes to be made.

- We create a new shadow space called operator. If a memory location x is updated via an operator op then we write op to location x in the operator shadow space.
- Each sync block is now assigned a unique ID (B).

Let us understand with the psudocode below. Comments start with //.

```c
write a shared location l with operator op by procedure F in sync block B:
    if FIND-SET(reader(l)) is a P-bag
        then a determinacy race exists

     
    // In addition to the normal write race detection
    // race could be illegal if the operators are not
    // commutative or the procedure performing a commutative
    // write is part of another sync block

    if FIND-SET(writer(l)) is a P-bag
        and (writer(l) ≠ B or op does not commute with operator(l))
            then a determinacy race exists
    // update if all good
    writer(l) ← F
    operator(l) ← op

    // Introduce a new action accumulate.
    // performed each time there is an accumulation.

accumulate returned result of spawned procedure into a shared
location l with operator op by procedure F in sync block B:
    if FIND-SET(reader(l)) is a P-bag
        then a determinacy race exists
    if FIND-SET(writer(l)) is a P-bag
        and (writer(l) ≠ B or op does not commute with operator(l))
            then a determinacy race exists
    // If sync block previously unseen, place id B of this sync block
    // in P bag of curr procedure
    if FIND-SET(B) = Ø
        then Pf ← UNION(Pf, MAKE-SET(B))
    writer(l) ← B
    operator(l) ← op

```

### 4. The Nondeterminator

The authors introduce a novel debugger named the Nondeterminator. It takes as an input the Cilk program on which race detection is to be done. It either certifies that thwe program is race free, or it isolates and reports the locations in the program subject to determinacy races. It localizes a bug, reports the variable name, filename, line number as well as the dynamic context, which the authors define as the "state of the runtime, stack, heap, etc."
<br>
It is important to note that the Nondeterminator reports a program as bug free or not only for a particular input data, and is not a general program verifier. It checks whether every possible scheduling of threads within the program for a particular input dataset produces the same output or not.
<br>
#### Operation

- The Nondeterminator executes the program in a serial, depth-wise fashion. It is implemented by modifying the Cilk compiler and runtime system.
-  As we know it utilizes the SP Bags algorithm. Hooks are inserted by the compiler so that particular actions are taken when spawn, sync and return statements are encountered.
- The addresses of shadow spaces are fixed via mmap() system call.

#### Modifications to original Algorithm

- The Nondeterminator uses a slightly modified version of SP bags to improve performance.
- If the compiler determines that the memory reference is to a non-shared variable then determinacy race check is not performed on it. 
- Thread local software caches are introduced so as to avoid the full overhead of the SP bags algorithm. Addresses previously checked by a thread are cached to avoid them being rechecked by same thread.
