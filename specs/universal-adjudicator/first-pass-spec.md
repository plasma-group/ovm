# Predicate OVM Enforcement

## Universal Decision Contract
Optimistically manages global decisions on properties of the state.
### Structs
```typescript=
Property = {
    predicate: address, // address delgated to decide properties true or false and whether other properties are their implications
    input: bytes // specifies a particular property via the input to its predicate 
}
Contradiction = [Property, Property]
ClaimStatus = {
    property: Property,
    numProvenContradictions: uint,
    decidedAfter: uint
}

ImplicationProofElement = {
    implication: Property, // property to verify is a implication of some premise (e.g. the root premise)
    witness: bytes[] // arbitrary proof that the `implication` is valid
}

ImplicationProof = ImplicationProofElement[]
```
### Public Variables
```typescript=
DISPUTE_PERIOD: uint // how long it takes us to decide
claims: mapping(bytes32, ClaimStatus) // status of claims at propertyId
contradictions: mapping(bytes32, bool) // whether a given contradiction is unresolved
```
### Contract
```typescript=
claimProperty(_claim: Property) {
    // get the id of this property
    claimedPropertyId: bytes32 = getPropertyId(_claim)
    // make sure a claim on this property has not already been made
    require(claims[claimedPropertyId] == EMPTY_CLAIM)
    // create the claim status. Always begins with no proven contradictions
    status: ClaimStatus = {
        property: _claim,
        numProvenContradictions: 0,
        decidedAfter: block.number + DISPUTE_PERIOD
    }
    // store the claim
    claims[claimedPropertyId] = status
}

decideProperty(_property: Property, _decision: bool) {
    // only the predicate can decide a claim
    require(msg.sender == _claim.predicate)
    _decidedPropertyId: bytes32 = getPropertyId(_property)
    if (_decision) {
        // NOTE: In our model, ExistsSatisfying is the only predicate authenticated to make decisions (it's the only contract which can make a call to this function `decidePropertyt`)
        // if the decision is true, automatically decide its claim
        claims[_decidedPropertyId].decidedAfter = block.number - 1
    } else {
        // then decision is false--delete its claim!
        delete claims[_decidedPropertyId]
    }
}

verifyImplicationProof(_rootPremise: Property, _implicationProof: ImplicationProof): bool {
    if (_implicationProof.length == 1) {
        // properties are always implications of themselves
        return _rootPremise == _implicationProof[0].implication
    }
    // Check first implication. (i.e. with the rootPremise)
    require(isWhitelistedProperty(_rootPremise)) // make sure all properties are on the whitelist
    require(_rootPremise.predicate.verifyImplication(_rootPremise.input, _implicationProof[0]))
    for (const i = 0; i < _implicationProof.length - 1; i++;) {
        premise: ImplicationProofElement = _implicationProof[i]
        implication: ImplicationProofElement = _implicationProof[i+1]
        require(isWhitelistedProperty(premise))
        // If this is the implication's conclusion property, also check that it is in fact whitelisted
        if (i == _implicationProof.length - 1) {
            require(isWhitelistedProperty(implication))
        }
        require(premise.predicate.verifyImplication(premise.input, implication))
    }
}

verifyContradictingImplications(_root1: Property, _implicationProof1: ImplicationProof, _root2: Property, _implicationProof2: ImplicationProof, _contradictionWitness: bytes): bool {
    require(verifyImplicationProof(_root1, _implicationProof1))
    require(verifyImplicationProof(_root2, _implicationProof2))
    implication1: Property = _implicationProof1[_implicationProof1.length - 1].implication
    implication2: Property = _implicationProof2[_implicationProof2.length - 1].implication
    // NOTE: if not whitelisting at the top level, we would have to
    // verify that both implications consider each other to contradict.
    require(implication1.predicate.verifyContradiction(implication1, implication2, _contradictionWitness))
}

proveClaimContradictsDecision(
    _decidedProperty: Property,
    _decidedImplicationProof: ImplicationProof
    _contradictingClaim: Property,
    _contradictingImplicationProof: ImplicationProof
    _contradictionWitness: bytes
) {
    decidedPropertyId = getPropertyId(_decidedProperty)
    contradictingClaimId = getPropertyId(_contradictingClaim)
    // make sure the decided claim is decided
    require(isDecided(decidedPropertyId))
    // make sure the two properties do in fact contradict oneanother
    require(verifyContradictingImplications(_decidedProperty, _decidedImplicationProof, _contradictingClaim, _contradictingImplicationProof, _contradictionWitness)

    // Delete the contradicting claim
    delete claims[contradictingClaimId]
}

proveUndecidedContradiction(
    _contradiction: Contradiction,
    _implicationProof0: ImplicationProof,
    _implicationProof1: ImplicationProof,
    _contradictionWitness: bytes
) {
    // get the unique ID corresponding to this contradiction
    contradictionId: bytes32 = getContradictionId(_contradiction)
    propertyIds = [getPropertyId(_contradiction[0]), getClaimId(_contradiction[1])]
    // Make sure both claims have been made and not decided false
    require(claims[propertyIds[0]] != EMPTY_CLAIM && claims[propertyIds[1]] != EMPTY_CLAIM)
    // make sure the contradicting properties have contradicting implications
    require(verifyContradictingImplications(_contradiction[0], _implicationProof0, _contradiction[1], _implicationProof1, _contradictionWitness))
    // increment the number of contradictions
    claims[propertyIds[0]].numProvenContradictions += 1
    claims[propertyIds[1]].numProvenContradictions += 1
    // store the unresolved contradiction
    contradictions[contradictionId] = true
}

removeContradiction(_contradiction: Contradiction, remainingClaimIndex: {0, 1}) {
    // get the claims and their Ids
    remainingClaim = _contradiction[remainingClaimIndex]
    remainingClaimId = getPropertyId(remainingClaim)
    falsifiedClaim = _contradiction[!remainingClaimIndex]
    falsifiedClaimId = getPropertyId(deledetClaim)
    // get the contradiction Id
    contradictionId = getContradictionId(_contradiction)
    // make sure the falsified claim was decided false
    require(claims[falsifiedClaimId] == EMPTY_CLAIM)
    // make sure the contradiction is still unresolved
    require(contradictions[contradictionId])
    // resolve the contradiction
    conlicts[contradictionId] = false
    // decrement the remaining claim numProvenContradictions
    claims[remaningClaimId].numProvenContradictions -= 1
}
```

