Note: The stuff at the bottom of this doc is more up-to-date than the stuff at the top. 

If reading top-down, note that some details may conflict with other parts of the doc, and what is at the bottom takes priority

# Client OVM Designs

## Types

```typescript=
type Property = {
    predicate: Predicate,
    input: {}
}
type Proof = ProofElement[]
type ProofElement = {
    property: Property,
    witness: {}
}

interface Decision = {
    outcome: boolean,
    implicationProof: ImplicationProofElement[] // constructed such that claim[N] --> claim[N-1] --> claim[N-2]... Claim[0]
}

interface ImplicationProofElement = {
    implication: Property,
    implicationWitness: any
}

```

## Proof Ingestion

```typescript=
function ingestProof(_proof: Proof): void {
    for (proofElement of _proof) {
        predicate = proofElement.property.predicate
        input = proofElement.property.input
        witness = proofElement.witness
        predicate.ingestProofElement(input, witness)
    }
}
```

## Preimage Exists Decider
```typescript=
db: Bucket

type PreimageExistsInput = {
    hash: bytes32
}

type DecisionStatus = boolean | undefined 

function decide(_input: PreimageExistsInput, _witness: {}): Decision {
    verify = _input.verifier
    parameters = _input.parameters
    preimage = _witness.preimage
    
    if (! hash(preimage) !== _input.hash) {
        throw new Error('Provided preimage does not hash to expected image, cannot decide.')
    }
    
    decisionKey = _input.hash
    decisionValue = {
        decision: true,
        witness: _witness
    }
    this.db.put(decisionKey, decisionValue)
    
    return {
        outcome: true,
        implicationProof: undefined
    }
}

function checkDecision(_input: PreimageExistsInput): DecisionStatus {
    decisionKey = _input.hash
    decisionValue = this.db.get(decisionKey)
    // if a decision has been made...
    if (decisionValue) {
        // ...return the decision
        return decisionValue.decision
    }
    // We have not made a decision, return undefined
    return undefined
}
```

## Universal Quantification Predicate

```typescript=
type UniversalQuantificationInput = {
    quantifier: Quantifier,
    parameters: {},
    proposition: (_variable: any) => Property
}

function decide(_input: UniversalQuantificationInput): Decision {
    quantifier = _input.quantifier
    quantifierParameters = _input.parameters
    quantification = quantifier.getAllQuantified(quantifierParameters)
    results = quantification.results
    allResultsQuantified = quantification.allResultsQuantified
    anyUndecided = false
    for (quantified of results) {
        // Assume variable substitution magic
        propertyToCheck = _input.proposition(quantified)
        predicateToCheck = propertyToCheck.predicate
        inputToCheck = propertyToCheck.input
        decision = predicateToCheck.checkDecision(inputToCheck)
        if (decision.outcome === false) {
            decision.implicationProof.push({
                quantified,
                property: propertyToCheck
            })
            return decision
        } else if (decision.outcome === undefined) {
            anyUndecided = true
        }
    }

    // Return undefined if any were undecided, or if not all the results were quantified.
    // Otherwise, we return true!
    return {
        outcome: (anyUndecided || !allResultsQuantified) ? undefined : true,
        implicationProof: undefined
    }
}
```

## Quantifiers

```typescript=

interface QuantifierResult = {
    results:  any[],
    allResultsQuantified: boolean
}

interface Quantifier {
    function getAllQuantified(parameters: any): QuantifierResult
}
```


## Less Than Quantifier
```typescript=
type LessThanQuantifierParameters = number

class LessThanQuantifier {
    function getAllQuantified(_lessThanThis: LessThanQuantifierParameters): [] {
        allQuantified = []
        for (int i = 0; i < _lessThanThis; i++) {
            allQuantified.push(i)
        }
        return {
            results: allQuantified,
            allResultsQuantified: true
        }
    }
}

```

