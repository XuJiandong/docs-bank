# RFC: Cardano Lock


## Introduction

Cardano lock is a new lock script which can be unlocked by [cardano](https://en.wikipedia.org/wiki/Cardano_(blockchain_platform)) signature.
It follows the method described in [cip8](https://cips.cardano.org/cips/cip8/). A public-key cryptography, [ED25519](https://en.wikipedia.org/wiki/EdDSA#Ed25519) is required in Cardano lock.

## Cardano Lock Script

A Cardano Lock script has the following structure:

```
Code hash: Cardano Lock script code hash
Hash type: Cardano Lock script hash type
Args: <payment pubkey hash, 32 bytes> <stake pubkey hash, 32 bytes>
```
The `args` in lock script represents two blake2b hashes of 32 bytes ED25519 pubkey:  one is for payment and the other for stake.

## Witness

When unlocking a Cardano lock, the corresponding witness lock must be a data structure in molecule format:

```
import blockchain;

table CardanoWitnessLock {
    pubkey: [byte; 32],
    signature: [byte; 64],
    new_message: Bytes,
}

```

The payment pubkey hash in lock script args should be identical to the blake2b hash of `pubkey` in `CardanoWitnessLock`. 


Then script performances same routine used by CKB to get a message described in [CKB System Scripts](https://github.com/nervosnetwork/ckb-system-scripts/blob/e08e6016f16072fc2f44cf889ae063fa5b7e10c7/c/secp256k1_blake160_sighash_all.c#L151-L219). 
It hashes tx_hash, witness length, witnesses to get a 32 bytes `message`.

The `cip8` requires a message to be boxed into a `new message` described in [Message Signing Example](https://github.com/Emurgo/message-signing/blob/d6736d3e97e58648b8585c7dabdaac5870adae30/examples/rust/src/main.rs#L13-L32). It uses [CBOR](https://en.wikipedia.org/wiki/CBOR) format.

A `payload` field should be extracted from `new_message` in `CardanoWitnessLock`. The `payload` should be identical to `message`. Otherwise, the verification fails.

Finally, the cardano lock script performances ED25519 signature verification with `new message`, `pubkey` and `signature` in `CardanoWitnessLock` structure. The verification result is the final result of cardano lock script.

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
            args: <blake2b hash of ED25519 pubkey 1 (payment), 32 bytes> <blake2b hash of ED25519 pubkey 2 (stake), 32 bytes>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        pubkey: <ED25519 pubkey 1 (payment), 32 bytes>
        signature: <ED25519 signature, 64 bytes>
        new_message: <CBOR encoded new message, variable length>
      <...>
```