## Predicate Contracts

### WitnessExists
Note on this predicate: it is possible to decide instantly that some data exists after a previous decision was made that it does not.  This sounds like a problem, but it is not--any time cryptoeconomic games are contingent on decisions that some witness does not exist, they will decide this because an interested party does not want to reveal the witness.  Therefore, in practice, we should not see cases in which deciding a witness doesn't exist is "overwritten"--because the it is not in the interest of the party which can produce the witness to publicize it.  If we are concerned about this problem for some game, then we simply remove the ability to make instant decisions on `exists == TRUE`.


```typescript=
WitnessExistsPredicateInput = {
    verifier: address,
    parameters: bytes
}

contract WitnessExistsPredicate {
    decideWitnessExistsTrue(_input: WitnessExistsPredicateInput, 
    _witness: bytes) {
        // make sure a NOT exists has not already passed the finality window
        require(_input.verifier.verify(_input.parameters, _witness))
        decidedExistsProperty: Property = {
            predicate: this.address,
            input: _input
        }
        MANAGER.decideProperty(decidedExistsProperty, true)
    }

    decideWitnessExistsFalse(
        _existsClaimInput: WitnessExistsPredicateInput,
        _decidedRootPremiseImplyingNOTExists: Property,
        _implicationProof: ImplicationProof
    ) {
        // get the property objects for exists and NOT exists
        existsClaim = {
            predicate: self.address,
            input: _existsClaimInput
        }
        doesNOTExistProperty = {
            predicate: NOT_ADDRESS,
            input: existsClaim
        }
        // verify we got a valid root premise which implies the NOT(exists) property
        MANAGER.verifyImplicationProof(_decidedRootPremiseImplyingNOTExists, _implicationProof)
        require(_implicationProof[_implicationProof.length - 1] == doesNOTExistProperty)
        // get Id of the root premise
        _rootPremiseId = MANAGER.getPropertyId(_decidedRootPremiseImplyingNOTExists)
        // if the root claim dependent on a NOT Exists, and the root claim has not been deleted, we take that as a decision on non-existence.
        // If the claim status is non-empty and past its dispute window...
        if (block.number >= MANAGER.claims[_rootPremiseId].decidedAfter) {
            // ...make the decision that existence is false
            existsClaimId = MANAGER.getClaimId(existsClaim)
            MANAGER.decideProperty(existsClaimId, false)
        }
    }

    verifyContradicts(_claim0: Property, _claim1: Property, _contradictionWitness: bytes)) {
        if (isNOTOfOther(_claim0, claim1)) {
            return true
        } else {
            return false
        }
    }
}

```

