+++
title = "Mutator Sets"

[extra]
author = "Alan Szepieniec"
ogdescription = "Mutator sets are a new cryptographic authenticated data structure facilitating scalable privacy."
ogimage = "mutator-set-midjourney-medium.png"
ogtype = "article"
+++

# Mutator Sets

A mutator set is a cryptographically authenticated data structure used to effect privacy in a scalable way. For a comprehensive presentation, have a look at the [paper](https://eprint.iacr.org/2023/1208.pdf) which is online as of today. This article is a less formal overview of mutator sets.


| ![A mutator set using MMRs](mutator-set-midjourney-medium.png) |
|:--:|
|     **Fig. 1: a mutator set, as imagined by Midjourney.**      |

## The Problem: Scalable Privacy

The [problem of scalable privacy](../scalable-privacy-problem) consists of marrying light clients with transaction graph obfuscation. Light clients shrink the set of unspent transaction outputs (UTXO set) down to a short cryptographic commitment and verify succinct updates to this commitment, but at the expense of having to produce and maintain membership proofs for UTXOs they themselves own – and continuously synchronize them as the UTXO set is updated. Existing architectures for effecting privacy on-chain impose large workloads on their users and ultimately undermine the point of having light clients to begin with.

To make this claim concrete, let's survey these privacy architectures and analyze their deficiencies. Privacy is a multi-faceted and varied feature and defies projection onto one dimension for the purpose of quantitative comparison. Nevertheless, we give it a try here – but note that a lot of nuance is lost.

All privacy architectures build on top of confidential transactions (or confidential assets). These keep the amounts (or states) of the UTXOs hidden behind commitments, while supplementing the transactions with a zero-knowledge proof that establishes that the amount logics (or state update rules) are satisfied.

 1. Standard UTXO-based ledgers, such as used in [Bitcoin](https://bitcoin.org/en/). These are the baseline. There is no privacy but on the upside it takes no work to keep a UTXO up to date. The verifier needs to keep track of the UTXO set in order to verify non-membership; this task takes $O(U)$ work where $U$ is the size of the UTXO set.
 1. Basic decoy-and-nullifier scheme, as used by [ZCash](https://z.cash/) and [Monero](https://www.getmonero.org/). In this scheme, every transaction defines a list of true origin UTXOs along with plausible decoys, and every spent UTXO defines a nullifier that is added to a sorted nullifier set. A double-spending attempt will be caught because it results in a collision of nullifiers. The disadvantage of this approach is that the nullifier grows with the age and popularity of the protocol and cannot be pruned or culled. The verifier of a transaction has to store this set and test non-membership in it; the workload of this task is $O(N)$, where $N$ is the size of the nullifier set.
 2. Decoy-and-nullifier with [accumulators](https://en.wikipedia.org/wiki/Accumulator_(cryptography)). This construction shrinks the nullifier set to a short cryptographic commitment under some set commitment scheme that admits updates and non-membership proofs. The verifier needs to store only the commitment to this set, but the workload shifts to the transaction initiator who now has to produce a zero-knowledge proof relative to the whole thing. The verifier has an easy workload of $O(\log N)$ but the initiator has a workload of $O(N)$ (not counting the overhead of a superlinear prover).
 3. Decoy-and-nullifier with snapshots. In this construction, every UTXO that is generated takes a snapshot of the nullifier set at the time of its birth. When it is spent, the zero-knowledge proof establishes that the (pseudorandom) nullifier was not added to the nullifier set in the meantime. In particular, the history before the UTXO was generated, can be ignored. However, the zero-knowledge proof still needs to integrate knowledge of all nullifiers that are announced after in the UTXO's lifetime. Let $L < N$ be the number of UTXOs generated after the UTXO in question; then the transaction initiator's workload scales with $O(L)$ instead of $O(N)$.
 4. Coinjoin and cut-through. [Mimblewimble](https://scalingbitcoin.org/papers/mimblewimble.txt)-style transactions can be merged in such a way as to drop all trace of the boundaries separating the original transactions. In the extremal case, every block contains thousands of inputs and thousands of outputs but no information about how to bundle them into original transactions. The mergers happen before the block is mined but after the transaction is sent over the network. Transaction initiators either broadcast the transaction, in which case it can be merged with other transactions but the passive network observer will learn the original transaction before that happens; or communicate the transactions selectively to authorities entrusted with merging. The problem with the latter strategy is that the preferred merger-authority is the one with the larger pool of other transactions as this pool is the anonymity set. The merger authorities have an incentive to cooperate and in the limit there is no distinction left between selective communication and broadcasting. In short, there is no on-chain obfuscation of the transaction graph; and the obfuscation that occurs before the data hits the chain, is self-defeating. On the plus side though, the work of the transaction initiator and its verifier is constant independent of $N$ or $L$, or even the UTXO set size $U$.
 5. [Mixing](https://en.wikipedia.org/wiki/Mix_network). UTXOs are hidden behind *publicly rerandomizable* commitments. At given points in time, a mixer takes a subset of UTXOs and shuffles them and rerandomizes their commitments, outputting a new list along with a zero-knowledge proof of correct mixing. The downside is that the owner of a UTXO has to track his UTXO across every mix it is involved in. At this point, we need to distinguish between two cases:
   - **a)** UTXOs can be mixed any number of times. The probability that the user's UTXO is selected for the next mix is proportional to the reciprocal of the UTXO set size $U$, and the batch size $b$ for the mix. The user's workload in case his UTXO is involved in a mix, is to try and unlock all the outputs until he succeeds; this takes $O(b)$ attempts. Lastly, every newly announced UTXO generates a new mix, and there are $L$ of them. All together, the workload of the UTXO owner consists of tracking the UTXO across mixes and scales with $O\left(\frac{Lb^2}{U}\right)$.
   - **b)** UTXOs can be mixed only once. The UTXO is mixed with other UTXOs generated at roughly the same time, as part of the confirmation process. In order to recover his UTXO, the user needs to try to unlock all the UTXOs in the batch until he is successful. As UTXOs can only be involved in one mix, in order to achieve the same anonymity[^anonymity], the batch size has to be larger compared to when the UTXOs can be involved in more than one. Specifically, for the single mix architecture, the batch size must be $B = b^{1+\frac{Lb}{U}} > b$, in order to have the same anonymity as the multiple mix architecture with batch size $b$.
 - In either case, an additional downside comes from the fact that the mixing authority must be trusted not to leak the secret permutation.

The next table shows how the surveyed architectures compare against mutator sets in terms of the relevant dimensions. Note that asymptotically speaking, $N > L > U > B > b$, and in particular $L > U$ assumes realistically that the set of transaction outputs that remain unspent grows slower than the set of new transaction outputs, precisely because a significant portion of them are spent. The dimensions are

 - (network) scalability, where we consider the workloads of the verifier and the initiator of transactions;
 - privacy, where we consider the confidentiality of UTXO amounts (or states), and the obfuscation of the transaction graph;
 - trust assumptions, added because if they are broken they ultimately undermine whatever feature is otherwise claimed.

We are not terribly concerned with the size of the anonymity set, as long as it is non-trivial. The reason is that transactions are linked to other transactions, and so anonymity sets naturally compound exponentially. Moreover, supporting large anonymity sets at the expense of scalability is purpose-defeating as the resulting network will have fewer users.

|                                                    | verifier workload | transaction initiator workload       | confidential transactions / assets | transaction graph obfuscation | trust assumptions |
|:---------------------------------------------------|:------------------|:-------------------------------------|:-----------------------------------|:------------------------------|:------------------|
| standard UTXO-based ledger                         | $O(U)$            | $0$                                  | ✗                                  | ✗                             | ✗                 |
| basic decoy-and-nullifier                          | $O(N)$            | $O(1)$                               | ✓                                  | ✓                             | ✗                 |
| decoy-and-nullifier with accumulators              | $O(\log N)$       | $O(N)$                               | ✓                                  | ✓                             | ✗                 |
| decoy-and-nullifier with snapshots                 | $O(\log N)$       | $O(L)$                               | ✓                                  | ✓                             | ✗                 |
| coinjoin and cut-through (broadcast)               | $O(U)$            | $O(1)$                               | ✓                                  | ✗                             | ✗                 |
| coinjoin and cut-through (selective communication) | $O(U)$            | $O(1)$                               | ✓                                  | ✓                             | ✓                 |
| mixing (multiple mixes)                            | $O(U)$            | $O\left(\frac{Lb^2}{U}\right)$       | ✓                                  | ✓                             | ✓                 |
| mixing (single mix)                                | $O(U)$            | $O(B)$                               | ✓                                  | ✓                             | ✓                 |
| **mutator set**                                    | **$O(\log N)$**   | **$O((U + \log L) \cdot (\log N))$** | **✓**                              | **✓**                         | **✗**             |

## The Solution: Mutator Sets

Mutator sets solve the shortcomings of prior privacy architectures. Depending on your perspective, a mutator set is a) a succinct decoy-and-nullifier construction; b) an accumulator scheme with privacy; or c) an operator-free mixnet.

The name originates from perspective (b). The operation that adds an item to the set cannot be linked to the operation that removes it. However, both units of information that induce the change – the *addition record* and the *removal record* – are cryptographic commitments to the same data item. Between being added and removed, the item's representation has mutated.

Here's how they work. A mutator set decomposes into two separated but linked data structures, the *append-only commitment list (AOCL)*, and the *sliding-window Bloom filter (SWBF)*. Both are compactified using [MMRs](https://neptune.cash/learn/mmr/), although a portion of the SWBF called the *active window* is represented without compaction. The accumulator value (black) is given by the two lists of Merkle roots, along with the active window.

The AOCL keeps track of which items are added by storing their addition records in an MMR. An element of the AOCL (an addition record, red) is a commitment to an item $t$ of the mutator set of the form $\mathsf{H}(t \Vert r)$, where $r$ is randomness ensuring that the commitment is [hiding](https://en.wikipedia.org/w/index.php?title=Commitment_scheme#Perfect,_statistical,_and_computational_hiding). When an item is added to the mutator set, its addition record is added to the AOCL MMR.

The SWBF keeps track of which items are removed from the mutator set by setting bits in the Bloom filter. The indices of these set bits are determined pseudorandomly from the item $t$, its index of insertion $l$ into the AOCL, and its randomness $r$. The window in which bits can be set slides periodically. The inactive chunks of the SWBF are stored in an MMR whereas the active window is represented explicitly. A removal record (blue) consists of:

 - an item's index set;
 - for all indices that live in the inactive part of the Bloom filter, their MMR membership proofs; and
 - a zero-knowledge proof proving that the bit indices were derived correctly.

The zero-knowledge proof establishes that the announced indices are derived correctly from some item that was already added to the mutator set. Specifically: the bits were determined by hashing $t, l, r$; where the addition record $\mathsf{H}(t \Vert r)$ has a valid membership proof in the AOCL MMR. The information involved in this proof is shown in light blue.

| ![A mutator set using MMRs](mutator-set.svg) |
|:--:|
| **Fig. 2: Construction of a mutator set using MMRs.** |

The malicious user who attempts to remove an item twice gets caught because the second time all the bits in the SWBF are already set. The malicious user who attempts to remove an item that was never added to the mutator set cannot successfully produce the zero-knowledge proof. The malicious observer who wants to link removals to additions is blocked by the hash functions and the zero-knowledge proof, which hide the information that would enable him to draw the link.

Moreover, the parameters of the Bloom filter are such that while it is *possible* to find removal records for distinct items but with colliding index sets, the probability of this event happening at random is cryptographically small. And as the recipient contributes secret randomness to the addition record before the indices are determined, no attacker can trigger this event by grinding.

## Mutator Sets and Scalable Privacy

At this point, the use of mutator sets for anonymous cryptocurrencies suggests itself. Just replace the UTXO set with the mutator set. The blockchain tip defines only the accumulator value. Every block has a list of transactions. These transactions are valid if (in addition to standard transaction validity requirements like no inflation) all the inputs are valid removal records. The transactions' outputs are addition records, and the mutator set is updated with them in every block.

| ![Transparent ledger fragment](transparent-ledger.svg) |
|:--:|
| **Fig. 3: Fragment of a transparent ledger.** |

| ![Opage ledger fragment](opaque-ledger.svg) |
|:--:|
| **Fig. 4: Fragment of a ledger that uses a mutator set for privacy.** |

Privacy comes from the fact that the addition and removal records are *hiding* commitments and from their unlinkability. But where does scalability come from?

The user who wants to keep his UTXOs' membership proofs synchronized has to take into account three types of events.

 1. A UTXO that is older than (or close enough in age to) the user's own is being spent. This expenditure sets bits in the SWBF and so the user must update his own SWBF membership proofs to account for this modification. Since the indices of the set bits of the removed item live in a part of the SWBF that is older than the user's own set bits, the MMR membership proofs are guaranteed to require update. The number of older UTXOs that are spent over the lifetime of the user's UTXO can scale in the worst case with the UTXO set size, $U$. The workload required for each such update is linear in the height of the MMR, which itself is logarithmic in the number of UTXOs. So the total workload for accomodating this type of event is $O(U \cdot \log N)$.
 2. A new UTXO is generated. A new addition record is added to the AOCL. The number of times this can happen is $L$, but in all but a logarithmic number of cases no update is required because only smaller Merkle trees are affected and not the one that contains the user's addition record. If an update is required, the workload is linear in the height of said Merkle tree, which itself is logarithmic in the total number of addition records, $N$. So the total workload for accomodating this type of event is $O((\log L) \cdot (\log N))$.
 3. A UTXO that is younger (with a sufficiently large age gap) than the user's own is being spent. This can happen at most $L$ times, and this expenditure does set bits in the SWBF, but once again in all but $\log L$ many cases no work from the user is required as only smaller trees are affected. So the total workload for accomodating this type of event is the same for the previous item, $O((\log L) \cdot (\log N))$.

All together, the workload of the transaction initator is $O((U + \log L) \cdot (\log N))$. But this formula fails to capture some nuance, which is that the term $U$ holds only in the worst case. In practice, most users spend their UTXOs long before a meaningful fraction of the UTXO set had a chance of being spent. So an average case complexity of $O((\log L) \cdot (\log N))$ paints a better picture.

As for the workload of the verifier, his task is merely to verify a constant number of MMR membership proofs, the zero-knowledge proof, and some trivial index arithmetic. The complexity of this task is dominated by the length of the authentication paths, which is logarithmic in the total number of transaction outputs: $O(\log N)$.

[^anonymity]: One way to quantify anonymity is to cast the adversary's knowledge as a probability distribution and then measure the information entropy of this distribution.