## Putting it all together
```typescript=
hashes = [0x123, 0x987, 0x111, ...]
orderedPreimages = ['pre1', 'pre2', 'pre3', ...]

isNthPreimageVerifier (_n: number, _preimage: bytes) {
    return hash(_preimage) == hashes[_n]
}

proofToIngest = [
    {
        property: {
            predicate: nthPreimageExistsPredicate
            input: { index: 0 }
        }
        witness: orderedPreimages[0]
    },
    {
        property: {
            predicate: nthPreimageExistsPredicate
            input: { index: 1 }
        }
        witness: orderedPreimages[1]
    }
    , ... , // for 0-9
    {
        property: {
            predicate: universalQuantificationPredicate,
            input: {
                quantifier: lessThanQuantifier,
                parameters: 10,
                proposition: (preImageIndex): Property => {
                    return {
                        predicate: nthPreimageExistsPredicate,
                        input: { index: preImageIndex }
                    }
                }
            }
        },
        witness: {}
    }
]
```


```typescript=
// let's decide on a 2D array of preimages.
nestedQuantification = {
    property: {
        predicate: universalQuantificationPredicate,
        input: {
            quantifier: lessThanQuantifier,
            parameters: 10,
            proposition: (preImageXIndex): Property => {
                return {
                    predicate: universalQuantificationPredicate,
                    input: {
                        quantifier: lessThanQuantifier,
                        parameters: 10,
                        proposition: (preimageYIndex): Property => {
                            return {
                                predicate: isXthYthPreimagePredite, // this verifies that the witness is the preimage at X,Y in a 2D array of hash images,
                                input: {
                                    xIndex: preImageXIndex,
                                    yIndex: preimageYIndex
                                }
                            }
                        }
                    }
                }
            }
        }
    },
    witness: {}
}

```

### Non-existence in a Comprehensive Finite Range
Instead of deciding properties of potentially infinite ranges, we can transform our property to be decided based on a finite quantification. 

One example is state channels where instead of framing the problem as: "For all nonces greater than n, there does not exist a signature by me." into "For all state updates which I have signed, the nonce is less than or equal to n"

NOTE: This decision can only be made if we know our knowledge of the set is comprehensive.

## Coin Ranges

For all ranges between `[start, end)`, the property included is true.

This to check:
- Covers the full range

### Range Witness At Block Ingestion

```typescript=
db: RangeBucket

interface Range = {
    start: number,
    end: number
}

interface RangeWitnessExistsAtBlockInput = {
    blockNumber: number,
    coinRange: Range
}

interface PlasmaDataBlock = {
    updatedRange: Range,
    property: Property // assuming properties are always "the included thing" -- the block entries. Note that often times the property will instead be the `property hash` for exclusion proofs (this way we don't reveal the property contents)
}

interface RangeAtBlockWitness = {
    inclusionProof: MerkleIntervalInclusionProof,
    dataBlock: PlasmaDataBlock
}

interface InclusionDecisionValue extends RangeAtBlockWitness {
    decision: boolean
}

// TODO: Rename from `RangeWitnessExistsAtBlock` to `IncludedInIntervalTreeAtBlock` or something. 
// Witness is gerneric and this isn't, and IntervalTree implies Range.

function decide(_input: RangeWitnessExistsAtBlockInput, _witness: RangeAtBlockWitness): Decision {
    plasmaBlockNumber = _input.blockNumber
    coinRange = _input.coinRange
    
    inclusionProof = _witness.inclusionProof
    includedProperty = _witness.dataBlock.property

    wasIncluded = merkleIntervalTree.verifyInclusion(plasmaBlockNumber, coinRange, includedProperty, inclusionProof)
    if (!wasIncluded) {
        // Inclusion proof was wrong, therefore we cannot decide.
        return {
            outcome: undefined,
            implicationProof: undefined
        }
    }
    
    inclusionBounds = merkleIntervalTree.getBounds(inclusionProof)
    
    if (!isRangeSubset(coinRange, inclusionBounds)) {
        // The range which we are interested in is not contained by the inclusion proof, therefore we cannot decide.
        return {
            outcome: undefined,
            implicationProof: undefined
        }
    }
    
    // 1. store the inclusion if it intersects the requested coinRange
    relevantInclusion = getOverlappingRange(
        coinRange,
        updatedRange
    )
    
    inclusionDecisionRange = relevantInclusion
    inclusionDecisionValue = {
        decision: true,
        dataBlock: _witness.PlasmaDataBlock,
        inclusionProof: _witness.inclusionProof
    }
    this.rangeDB.getBlockStore(plasmaBlockNumber).put(inclusionDecisionRange, inclusionDecisionValue) 
    
    if (coinRange.end === updatedRange.end) {
        return
    }
    
    // 2. store the exclusion if it intersects the requested coinRange
    relevantExclusion = {
        start: updatedRange.end,
        end: inclusionBounds.end
    }
    
    exclusionDecisionRange = relevantExclusion
    exclusionDecisionValue = {
        decision: false,
        dataBlock: _witness.PlasmaDataBlock,
        inclusionProof: _witness.inclusionProof
    }
    this.rangeDB.getBlockStore(plasmaBlockNumber).put(exclusionDecisionRange, exclusionDecisionValue)
    
    return {
        outcome: true,
        implicationProof: undefined
    }
}

function checkDecision(_input: RangeWitnessExistsAtBlockInput): Decision {
    decisionKey = _input.coinRange
    decisionValue = this.db.get(decisionKey)
    // if a decision has been made...
    if (decisionValue) {
        // ...return the decision
        implicationProof = (decisionValue.decision) ? [] : [decisionValue.inclusionProof]
        return {
            outcome: decisionValue.decision,
            implicationProof
        }
    }
    // We have not made a decision return undefined
    return {
        outcome: undefined,
        implicationProof: undefined
    }
}
```