### NOT Predicate
Predicate returning true if a given property is false.

```typescript=
NOTPredicateInput = {
    isNOTPropertyOn: Property
}

NOTContradictionWitness = {
    NOTIndex = {0, 1}
}

contract NOTPredicate {
    verifyContradiction(_claim0: Property, _claim1: Property, _contradictionWitness: NOTContradictionWitness)) {
        // a NOT property conflicts with a base property if notProperty.input == baseProperty
        bothClaims = [_claim0, _claim1]
        baseClaim = bothClaims[!_contradictionWitness.NOTIndex]
        NOTBaseClaim = bothClaims[_contradictionWitness.NOTIndex]
        return ((NOTBaseClaim.predicate == this.address) && (NOTBaseClaim.input == baseClaim))
    }
    // The premise of NOT(P) implies P' if and only if P' == NOT P.
    // In other words, we allow NOT(NOT(P)) implies P.
    verifyImplication(
        _thisClaimData: NOTPredicateInput, _implication: Property)
        if (_thisClaimData == {
            propertyAddress: NOT.address, // "the inner NOT"
            propertyData: _implication
        }) {
            return true
        } else {
            return false
        }
}
```

### AND Predicate
AND is a predicate whose inputs are two properties. It implies both of its inputs.
```typescript=
ANDPredicateInput = [Property, Property]

ANDImplicationWitness = 0 | 1

verifyImplication(_ANDData: ANDPredicateInput, _implicationProofElement: ImplicationProofElement): bool {
    implication =  _implicationProofElement.implication
    indexInAND: ANDImplicationWitness = _implicationProofElement.witness
    return (_ANDData[indexInAND] == implication)
}

decideTrue(_leftDecidedRootProperty: Property, _leftImplicationProof: ImplicationProof, _rightDecidedRootProperty: Property, _rightImplicationProof: ImplicationProof) {
    // if a decided property implies the left side of an AND, and another deicded property implies the right side,
    // Then we can decided on AND(left, right).
    // TODO: write this functionality (non-critical ATM).
}
```

### Universal Quantifier Predicate
And important predicate is the universal quantifier "for all".  Its input is a quantifier function `quantify(_property: Property): bool` which return true or false on a property.  The universal quantifier predicate implies all properties satisfying this quantifier. It's like taking an AND on all properties `prop` satisfying `quantify(prop) == true`


