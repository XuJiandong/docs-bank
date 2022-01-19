# RFC: Cardano Lock


## Introduction

Cardano lock is a new lock script which can be unlocked by [cardano](https://en.wikipedia.org/wiki/Cardano_(blockchain_platform)) signature.
It follows the method described in [cip8](https://cips.cardano.org/cips/cip8/). A public-key cryptography, [ED25519](https://en.wikipedia.org/wiki/EdDSA#Ed25519) is required in Cardano lock.

## Cardano Lock Script

A Cardano Lock script has the following structure:

```
Code hash: Cardano Lock script code hash
Hash type: Cardano Lock script hash type
Args: <21 byte auth>
```
An auth is a 21-byte long data structure described in [Omni Lock](https://github.com/XuJiandong/docs-bank/blob/master/omni_lock.md#auth).
In cardano lock, 1 byte flag in auth is 0x7. The 20 bytes auth content represents the blake160 hash of an 32 bytes ED25519 public key.

## Witness

When unlocking a Cardano lock, the corresponding witness lock must be a 96 bytes data structure:

```
<ED25519 pubkey, 32 bytes><ED25519 signature, 64 bytes>
```

The auth content in lock script args should be identical to the blake160 hash of ED25519 pubkey in witness lock. 


Then script performances same routine used by CKB to get a message described in [CKB System Scripts](https://github.com/nervosnetwork/ckb-system-scripts/blob/e08e6016f16072fc2f44cf889ae063fa5b7e10c7/c/secp256k1_blake160_sighash_all.c#L151-L219). 
It hashes tx_hash, witness length, witnesses to get a 32 bytes `message`.

The `cip8` requires a message to be boxed into a `new message` described in [Message Signing Example](https://github.com/Emurgo/message-signing/blob/d6736d3e97e58648b8585c7dabdaac5870adae30/examples/rust/src/main.rs#L13-L32). It uses [CBOR](https://en.wikipedia.org/wiki/CBOR) format.

Finally, the Cardano lock script performances ED25519 signature verification with `new message`, pubkey and signature in witness lock. The verification result is the final result of cardano lock script.

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
            args: <flag: 0x7> <blake160 hash of ED25519 pubkey 1, 20 bytes>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        pubkey: <ED25519 pubkey 1, 32 bytes>
        signature: <ED25519 signature, 64 bytes>
      <...>
```