### Block Range Quantifier

```typescript=
interface IncludedCoinRangeQuantifierParameters = {
    plasmaBlockNumber: number,
    range: Range
}

function getAllQuantified(params: IncludedCoinRangeQuantifierParameters): QuantifierResult {
    blockRangeDB = this.rangeDB.getBlockStore(params.plasmaBlockNumber)
    includedDecisionValues = blockRangeDB.get(params.range)
    // Check if we have returned ranges which combined span the range we are attempting to quantify
    fullRangeIncluded: bool = rangesSpanRange(includedDecisionValues, params.range)
    // Filter our included properties to all properties we know were included
    // (excluding the ones we know to exclude)
    includedProperties = includedDecisionValues
        .filter((decisionValue) => decisionValue.decision)
        .map((decisionValue) => decisionValue.dataBlock.property)
        
    return {
        results: includedProperties,
        allResultsQuantified: fullRangeIncluded
    }
}
```

## Plasma Checkpoint Property

```typescript=
CHECKPOINT_BLOCK = 10
CHECKPOINT_RANGE = { start: 50, end: 100 }
plasmaCheckpointProofElement = {
    property: {
        predicate: universalQuantificationPredicate,
        input: {
            quantifier: lessThanQuantifier,
            parameters: CHECKPOINT_BLOCK,
            propertyFactory: (plasmaBlockNumber): Property => {
                return {
                    predicate: universalQuantificationPredicate,
                    input: {
                        quantifier: includedCoinRangeQuantifier,
                        parameters: {
                            plasmaBlockNumber,
                            range: CHECKPOINT_RANGE
                        },
                        propertyFactory: (includedProperty): Property => {
                            return includedProperty
                        }
                    }
                }
            }
        }
    },
    witness: {}
}

```


## State Channels

### State Channel Quantifier

```typescript=

// SOMEWHERE ELSE
const MY_ADDRESS = '0xabcdef0123456789'
const ADDRESS_ONLY_I_CAN_SIGN_FOR = 'stateChannel'
// ...

interface StateChannelParticipantMessagesQuantifierParameters = {
    channelId: any,
    participantAddress: address
}

function getAllQuantified(params: StateChannelParticipantMessagesQuantifierParameters): QuantifierResult {
    signedMessages = this.db
        .bucket(`${STATE_CHANNNEL_KEY_PREFIX}${params.chanelId}`)
        .bucket('messages')
        .getAll() // Really would use .iterator
        .filter((message) => message.signers.contains(participantAddress))
    return {
        results: signedMessages,
        allResultsQuantified: participantAddress === ADDRESS_ONLY_I_CAN_SIGN_FOR
}

```

