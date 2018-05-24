---
layout: post
title:  "Sprites State Channels"
date:   2018-05-24 17:30:00 +0800
categories: update
author: Mathis Antony
---
  Enuma would like to thank the [Ethereum Foundation](https://www.ethereum.org/foundation) for a grant to implement [Sprites state channels](https://arxiv.org/abs/1702.05812). In this blog post we would like to provide an accessible introduction to Sprites and our motivations for pursuing its implementation.

# Moving Off Chain

  One of the main challenges for widespread adoption of blockchain technology is scalability. Ideally any two parties should be able to make transactions within a reasonable time frame. Currently the Ethereum blockchain handles about 20 transactions per second. If each of 7 billion people in the world were to make one transaction per day, we would need to handle about 80,000 transactions per second. Visa handles about 2,000 transactions per second.

   Decentralization and censorship resistance are fundamentally at odds with some "solutions" to scalability problems. For instance larger block sizes allow more transactions per block but conversely require more storage to store the entire blockchain and thus reduce decentralization. Scaling efforts for blockchains can be separated into two categories: scaling of the blockchain itself (layer 1 scaling) and development of off-chain technologies (layer 2 scaling). Sprites state channels fall into the second category. 

# Composable

  In the Sprites paper, the authors construct a general *state channel* abstraction and prove its security guarantees within in the [Universal Composability](https://en.wikipedia.org/wiki/Universal_composability) framework. They later derive the Sprites payment channel from the more general state channel construction. Other types of state channel can be derived from the general state channel construction to provide functionality as desired by the application logic. The construction caters to arbitrary state channel update functions as well as side effects on the blockchain. In this we see the real potential of the Sprites state channel construction for the Ethereum ecosystem.

# State Channels

  In a nutshell, state channels allow parties to evolve [state machines](https://en.wikipedia.org/wiki/Finite-state_machine) without relying on the blockchain to confirm each state change. In optimistic cases where all participants act honestly, transactions are carried out very quickly with signed peer to peer messages between the parties. Only in case of dispute do we fall back to the blockchain and delegate the resolution to a smart contract. In a loose real world analogy,  parties do their everyday business with each other but know they can go to court if they have a severe disagreement. State channels therefore handle the common, optimistic case swiftly but nevertheless provide security guarantees by optionally falling back to the blockchain in case of disputes. This makes state channels an attractive scaling option by moving the handling of the majority of transactions away from the blockchain.

  A prominent application of state channels are payment channels, where the two parties might exchange ERC20 tokens with each other.

  A simplified payment channel in practice could evolve as follows

  - Alice and Bob open a payment channel and make deposits into the channel.
  - Alice and Bob exchange rounds of signed messages,  for instance to send Ether.
  - Once they are done, one of the parties initiates the closure of the channel.
  - The channel waits some time for parties to submit evidence and then disburses the collateral in accordance with the payments.

  A simplified depiction of such an interaction in a Sprites payment channel is shown below.

  ![](/images/sprites/simple-channel.svg)

  At this time payment channels require participants to freeze collateral (the deposit) because we can no longer rely on the blockchain directly to prevent double spending of funds inside payment channels. While a channel is open, the collateral is locked and can't be used in any other way. The cost of maintaining a payment channel is quantified as not just the amount of collateral but as the amount of collateral multiplied by the time for which the collateral is locked.

  In the optimistic case, where the parties agree with each other, the payments can be completed quickly and without the use of the blockchain.

  One of the appealing properties of state channels is that channels can be linked together. For instance Alice can send $X via Bob to Charlie over two channels.

  ![](/images/sprites/linked.svg)

  In Sprites linked payments parties first exchange conditional payments, conditioned on the revelation of a secret message (the *preimage*) before a certain deadline. Once an honest party receives the evidence to claim the conditional payment via a dispute they send an unconditional payment message to supersede the conditional payment. As a result, in the case where all parties behave honestly the payment can be completed with peer to peer messages and we don't need to wait for any blockchain transactions to be confirmed.

# Resolving Disputes

  If at some point during the process a party refuses to cooperate, is disconnected, or for any other reason fails to continue, we require a mechanism to assure a fair termination of the state channel. This is commonly referred to as *dispute resolution*. 

  One of the main practical advantages of Sprites compared to [Lightning](https://en.wikipedia.org/wiki/Lightning_Network)/Raiden channels is that disputes can be resolved in parallel with the help of a *Preimage Manager* smart contract. In a linked payment a preimage (the input of a hash function that generates a particular hash)  is created by the sender and is initially shared with the final recipient. The parties then make payments conditional to the public revelation of the preimage before a deadline. The hash of the preimage is sent as part of the conditional payments. The Preimage Manager  serves as a global source of truth for the time of revelation of preimages. The Sprites channel contract queries the Preimage Manager when resolving disputes.

  Below is a simplified example of how a dispute is resolved with Sprites.

  - Bob triggers a dispute.
  - Bob submits the preimage.
  - All parties can submit evidence for a fixed amount of time.
  - Charlie initiates termination.
  - Sprites Contract disburses fairly according to received evidence.

  ![](/images/sprites/dispute.svg)

  In a linked payment channel the collateral cost then becomes proportional to the block time instead of proportional to the block time times the number of links in the channel which is the case with lightning channels. This in turn makes a Sprites channel network more resistant to petty attacks because such attacks can no longer slow down the average transaction time as much,  even when going through linked channels with many hops. More details and simulation results can be found in the Sprites paper.

# Atomic Swaps

  Atomic swaps are useful in scenarios where parties would like to do a bi-directional exchange. This can be constructed from Sprites payment channels by making a linked payment from the first party via the second party and back to the first party.

  Below we depict Alice sending 10 T1 tokens to Bob in exchange for 20 T2 tokens via two intermediaries decentralized exchanges DEX 1 and DEX 2.

  ![](/images/sprites/swap.svg)

  Thanks to the guarantees of the channel construction either all of the intermediate payments complete or none do. The swap is therefore atomic and the balance changes of the intermediaries cancel each other out.

# Channel Networks

  Given the collateral requirements it is unlikely that most parties will not have the required funds to maintain a large number of open payment channels. Opening and closing channels requires blockchain transactions. It is therefore not desirable to open a channel for the purpose of making a single payment.

  We envision that by providing small fees for the use of payment channels we can entice users to keep payment channels open at times when they are not actively using them. As a consequence we hope to see a channel network emerge where vertices are participants and edges are state channels. Ideally all participants can reach each other via a few channels through this network.

  To illustrate consider the tiny payment network below on the left. First (in the middle) Alice sends a linked payment of $X to Charlie. Afterwards (on the right) Alice sends a liked payment of $Y to David. Bob is used as an intermediary for both payments because Alice doesn't have an open channel with Charlie or David.

  ![](/images/sprites/blah.svg)

In a semi decentralized setting with a spoke and hub topology users could trade with most other users by having just a single channel with one of the larger decentralized exchanges.

**Network of trades on 0x contract**

At this stage it is speculative to talk about the topology of large scale crypto-currency payment channel networks. In order to get an idea of how such a payment network topology might look like, we take a look at the history of token trades done through the [0x](https://0xproject.com) exchange contract as of May 7 2018. For this we draw an edge connecting two addresses if they have done at least one trade with each other. We ignore the individual tokens, the time component and the balances.

![](/images/sprites/zerox.png)

- 5674 vertices (addresses)
- 11945 edges
- Large connected component
  - 95% of vertices
  - 3.8 average shortest path length

We observe a sparse graph with a relatively low average shortest path length and a large connected component containing most vertices.  The degree distribution follows approximately a power law which is the main characteristic of *[scale-free* networks](https://en.wikipedia.org/wiki/Scale-free_network) like the BA network used for simulations in the Sprites paper. 

![](/images/sprites/degree_distribution.svg)

If such a channel network existed and was sufficiently backed with collateral most parties could interact with each other through channel network through relatively few channels along the network.

## Summary

In summary we at Enuma are excited about the Sprites because it will enable fast, attack-resistant off chain interactions in synergy with the Ethereum blockchain. We see the generic Sprites state channel construction as a valuable building block for future DApps.

## What we hope to achieve

We find the Sprites construction both elegant and generally useful we also think that it is non-trivial in its implementation because the majority of logic happens off-chain. Therefore we think most DApp developers would prefer not to implement state channels themselves and rather focus on their application logic.

Over the next months we will be working on reference implementation consisting of supporting smart contracts, a library and examples for common use cases such as payment channels.

## Further Reading

- Sprites paper: [https://arxiv.org/abs/1702.05812](https://arxiv.org/abs/1702.05812)
- Example code: [https://github.com/amiller/sprites](https://github.com/amiller/sprites)
- Universal Composability
  - [https://en.wikipedia.org/wiki/Universal_composability](https://en.wikipedia.org/wiki/Universal_composability)
  - Video tutorial: [https://youtu.be/5kF3TxEt-_A](https://youtu.be/5kF3TxEt-_A)
  - Paper: [https://eprint.iacr.org/2000/067](https://eprint.iacr.org/2000/067)

# Author
[Mathis Antony](http://github.com/sveitser/)
