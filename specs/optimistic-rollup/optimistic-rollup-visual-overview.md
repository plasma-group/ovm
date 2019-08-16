# Optimistic Rollup: Smart Contracts in L2
This post extends https://plasma.build/t/rollup-plasma-for-mass-exits-complex-disputes/90/21 to describe a scheme, optimistic rollup, which enables autonomous smart contracts on L2 using the [OVM](https://medium.com/plasma-group/introducing-the-ovm-db253287af50). It also compares optimistic rollup to plasma, including why using an availability oracle gives you smart contracting capabilities similar to those on Ethereum today but much improved scale. Lastly it describes how rollup fits into the OVM as the third component to the currently explored layer 2 stack.

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
