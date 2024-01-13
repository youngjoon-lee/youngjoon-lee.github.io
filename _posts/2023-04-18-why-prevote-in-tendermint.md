---
layout: post
title: "Why prevote in Tendermint"
date:   2023-04-18 21:06:00 +09:00
---

Tendermint Byzantine Fault Tolerance (hereafter BFT) consensus algorithm has two stages of voting before committing a block to the state,
while Raft consensus algorithm (that doesn't cover the Byzantine problem) has a single stage of preparation (aka. log replication) for committing a transaction.
When I first met Tendermint, I was most curious why Tendermint has the two-stage voting (prevote-precommit).

In [Ethan Buchman 2016](https://atrium.lib.uoguelph.ca/xmlui/handle/10214/9769), he mentioned as below:
> In asynchronous environments with Byzantine validators, a single stage
of voting, where each validator casts only one vote, is not sufficient to ensure
safety. In essence, because validators can act fraudulently, and because there
are no guarantees on message delivery time, a rogue validator can co-ordinate
some validators to commit a value while others, having not seen the commit,
go to a new round, within which they commit a different value.
<br> 
A single stage of voting allows validators to tell each other what they
know about the proposal. But to tolerate Byzantine faults (which amounts,
essentially to lies, fraud, deceit, etc.), they must also tell each other what
they know about what other validators have professed to know about the
proposal. In other words, a second stage ensures that enough validators
witnessed the result of the first stage. 

That sounds reasonable, but I wanted to see a specific counterexample that explains why only precommit without prevote is not sufficient.

Fortunately, I found a counterexample from a [Reddit post](https://www.reddit.com/r/cosmosnetwork/comments/8zwais/comment/e2nyg92/?utm_source=reddit&utm_medium=web2x&context=3).
Below, I am going to go into details on this counterexample.

Let's assume that Tendermint has only single stage of voting (aka. precommit), and the network consists of the following validator nodes:

- 100 validator nodes in the network
    - 28 offline nodes (e.g. due to network partitioning)
    - 65 honest nodes
    - 1 honest node (aka. `H`)
    - 1 Byzantine node (aka. `B`)

Let's say the current round is `n`. A "valid" block is proposed by one of the validators.
Then, all 66 (= 65 + 1) honest nodes broadcast a precommit for this valid block.
But, let's imagine the case when the Byzantine node `B` sends `nil` to 65 honest nodes and itself, but a precommit to the honest node `H`.

In this case, `H` has 67 precommits (`> 100 * 2/3`) with its own vote. Then, `H` will commit the block.
However, all other nodes (65 honest + `B`) move to the next round `n+1` because they have only 66 precommits (`< 100 * 2/3`).

This result is a breach of safety because the honest node `H` committed the block that all other honest nodes didn't commit.

If Tendermint has the prevote-precommit mechanism, the above example can be resolved safely.
Let's replace all precommits in the above example with prevotes.
Then, `H` will broadcast a precommit to all other nodes instead of committing the block to its state,
but it won't receive 2/3+ precommit from other nodes because all other nodes except `H` won't broadcast a precommit since they didn't receive 2/3+ prevotes.
As a result, all nodes will move to the next round `n+1` without committing the block.
Then, the consensus remains safe, even if any new block never be committed due to `B` in subsequent rounds.