```typescript=
QuantifierPredicateInput = {
    quantifier: address,
    parameters: bytes
}
```
(Note from above that in practice we do not just check `quantifier(prop)`, we call `quantifier(params, prop)` as this is more expressive.)`

```typescript=
contract UniversalQuantifierPredicate {
QuantifierImplicationWitness

    verifyImplication(_quantifierInput: QuantifierPredicateInput, _implicationProofElement: ImplicationProofElement): bool {
        quantifier = _quantifierInput.quantifier
        parameters = _quantifierInput.parameters
        implication = _impicationProofElement.implication
        return (quantifier.quantify(parameters, implication))
    }
}
```

#### Quantifier Contracts
This section outlines example quantifier contracts which will be inputs into the universal quantifier predicate.

##### Predicate Type Quantifier
This quantifier quantifies whether or not a given property `prop` has a given predicate address.
```typescript=
contract PredicateTypeQuantifier {
    quantify(_parameters: address, _toQuantify: Property) {
        return (_parameters == _toQuantify.predicate)
    }
}
```

##### AND Quantifier
This quantifier accepts two sub-quantifiers and returns the boolean AND on each sub-quantifier's `.quantify`.  This is one way to compose quantifiers.

```typescript=
ANDQuantifierParameters = {
    subQuantifier0: address,
    subParameters0: bytes,
    subQuantifier1: address,
    subParameters1: bytes
}

contract ANDQuantifier {
    quantify(_parameters: ANDQuantifierParameters, _toQuantify: Property) {
        // get the first subquantifier's output
        quantification0 = _parameters.subQuantifier0.quantify(_parameters.subParameters0, _toQuantify)
        // get the second subquantifier's outut
        quantification1 = _parameters.subQuantifier1.quantify(_parameters.subParameters1, _toQuantify)
        // return the boolean AND of each quantification
        return (quantification0 && quantification1)
    }
}
```

#### Less Than Quantifier
The less than quantifier takes as its parameters a numeric `upperBound` and a `reducer(_toReduce: Property): number` function which accepts a property to reduce into a numeric value.  The less than quantifier returns true if `reducer(property) < upperBound`.

```typescript=
LessThanQuantifierParameters = {
    reducer: address,
    upperBound: uint
}

contract LessThanQuantifier {
    quantify(_parameters: LessThanQuantifierParameters, _toQuantify: Property) {
        // reduce the property down to a single value
        valueToQuantify = _parameters.reducer.reduce(_toQuantify)
        // return the boolean expression of whether this reduction is less than our upper bound.
        return valueToQuantify < _parameters.upperBound
    }
}
```

One example reducer is one which returns the "index of preimage claimed to exist" that may be an input to a "WITNESS_EXISTS(verifyNthPreimage, N)" property. This could be implemented as:
```typescript=
contract IndexOfPreimagesRevealedReducer {
    reduce(_toReduce: Property) {
        return _toReduce.input.indexOfPreimageRevealed
    }
}
```

#### Reducer Equality Quantifier
Applies two reducers `reducer0` and `reducer1` and returns true if `reducer0(prop) == reducer0(prop)`

```typescript=
ReducerEqualityQuantifierParameters = {
    reducer0: address,
    reducer1: address
}

contract ReducerEqualityQuantifier {
    quantify(_parameters: ReducerEqualityQuantifierParameters, _toQuantify: Property) {
        reduced0 = _parameters.reducer0.reduce(_toQuantify)
        reduced1 = _parameters.reducer1.reduce(_toQuantify)
        // return the boolean expression of whether these two reductions are equal.
        return reduced0 == reduced1
    }
}
```


## Examples
### Preimage Reveal
We can construct a claim that a given preImage exists. This uses the `WITNESS_EXISTS` predicate with a `verifier` that checks whether the hash of a witness equals its parameter.

#### Verifier Contract
```typescript=
PreimageVerifierParameters = {
    hashImage: bytes32
}

