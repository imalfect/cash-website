+++
title = "Post-Quantum Security at Genesis"

[extra]
author = "Alan Szepieniec"
+++

# You Cannot Fix Quantum Insecurity with an Upgrade

You need post-quantum security at genesis.

## Introduction

Shor's quantum algorithm is known to be capable of computing elliptic curve discrete logarithms efficiently, thereby breaking the ECDSA and ECSchnorr signature schemes used in Bitcoin, Ethereum, and a host of other popular cryptocurrencies. The catch is that you need a quantum computer to deploy this attack, and the quantum computers that exist today do not seem to be big enough for this task – otherwise we would have seen Satoshi's hoard of Bitcoins move already. But what happens when large enough quantum computers are built?

## That's a Big "If"

It is entirely reasonable to question the feasibility of large scale quantum computers. It is possible that a not-yet discovered law of nature forbids their construction. Or perhaps no such law exists but the engineering effort explodes, exposing the attack to merely a new point on the tradeoff curve between cryptographically intractable running times and cryptographically intractible engineering work. There are educated opinions on both sides.

However, from a risk management point of view, you need to take the construction of large scale quantum computers seriously. Even if you judge the probability of success to be 0.1%. An insurance against quantum attacks is worth billions, if trillions of dollars are at stake.

## Hashing is Not Enough

It is true that quantum computers cannot break hash functions.[^1] So if the quantum attacker only sees a script hash or public key hash, then he won't be able to spend the locked coins. However, neither will you! When you do want to spend the coins you must first release the preimage of this hash digest, which contains the vulnerable public key. At this point a race begins: can the quantum attacker compute the secret key before the transaction is confirmed? Because if he can, then he can replace your transaction with his own by bumping the fee. And even if he can't crack the key in 10 minutes, the price to pay to guarantee that your transaction is confirmed inside this window is enough fees to outbid all the other hodlers who are scrambling to move their coins to safety.

## Upgrades will Not Suffice

So what happens if we upgrade the protocol to integrate native support for post-quantum signature schemes such as the ones set to be standardized by [NIST](https://csrc.nist.gov/projects/post-quantum-cryptography)? Quantum computers will be unable to attack the public keys introduced by this upgrade.

With a sophisticated deployment, hodlers will have the opportunity to move their coins from quantumly-insecure addresses to quantumly-secure addresses in the course of a transition period, which hopefully comes to a planned end before quantum computers are capable of deploying their attacks. At the end of the transition period, all coins still tied to vulnerable public keys will be burned. When quantum computers finally are large enough to attack elliptic curve discrete logarithms, there will be no vulnerable coins left to attack.

This sophisticated deployment is unlikely because it is controversial. 

 1. The reason why post-quantum signature schemes are not more widely used is because they inevitably induce cumbersome tradeoffs. Either the public key is huge, or the signature is huge, or it takes on the order of seconds to generate a signature. Relative to ECSchnorr, which has compact public keys, compact signatures, and fast signature generation (not to mention other fancy properties such as batch verification and native support for adaptor signatures), migrating to any post-quantum signature scheme seems like an unnecessary performance downgrade – particularly in the eyes of folks who religiously believe in the impossibility of quantum computers.
 2. The policy of burning vulnerable coins when the transition period ends goes against the most fundamental spirit of cryptocurrencies. Your capacity to spend your coins should be independent of what other people think, whether they be central bankers or shadowy supercoders. Burning the coins of users who refuse to or cannot participate in the upgrade ceremony sets the worrying precedent that coins could be burned for other reasons as well.

## Burn or Tank

The most likely scenario (in my eyes) is that the transition period starts and no expiration date is set. No coins are ever burned. Users can migrate their coins whenever they judge the time to be right. If they move their coins too soon, they pay the cost of higher fees as a result of consuming more scarce block space. If they move their coins too late, they risk losing them to the quantum attacker.

At this point, the relevant question is what fraction of coins will have been moved to post-quantum signatures before quantum computers arrive. Because the complement of this fraction is vulnerable to attack; and the rational attacker will sell it to make a profit. Even assuming that all the *active* users migrate their coins, the coins of *inactive* users might be enough of a supply shock to tank the price.

Users anticipating this supply shock will sell their coins before it hits. If the progress of quantum computers is slow and public, the price of vulnerable cryptocurrencies will drop gradually as more and more users opt out. Conversely, if the progress is secret or sudden or both, users will be surprised by the supply shock. In either case, the price will drop.

## Cryptoeconomic Security

Cryptocurrencies do not exist for the purpose of price stability, or at least not all of them do. The price is allowed to drop. And after all the vulnerable coins were either migrated-early-enough or stolen-then-migrated to post-quantum addresses, the economy can resume. What's the problem?

The problem with this line of reasoning is that a cryptocurrency's capacity to reach consensus is intricately linked to its market price. If the price drops, miners have less incentive to mine. The reduction in mining power makes the network more vulnerable to reorganization attacks. More confirmations are needed before a transaction of a constant monetary value (*i.e.*, measured in an external unit) is as difficult to overturn. The cryptocurrency is a less suitable medium for exchange, or concisely: it is worse money.

## Immunity

Vulnerable coins do not only weaken their owners' portfolios; they weaken their host cryptocurrency through the cryptoeconomic incentive system. To provide immunity against this degrading effect due to the threat of quantum computers, it is necessary to make *all* coins invulnerable to them, down from the very first coins minted by the very first block. In short, we need *post-quantum security at genesis*.

[^1]: Quantum computers cannot break hash functions *as far as we know*. Quantum algorithm researchers widely conjecture as much. The discovery that quantum computers could break hash functions would be one of the most surprising scientific breakthroughs of all time.
