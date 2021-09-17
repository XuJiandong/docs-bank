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
It takes 21 bytes memory on chain. The content can be blake160 hash of pubic key, preimage, or some others. 

### Auth Algorithm Id
Here we list the known id which have been used already:
- ckb = 0

[reference implementation](https://github.com/nervosnetwork/ckb-system-scripts/blob/master/c/secp256k1_blake160_sighash_all.c).
- ethereum = 1

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L199)
- eos = 2

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L206)
- tron = 3

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L213)
- bitcoin = 4

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L220)
- dogecoin = 5

[reference implementation](https://github.com/XuJiandong/pw-lock/blob/e7f5f2379185d4acf18af38645559102e100a545/c/pw_lock.h#L227)
- ckb_multisig = 6

[reference implementation](https://github.com/nervosnetwork/ckb-system-scripts/blob/master/c/secp256k1_blake160_multisig_all.c)
- schnorr/taproot = 7

- iso_9796_2 = 8

[reference implementation](https://github.com/nervosnetwork/ckb-production-scripts/blob/e570c11aff3eca12a47237c21598429088c610d5/c/validate_signature_rsa.h#L115)
- RSA = 9

[reference implementation](https://github.com/nervosnetwork/ckb-production-scripts/blob/e570c11aff3eca12a47237c21598429088c610d5/c/validate_signature_rsa.h#L115)

- owner_lock = 0xFC


### Low Level APIs

We define some low level APIs to auth (Authentication), which can be also used for other purpose.
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
We define the follow functions when entry category is `dynamic liking`:
```C
int ckb_auth_validate(uint8_t auth_algorithm_id, const uint8_t *signature,
    uint32_t signature_size, const uint8_t *message, uint32_t message_size,
    uint8_t *pubkey_hash, uint32_t pubkey_hash_size);
```
The first argument denotes the `auth_algorithm_id` described above.

The names of last 2 arguments are changed into input arguments instead of output arguments: make them same to [Entry category: exec](#entry-category-exec). 
Other arguments are kept as same to [RFC: Swappable Signature Verification Protocol Spec](https://talk.nervos.org/t/rfc-swappable-signature-verification-protocol-spec/4802).

A valid dynamic library denoted by `EntryType` should provide `ckb_auth_validate`.

### Entry Category: Exec

This category actually shares same arguments and behavior to [Entry Category: Exec](#entry-category-exec). It uses `exec` instead of `dynamic linking`. When entry category is `exec`, it follows the rules described in [Ideas on chained locks](https://talk.nervos.org/t/ideas-on-chained-locks/5887) with update of arguments format:

```text
<code hash>:<hash type>
:<auth algorithm id 1>:<signature 1>:<message 1>:<pubkey hash 1>
:<auth algorithm id 2>:<signature 2>:<message 2>:<pubkey hash 2>
...
:<auth algorithm id n>:<signature n>:<message n>:<pubkey hash n>
```
Here 2 new fields are added:
- auth algorithm id
- pubkey hash

We can implement different auth algorithm id in same code binary. For example, the secp256k1 and shcnorr algorithm can be implemented from [bitcoin secp256k1](https://github.com/bitcoin-core/secp256k1). The RSA and ISO_9796_2 can be from [mbedtls](https://github.com/ARMmbed/mbedtls). The `auth algorithm id` occupies 1 byte.

### High Level APIs
The following API can combine the low level APIs together:
```C
int ckb_auth(EntryType* entry, CkbAuthType *id, uint8_t *signature, uint32_t signature_size, const uint8_t *message32)
```
Most of developers only need to use this function without knowing the low level APIs.


### Signature Format

TODO

Some signatures are not standardized. e.g. RSA, ISO-9796-2, BLS12-381. Here we propose a data structure for each.