contract IS_PREIMAGE_VERIFIER {
    verify(_parameters: PreimageVerifierParameters, _witness: bytes): bool {
        return (hash(_witness) == _parameters.hashImage)
    }
}
```
#### Existence Claim
Then the following claim asserts that a preImage for some `HASH_IMAGE_0x123` exists:
```typescript=
PREIMAGE_EXISTS_CLAIM = {
    predicate: WITNESS_EXISTS_PREDICATE.address,
    input: {
        verifier: IS_PREIMAGE_VERIFIER.address,
        parameters: HASH_IMAGE_0x123
    }
}
```
This can be instantly decided by the `WITNESS_EXISTS` contract if the hash preimage is published by a prover. Or it will be decided true after the `DISPUTE_PERIOD` elapses and there are no contradicting claims.

#### Non-Existence Claim
We can claim that a given preImage does not exist for some `HASH_IMAGE_0x123` via the `NOT` predicate.  This claim is as follows:
```typescript=
PREIMAGE_DOES_NOT_EXIST_CLAIM = {
    predicate: NOT_PREDICATE.address,
    input: PREIMAGE_EXISTS_CLAIM
}
```
`PREIMAGE_DOES_NOT_EXIST_CLAIM` can be proven to contradict `PREIMAGE_EXISTS_CLAIM` since one is the NOT of the other.  If the preimage is published by a prover, then the decision `true` is made on the existence claim, and the `PREIMAGE_DOES_NOT_EXIST_CLAIM` claim can be removed via `MANAGER.proveClaimContradictsDecision()`.  If the preimage is not published by a prover, then we could have the following scenario: both claims are made, and shown to be an unresolved conflict.  However, if no existence decision is made, the `PREIMAGE_EXISTS_CLAIM` can be decided `false` with a call to `WITNESS_EXISTS.decideWitnessExistsFalse()`

### AND on preimage reveal
#### Two Preimage Non-Existence Claim
```typescript=
TWO_PREIMAGES_DONT_EXIST = {
    predicate: AND_PREDICATE.address,
    input: [
        {
            predicate: NOT_PREDICATE.address,
            input: { 
                predicate: WITNESS_EXISTS_PREDICATE.address,
                input: {
                    verifier: IS_PREIMAGE_VERIFIER.address,
                    parameters: HASH_IMAGE_0x123
                }
            }
        },
        {
            predicate: NOT_PREDICATE.address,
            input: { 
                predicate: WITNESS_EXISTS_PREDICATE.address,
                input: {
                    verifier: IS_PREIMAGE_VERIFIER.address,
                    parameters: HASH_IMAGE_0x456
                }
            }
        },
    ]
}
```
Because the AND predicate can verifyImplication for either of its input elements, we execute the same protocols as above to maintain safety on this claim's decision.


### FOR ALL LESS THAN of a list of commitments
There exists a list of commitments to images for which the preimage is not necessarily known upfront. We will then make a claim over a range of commitment's preimages which we assert exist.
#### Verifier Contract
```typescript=
IsNthPreimageParameters = {
    index: uint
}
contract IsNthPreimage {
    hashImages = [hashImage0: bytes32, hashImage1: bytes32, ...]
    verify(_parameters: IsNthPreimageParameters, _witness: bytes): bool {
        nthImage = hashImages[_parameters.index]
        return hash(_witness) == nthImage
    }
}
contract PropertyIndexReducer {
    reduce(_toReduce: Property) {
        return _toReduce.input.index
    }
}
```

#### Claim
```typescript=
PREIMAGE_RANGE_EXISTS_CLAIM = {
    predicate: UNIVERSAL_QUANTIFIER_PREDICATE.address,
    input: {
        quantifier: AND_QUANTIFIER.address,
        parameters: {
            subQuantifier0: LESS_THAN_QUANTIFIER.address,
            subParameters0: {
                reducer: PROPERTY_INDEX_REDUCER,
                upperBound: CLAIMED_UPPER_BOUND
            },
            subQuantifier1: PREDICATE_TYPE_CHECKER_QUANTIFIER,
            subParameters1: WITNESS_EXISTS_PREDICATE
        }
    }
}
```

### Preimage Reveal Payment Chain
We can trivially move from this range-based preimage reveal to a chain of transference of ownership. Instead of simply committing to `[hash1, hash2,...]`, we will commit to `[[hash1, newOwner1], [hash2, newOwner2],...]`. From this we create a "custody" contract which looks at which EXISTS_CLAIM was decided, and sends some payout to the corresponding owner.

For example, let's say we have the array `[[hash1, alice], [hash2, bob]]`. For Alice to optimistically transfer custody of the coin to Bob, all she must do is reveal `preimage` where `hash(preimage)==hash1`. Then Bob may claim that hash1's preimage exists, and hash2's preimage (Bob's preimage) does not exist. 


### Transition Property Plasma
If we have an array of properties `commitments: Property[]`, we can construct a plasma-like OVM where we return the earliest unsatisfied entry in `commitments` as the "exitable block."  We call each of these "transition properties," because they must be decidable true to transition to the next exitable block.

#### Commitment Contract
```typescript=
contract CommitmentChain {
    commitments = [
        // Block 0:
        [transition0: Property, transition1: Property, transition2: Property...transition_N],
        // Block 1:
        [transition0: Property, transition1: Property, transition2: Property...transition_N],
        // Block...n
    ]
    verifyInclusion(_transition: Property, blockNumber: uint, index: uint, witness: bytes): bool {
        return _transition == this.commitments[blockNumber][index]
    }
    addBlock(newBlock: Property[]) // ... adds a block to our commitments list
}
```

#### Transition Inclusion Verifier
This is used to verify that a transition is actually included in our commitment chain.
```typescript=
IsIncludedTransitionParameters = {
    block: uint,
    index: uint
}
contract IsIncludedTransition {
    verify(_parameters: IsIncludedTransitionParameters, _witness: bytes): bool {
        isIncluded = COMMITMENT_CHAIN.verifyInclusion(_witness.property, _parameters.block,  _parameters.index, _witness.inclusionProof)
        return isIncluded
    }
}
```

#### Exitable Quantifier
```typescript=
ExitableQuantifierParameters = {
    blockNumber: uint,
    coinId: uint
}

