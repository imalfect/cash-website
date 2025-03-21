+++
title = "Authentication Structure Authentication"
date = "2024-10-26"
weight = 0

[extra]
author = "Alan Szepieniec"
+++

# Fast Authentication Structure Authentication

**TLDR.** Non-deterministic algorithms can efficiently verify two crucial
aspects of Merkle trees:

 1. That a list of leafs belongs to a specific Merkle tree.
 2. That the provided authentication structure for these leafs is correct and
    complete.

This note presents and motivates a non-deterministic algorithm for the second
task.

## Introduction

Here is a nice little computer science challenge. It is probably a bad idea to
use this for a job interview question because a) the solution follows below, and
b) you might be setting the bar too high. Nevertheless, it's an excellent brain
teaser if you are into that sort of thing.

The task is as follows. You are given a Merkle root, a list of leafs[^1] along
with indices, and an authentication structure that establishes membership of the
leafs in the Merkle tree[^2]. The authentication structure is minimal: it does not
repeat information and nor does it contain information that can be computed. See
the diagram below for a non-trivial example of an authentication structure.

|  ![🌲](non-trivial-authentication-structure.svg)    |
| :-------------------------------------------------: |
| **Fig.1.**: A non-trivial authentication structure. |

The task is to *authenticate the authentication structure*. Specifically: to
decide whether the given authentication structure really is an authentication
structure for the given leafs in the Merkle tree determined by the given root.
This task is to be solved in the model of computation where
 - computing a hash has a unit cost – equivalent to one cycle or one arithmetic
   operation;
 - the solver has access to *non-determinism* – that is to say, it can guess
   helper information and gets to proceed assuming the helper information is
   correct.

There is obviously the naïve deterministic algorithm to verify authentication
structures. It involves storing all known nodes with their node indices in a
mutable list. Initially this list contains all leafs and all authentication 
structure elements. Until there is only one node left, do the following
 - find a pair of sibling nodes
 - take them out of the list
 - compute the siblings' parent by hashing
 - insert the parent (with node index) into the list.
When the list has only one element left, compare it against the given root.

The disqualifying feature of this algorithm is its large complexity, which it
owes to either searching the list in every iteration, or of it is sorted, to
inserting new elements in the right position. Either strategy induces an
asymptotical $\log N$ multiplicative overhead, where $N$ is the size of the
initial list, thus making the complexity $O(N \log N)$. The task is to find an
$O(N)$ algorithm – and not just an asymptotically linear one, but one that is
concretely efficient as well.

You might ask, *if the algorithm is allowed to make guesses that are guaranteed
to be correct, why can't it just guess the answer?* To dispel any confusion,
such algorithms are also disqualified because they are not *sound*. By
maliciously forging the nondeterministic guess, the host machine can trick such
an algorithm into accepting an invalid authentication structure. The goal is to
devise an algorithm that is correct when the nondeterministic guess is correct,
but errs on the side of caution when it is not – specifically by rejecting the
input regardless of whether it was valid. Phrased differently, false negatives
are allowed; false positives must occur with at most a negligible probability.

## Real Life Motivation

Before detailing our solution, let us motivate the problem by explaining how it
arises in practice. It is not a problem of purely theoretical interest; it has
real life applications.

