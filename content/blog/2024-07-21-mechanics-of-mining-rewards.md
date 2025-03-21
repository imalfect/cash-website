+++
title = "Mechanics of Mining Rewards"

[extra]
author = "Alan Szepieniec"
ogdescription = "A summary of the steps involved in obtaining mining rewards."
ogimage = "mechanics-of-mining-rewards-concept-art.jpg"
ogtype = "article"
+++

|                ![Concept art.](mechanics-of-mining-rewards-concept-art.jpg)                |
| :----------------------------------------------------------------------------------------------------------: |
| **Fig.1.**: Mechanics of mining reward distribution. Imagination by [Perchance](https://perchance.org/ai-text-to-image-generator). |

# Mechanics of Mining Rewards

The distribution of mining rewards is conceptually simple, but certain architectural
decisions in Neptune make the concrete mechanics of this process rather challenging
to specify and implement. This note articulates the problems that arise and motivates
the solutions we came up with.

## Concept

The limit on Neptune's token supply, 42 million units, is as good as set in stone[^1].
These coins come into existence through mining, which is the process of binding
proof-of-work with a selection of transactions in order to establish consensus on
their finality. Miners are rewarded for this task in two ways:

 1. The *block subsidy* is distributed to a spending address, or collection of
 spending addresses, under their control.
 2. The *transaction fees* with which transaction initiators prioritize their own
 transactions.

 Where the transactions only redistribute existing coins, the block subsidy mints
 new ones. Almost all coins come into existence through block subsidies –
 and if the term "block subsidy" covers the premine, then this
 statement is true for *all* coins.

The supply is limited because the magnitude of the subsidy is automatically
reduced in accordance with an exponential decay curve. As a result, the total
sum of all block subsidies ever is finite, and if you include the premine in that
sum, the total is exactly 42 million.

To avoid any doubt it is worth reiterating a second feature that is also as good
as set in stone. The premine consists of 1.98% of the supply limit, or 831600 coins.
They are allocated to persons who made the project possible prior to launch.

## Challenges

While the conceptual framework of Neptune's mining rewards is straightforward,
several practical challenges arise when implementing this system.

 1. Two-step mining. To produce a block, a miner must both *a)* prove its correctness,
    and *b)* solve a proof-of-work puzzle.
 2. Time-locked mining reward. Of entire block reward, *i.e.*, subsidy plus fees,
    only half is immediately liquid. The remainder is released after three years.
 3. Dynamic block intervals. If the protocol observes good network conditions,
    short proving times, and full blocks, the target block interval is automatically
    reduced.

## Two-Step Mining

