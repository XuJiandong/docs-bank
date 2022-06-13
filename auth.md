In this document, we will introduce a new concept, `auth` (authentication).
Previously it's called `identity` which was firstly introduced in [RFC: Regulation Compliance Lock](https://talk.nervos.org/t/rfc-regulation-compliance-lock/5788).
It's used for authentication by validating signature.

### Definition

```C
typedef struct CkbAuthType {
  uint8_t algorithm_id;
  uint8_t content[20];
} CkbAuthType;

```
It is a data structure with 21 bytes. The content can be hash (blake160 or some other hashes) of pubic key, preimage, or
some others. The blake160 hash function is defined as first 20 bytes of [blake2b
hash](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md#crypto-primitives).

### Auth Algorithm Id
Here we list some known `algorithm_id` which have been implemented already:

#### CKB(algorithm_id=0)

It is implemented by default CKB lock script: secp256k1_blake160_sighash_all. More details in [reference implementation](https://github.com/nervosnetwork/ckb-system-scripts/blob/master/c/secp256k1_blake160_sighash_all.c).

Key parameters:
* signature: a 65-byte signature defined in [secp256k1 library](https://github.com/bitcoin-core/secp256k1)
* pubkey: 33-byte compressed pubkey
* pubkey hash: blake160 of pubkey


#### Ethereum(algorithm_id=1)

It is implemented by blockchain Ethereum.
[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L199)

Key parameters:

  - signature: a 65-byte signature defined in [secp256k1 library](https://github.com/bitcoin-core/secp256k1)
  - pubkey: 64-byte uncompressed pubkey
  - pubkey hash: last 20 bytes of pubkey keccak hash

#### EOS(algorithm_id=2)

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L206)

Key parameters: Same as ethereum

#### Tron(algorithm_id=3)
[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L213)

Key parameters: Same as ethereum

#### Bitcoin(algorithm_id=4)

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L220)

Key parameters:
- signature: a 65-byte signature defined in [secp256k1 library](https://github.com/bitcoin-core/secp256k1)
- pubkey: 65-byte uncompressed pubkey
- pubkey hash: first 20 bytes of sha256 and ripemd160 on pubkey

#### Dogecoin(algorithm_id=5)

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L227)

Key parameters: same as bitcoin

#### CKB MultiSig(algorithm_id=6)

[reference implementation](https://github.com/nervosnetwork/ckb-system-scripts/blob/master/c/secp256k1_blake160_multisig_all.c)

Key parameters:
- signature: multisig_script | Signature1 | Signature2 | ...
- pubkey: variable length, defined as multisig_script(S | R | M | N | PubKeyHash1 | PubKeyHash2 | ...)
- pubkey hash: blake160 on pubkey

`multisig_script` has following structure:
```
+-------------+------------------------------------+-------+
|             |           Description              | Bytes |
+-------------+------------------------------------+-------+
| S           | reserved field, must be zero       |     1 |
| R           | first nth public keys must match   |     1 |
| M           | threshold                          |     1 |
| N           | total public keys                  |     1 |
| PubkeyHash1 | blake160 hash of compressed pubkey |    20 |
|  ...        |           ...                      |  ...  |
| PubkeyHashN | blake160 hash of compressed pubkey |    20 |
```


#### Schnorr(algorithm_id=7)

Key parameters:
- signature: 32 bytes pubkey + 64 bytes signature
- pubkey: 32 compressed pubkey
- pubkey hash: blake160 of pubkey

#### RSA(algorithm_id=8)
[reference implementation](https://github.com/nervosnetwork/ckb-production-scripts/blob/e570c11aff3eca12a47237c21598429088c610d5/c/validate_signature_rsa.h#L115)

Key parameters:
- signature: variable length, depending on key size. It includes the following data structure:
```C
typedef struct RsaInfo {
  uint8_t algorithm_id;
  uint8_t key_size;
  uint8_t padding;
  uint8_t md_type;
  uint32_t E;
  uint8_t N[KEY_SIZE_IN_BYTES];
  uint8_t sig[KEY_SIZE_IN_BYTES];
}
```
See more in [RsaInfo structure](https://github.com/nervosnetwork/ckb-production-scripts/blob/e570c11aff3eca12a47237c21598429088c610d5/c/validate_signature_rsa.h#L58).
- pubkey: variable length, depending on key size. It includes the following data structure:

```C
typedef struct RsaInfo {
  uint8_t algorithm_id;
  uint8_t key_size;
  uint8_t padding;
  uint8_t md_type;
  uint32_t E;
  uint8_t N[KEY_SIZE_IN_BYTES];
}
```
The KEY_SIZE_IN_BYTES can be 128, 256, 512 in bytes for key size 1024-bit, 2048-bit and 4096-bit respectively.
- pubkey hash: blake160 of pubkey

#### ISO9796-2(algorithm_id=9)
This signature is only used in digital passport.
[reference implementation](https://github.com/nervosnetwork/ckb-production-scripts/blob/e570c11aff3eca12a47237c21598429088c610d5/c/validate_signature_rsa.h#L115)

Key parameters:
- signature: RsaInfo | sig2[KEY_SIZE_IN_BYTES] | sig3[KEY_SIZE_IN_BYTES] | sig4[KEY_SIZE_IN_BYTES] 
- pubkey: Same as RSA
- pubkey hash: blake160 of pubkey

Note, the final signature is composed by 4 signatures with same key.

### Low Level APIs

We define some low level APIs to auth (Authentication), which can be also used for other purposes.
It is based on the following idea:
* [RFC: Swappable Signature Verification Protocol Spec](https://talk.nervos.org/t/rfc-swappable-signature-verification-protocol-spec/4802)
* [Ideas on chained locks](https://talk.nervos.org/t/ideas-on-chained-locks/5887)

First we define the "EntryType":
```C
typedef struct EntryType {
    uint8_t code_hash[32];
    uint8_t hash_type;
    uint8_t entry_category;
} EntryType;
```

* code_hash/hash_type

  the cell which contains the code binary
* entry_category

  The entry to the algorithm. Now there are 2 categories:
  - dynamic linking
  - exec

### Entry Category: Dynamic Linking
We define the follow functions when entry category is `dynamic linking`:
```C
int ckb_auth_validate(uint8_t auth_algorithm_id, const uint8_t *signature,
    uint32_t signature_size, const uint8_t *message, uint32_t message_size,
    uint8_t *pubkey_hash, uint32_t pubkey_hash_size);
```
The first argument denotes the `algorithm_id` in `CkbAuthType` described above. The arguments `signature` and
`pubkey_hash` are described in `key parameters` mentioned above.

A valid dynamic library denoted by `EntryType` should provide `ckb_auth_validate` exported function.

### Entry Category: Exec

This category shares same arguments and behavior to dynamic linking. It uses `exec` instead of `dynamic linking`. When
entry category is `exec`, it follows the rules described in [Ideas on chained
locks](https://talk.nervos.org/t/ideas-on-chained-locks/5887) with update of arguments format:

```text
<code hash>:<hash type>
:<auth algorithm id 1>:<signature 1>:<message 1>:<pubkey hash 1>
:<auth algorithm id 2>:<signature 2>:<message 2>:<pubkey hash 2>
...
:<auth algorithm id n>:<signature n>:<message n>:<pubkey hash n>
```

The `auth algorithm id n` denotes the `algorithm_id` in `CkbAuthType` described above. The fields `signature` and
`pubkey_hash` are described in `key parameters` mentioned above.

We can implement different auth algorithm id in same code binary. For example, the secp256k1 and shcnorr algorithm can be implemented from [bitcoin secp256k1](https://github.com/bitcoin-core/secp256k1). The RSA and ISO_9796_2 can be from [mbedtls](https://github.com/ARMmbed/mbedtls). 

### High Level APIs
The following API can combine the low level APIs together:
```C
int ckb_auth(EntryType* entry, CkbAuthType *id, uint8_t *signature, uint32_t signature_size, const uint8_t *message32)
```
Most of developers only need to use this function without knowing the low level APIs.


### Implementation

[See this PR](https://github.com/nervosnetwork/ckb-production-scripts/pull/37).