### State Channel Exit Property

```typescript=

CHANNEL_ID = '0x3423235fff'
CHANNEL_COUNTERPARTY = '0x1234455667'
CHANNEL_NONCE = 10
stateChannelExitProperty = {
    predicate: andPredicate,
    input: {
        properties: [
            {
                predicate: universalQuantificationPredicate
                input: {
                    quantifier: stateChannelParticipantMessagesQuantifier,
                    parameters: {
                        channelId: CHANNEL_ID,
                        participantAddress: ADDRESS_ONLY_I_CAN_SIGN_FOR
                    }
                },
                propertyFactory: (channelMessage) => {
                    return {
                        predicate: hasLowerNoncePredicate,
                        input: {
                            nonce: CHANNEL_NONCE + 1
                        }
                    }
                }
            },
            {
                predicate: channelUpdateSignatureExists,
                input: {
                    channelId: CHANNEL_ID,
                    nonce: CHANNEL_NONCE,
                    participantAddress: COUNTERPARTY_PARTICIPANT
                }
            }
        ]
    }
}

CHECKPOINT_BLOCK = 10
CHECKPOINT_RANGE = { start: 50, end: 100 }
plasmaCheckpointProofElement = {
    property: {
        predicate: universalQuantificationPredicate,
        input: {
            quantifier: lessThanQuantifier,
            parameters: CHECKPOINT_BLOCK,
            propertyFactory: (plasmaBlockNumber): Property => {
                return {
                    predicate: universalQuantificationPredicate,
                    input: {
                        quantifier: includedCoinRangeQuantifier,
                        parameters: {
                            plasmaBlockNumber,
                            range: CHECKPOINT_RANGE
                        },
                        propertyFactory: (includedProperty): Property => {
                            return includedProperty
                        }
                    }
                }
            }
        }
    },
    witness: {}
}


```

## Processing L1 events
### BeginClaim
What we do:

- decide() on the claim.
    - if we decide false, submit counterclaim and challenge.
    - else, store pending L1 decision in the same place we store L2 decisions. (note: need to choose whether to lazy or eager purge proof data once decision is finalized)

### Challenges
What we do
- attempt decision on the challenged claim
- use the failed decision to submit counterclaim



## Next Steps
- [x] Add getAllQuantifiedBy
- [x] Write up a plasma exitable decision example
- [x] "my signatures don't exist"
- [x] State Channel Exit Property
- [x] Disputes (get smallest evidence possible to invalidate a claimed property)
- [ ] Storing 2 classes of on-chain events:
    - [ ] claims/finalizations
    - [ ] challenges
- [ ] Actually sending/transferring
- [ ] *PRIORITIZED* ISSUES BABY

- [ ] [low priority--only implement after we have a working OVM] Generate two sided implecation proofs. This is relevant when on chain we have decided a property that contradicts a new claim. You must show that the decided property implies a particular contadiction of the new claim.

## Issue Brainstorming

### Basic OVM Implementation
- OVM Interfaces
    - Decider Interface:
        - decide()
            - Put note in method header that this should store decisions that are able to be made
        - checkDecision()
        - Related Interfaces:
            - Decision
    - Proofs
        - Proof
        - ProofElement
    - Properties
        - Property
    - Implication Proofs
        - ImplicationProof
        - ImplicationProofElement
    - Quantifier Interface
        - `getAllSuchThat(parameters): AllSuchThatResult`
        - `interface AllSuchThatResult = { results:  any[], isCertainlyAll: boolean }`
    - PropertyFactory
        - getProperty(any): Property

     
- Implement HashPreimageExistenceDecider
    - HashPreimageExistenceInput (includes a hash image)
    - HashPreimageExistenceWitness
    - Implement decide() & checkDecision()