Mining consists of two steps.

 1. *Proving.* The miner produces a [STARK](https://triton-vm.org/) proof of the
    correctness of the block body, ensuring that the transaction is valid and that
    the previous block (body *and* header) was also valid. Proving is a 
    labor-intensive task
    requiring tens of gigabytes of RAM and either several dozen CPU cores or a GPU. 
    It is difficult to parallelize, meaning that the prover with the faster hardware 
    has a consistent advantage in producing the proof first. The motivation for
    including the block proof is *protocol succinctness*, which is the 
    *non-reliance* on more-than-negligible amounts of historical blockchain data for 
    evaluating the fork  choice rule. Specifically, in order to decide which of two 
    contender blocks is canonical, Neptune nodes need no information besides those 
    contender blocks and the common hash of the genesis block. The blocks' 
    correctness proofs establish that they are both valid descendants of the genesis 
    block and that their advertised aggregate proof of work numbers were computed 
    correctly. Evaluating the fork choice rule involves verifying the proofs and 
    comparing these numbers. As a result, nodes can synchronize to the network in 
    milliseconds, without trust assumptions, and with the same robustness as if they 
    had verified all of history.
 2. *Preimage search.* A block header contains, among other things, random data
    often called a nonce[^2]. Consequently, the hash digest of the block is hard
    to predict without computing the hash function. The preimage search involves
    sampling this random data in the hope that the resulting block digest is smaller
    than a target difficulty. If this condition is met, then the miner has found a
    new block which, when broadcast, will likely be accepted by peers as the new
    tip of the canonical chain. Preimage search is stateless, and therefore this
    step is a [Poisson process](https://en.wikipedia.org/wiki/Poisson_point_process)
    allocating the expectation of success to searchers in proportion to their hash
    rate. It is also [embarrassingly parallelizable](https://en.wikipedia.org/wiki/Embarrassingly_parallel),
    meaning that the searcher with the cheaper electricity bill
    (whether due to energy-efficient hardware or availability of cheap electricity)
    has a marginal advantage over the searcher with the more expensive one. All
    other things equal, preimage search favors searchers with access to cheap 
    electricity. The geographical distribution of cheap power is the physical
    decentralization that underpins the virtual decentralization of the 
    cryptocurrency that utilizes it.

The challenge arises because proving is stateful and breaks the Poisson nature of
mining. In a naïve implementation, the miner with the faster hardware has
a consistent head start over the slower miner in the preimage search step, leading
to a distribution of mining power that biased away from the decentralized
distribution of cheap energy and towards something that is emphatically *not*
decentralized. Worse still, the advantage of fast provers constitutes a strong
centralizing force in the evolution of mining pools.

The physics of power consumption in processors is the relevant constraint. The power
consumed by a chip grows polynomially[^3] with the frequency. Therefore, a miner
who buys twice as much hardware running at half the frequency will consume less
power while computing the same number of hashes. In a Poisson mining environment,
this trade-off benefits the miner because his electricity cost is less and his 
reward is the same. As a consequence, miners prefer energy-efficient hardware, and
over time the distribution of mining power will tend to match the distribution of
cheap energy. In contrast, in a stateful mining environment, this trade-off makes
no sense because while the miner's power consumption is reduced, so is his
expectation of mining rewards. Consequently, for the same power budget, miners 
prefer fewer units of faster hardware over more units of energy-efficient hardware.
The optimal configuration is one chip per power source, and ultimately some power
sources are too small to be competitive while large power sources can justify
acquiring the ultra-high-end chip. This economy of scale punishes small power
sources and rewards large ones even when the price per kilowatt-hour is the same.

This deficiency is addressed by splitting the block reward into two parts. The
prover, who assembles a block and
proves its correctness but does not participate in the preimage search, chooses
which portion of the block reward to allocate to himself. The remainder of the block
reward goes to the searcher who completes the block by finding the right randomness
in the header such that the block hash is small enough. 

Fast provers can allocate to themselves a large portion of the block reward. Slow
provers can compete only by shrinking the allocated block reward. Searchers will
rationally switch to whichever block leaves the largest reward for the searcher,
and so they should constantly test announced block proposals to see if the searcher
reward is larger. The premium for fast proven-but-unworked block proposals will
tend to equal the advantage that the corresponding head start gives in the 
searching step.

This dynamic benefits from an example. Suppose we have a configuration of provers $1, 2, 3, \ldots$ with proof times
$(t_1, t_2, t_3, \ldots)$ as a fraction of the target block interval and satisfying
$t_1 < t_2 < t_3 < \ldots$. And suppose the claimed prover rewards as a fraction
of the total block reward are $(c_1, c_2, c_3, \ldots)$. Furthermore, let $(s_1, s_2, s_3, \ldots)$
be the fractions of the searching population for which it is economical to search
on top of a block proposal where respectively a fraction $(c_1, c_2, c_3, \ldots)$
of the block reward is claimed by the prover. In this environment:

 - Before block proposal 1 is broadcasted, searchers can do nothing.
 - When block proposal 1 is broadcasted, a fraction $s_1$ of the total searching
   power will switch on.
 - In general, when the $n$ th proven-but-unworked block proposal is published,
   the existing population of searchers will rationally switch to the new proposal,
   and a fraction $s_n - s_{n-1}$ will switch on.
 - The probability $P_n$ that a block is found in the time period $(t_n; t_{n+1})$ is
   approximately[^4] proportional to the total number of hashes computed in that 
   period, which is proportional to the product of the duration with the active 
   proportion of searching power: $P_n \propto (t_{n+1}-t_n) \cdot s_n$.
 - The prover's reward is proportional to this number *and* to the claimed portion,
   so $(t_{n+1}-t_n) \cdot s_n \cdot c_n$.
 - The proportion of searching power $s_n$ is a monotonically decreasing function
   of the proportion of the reward claimed $c_n$, so there is some $c_n$ which
   maximizes the miner's expectation of profit. Above this maximum, too few
   searchers will switch on. Below this maximum, the prover could have charged more.
 - As a first-order approximation and all other variables remaining equal, if prover 
   $n$ reduces his proof generation time to $t_n - \epsilon$ then he can command
   $1 + \frac{\epsilon}{t_{n+1}-t_n}$ times as much profit. Whether the increased
   profit justifies purchasing the hardware (and perhaps also software) needed to
   produce proofs that much faster depends on the cost of that capital investment.

Is there room for malicious behavior? It is difficult to analyze all possible
deviations from the protocol but the following behaviors come to mind at first 
thought.

 1. Searchers decline to switch to a proven-but-unworked block proposal whose
    searcher share is larger. In isolation, this strategy is costly. In combination
    with a colluding prover, it might make sense if the bribe from the prover
    outweighs the advantage of switching. This scenario is identical to the prover
    reducing the prover reward, and increasing the searcher reward, by the 
    equivalent amount.
 2. Provers decline to broadcast proven-but-unworked block proposals. In isolation,
    this strategy is costly. In combination with a colluding searcher (or collection
    of searchers) if the prover gets a bribe from them for keeping the proposal from 
    all other searchers. This scenario is identical to the prover increasing the
    prover reward, and decreasing the searching reward, by the equivalent amount.
 3. Provers charge out-of-market prices. The market price is precisely the price
    that optimizes for profit, so provers that follow this strategy necessarily
    incur a cost.
 4. Mining pools, wherein one centralized operator produces the proof and
    subscribers contribute searching power. This strategy is certainly rational
    for searchers looking to reduce the variance on their rewards. And since there
    is a demand for reduced variance, there is a potential profit for pool operators,
    all other things equal. Since there is no mechanism for operators to lock in
    subscribers, they are bound to either charging market prices (up to a fee for
    variance reduction) for their proofs or incurring a loss. The incentive for
    searchers to join the bigger pool must come from a loss on the part of the
    pool operator relative to just publishing his block proposal in the same time
    without administering the pool. Commanding a larger army of subscribers who 
    contribute searching power can justify the loss incurred to acquire it, but
    this dynamic has an identical counterpart in single-step mining architectures
    such as Bitcoin. It follows that two-step mining introduces no new
    centralization hazard to the mining pool environment.

A second challenge arises from the possibility that the searcher is a different party
from the prover. The block reward needs to be distributed to distinct
addresses. However, the block proof only pertains to the block body and not the
header, because searching involves modifying part of the block (the header) *without*
invalidating the proof. The prover does not know in advance who will be the winning
searcher, and so the searcher's address for receiving his block reward cannot
possibly be in the block body.

The solution is to populate the randomness in the block header with this data.
Specifically, instead of a nonce field, the block header in Neptune has a
`receiver_digest` field[^5] and a `lock_script_hash` field. The next block is responsible
for assembling coinbase UTXOs (one timelocked and one without locks) destined for the
winning searcher.

In the context of a mining pool with hundreds of subscribers contributing searching
power, these two fields are inadequate for distributing rewards to more than one
searcher. However, the mining pool operator, who assembles a block and proves its
correctness, can claim a large prover reward and distribute it in the block body
to all subscribers in proportion to their search power contribution.

## Time-Locked Mining Reward

The entire block reward is split into two parts, orthogonal to the miner-searcher
split. The first part constitutes 25% of the block reward and is liquid immediately
(or as soon as possible in the case of the searcher's reward). The remaining 75%
is time-locked for three years and becomes spendable at that point. This time lock
ensures that miners' incentives are aligned with the project's long-term success. [^6]

The prover's portion of the block reward is encapsulated as two UTXOs (one 
timelocked and one without locks), both of which are listed as explicit outputs of 
the block's transaction. (Recall that the block has only one transaction, which 
represents the merger of all transactions that the miner (and in fact: prover) 
decided to confirm.) Also listed as explicit outputs of the block's transaction
are the UTXOs associated with the previous block's searcher's reward. The block
proof establishes, among other things, that all the amounts, as well as the time 
locks, are correct.

## Dynamic Block Intervals [^7]

The protocol automatically adjusts the target difficulty so that blocks are found
at an expected rate equal to a target block interval. The protocol furthermore 
tracks the block orphan rate. A low block orphan rate indicates good network
conditions resulting in little power wasted on blocks that end up
losing a block race. When the orphan rate is low and has been for a sustained period
of time, the network can tolerate either larger blocks or more frequent ones.

Besides the block orphan rate, the protocol also tracks an estimate of the block
proving time and the fullness of blocks. If the orphan rate is low, blocks are full, 
and the proving time is small (relative to the block time), the target block 
interval is lowered. If the orphan rate is low, blocks are full, and the proving 
time is large, the block size bound is increased.

To track these signals in a way that is compatible with protocol succinctness – 
meaning that we do not have access to a large window of recent blocks – Neptune uses
[exponential moving average](https://en.wikipedia.org/wiki/Exponential_smoothing)s.
Specifically, the header contains values that are updated with every block according
to the formula $x \mapsto \alpha \cdot x + (1 - \alpha) \cdot s$, where $\alpha$ is
a smoothing parameter determining how quickly the signal-tracker adapts to changes
in the signal it tracks; and $s$ is the signal itself, which could be

 - the number of orphans observed since the last block; or
 - the ratio between the block's size and its size bound; or
 - the difference between the current block's timestamp and the previous, adjusted
   for proving time by subtracting the proving time tracker; or adjusted for
   searching time by subtracting the target block interval.

The capability for the target block interval to drop makes the question about the
block time moot; and it is also the reason why the follow-up question about the block
subsidy is impossible to give a straight answer to. The block reward must depend on
the target block interval because otherwise the issuance schedule of new coins is
broken.

Neptune defines a *base interval*, which is a fixed time interval; and a *base 
subsidy*, which is as number of coins that is reduced every epoch independently of
variable network parameters. Furthermore, a variable called the *acceleration 
factor*, which for convenience is always a power of two and initially set to 1,
determines the concrete block interval and concrete subsidy simply by dividing the
the base interval and base subsidy by it. If the protocol reduces the target block
interval, it does so by doubling the acceleration factor.

## Conclusion

This article covers three features impacting Neptune's block reward distribution.
To the best of our knowledge, Neptune is the first blockchain featuring two-step 
mining and partially time-locked block rewards. And while dynamic block times and 
sizes already exist, Neptune is the first to marry this feature with protocol 
succinctness. These novel features induce challenges for implementing the block 
reward distribution mechanism, but ultimately these challenges can be resolved
through clever architecture and engineering.

That said, present tense notwithstanding, this article is the first or maybe second
step in the architecture and engineering process. At the time of writing, these 
features and the mechanisms for achieving them have not been implemented yet. The 
good news is that you get to play a part in their realization – if you want.

**Acknowledgements.** Thanks to [danda](https://talk.neptune.cash/user/8) for fruitful discussions.

**Edits:**

 - 2025-02-14: Add footnotes to clarify difference with respect to mainnet.

[^1]: In the present context, the phrase "as good as set in stone" means
      "exceedingly unlikely to change given previous public statements and promises".
      As for for the other features alluded to in this note – take them with a grain
      of salt as they might change between now and launch, or be postponed until
      Hard Fork One.

[^2]: The choice of word is objectionable because it breaks with its origin in the
      cryptographic literature where it connotes a norm that one should not ever,
      under any circumstances, reuse the same nonce. This norm does not apply in
      the context of proof-of-work.

[^3]: The power dissipation equation $P = C V^2 f$ is of course correct but
      misleading because the frequency that you can run a chip at depends on the
      voltage.

[^4]: This approximation holds for small periods and small fractions of searching
      power, but is inaccurate in general because the probabilities of events in a disjunction cannot be added. However, *failures* to find blocks are independent
      events whose probabilities can be multiplied. Let $p$ be the probability of
      single hash trial resulting in a valid block, then $1-P_n = (1-p)^{(t_{n+1}-t_n)\cdot s_n \cdot c}$, where $c$ is the total hash power in terms of number
      of hashes per unit of time. So, the *logarithm* of the 
      probability of *not* finding a block in time period $(t_n; t_{n+1})$ is 
      proportional to the number of hashes computed in that 
      period: $\log (1-P_n) = (t_{n+1}-t_n) \cdot s_n \cdot c \cdot \ln (1-p)$. Recall
      the Taylor series of $\ln (1-x) \approx -x - \frac{x^2}{2} - \frac{x^3}{3}
      \ldots$ which converges for $x \in [-1;1)$. Ignoring higher order terms gives
      $P_n \propto (t_{n+1}-t_n) \cdot s_n$ as an approximation.
      
[^5]: The `receiver_digest` field does *not* identify the recipient (or in this case,
      the miner); instead, it is a requirement of the [mutator set](../mutator-sets) which serves 
      precisely to protect their anonymity. In order to spend the UTXO the recipient 
      will need to derive data from the preimage of the `receiver_digest` – and 
      prove the integrity of this derivation in zero knowledge.

[^6]: The distribution 75%-25% is outdated. As of
      [mainnet launch](../blog/mainnet-launch), the actual distribution is
      50%-50%.

[^7]: The present tense in this section is used to describe an architecture.
      While it might be part or all of a future upgrade, this construction is
      currently not part of mainnet.