contract ExitableQuantifier {
    quantify(_parameters: ExitableQuantifierParameters, _toQuantify: Property, _implicationProofElement: ImplicationProofElement) {
        isAND: bool = _toQuantify.predicate == AND_PREDICATE.address
        leftIsExistenceAtCoinAndIndex = (
            _toQuantify.input[0].predicate == EXISTS_SATISFYING.address &&
            _toQuantify.input[0].input.verifier == TRANSITION_INCLUSION_VERIFIER
        )
        isOnCoinId = _toQuantify.input[0].input.parameters.coinId == _parameters.coinId
        isLessThanExitableIndex = _toQuantify.input[0].input.parameters.blockNumber < _parameters.blockNumber
        
        wasIncluded = COMMITMENT_CHAIN.verifyInclusion(_toQuantify.input[1], _parameters.blockNumber, parameters.coinId, _implicationProofElement.implicationWitness.inclusionProof) // Check that the property was included in a particular commitment
        return (isAND && isOnCoinID && isLessThanExitableIndex && wasIncluded)
    }
}
```

#### Claim

```typescript=
PlasmaExitableClaim = {
    predicate: UNIVERSAL_QUANTIFIER.address,
    input: {
        quantifier: EXITABLE_QUANTIFIER,
        parameters: {
            coinId: COIN_ID,
            blockNumber: BLOCK_NUMBER
        }
    }
}
```

#### Atomic Swap Transition Predicate

```typescript=
AtomicSwapTransitionInput = {
    desiredPostSwapTransitionQuantifier: address
}

contract CustomAlicePostSwapTransitionQuantifier {
    quantify(_toQuantify: Property): bool {
        require(MANAGER.isDecided(_toQuantify))
        require(isPlasmaExitableClaim(_toQuantify))
        require(willAliceGetTheExit(_toQuantify))
        return true
    }
}

AtomicSwapTransitionPredicate = {
    decideDecidedPostSwapExit(_input: AtomicSwapTransitionInput, _decidedProperty: Property) {
        passesQuantifier = _input.desiredPostSwapTransitionQuantifier.quantify(_decidedProperty)
        isDecided = MANAGER.isDecided(_decidedProperty)
        if (isDecided && passesQuantifier) {
            MANAGER.decideProperty(
                {
                    predicate: self.address,
                    input: _input
                },
                true
            )
        }
    }
}
```
