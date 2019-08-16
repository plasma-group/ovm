# Decisions & Properties for Optimistic Rollup
## Rollup Components
- transactions specify input as all touched accounts, and some calldata: `tx = {inputs: address[], callData: bytes}`
- nested existence of `(contract state, inclusion proof)` in the pre-root, for each of the `tx.inputs`
- execution of transaction against each `contract state`, merkle insertion of each new `contract state`  using its `inclusion proof` results in the `post state`

## Attempt to write out the claim

$$\text{claimValidTransition}(preRoot, inputs, txData, postRoot):=  \\ 
\exists (incl_1,data_1), dataIncludedAt(inputs_1, data_1, preRoot, incl_1): \\ 
\exists (incl_2,data_2), dataIncludedAt(inputs_2, data_2, preRoot, incl_2): \\ 
\text{...} \\
\exists (incl_n,data_n), dataIncludedAt(inputs_n, data_n, preRoot, incl_n): \\ 
batchMerkleUpdate(executeTransaction(data_1,...,data_n, inputs, txData), incl_1,...incl_k) = postRoot
$$

**Note** // Correctly evaluating this on-chain requires the ability to say that $incl_1, data_1 ... incl_n, data_n$ are the only such ones possible, so that $merkleInsert(executeTransaction(...$ evaluating false will be sufficient to falsify.  In game semantics I think this is reflected in the fact that a prover is required to choose the $incl_1, data_1 ... incl_n, data_n$ themselves, which would be impossible.  Further work is required to figure this out on the contract side, but we should be able to move forward client-side regardless.

## Big-ticket Issues

### MerkleInclusion Quantifier
We need a quantifier which functions as the $dataIncludedAt$ in the above claim. Particularly, it would return some `includedData` included at a particular point in a vanilla Merkle tree (above, each are called $inputs_i$) for `getAllQuantified`, or none if an exclusion proof is relevant.

Specifically, we would have 
```typescript=
type MerkleIntervalInclusionParams =
    {
        root: bytes32,
        key: bytes32
    }
```
The `getAllQuantified` would return some generic `data: bytes` to be fed to a property factory.

Note that the length of `key` above should be parameterizable, i.e. 20 bytes for an address or 16 if we want to be efficient.  I think it's too complex and maybe infeasible to have different-length keys in a single tree, though.

### executeTransaction
A pure function which runs the `txData` according to our smart contracting rules. Accepts an array of `inputs` based on the transaction, and `data`--the data included at the `input`th position in the tree. Also accepts some arbitrary `txData` e.g. a signature.

Returns an array of `outputData[]` corresponding to the new state at each `input`.

```typescript=
// transaction accounts being used in a transaction.
type txInput = {
    account: address,
    data: any
}
// function being used to execute the transaction
function executeTransaction(
    inputs: txInput[],
    txData: any
): bytes[]
```

The behavior of this function should directly map to its on-chain equivalent, which should be built by the time this is being tackled.

Note // this could be implemented as a wrapper around some evm-js implementation. Possibly Kelvin did some work on this in the past.  For this MVP though, the functions should be pretty simple, so probably fine to simulate/rewrite in JS.


### batchMerkleUpdate
A pure function which performs a repeated set of merkle insertions/replacements to produce a new merkle root based on inclusion proofs for the elements to be replaced.  Accepts an array of `{position, newValue, inclusionProof}[]` to produce the new root.  Sometimes the `inclusionProof` is an exclusion proof.

```typescript=
// note: not sure if this is the right data type for append operations; the inclusion proof may be for an adjacent element in that case.
type individualMerkleUpdate = {
    key: uint256, // or whatever 2^tree height is decided.  uint128?
    newValue: bytes, //
    inclusionProof: MerkleInclusionroof
}
// batch merkle replace algorithm should be trickier than individual insertions--has to happen in one step.
function batchMerkleUpdate(
    insertions: individualMerkleUpdate[],
): bytes32 // new root to output
```

**NOTE** // There might be a better usage of types here to take advantage of, `individualMerkleUpdate` is a little ham-fisted. Interface should be changed as it becomes clearer during implementation.

### Equality Decider
If we don't have this already, we need an equality decider to compare a left and right input as bytes.

### Property Factories
The claim will utilize two property factories:
#### Nested Existence
Each of the $\exists$ above are nested within each other.  This means that, in practice, there is a propertyFactory which produces the nested array of these claims based on the `inputs`.  This will accept as input all of the params to the top-level claim.
#### merkleInsert-postRoot equality
Constructs an equality decision comparing the result of `merkleInsert` to the `postRoot`.  Accepts as input the postRoot as well as the inputs to `merkleInsert`, which it evaluates.

//note: all inclusion proofs above are assumed to include a `leafPosition` which specifies the tree path from the root to the leaf.
### Rollup State Manager
Although we can check the property for the batch merkle insertion and replacements, to generate/check existence of future state inputs, we need to have a helper to track the global state after each transaction.   This would involve performing the `batchMerkleUpdate` operation via not just inclusion proofs, but by modifying a full on-disk tree of all accounts.  This would be fed to the localInfoStore to be used by `MerkleInclusionQuantifier`