- Implement StateManager.ingestProof()
    - Remove ingestHistoryProof()
    - Generate a proof of multiple preimage exists properties
    - Ingest this proof, checking that all preimage exists properties are now set as decided.
- Implement AND Decider
    - Accept two properties & decide `true` if both of those properties are decided true. Decides `false` if either `false`, decides `UNDECIDED` if neither false and at least one `UNDECIDED`
    - ANDInput
    - ANDImplicationWitness (= `left` or `right`, whichever returned false)
- Implement NOT Decider
    - NOTDeciderInput
    - Accepts a property & decides `true` if that property is decided false. Decides `UNDECIDED` if the property returns `UNDECIDED`.
- ForAllSuchThatDecider
    - ForAllSuchThatInput
        - quantifier
        - quantifierParameters
        - propertyFactory
    - decides false if any of the results gets decided false, only decides true if all results decided true `&& isCertainlyAll`
    - if false, pushes the `AllSuchThatResults.results[i]` which failed to the `implicationProof` witness
- Add Simple Quantifiers
    - IntegerLessThan
    - IntegerRange
- Integration Test: Preimage Existence on a range of hash images
    - uses IntegerRangeQuantifier and the PreimageExistenceDecider to make decisions on ranges of preimages
    - uses a PropertyFactory which contains a list of `hashImages: bytes32[]` and constructs a PreimageExists Property with input as `hashImages[result]` via `for (result of IntegerRangeQuantifier.getAllSuchThat(...).results)`
- IntervalTreeDB
    - getLeaves(rootHash, range) => RangeEntry
        - Does something like rangeDB.bucket(key).get(interval)
    - putLeaf(rootHash, range, dataBlock) => void
        - Does something like rangeDB.bucket(key).put(interval, val)
- IntervalTreeInclusionDecider`<T>` implements Decider
    - IntervalTreeInclusionInput`<T>`
        - rootHash: bytes32
        - interval: Range
        - dataBlock: T
    - IntervalTreeInclusionWitness
        - inclusionProof: IntervalInclusionProof
    - decide(input: IntervalTreeInclusionInput, witness): Decision
        - check that inclusion proof is valid and matches input.rootHash & input.interval
        - if decided, store DataBlock in intervalTreeDB.putLeaf(rootHash, interval, dataBlock)
    - checkDecision(input: IntervalTreeInclusionInput): Decision
        - results = intervalTreeDB.getLeaves(input.rootHash, input.interval)
        - if results is [] / undefined (TBD), return `undecided`
        - else `(return result[0].range == input.interval && result[0].dataBlock == input.dataBlock)`
- IntervalTreeIntersectionQuantifier
    - IntervalTreeIntersectionParameters
        - rootHash: bytes32
        - intersectionInterval: Range
    - getAllQuantified
        - results = intervalTreeDB.getLeaves(parameters.rootHash, parameters.intersectionInterval)
        - Make sure no gaps --> areNoGaps (implemented above with `rangesSpanRange(...)`)
        - return { results: results, isCertainlyAll: areNoGaps }
- Integration Test: checking the existence of preimages which were commited to an interval tree intersecting some range
    - Same factory scheme as above integration test, takes all the hash images committed in a data block and then generates a PreimageExists property based on that
    - Property would look something like:
```typescript=
intervalHashCommitmentRevealProperty = {
    predicate: ForAllSuchThatDecider,
    input: {
        quantifier: intervalTreeIntersectionQuantifier,
        parameters: {
            rootHash: BLOCK_ROOT_HASH,
            intersectionRange: RANGE_TO_CHECK,
            propertyFactory: (dataBlock: bytes32) => {
                return {
                    predicate: preimageExistsDecider,
                    input: {
                        hashImage: dataBlock
                    }
                }
            }
        }
    }
}
```
- SignedByDecider
    - allows us to verify existence of and store a signature by some pubkey.
    - `type SignedByInput = {pubkey: address, message: bytes}`
    - type `type SignedByWitness = {v, r, s}` (standard ECDSA signature type)
    - stores in a DB indexed by `[pubkey][message]`
