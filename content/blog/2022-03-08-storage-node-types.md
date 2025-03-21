+++
title = "Storage Requirements and Node Types in the Neptune Protocol"

[extra]
author = "Alan Szepieniec"
+++

# Storage Requirements and Node Types in the Neptune Protocol

The [Neptune White Paper](/whitepaper) makes, among many others, two statements of interest:

 - SNARKs enable fast synchronization.
 - Full nodes have a constant memory footprint.

This note qualifies these claims.

## Introduction

Neptune uses *recursive block validation* as an integral part of the mining process. What this means is that every block contains a STARK proof that the previous block was correct. New entrants to the network need only verify that the proof in the most recent block is correct, and can avoid running through all historical blockchain data to synchronize. Variations to this theme are variously called [Proof of Necessary Work (PoNW)](https://eprint.iacr.org/2020/190.pdf), [Non-Interacive Proof of Proof-of-Work (NIPoPoW)](https://nipopows.com/), and  [Succinct Non-ineractive Argument of Chain Knowledge (SNACK)](https://eprint.iacr.org/2022/240.pdf). The layer-1 blockchain project called [Mina](https://minaprotocol.com/) captures the same principle in the phrase "constant-size blockchain".

Every Neptune block contains a short cryptographic commitment to the UTXO set, called the *Merkle Mountain Range Accumulator (MMRA)*. To verify that the inputs of a transaction really are spendable UTXOs, the node verifies the membership of the input UTXOs in this data structure. This verification is relatively straightforward; it boils down to verifying one Merkle authentication path. The node does not need to verify the MMRA because *its* validity is established by the block's correctness proof. Therefore, the newcomer who trusts no-one except the genesis block and who wants to participate in the network needs to download and verify only one block -- *independently of the age or popularity of the network*. However, this claim is not quite as categorical as it seems, as it relies on hidden assumptions about the bounded depth of chain reorganizations. These assumptions are better illustrated in relation to nodes who don't just want to verify transactions but who want to initiate them also -- but without storing the entire chain history.

In order to initiate a transaction, the user needs to supply the membership proofs of the transaction's input UTXOs. These membership proofs change with each block. Therefore, nodes that own assets and might want to spend them have two options:

 1. They remain online and monitor in real time how their UTXOs' membership proofs evolve as new blocks are mined.
 2. They outsource this monitoring to other parties (who may or may not be remunerated for their service), who supply this up-to-date information when queried for it.

We use the following terminology for the various types of nodes:

 - *Spectator nodes* do not own UTXOs and serve only to answer questions about the status and validity of transactions initiated by other nodes, in addition to generic questions about the state of the blockchain.
 - *Light nodes* store only the indices of the UTXOs they own. They can go offline because they outsource the calculation of their membership proofs to support nodes.
 - *Full nodes* are nodes that stay online and monitor in real time how their UTXOs' membership proofs evolve.
 - *Support nodes* stay online and monitor in real time how the membership proofs of UTXOs they do not own, evolve.
 - *Archival nodes* stay online and store the entire history of the blockchain, and update the membership proofs of all UTXOs.

These categories are not mutually exclusive. Nothing prevents a node from managing both their own UTXOs and those of clients, or all UTXOs for that matter.

Another, rather more continuous, dimension for classifying nodes is by their storage requirements. In this regard, the distinguishing feature of full nodes and support nodes on the one hand, and archival nodes on the other, is that the former saves on storage cost by pruning obsolete data. How aggressively to judge data as obsolete (in order to prune it) is the subject of a trade-off. Historical blockchain data *might* be needed to resynchronize in the event of a blockchain reorganization. Nodes trade off reduced storage requirements against increased intolerance for reorganizations. Archival nodes merely occupy the extremal point of this spectrum.

![](node-types-plot.svg)

To illustrate, let's explore how these nodes behave in the event of a reorganization.

# Reorganizations

A reorganization occurs when a node is presented with a new chain of blocks, whose last few blocks are different, and that has a greater amount of accumulated proof-of-work. The protocol stipulates that the new chain is canonical, and that the old one must be discarded. This stipulation is known as the *longest chain rule*, even though it is not the chain's length that counts but the amount of accumulated proof-of-work.

Regardless of the blockchain protocol, a node that is presented with a new chain tip needs to decide two things: 1) is it valid?; and 2) is it canonical? In Neptune, the block validity is guaranteed by the validity proof; no further validation is needed. Furthermore, Neptune blocks come with fields counting the amount of accumulated proof-of-work; a comparison of this field in two or more candidate blocks determines which is canonical. This fork choice logic is much less complex than in Bitcoin, where nodes must validate the entire sequence of blocks (along with their transactions) from the point of divergence to the tip, and then count the amount of accumulated proof-of-work manually. Regardless, reorganizations in Bitcoin tend to be rare and relatively shallow and so the cost of deciding which fork is canonical, is negligible.

