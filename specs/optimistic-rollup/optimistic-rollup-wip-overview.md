# Optimistic Rollup: Smart Contracts in L2
This post outlines optimistic rollup: a construction which enables autonomous smart contracts on layer 2 using the [OVM](https://medium.com/plasma-group/introducing-the-ovm-db253287af50). Optimistic rollup barrows heavily from both plasma and rollup designs, and builds on [shadow chains](https://blog.ethereum.org/2014/09/17/scalability-part-1-building-top/) as described by Vitalik. This construction resembles plasma but trades off some scalability to enable fully general (eg. Solidity) smart contracts to be run in layer 2, secured by layer 1. The scalability bottleneck present here but not in plasma is the bandwidth of the rollup data availability oracle. However, this can be mitigated using other data availability oracles like Eth2 or even [Bitcoin Cash](https://ethresear.ch/t/bitcoin-cash-a-short-term-data-availability-layer-for-ethereum/5735), providing a near term scalable EVM-like chain in layer 2.

## Overview
To provide intuitions for how optimistic rollup works, we will chronicle the life of a smart contract named Sam:

1. Developer writes a Solidity contract, Sam.
2. Developer sends transaction to a bonded **aggregator** which deploys the contract. 
    - Fees are paid however the aggregator wants.
    - There are multiple aggregators per-chain.
    - Developer gets an instant guarantee that the transaction will be included or else the aggregator loses their bond.
4. Aggregator computes the new state root after applying the transaction.
5. Aggregator submits an Ethereum transaction (paying gas) which contains the transaction & state root (an optimistic rollup block).
6. If **anyone** downloads the block & finds that it is invalid, they may prove the invalidity with `verify_state_transition(prev_block, block)` which:
    - Slashes the malicious aggregator & any aggregator who built on top of the invalid block.
    - Rewards the prover with the aggregator bond.
7. Sam the smart contract is safe happy & secure knowing her deployment transaction is now a part of every valid future optimistic rollup state (assuming Ethereum mainnet doesn't revert). Yay!

You'll see through this example that optimistic rollup resembles Ethereum mainnet today. Still, it's important to review how we we can get to a construction so similar to L1, and even what what L1 gives us today.

## Optimistic Rollup Security Analysis
Any blockchain state machine should provide the following three guarentees:

1. **Available head state** -- Any relevant party can download the current head state.
2. **Valid head state** -- The head state is valid.
3. **Live head state** -- Any interested party can submit transactions which transition the head state.

You'll notice that Ethereum layer 1 satisfies these three properties because 1) miners do not mine on unavailable blocks, 2) miners do not mine on invalid blocks*; and 3) we believe layer 1 does not censor transactions. However, unfortunately for us, layer 1 doesn't currently scale.

Thankfully, under some light security assumptions optimistic rollup provides all three guarentees--enough to serve as a public EVM/WASM enabled state machine for running smart contracts at scale. To understand the construction & security assumptions we'll go over each property we'd like to ensure individually.

### Available head state
Optimistic rollup uses classic rollup techniques ([first outlined here](https://ethresear.ch/t/on-chain-scaling-to-potentially-500-tx-sec-through-mass-tx-validation/3477)) to ensure data availability of the current state. The technique is simple--block producers (called aggregators) pass all blocks which include transactions & state roots through calldata (ie. the input to an Ethereum function) on Ethereum mainnet. The calldata block is then merklized & a single state root is stored. This ensures that the data is available without incurring the high gas cost of state storage. It is also notable that the gas cost of calldata will be reduced by almost 5x in the [Istanbul hard fork](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1679.md).

Notably, we can use data availability oracles other than the Ethereum mainnet, including Bitcoin Cash and Eth2. With Eth2 phase 1, data availability is going to be _cheap_ and so data availability will be much less of a bottleneck.

With all transactions and state roots made available anyone can download the current head state, satisfying property #1.

#### Security Assumption
Here we assume honest majority on Ethereum mainchain. In addition, if we use Eth2 or Bitcoin Cash, we similarly inherit their honest majority assumptions.


### Valid head state
The next property we need to ensure is a vaild head state. In zkRollup we use zero-knowledge proofs to ensure the head state is valid. While this is a great solution in the long term, for now it is not possible to create efficient zkProofs for arbitrary state transitions. However, there's still hope for a general purpose EVM-style state machine! We can use a cryptoeconomic validity game similar to plasma / truebit.

#### Cryptoeconomic Validity Game
At a high level the block submission & validity game is as follows:

1. Aggregators (eg. block producers) post a security deposit to start producing blocks.
2. Each block contains `[access_list, transactions, post_state_root]`.
3. All blocks are committed to a `ROLLUP_CHAIN` contract by a bonded aggregator.
4. **Anyone** may prove a block invalid, *winning the security deposit of the aggregator*.

To prove a block invalid you must prove one of the three following properties:

```
1. The committed block is *invalid*. 
   This is calculated with `isValidTransition(prev_state, block) => boolean`
2. The committed block "skipped" a valid block.
3. The committed block's parent is invalid.
```

These three state transition rules can be visualized as:

![](https://i.imgur.com/eFz8Dzw.jpg)

##### Validity Checkers
Validity checkers for `isValidTransition(...)` are pluggable. It is even possible to use multiple validity checkers for the same chain! Examples of validity checkers include:
1. Custom subsets of the EVM
2. Full EVM
3. WASM
4. ...whatever!

##### "Skipped" Blocks
Aggregators are able to skip blocks which they believe are invalid. These are not the same sort of forks which are common on the mainchain. Because Ethereum mainnet gives us a total ordering of blocks and we disallow skipping valid blocks, users able to deterministically calculate what the **valid** head state is. This determination is done 'optimistically,' meaning the client can compute the valid head state before mainnet is aware of an invalid block.

##### Sharding the State with UTXOs
We can use a UTXO model to shard state validation (similar to how Plasma Cash sharded state validation). UTXOs allow for the verification & invalidation of independent state transition histories. This means we can validate only UTXOs for contracts we care about to secure our state. Exciting! To learn more about how UTXOs enable parallelism [check out this Cryptoeconomics.study video!](https://www.youtube.com/watch?v=-xoCoZGJ9AQ).

##### Cryptoeconomic Light Clients


##### A Note on Plasma
Many plasma constructions also rely on this approach; however, in plasma state enforcement is impossible without a [fishermen's game](https://github.com/ethereum/research/wiki/A-note-on-data-availability-and-erasure-coding#what-is-the-data-availability-problem) during a data withholding attack (the data availability problem). Thankfully rollup gets around the data availability problem by simply posting everything on-chain and so, in a sense, we can use a plasma



# WORK IN PROGRESS...
Will finish this soon!

---

This post extends https://plasma.build/t/rollup-plasma-for-mass-exits-complex-disputes/90/21 to describe a scheme, optimistic rollup, which enables autonomous smart contracts on L2 using the [OVM](https://medium.com/plasma-group/introducing-the-ovm-db253287af50). It also compares optimistic rollup to plasma, including why using an availability oracle gives you smart contracting capabilities similar to those on Ethereum today but much improved scale. Lastly it describes how rollup fits into the OVM as the third component to the currently explored layer 2 stack.



L2 smart contracts are possible through the use of a data availability oracle to get around data withholding attacks--a major limiting factor in plasma designs. Then 

In addition, optimistic rollup's security properties are so similar to L1 that it can host adjudication contracts for plasma & state channels.

With the advent of 

## Motivation
Through building Plasma we've had to explore the design space of secure computation on top of a slow top level blockchain. One property we discovered is the relationship between the authority to transition state & the enforcement of the validity of the state in the context of data withholding.


## Background

Why are we reviewing this?

![](https://i.imgur.com/T700dPg.png)

### Let's Review Etherem's State Transition Function

![](https://i.imgur.com/JWWV8SQ.png)

### Committing to Multiple State Machines

![](https://i.imgur.com/mDoIwEH.png)

### Verifying State

![](https://i.imgur.com/6DaXpL9.png)

## Exploring the Design Space
As we've built Plasma we realized the design space for L2 

### Plasma vs Optimistic Rollup

![](https://i.imgur.com/gRqBh9n.png)


# Adding a Cryptoeconomic Validity Game Part 1: Universal Dispute Contract
We can use this foundation to build a validity game which allows for cryptoeconomic light-clients on top of optimistic rollup. This allows us to create a "validity" oracle that uses bonded validators to incentivize invalid states to be pruned.

In order to make this formulation of layer 2 useful, we created a smart contract which interprets "claims" and is able to make inferences based on the claim contents. This will allow us to express all these layer 2 protocols in the same language which is directly interpreted on & off chain.

The details of this contract are specified here: https://hackmd.io/@aTTDQ4GiRVyyce6trnsfpg/S1yGmXVxB

At a high level, this contract simply makes decisions on claims by evaluating logical implications, contradictions, and proofs.

## Adjudication Contract Logical Foundations

Let's review what these claims, implications, contradictions, and proofs look like with an intuitive example.

We start with the following **claim**:

> It is raining.

Notice that the claim `It is raining` **implies** at least two sub-claims:

> 1. It is cloudy.
> 2. The ground is wet.

Further, notice that there are at least 3 claims which would **contradict** the initial claim of `It is raining`. These are:

> 1. It is NOT raining.
> 2. It is NOT cloudy.
> 3. The ground is NOT wet.

Now, armed with this understanding let's introduce the Universal Decision Contract (UDC). The UDC is a smart contract which evaluates these logical expressions and makes decisions based on claims & evidence submitted.

### Example Adjudication Process
We will use the above claim of `It is raining.` to review the process the UDC follows for all of it's decision making. In this case, Mallory will claim it is raining when really it's a sunny day.

1. Mallory submits a claim `It is raining`.
2. The UDC begins a dispute peroid where anyone can dispute this claim. If no one disputes after the allotted dispute peroid, the claim will be decided TRUE.
3. Alice sees Mallory's claim & looks outside. Oh no Mallory was lying! It's a bright and sunny day! She's got to prove this malicious claim FALSE!
4. Alice knows that if it's raining, it must be cloudy. Therefore all Alice must do to disprove Mallory's claim is take a photo of the clear blue sky & send it to the UDC.
5. Alice takes a photo and proves her own claim: `It is NOT cloudy` to the UDC.
8. Alice then proves to the UDC that the claim `It is raining` **implies** `It is cloudy`. Further, she shows that we've already decided that `It is NOT cloudy`--a decision which **contradicts** the Mallory's original claim. Because claims cannot contradict decisions, the UDC decides that Mallory's claim of `It is raining` is FALSE.
10. Alice can now celebrate that this evil claim did not prevail! Woot!

## OVM Formulations
The following are a few "plain English" definitions for properties which will come in handy for different "layer 2" solutions.

### 1) State Channels
A state channel can be defined with the following data types & claim structure:

#### Claim Structure:
```
There exists a SignedStateUpdate with all participant signatures `lastUpdate`, 
AND
all SignedStateUpdates which are signed by me have a nonce <= lastUpdate+1.
```

#### Contradictions
To contradict this claim you can show either a signed transaction with a higher nonce, or you can claim that the signed message with all participants does not exist.


### 2) Plasma
Plasma is complex & so to save time this one is quite high level.

In order to exit a "coin" at plasma commitment `b`, and coinID `c` we can use the following claim:

#### Claim Structure
```
NOT: is_deprecated(b, c)
AND
For all coins such that `coinId == c`:
    For all blocks such that `blockNumber < b`:
        NOT:
            (
                    included_state_update(blockNumber, coinId)
                AND
                    NOT: is_deprecated(blockNumber, coinId)
            )
```

#### Contradictions

A plasma exit claim can be contradicted by showing an undeprecated (eg. "unspent") state in that coin’s history, or by showing that the state being exited is deprecated (eg. "spent").

Notice how complex this dispute process is. This is *why* we came up with this language—-it got too hard to reason about!
                
### 3) Optimistic Rollup

We can visualize block commitments as:

![](https://i.imgur.com/0Qbl0W1.jpg)

#### Single-Contract Transition Validity
Now we have commitments, but we also want to determine the validity of the commitments. To do this, every time there is a commitment submitted, a cooresponding claim will be submitted to the UDC. The claim is of the form:

> commitment #x is entirely valid

To contradict the validity of a commitment you can prove any of the following:

```
1. The committed state transition is *invalid*. 
   This is calculated with `isValidTransition(...) => boolean`
2. The committed state "skipped" a valid transition.
3. The committed state's parent is invalid.
```

![](https://i.imgur.com/eFz8Dzw.jpg)

Note that a validity checkers for `isValidTransition(...)` are pluggable. Examples of validity checkers include:
1. Custom subsets of the EVM
2. WASM
3. Full EVM as implemented today
4. ...whatever! This is especially easy if we move to WASM as a base layer... but not required.

#### Multi-Contract Transition
Realisitically we want to make commitments which transition *multiple* seperate contracts. These transitions should be *entirely independent* of oneanother so that users only need to download the transitions of contracts they care about. 

Note: this is the same problem "plasma cash" was created to solve.

Making each contract their own state machine introduces a new problem--atomicity. We can solve this problem by making our transition validity coniditions more complex. In particular we:

1. introduce "access lists" to our transactions
2. change `isValidTransition(...)` to accept an array of prevTransitions & postStates
3. alter our claim so that a) the transition must be included in *all* slots, and b) *any* of our prevTransitions being invalid invalidates the transition.

![](https://i.imgur.com/YFlqRn5.jpg)


# Adding a Cryptoeconomic Validity Game Part 2: Bonded Aggregators

1. Aggregators must submit a bond of size `SECURITY_DEPOSIT`
2. If an aggregator submits an *invalid* transition (as defined above), their bond is at risk & can be transfered to **anyone** who shows the invalid transition.
3. Notice that:
    A) Proving an invalid chain that `N` aggregators have built on top of rewards the prover with a security bond equal to `N*SECURITY_DEPOSIT`--naturally incentivizing "watchtowers".
    B) No "finality threshold" must be encoded into the core protocol.
    C) Aggregators can "transition" any sub-state machine, removing complexity around load balancing.
    D) No "congestion" problem common to L2 protocols. New blocks can be submitted even while invalidity games are pending, & users can fork around invalid chains before L1 catches up.
    E) Plasma & State Channels can be adjudicated **inside of a optimistic rollup chain**.
    
## Scalability!
### With ETH1 Data availability
The following is a little call-data calculation python script -- https://gist.github.com/karlfloersch/1bf6ab7871f41e3a5a921c0a007ad5c6

#### ~5x Without EIP 2028
Further optimizations can be made to reduce tx calldata size.

#### ~24x With EIP 2028
Once again these optimizations could be made.

### With external availability oracles (eg. ETH2, Bitcoin Cash)
#### ~linear in relation to the amount of throughput the availability oracle can handle.
That's a lo

## That's Optimistic Rollup! Questions?
I <3 Simplicity!

### Notes relating to to ETH2
- Load balancing not a problem!
- Stateless clients are great

### What is the Optimistic Virtual Machine (OVM)
A smarter wallet software which uses fork choice evaulation to derive the state from both on-chain state & local local information.

![](https://i.imgur.com/v69v6lv.gif)

Here is the math!

![](https://i.imgur.com/10mGsIp.png)


---

For more information check out http://medium.com/plasma-group/introducing-the-ovm-db253287af50
