Miniflow
===

Data
---

### Base Types

Miniflow types are defined as RLP-encoded recursive lists of byte arrays, which are interpreted in one of a few ways:

* `WORD` is a nullable 256-bit unsigned integer represented by a variable number of bytes, from 0 to 32. 
It is not a variable-length encoding. The length is given by the surrounding RLP encoding. It is a `uint` with N bytes, where 0 bytes means null. There are additional validation checks requiring some fields to be non-null which are documented for those fields.
* `BLOB` is a big array of bytes. The maximum size is defined per-field. Again, the length is given by the surrounding RLP encoding, and again a length of 0 is a null blob. Max blob size is defined per-field.
* `HASH` is a 32-byte blake2b hash (`Blake2b.Sum256`). `HASH(var1, var2)` is a hint about how the hash is constructed, but the data type is still a 32-byte hash.
* `PUBKEY` is a 32-byte public key
* `SIGNATURE` is a 64-byte signature


### UTag and Input

A `UTag` is a pair consisting of `HASH, VARNUM`. It is an index into some hash-addressed data type.

```
// UTXO Tag
UTag: RLP(
  0 / actID : HASH(Action)
, 1 / index : WORD
)
```

`Input` is a kind of `UTag` used in the list of `inputs` in an [Action]().

```
// Familiar alias for RTXI case
Input = UTag
```

`UTag`s are more general than "Inputs", but the distinction is not meaningful in miniflow.

### Output

```
Output: RLP(
  0 / left    : WORD    1 to 32 bytes
, 1 / right   : WORD    1 to 32 bytes
, 2 / odata   : BLOB      0 to 2^16 bytes 
, 3 / quorum  : WORD    0 to 1 bytes
, 4 / pubkeys : [Pubkey]  max 255 * sizeof(Pubkey)
)
```

* `O.left` and `O.right` are detailed in [interval validation]().
* `odata` is arbitrary extra data with a maximum size of 64KiB.
* `quorum` is the **key quorum**. This is the multisig quorum for the pubkeys in `pubkeys`.
* `pubkeys` is the list of public keys.


### Action

```
Action: RLP(
  0 / validSince : WORD
, 1 / validUntil : WORD
, 2 / inputs     : [Input]
, 3 / outputs    : [Output]
, 4 / signatures : [SIGNATURE(HASH(RLP(validSince,...,pubkeys)))] // sign hash of above fields RLP encoded in order (not raw prefix)
, 5 / mdata      : BLOB // can be modified by block producer
)
```

* `validSince` gives the earliest time (header.time) that this action may be included in a block
* `validUntil` gives the first millisecond at which this is action may no longer be included in a block
* `inputs` is the list of `UTags` that must be unspent and are consumed by this action.
* `outputs` is a list of `Outputs`
* `signatures` is a list of signatures. Note they sign the 'action mixhash' which is 1) hash of RLP encoded (not concatenated) fields and 2) does NOT contain `mdata`.
* `mdata` is any data and can be modified by the miner, which "malleates" the transaction. The concept of "mixhash vs confirmation hash" is fundamental and cannot be collapsed into "transaction hash".

### Header

```
Header: RLP(
  0 / prev : HASH(Header)  
, 1 / root : HASH(...)
, 2 / xtra : WORD
, 3 / node : PUBKEY
, 4 / time : BIGNUM
, 5 / fuzz : WORD
, 6 / work : HASH(mixhash, fuzz)
)
```

* `prev` is the previous block `headID`.
* `root` is the action merkle root.
* `xtra` is 0-32 bytes that are ignored by validation
* `mixhash = CONCAT(prev, root, xtrb, node, time, fuzz)`
* `node` is a public key
* `time` is a timestamp
* `fuzz` is 1-32 bytes of nonce data for the proof-of-work
* `work` is the proof-of-work

### Block

A block is a header and a list of actions.
A block is identified by the header's [headID](), which contains the [action merkle root]().

```
Block: RLP(
  0 / header  : Header
, 1 / actions : [Action]
)
```

Validation
---

This state definition and pseudocode describe a sequential validation algorithm. 

### Chain State

The chain state is best understood as an immutable data structure providing key->value maps, with references to snapshots of this state saved for each block header.
The parenthesized header argument `(Header ->)` at the start of some signatures is included to emphasize that each lookup has an implicit 'atBlock' argument, even though it is normally ignored for clarity.

```
// These are proper hash tables, and can be defined without regard for state or validation. The state itself has only references to hashes.
allHeaders: HeaderHash -> Header
allBlocks: HeaderHash -> Block  (indirect hash ID)
allActions: ActionHash -> Action

// This is a map from a UTXO to which header it was confirmed in. (*In this branch*. In different histories, a transaction may have been confirmed in different blocks.)
utxos: (Header ->) UTXO -> HEADER  

// This is a set of which headers are in the chain. (*In this branch*.)
pastHeaders: (Header ->) -> Header -> bool
```

### State -> Block -> State

This code takes a block, and inserts it after its parent in the blocktree if it is valid.
It is not concerned with determining which of many valid states is the 'latest'.
Again, it is an 'implementation', and there is a parallel algorithm that is equivalent.

```
let block be given
let header = block.header

prevHeader = header.prev
FAIL IF prevHeader not in allHeaders // No known previous header
FAIL IF prevHeader not in allBlocks // No known block for header - maybe a thin client?

FAIL IF header.time > clock() // wall clock time
FAIL IF header.time <= prevHeader.time

let mixHash = hash(...) // see data definitions
FAIL IF header.work != hash(mixHash, header.fuzz)

FAIL IF header.work > prev.header.work * (3 / 2) // literal numeric values ("must be at least 2/3 as hard")

FAIL IF header.actroot != block.actions.merkelize() // regular (dense binary) merkle tree from list


FOR (action) IN (block.actions)
    FAIL IF action.validSince > header.time
    FAIL IF action.validUntil <= header.time

    FAIL IF requireHeader not in pastHeaders // not allHeaders, *this branch's* headers

    FAIL IF any signature fails from action.signatures // not related to spending logic, reject invalid signatures outright
    // Note which fields are signed -- nodes can mutate extraData (`mdata`) on transactions. But whole trx is merkelized.

    var garbageIn = 0
    var intervals = {} // this interval matching itself be parallelized internally for huge transactions
    FOR (input) IN (action.inputs)
        let inAction = allActions(input.actID)
        let inOutput = inAction.outputs[input.idx]

        check key quorum (multisignature)
        check locks quorum (tag must not exist in UTXO)
        check needs quorum (tag must exist in UTXO)

        if left == 0
            garbageIn += (right - left)
        intervals[inOutput.right] = inOutput.left // left has multiple values at 0, so map this way
        STATE: REMOVE UTXO(input) // utag

    var garbageOut = 0
    var outervals = 0
    FOR (output) IN (action.outputs)
        FAIL IF output.right <= output.left
        FAIL IF data.size > (output.right - output.left) // a novelty? or a real thing?

        outervals[output.right] = output.left

        STATE: ADD UTXO(output, header.hashID())

    checkOnlySplit(intervals, outervals) // aware of slide-to-zero / garbage
    checkNetZero(intervals, outervals)
    DHT: ADD ACTION

STATE: ADD HEADER (this branch's set ref)
DHT: ADD HEADER
DHT: ADD BLOCK
```
