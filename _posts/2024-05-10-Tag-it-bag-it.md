---
layout: post
title: "Tag it, Bag it, but Prove it First : On the Correctness of the SP Bags Algorithm"
date: 2024-05-10
---
### Anurag Choubey
---

### Full bibliographic citation for research paper<br>
Mingdong Feng and Charles E. Leiserson. 1997. Efficient detection of determinacy races in Cilk programs. In Proceedings of the ninth annual ACM symposium on Parallel algorithms and architectures (SPAA '97). Association for Computing Machinery, New York, NY, USA, 1–11. https://doi.org/10.1145/258492.258493

Note: All images and mathematical formulae / notations have been taken from the original paper, unless stated otherwise.

---

## Introduction

In the previous blogpost titled ["You Can Race, But You Can't Hide"](paper_blogpost.md), we introduced the reader to the SP Bags algorithm, which detects race conditions in multithreaded Cilk programs using a time-efficient algorithm known as the SP Bags algorithm.<br>
Let us recap the SP bags algorithm using the figure below:

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images/fig_8.png" width="350" height="480">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 1: SP Bags Algorithm</p>
</div>
<br>

According to the figure above, upon a read or a write within a cilk procedure, the algorithm employs FIND-SET operations on a shared location l to determine whether the [UID](paper_blogpost.md#description-1) of the last write or read on l is in a [P bag or S bag](paper_blogpost.md#bags). Accordingly it reports the presence or absence of a race condition.<br>
What we are about to see is how the algorithm can report a race condition by just determining the bag (S or P) in which the procedure id of the last reader or writer to a location is in. We shall now proceed to discuss some key ideas using which we shall slowly work our way upto the correctness proof of the SP bags algorithm. Basically we want to know how and why it correctly detects determinacy races.

## Main Idea #1: Least Common Ancestors

The least common ancestor of two nodes A and B in a tree is the node furthest away from the root (by depth) which is a common ancestor of both A and B. We shall see later how the SP bags algorithm uses the [LCA algorithm developed by Tarjan](https://cp-algorithms.com/graph/lca_tarjan.html). Fow now we must know that the SP bags algorithm uses the LCA algorithm to determine whether a UID belongs to a P bag or an S bag.

---

## Main Idea #2: Thread Notation & Series-Parallel Dags

To proceed forward in our correctness argument, we must familiarize ourselves with some notation and concepts first.
Consider the Cilk procedure below. Our goal in this section is to transform a user written Cilk program into a data structure on which we can run the SP Bags algorithm.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images/fig_1.png" width="400" height="400">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 2: Program</p>
</div>
<br>

- We can see that thread `e0` is in series with thread `e1` (note here that the [original paper](http://supertech.csail.mit.edu/papers/spbags.pdf) defines threads as "maximal sequence of instructions not containing any parallel control constructs"). We can represent such a series relation with notation `e0 < e1` hereafter.
- Had `e0` been parallel to `e1`, we would have written the relation as `e0 || e1`. Also to be noted here is that `e0 || e1` if and only if `e0 <! e1` and `e1 <! e0`.


### Series Parallel Dags

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images/spDag.png" width="500" height="150">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 3: 3 types of series parallel dags</p>
</div>
<br>

A series-parallel dag is a recursive construction, with two unique vertices, a source `s` and a sink `t`. A series-parallel dag (sp dag) can be constructed as follows:

- Base: A single edge connects the source and the sink.
- Series Composition: Two sp dags such that the sink of the first is the source of the second.
- Parallel Composition: Two disjoint sp dags such that their sources and sinks are the same respectively.

Following the definition of the sp dag, we can declare 4 properties of sp dags which the [original paper](http://supertech.csail.mit.edu/papers/spbags.pdf) presents without proof:

Let `G'` be a series-parallel dag and `G` as a subdag of `G'`. If `s` and `t` are the source and sink of `G`, then the following properties are true:

- There exists a path in `G` from `s` to any other edge in `G`.
- There exists a path from any edge in `G` to `t`.
- Every path in `G'` that begins outside of `G` and enters `G` passes through `s`.
- Every path in `G'` that begins within `G` and leaves `G` passes through `t`.

These properties will be crucial as we build up our correctness argument piece-by-piece.

### Sub Idea 2.1: A Cilk program may be represented as a series parallel dag

Consider a Cilk procedure structure like that below:
```
e; spawn F;e; spawn F; e;....spawn F;e; sync;
e; spawn F;e; spawn F; e;....spawn F;e; sync; 
...
e; spawn F;e; spawn F; e;....spawn F;e; sync; 
e;return;
```

- The procedure is composed of Cilk sync blocks, which in turn are threads interleaved with calls to spawned procedures. 
- Each sync block terminates with a sync statement and the procedure terminates with a return statement.
- We can represent the procedure through a ["parallel control flow dag"](paper_blogpost.md#some-terms-and-concepts). 

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images/cfdag.png" width="1500" height="250">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 4: parallel control flow dags</p>
</div>
<br>

- Figure 4 is a parallel control flow dag of the of the earlier mentioned procedure. This concept was also discussed [in the previous blog post](paper_blogpost.md#some-terms-and-concepts)
- Each thread e having no spawn and sync statements within it is a [base sp dag](#series-parallel-dags) (single line from source to sync).
- The sync blocks are composed in a series manner, with each sync block being a parallel composition of threads e and procedures F.
- The spawned subprocedures F are themselves sp dags.

### Sub Idea 2.2: A series-parallel dag may be represented as a binary parse tree

- The SP bags algorithm is a serial algorithm, meaning that it runs any multithreaded Cilk program depthwise on a single processor. 
- For that __we need a tree data structure that "tags" and places serial and parallel constructs of the program in such a way the nodes of the tree are visited in the same order that the threads of computation are visited in the serial execution of a Cilk program__.
- Basically we need a __one-to-one mapping__ between a tree-walk and a serial execution of all the threads that make up a serial execution of a Cilk program.
- For that we need to convert our sp dags to a binary parse tree representation.
- A binary parse tree representing the program in this section is given below. It also thereby represents Fig. 4.

<div align="center" style="text-align: center;">
    <img src="/Blogposts/images/parseTree.png" width="700" height="450">
</div>
<div align="center" style="text-align: center;">
    <p>Figure 5: Binary Parse Tree of Fig.4</p>
</div>
<br>

- The parse tree is a recursive structure, here representing SP dag of figure 4 that holds the same code as shown in this subsection.
- Each dark shaded region represents a single sync block.
- The sync blocks are connected via a spine of `S` nodes that represent serial in-order precedence. Sync blocks are composed serially because all the threads within a sync block have to be complete their execution before the next sync block can begin execution.
- - Children of `S` nodes imply series composition of their respective sp subdags. Children of `P` nodes imply parallel composition of their respective sp subdags.
- Each node is a single thread as represented in Fig.4

An in-order depth-first tree walk of such a binary parse tree corresponds to the serial depth-first execution of a Cilk program on a single processor. Please see Theorem 3 of the [original paper](http://supertech.csail.mit.edu/papers/spbags.pdf) for references.

#### An Exercise For the Reader

Write a simple parallel procedure with atleast two spawned subprocedures. Follow the thread labelling convention as given in [Fig.1 of previous blogpost](paper_blogpost.md#some-terms-and-concepts). Now convert the procedure to a parallel control flow dag as well as a binary parse tree composed of `S` nodes and `P` nodes. Do a dry run of your program and make sure it corresponds to the in-order tree walk of your parse tree.

## Main Idea #3: LCA's, Parse Trees and Parallel Execution

We have all the necessary concepts now to analyze what looking at the LCA of two nodes within our parse trees can tell us about the nature of their execution (in series or in parallel?)

#### Q. If the LCA of two nodes e1 and e2 within our parse tree is a P node? Then would the threads e1 and e2 in the program be in series or in parallel? Do they have to be in series? Do they have to be in parallel?

Ans. 
- Say the LCA is a `P` node and the threads are in series in the Cilk program, from [sub-idea 2.1](proof_blogpost.md#sub-idea-21-a-cilk-program-may-be-represented-as-a-series-parallel-dag) we can convert it to a series parallel dag. Let `G1` be sp dag of `e1`, `G2` be the sp dag of `e2`
- Now since the LCA of `e1` and `e2` is a P node, we know this implies parallel composition. Therefore the [source and sink of G1 and G2 are the same](proof_blogpost.md#main-idea-2-thread-notation--series-parallel-dags).
- But it is also given that the threads `e1` and `e2` are scheduled to run in series, which means that in actual execution the execution path goes from [sink of `G1` and then through source of G2](proof_blogpost.md#main-idea-2-thread-notation--series-parallel-dags). Since their source and sink are the same, this would imply a cycle, which would contradict the fact the sp dag is a dag.
- Let us understand with another scenario....
- Say the LCA is a `S` node and `e1` || `e2` in the Cilk program, from [sub-idea 2.1](proof_blogpost.md#sub-idea-21-a-cilk-program-may-be-represented-as-a-series-parallel-dag) we can convert it to a series parallel dag. Let `G1` be sp dag of `e1`, `G2` be the sp dag of `e2`.
- We know from properties of SP dags that there exists a path in `G1` from `e1` to sink of `G1`, and that there exists a path in `G2` from its source to `e2`. Since their LCA is an `S` node, this implies series composition, meaning the sink of `G1` is the source of `G2`, which implies that the threads in the actual program have the relation `e1` < `e2`, which contradicts  `e1` || `e2`.

Through such an exercise we can conclude the following:

- __For two threads `e1` and `e2` in a Cilk dag, they are scheduled to run in parallel if and only if their LCA is a P node.__
- __For two threads `e1` and `e2` in a Cilk dag, they are legally scheduled to run in series if and only if their LCA is an S node. `e1` < `e2` if and only if `e1` is the left subchild of the LCA and `e2` is the right subchild of the LCA.__

This main idea is presented by Lemma 4 and Corollary 5 in the [original paper](http://supertech.csail.mit.edu/papers/spbags.pdf).

## Main Idea #4: How Cilk Threads Relate to Each Other

So far we have taken user written code and transformed it to two representations, a parallel control flow dag which may be seen a being composed of threads in the program, and a binary parse tree with S and P nodes which can be seen as being composed of sp dags, which in turn are composed of threads.<br>
We then proceeded to establish some ground truths as to how threads of a procedure relate to nodes in the parse tree. We shall in this section further explore relations among threads which will then be utilised by the SP bags algorithm in correctly determining determinacy races. 

#### Q. Let there be three threads in a serial execution of a Cilk program named e1, e2 and e3. if e1 < e2 and e1 || e3, can e2 < e3?

Ans. No. Why? Suppose `e2` < `e3`. Since we know `e1` < `e3`, transitivity would tell us that `e1` < `e3`, contradicting our initial condition that `e1` || `e3`.

#### Q. Let there be three threads in a serial execution of a Cilk program named e1, e2 and e3. if e1 || e2 and e2 || e3, can e1 < e3?

Ans. Let's think through this. The LCA of `e1` and `e2` (say `a`) would defenitely be a `P` node, and the LCA of `e2` and `e3` (say `b`) would also be a `P` node. Since the 3 threads execute serially depth-wise on a single processor, we can see that the LCA of `e1` and `e3` would be either `a` or `b`, both of which are `P` nodes, hence are scheduled to run in parallel.

## Main Idea #5: SP Nodes and SP Bags

Let us revisit the `S` bags and `P` bags that are used by the SP Bags algorithm to actually determine whether or not a determinacy race occurs. While the algorithm executes, the algorithm maintaines two bags [(S and P)](paper_blogpost.md#bags) for each procedure on the call stack.<br>
In this section we will see how the authors of the [original paper](http://supertech.csail.mit.edu/papers/spbags.pdf) derived a relation between the LCA of nodes in a parse trees, and what th nature of the LCA can tell us about whether the procedures corresponding to the nodes are in a `S` bag or a `P` bag. This section is a where all our previous main ideas are tied together. The section immediately following this section will be presenting the final correctness argument.

### Mapping Nodes to Actual Procedures

The serial execution of the program as done by the SP bags algorithm does a tree walk of the parse tree. This alone does not convey the sufficient information it needs to determine race conditions. The algorithm checks for races by determining whether the UID of a procedure is in a `S` bag or a `P` bag.<br>
Currently it has only parse tree nodes to work with. Hence it needs two "mechanisms" to read in nodes and:

1. Map the nodes to the procedures in the Cilk program.
2. What the nature of a node (`S` or `P`) can tell us about how it relates to an `S` bag or a `P` bag

### Sub Idea 5.1: Procedurification Function

- The procedurification function is a mapping h that maps nodes of a parse tree to the actual procedures to which they belong in the Cilk program.
- It takes as input a node (`S`, `P` or a leaf node) and outputs the UID of the procedure for which the node exists on the parse tree.
- h :: Node -> UID (__This notation is not in the paper and is used by us to convey what the mapping does.__)


### Lemma #8 from the paper (paraphrased)

 If two threads (leaf nodes on parse trees) `e1` and `e2` exist such that:
- during serial execution of the tree `e1` executes before `e2`
- `a` = LCA(`e1`, `e2`), and
- `h` is the procedurification function mapping the nodes to the UID's

then,

1. if `a` is a `S` node, then `h(e1)` belongs to `S` bag of `h(a)` when `e2` executes.
2. if `a` is a `P` node, then `h(e1)` belongs to `P` bag of `h(a)` when `e2` executes.

#### Proof (paraphrased)

We will begin by proving `1.` first.

1. Take a look at the [algorithm](#introduction) again along with how a [parse tree is structured](#sub-idea-22-a-series-parallel-dag-may-be-represented-as-a-binary-parse-tree)
   - If `a` lies on the spine of the parse tree, then [e1 belongs to the left subchild of a and e2 belongs to the right subchild](#main-idea-3-lcas-parse-trees-and-parallel-execution). Since instructions of `e1` logically in series with `a`, when `e2` executes the child procedure that contains `e1` has already synced and hence the UID `h(e1)` is already in the `S` bag of `h(a)`.
   - If `a` is not on the spine and belongs to one of `h(a)`'s sync blocks, then `e1` is a left child of `a`. This means `a` and `e1` are nodes mapping to the same procedure and `h(a)` = `h(e1)`. From the algorithm we can know that `h(a)` places itself in its own `S` bag right when it is spawned.

We will now go on to prove `2.`

2. Now is `a` is a `P` node in the parse tree, it cannot (by construction) be on the spine. Hence `a` would belong to a sync block and `e1` would belong to the left subtree of `a` and `e2` would belong to the right subtree. When `h(e1)` returns to `h(a)`, `h(e1)` is placed in `P` bag of `h(a)`, as per the algorithm. Since `e2` is the right subchild of `a`, it has to execute before the sync statement executes. Therefore when `e2` executes `h(e1)` would belong to the `P` bag of `h(a)`.


### Corollary #9 from the paper (paraphrased)

If two threads (leaf nodes on parse trees) `e1` and `e2` exist such that:
- during serial execution of the tree `e1` executes before `e2`
- `a` = LCA(`e1`, `e2`), and
- `h` is the procedurification function mapping the nodes to the UID's

then,

1. `e1` < `e2` if and only if `h(e1)` belongs to a `S` bag when `e2` is executed.
2. `e1` || `e2` if and only if `h(e1)` belongs to a `P` bag when `e2` is executed.

#### Proof (paraphrased)

Let us prove `1.` first

- If `e1` < `e2` then we know that [LCA(`e1`, `e2`) is an S node with e1 as left child and e2 as right child](#main-idea-3-lcas-parse-trees-and-parallel-execution). And since that is the case then we know from [lemma 8](#lemma-8-from-the-paper-paraphrased) that `h(e1)` has to belong to the `S` bag of `h(LCA(`e1`, `e2`))` when `e2` executes.

Now we prove `2.`

- If `e1` || `e2` then we know that [LCA(`e1`, `e2`) is a P node with e1 as left child and e2 as right child](#main-idea-3-lcas-parse-trees-and-parallel-execution). And since that is the case then we know from [lemma 8](#lemma-8-from-the-paper-paraphrased) that `h(e1)` has to belong to the `P` bag of `h(LCA(`e1`, `e2`))` when `e2` executes.


## Proving the Correctness

Since we have setup all the necessary constructs and established all the relations between those constructs in the previous sections, our task of proving the correctness here is quite simplified.

Consider the same serial depthwise execution of a Cilk procedure. There are threads `e1` and `e2` in the program such that `e1` executes before `e2`. Now when `e2` executes it accesses a memory location `l`, last accessed by `e1`.

1. If `e2` performs a write, then a race could occur when:
   1. `e1` previously wrote to `l` and currently is in a `P` bag. This would imply parallel writes when executed in a multiprocessor environment to a common memory location, __resulting in a race__.
   2. `e1` previously read from `l` and currently is in a P bag. This would imply parallel write-read when executed in a multiprocessor environment to a common memory location, __resulting in a race__.
2. IF `e2` performs a read from location `l` and `e1` has previously written to `l`, and if `h(e1)` is in a P bag this would imply a parallel read-write when executed in a multiprocessor environment, leading to a determinacy race.

## Conclusion

We built the correctness argument piece-by-piece using necessary constructs along the way to understand how executing a Cilk program in a __serial , depthwise order on a single processor can catch all determinacy races when the same program executes in a multiprocessor environment__. The original paper also proves how if there are multiple determinacy races on a memory location, the SP bags algorithm catches the first such occurence. This part of the argument has not been discussed in this blogpost for the sake of maintaining conciseness. The curious reader make take a look at it in the original paper.