Every block of the blockchain protocol [Neptune Cash](https://neptune.cash/) 
confirms a transaction which consumes some UTXOs and produces new ones. Glossing
over details that are not relevant here, the way the blockchain knows that the
UTXOs that are being spent really do exist, is by verifying authentication
authentication structures. Furthermore, the authentication structures published
in one transaction may be needed to update the authentication structures of
other UTXOs, which is necessary if their owners want to spend them in a later
transaction. And so the blockchain needs to do two things: it needs to ensure
that the UTXOs that are being spent really do exist, and it needs to ensure
that the authentication structure that testifies to this fact really is correct.

It is not obvious that these two tasks are different, unless you remember that
we are working in a non-deterministic model of computation There is an efficient
algorithm for verifying that a list of leafs (with indices) belong to a Merkle
tree that involves non-deterministically guessing the authentication structure.
But this algorithm fails to satisfy the second requirement, because the
non-determinism is not being broadcast.

Why is block verification non-deterministic to begin with? It is because Neptune
uses proofs everywhere, and proof systems natively live in a non-deterministic
world. The prover is being run on a classical deterministic machine, but what
matters for gauging its complexity is the cycle count of the computation that
it establishes the integrity of. And so if the host machine must spend a few
more cycles preparing the right non-deterministic guess for the block-verifier,
that is a worthwhile trade-off if the resulting block-verifier shaves off a
factor $\log N$ in complexity.

The blockchain context helps to internalize the important distinction between
verifying Merkle membership of a list of leafs; and verifying the integrity of
their authentication structure. Consider a malicious transaction-initiator who
does the following. He proves Merkle membership of all relevant leafs, thereby
establishing that the UTXOs that are being spent really do exist. But this proof
was produced using a non-deterministically-guessed authentication structure that
is different from the one that was published as part of the transaction. As a
result, the transaction will be confirmed (since its proof is valid) but
everyone else who tries to update the authentication structures of their own
UTXOs ends up corrupting them.

## The Algorithm

The trick is to represent the list from the naïve algorithm as a univariate
polynomial whose roots correspond to
$\mathsf{E}(\mathtt{node} , \mathtt{nodeindex})$, where $\mathsf{E}$ is
some embedding function into a sufficiently large finite field. Furthermore,
this polynomial is represented by its evaluation at $\alpha$, which is a scalar
sampled uniformly from a sufficienly large field. With these representations we
can think of adding and removing elements to and from the list, as adding and
removing factors $X - \mathsf{E}(\cdot)$ from the mutable polynomial, and these
operations can be computed efficiently through multiplication or division by
$\alpha - \mathsf{E}(\cdot)$.

The initial list can be computed in linear time by iterating over all leafs and
all authentication structure elements and accumulating one factor for each node.
Then for $N$ iterations:
 - guess $\mathtt{leftnode}, \mathtt{leftindex}$ and
   $\mathtt{rightnode}, \mathtt{rightindex}$ nondeterministically;
 - divide out $\alpha - \mathsf{E}(\mathtt{leftnode} , \mathtt{leftindex})$
   and $\alpha - \mathsf{E}(\mathtt{rightnode} , \mathtt{rightindex})$
   from the polynomial;
 - compute $\mathtt{parent} = \mathsf{H}(\mathtt{leftnode} \Vert \mathtt{rightnode})$ and $\mathtt{parentindex} = \lfloor \mathtt{leftindex} / 2 \rfloor = \lfloor \mathtt{rightindex} / 2 \rfloor$;
 - multiply in $\alpha - \mathsf{E}(\mathtt{parent} , \mathtt{parentindex})$.

After $N$ iterations the only remaining factor should be
$\alpha - \mathsf{E}(\mathtt{root} , \mathtt{rootindex})$. Compare this
factor to the one obtained from the given root.

## Conclusion

There is an important difference between verifying that a list of leafs are
members of a Merkle tree, and verifying that a given authentication structure
testifies to this fact. In contexts where the blockchain is used for data
availability, both tasks are important. In a non-deterministic model of
computation, which arises in the context of proof systems, there is a
linear-time and concretely efficient algorithm for the second task.

In the context of Neptune, we decided to postpone the implementation of this
algorithm for the time being. It is not clear that the performance benefits
kick in early on in the blockchain's lifetime, relative to a more naïve
approach; and since we plan to [soft-fork succinctness in](../blog/2024-10-14-launch-scope-and-hard-forks.md) at a time after launch
anyway, that might be the opportune moment to take another look at the
complexity of this step. The main purpose of this note is to record this 
algorithm so that we do not forget about it.

What's missing from this note is of course a proof of correctness – or more
specifically, of *soundness*. We leave that exercise to the job interviewee.

[^1]: "Leafs" being the correct plural of "leaf" when used in the context of 
abstract mathematical objects called trees.

[^2]: A Merkle tree is a cryptographically authenticated data structure
organized in the form of a balanced binary tree where each node is the hash
of the concatenation of its siblings. The main benefit are compact,
cryptographically-secure membership proofs for data items. [https://en.wikipedia.org/wiki/Merkle_tree](https://en.wikipedia.org/wiki/Merkle_tree)