- SignedByQuantifier
    - analagous to the inteval tree intersection quantifier, in that it interacts with the db used by `SignedByDecider`
    - communicates with wallet so that a `SignedByQuantifier` can check whether the `input.signer == MY_ADDRESS`, allowing it to return `true` for `isCertainlyAll`
- Integration test with signatures: simple state channel
    - Uses the property class for state channels as determined above



### Aggregator Utilities (CLI, Deployment, etc.)

- CLI with mocked Ethereum calls
    - Keygen & import
    - Aggregator Registry deployment
        - This deploys a new aggregator registry--done once per L1 chain.
        - Should be skipped if an aggregator registry address is added to config.
    - Aggregator registration
        - This includes an aggregator metadata smart contract which stores all of the aggregator specific configuration like the IP address of the aggregator, deposit contracts which the aggregator accepts (different tokens)
    - Add new token deposit addresses
- Allow write to config file with new aggregator information (eg. Aggregator Registry contract)
    - Do you want to default Aggregators created in the future to be registered with this Registry? Default yes
- Allow aggregator metadata contract address to be imported from config


### Ethereum Integration
- Communicate with L1 (should be generic if possible to support different L1s)
    - Subscribe to L1 events
    - Submit TXs to L1
- Marshall events to internal types
- Internal message Pub/Sub
    - Make reasonably generic so we can add different transport layers


### Basic LevelDB Public Message Queue (Mocked P2P Message Transfer network)
- Imagined Network Interfaces:
    - NetworkMessageSender
        - submitMessage(...)
    - NetworkMessageHandler
        - messageReceived(...)
- Description:
    - Takes in a NetworkMessageHandler and implements NetworkMessageSender
        - Used to send messages, periodically polls for messages and invokes member NetworkMessageHandler with results
- Note: If there is an open source package which implements this functionality then consider using it. However, it should be quite easy to reimplement.

### Basic Block History Server
- Description:
    - Accepts requests which ask for particular ranges spanning a number of blocks
    - NOTE: 
        - Block history is stored in new range bucket for each block.
        - For each range included, we store a BlockRangeContents which is defined by:
            - BlockRangeContents = { dataBlock, inclusionProof } 
        - Notably this is storing redundent data as many of the inclusionProofs will be nearly identical. However, this makes querying easier & reduces the number of DB reads.
    - Requests call getHistoricalRanges(startBlock, endBlock, range) => BlockRangeQueryResults
        - BlockRangeQueryResults = BlockRangeQueryResult[] 



### Some questions

#### Decision Caching
One of the main difficulties we face is how to structure the storage of decisions, especially "meta" decisions like the ForAllSuchThat decisions. The following is a proposal for how to deal with this complexity that builds on what Ben said about "monotonically increasing" decisions--basically keeping an invarient that decisions can never become False after deemed True.

If we have the property that a decision once made can never be reverted, then we can build meta decisions on top which themselves can never be reverted. This would allow us to very simply store decisions like "for all blocks before n all prior states are deprecated"--it reframes these queries as decisions themselves.

The complexity comes in when we have a decision like "This is the highest nonce in our state channel" but then we sign a message which no longer makes it the highest nonce.

A way to keep our decisions immutable even in this circumstance is by reframing our decision. These very "application specific" decisions can take into consideration a concept of the "local message chain" which stores messages a user signed at a particular time.

With this we can decide "at signed message #5, there is no higher nonce". Then when interpreting our decisions we can check if we have any locally signed messages which are not included in our last decision, and then requery for our result if we've signed a new message. Then we make use of all of our piror decisions.

TLDR: Make sure no decision can be reverted, and then structure your decisions in a way which holds this property, and then we don't have to deal with one of the two hard problems in computer science.

NOTE: This does mean that if our liveness assumption for some reason fails and the UDC (universal decision contract) decides on an invalid value, there's something weird where our "never change a true value to false" bit doesn't hold. That said, we can fake it for a bit before figuring that one out.
