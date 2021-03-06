# RFC: Cardano Lock


## Introduction

Cardano lock is a new lock script which can be unlocked by [cardano](https://en.wikipedia.org/wiki/Cardano_(blockchain_platform)) signature.
It follows the method described in [cip8](https://cips.cardano.org/cips/cip8/). A public-key cryptography, [ED25519](https://en.wikipedia.org/wiki/EdDSA#Ed25519) is required in Cardano lock.

## Cardano Lock Script

A Cardano Lock script has the following structure:

```
Code hash: Cardano Lock script code hash
Hash type: Cardano Lock script hash type
Args: <header byte, 1 byte> <payment pubkey hash (Blake2b224), 28 bytes> <stake pubkey hash (Blake2b224), 28 bytes, optional>
```
The `args` in lock script represents two Blake2b224 hashes of 32 bytes ED25519 pubkey:  one is for payment and the other for stake.

In the header byte, bits [7;4] indicate the type of addresses being used; we'll call these four bits the header type. According to the
[Shelley Addresses](https://cips.cardano.org/cips/cip19/#shelleyaddresses), the header byte should be in following values:

- 0b0000
- 0b0010
- 0b0100
- 0b0110

For 0b0110 case, the stake pubkey hash part can be optional.


## Witness

When unlocking a Cardano lock, the corresponding witness lock must be a data structure in molecule format:

```
import blockchain;

table CardanoWitnessLock {
    pubkey: [byte; 32],
    signature: [byte; 64],
    sig_structure: Bytes,
}

```

The payment pubkey hash in lock script args should be identical to the blake2b hash of `pubkey` in `CardanoWitnessLock`. 


Then script performances same routine used by CKB to get a message described in [CKB System Scripts](https://github.com/nervosnetwork/ckb-system-scripts/blob/e08e6016f16072fc2f44cf889ae063fa5b7e10c7/c/secp256k1_blake160_sighash_all.c#L151-L219). 
It hashes tx_hash, witness length, witnesses to get a 32 bytes `message`.

The `cip8` requires a message to be boxed into a `Sig_structure` described in [Message Signing Example](https://github.com/Emurgo/message-signing/blob/d6736d3e97e58648b8585c7dabdaac5870adae30/examples/rust/src/main.rs#L13-L32). It uses [CBOR](https://en.wikipedia.org/wiki/CBOR) format.

A `payload` field should be extracted from `sig_structure` in `CardanoWitnessLock`. The `payload` should be identical to `message`. Otherwise, the verification fails.

Finally, the cardano lock script performances ED25519 signature verification with `sig_structure`, `pubkey` and `signature` in `CardanoWitnessLock` structure. The verification result is the final result of cardano lock script.

## Examples

```
CellDeps:
    <vec> Cardano Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Cardano Lock
            args: <header byte, 1 byte> <Blake2b224 hash of ED25519 pubkey 1 (payment), 28 bytes> <Blake2b224 hash of ED25519 pubkey 2 (stake), 28 bytes>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        pubkey: <ED25519 pubkey 1 (payment), 32 bytes>
        signature: <ED25519 signature, 64 bytes>
        sig_structure: <CBOR encoded, variable length>
      <...>
```