What's not negligible is the cost of recalculating an up-to-date UTXO set. This workload is already significant for Bitcoin but on Neptune it is compounded by the accumulator data structure: not only does the UTXO set change but so too do the commitment and the membership proofs. This task is obviously not relevant for spectator nodes and light nodes: they can just switch to the new tip after validating the block.

![](reorganization-light.svg)

The task of recomputing the UTXO set (as well as the commitment and the membership proofs) is relatively straightforward for archival nodes. Since the archival node stores all history, it can simply revert to the state of the UTXO set at the block where the histories start to diverge. At this point, the archival node synchronizes with the longest chain by updating his UTXO set (and commitment and membership proofs) accordingly.

![](reorganization-archival.svg)

For full nodes and support nodes on Neptune, this task is rather more complicated. They have to undo updates that were induced by now-obsolete blocks, before they can apply the updates induced by the now-canonical chain. However, in order to walk back changes and to determine how far to walk them back, nodes need to store historical blocks. They can get away with storing, say, the 10 most recent blocks -- but only if they assume that no reorganization will be deeper than 10 blocks.

![](reorganization-full-success.svg)

If a deeper reorganization does occur, full and support nodes will be incapable of rolling back the changes far enough, let alone figuring out how far "far enough" is. In this event, full and support nodes rely on the information of archival nodes to resynchronize, or at least on the information of nodes whose block coverage is deeper than theirs.

![](reorganization-full-failure.svg)

Only archival nodes can tolerate arbitrarily deep reorganizations. Full nodes and support nodes that reduce their historical block coverage to a handful of most recent blocks do so at the risk of being unable to tolerate reorganizations deeper than that. It may be a viable strategy, considering that deep reorganizations are not only unlikely but costly as well. However, the point is that *there is no such thing as obsolete blockchain information* in the same sense that *there is no such thing as a fully confirmed transaction*. Full nodes and support nodes trade off reducing the storage cost of historical blockchain information against increasing the likelihood of failing to independently resynchronize after reorganization.

# Limiting Reorganization Depth

What if we stipulate that nodes only allow reorganizations that are at most, say, 100 blocks deep? If we do that, then full nodes and support nodes that store the most recent 100 blocks are guaranteed to be capable of handling any reorganization that can occur. Conversely, in this world a fully confirmed transaction does exist.

The reason why limiting reorganization depth is a bad idea is because it allows forks to exist that do not have a mutual resolution policy. An attacker can fork the network more than 100 blocks deep. The resulting blockchain is just as internally consistent as the "authentic" one. Furthermore, it is impossible to define objectively what "authentic" even means. Distinct nodes comparing the two views of history to determine which is canonical, cannot reach consensus. In both cases, the proposed protocol stipulates that the alternative view be ignored. Therefore, the question which blockchain is canonical, has an answer that is relative to who is asking the question. In other words, *canon has become subjective*.

Suppose you buy a house and the payment is confirmed on your blockchain but never arrived on the seller's. The court that decides who the owner of the house is, implicitly decides which blockchain is canonical. The whole point of money is to settle debts *objectively* -- without resorting to a court decision.

