Miniflow
===

Data
---

### Base Types

Miniflow types are defined as RLP-encoded recursive lists of byte arrays.
The base byte arrays are interpreted in one of a few ways:

* `WORD` is a nullable 256-bit unsigned integer represented by a variable number of bytes, from 0 to 32. 
It is not a variable-length encoding. The length is given by the surrounding RLP encoding. It is a `uint` with N bytes, where 0 bytes means null. There are additional validation checks requiring some fields to be non-null which are documented for those fields.
* `BLOB` is a big array of bytes. The maximum size is defined per-field. Again, the length is given by the surrounding RLP encoding, a length of 0 is a null blob, and the max blob size is defined per-field.
* `HASH` is a 32-byte blake2b hash (`Blake2b.Sum256`). `HASH(var1, var2)` is a hint about how the hash is constructed, but the data type is still a 32-byte hash.
* `PUBKEY` is a 32-byte public key
* `SIGNATURE` is a 64-byte signature

All timestamps are in **UTC milliseconds**.

### UTag and Input

A `UTag` is a pair consisting of `HASH, VARNUM`. It is an index into some hash-addressed data type.

```
// UTXO Tag
UTag: RLP(
  0 / actID : HASH(Action)
, 1 / index : WORD
)
```

`Input` is a kind of `UTag` used in the list of `inputs` in an [Action](Action).

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
, 2 / data    : BLOB    0 to 2^16 bytes 
, 3 / quorum  : WORD    0 to 1 bytes
, 4 / pubkeys : [Pubkey]  max 255 * sizeof(Pubkey)
)
```

* `left` and `right` are detailed in [interval validation]().
* `data` is arbitrary extra data with a maximum size of 64KiB.
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
, 5 / xtra       : WORD // can be modified by block producer, ignored by validation
)
```

* `validSince` gives the earliest time (header.time) that this action may be included in a block.
* `validUntil` gives the first time at which this is action may no longer be included in a block. This constraints confirmation, it is not related to expiration. An action (its set of UTags) can expire even if the `validUntil` time has not come.
* `inputs` is the list of `UTags` that must be unspent and are consumed by this action.
* `outputs` is a list of `Outputs`
* `signatures` is a list of signatures. Note they sign the 'action mixhash' which is 1) hash of RLP encoded (not concatenated) fields and 2) does NOT contain `mdata`.
* `mdata` is any data and can be modified by the miner, which "malleates" the transaction. The concept of "mixhash vs confirmation hash" is fundamental and cannot be collapsed into "transaction hash".

### Header

```
Header: RLP(
  0 / prev : HASH(Header)
, 1 / root : HASH(...)
, 2 / xtrb : WORD
, 3 / node : PUBKEY
, 4 / time : WORD
, 5 / fuzz : WORD
, 6 / work : HASH(mixhash, fuzz) where mixhash = CONCAT(prev, root, xtrb, node, time)
)
```

* `prev` is the previous block `headID`.
* `root` is the action merkle root.
* `xtrb` is 0-32 bytes that are ignored by validation
* `node` is a public key
* `time` is a timestamp
* `fuzz` is 1-32 bytes of nonce data for the proof-of-work
* `work` is the proof-of-work, `HASH(mixhash, fuzz)` where `mixhash = CONCAT(prev, root, xtrb, node, time)`.

##### HeadID

`headID` is `hash(Header)`, the hash of the entire RLP-encoded header.

### Block

A block is a header and a list of actions.
A block is identified by the header's [headID](HeadID), which contains the [action merkle root]().

```
Block: RLP(
  0 / header  : Header
, 1 / actions : [Action]
)
```

Validation
---

Some of these rules have special cases for block 0. It is simpler to imagine both block 0 and block 1 as part of the spec.

### Block

A block is valid if,

* The encoding is valid
* The actions are all valid
* The header is valid for action merkle root

### Header

A header is valid if:

* The encoding is valid
* The `prev` header exists
* The action merkle tree root equals `root`
* `xtrb` is ignored.
* `node` is ignored (during *header* validation. It is the miner's public key they will use to claim fees / expired objects).
* `time` must be greater than `headers[prev].time` and must not be in the future.
* `hash(mixhash, fuzz) == work`
* `work` is at least 2/3rds as difficult as prior work (less than 3/2 the numeric interpretation)

### Action

An action is valid if:

* The encoding is valid
* `validSince <= block.time`
* `block.time < validUntil`
* All inputs in `inputs` are valid
* All outputs in `outputs` are valid
* All signatures in `signatures` validate for `SIGN(HASH(RLP(validSince...pubkeys)))` of the entire transaction of inputs they consume. Any bad signature fails the transaction even if a quorum of good signatures exists.
* `xtra` is ignored by validation and can be malleated by miners.

Plus [interval validation](Intervals)

### Input

An input is valid if:

* The encoding is valid
* An input is valid if the `index` points into a valid output of the corresponding `actID`. This is validated during output evaluation.

### Output

An output is valid if:

* The encoding is valid
* `left` < `right`
* `data.size <= right - left  AND  data.size <= 2^16`  (`data.size` in bytes)
* `quorum <= 255`
* `pubkeys.length <= 255`


### Intervals

Intervals are the logical structures represented by output `left, right` pairs. Intervals come in two flavors:

* Fungible Garbage (if `left == 0`)
* Non-Fungible Sector (if `left != 0`)

Fungible garbage is a coin-like. The `right` represents the "amount". Two garbage regions can be joined and then split into regions of different sizes.

Sectors are non-fungible, non-intersecting `[left, right]` pairs. They can be split but NOT joined. They can be converted to fungible garbage, which sets `left = 0; right = right-left;`, but garbage cannot be converted back into sectors.

The sum of inputs and outputs (garbage plus non-garbage) in an action must be equal. The miner fee is always set explicitly with a zero-quorum output.

These rules are enforced by examining an action as a whole.

See [minibang]() to see the initial split of garbage / sectors.

Pseudocode (sketch / draft)
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

    // see interval validation section
    checkOnlySplit(intervals, outervals) // aware of slide-to-zero / garbage
    checkNetZero(intervals, outervals)

    DHT: ADD ACTION

STATE: ADD HEADER (this branch's set ref)
DHT: ADD HEADER
DHT: ADD BLOCK
```
