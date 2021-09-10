

### Low Level APIs

We define some low level APIs to identity, which can be also used for other purpose.
It is based on the following idea:
* [RFC: Swappable Signature Verification Protocol Spec](https://talk.nervos.org/t/rfc-swappable-signature-verification-protocol-spec/4802)
* [Ideas on chained locks](https://talk.nervos.org/t/ideas-on-chained-locks/5887)

First we define the "EntryType":
```C
typedef struct EntryType {
    uint8_t code_hash[32];
    uint8_t hash_type;
    uint8_t entry_category;
    uint32_t algorithm_id;
} EntryType;
```

* code_hash/hash_type

  the cell which contains the code binary
* entry_category

  The entry to the algorithm. Now there are 2 categories:
  - dynamic linking
  - exec

* algorithm id

  the cryptography algorithm:
  - secp256k1
  - schnorr
  - RSA
  - ISO-9796-2 
  - BLS12-381
  - ... many others
  

### Entry Category: Dynamic Linking
We define the follow functions when entry category is `dynamic liking`:
```C
int validate_signature(uint32_t algorithm_id, const uint8_t *signature,
    size_t signature_size, const uint8_t *message, size_t message_size,
    uint8_t *pubkey_hash, size_t pubkey_hash_size);
```
The first argument denote the algorithm id described above.

The name of last 2 arguments are changed because they're conflicted with the concept of `identity`. 
And they're changed into input arguments instead of output arguments: make them same to `Entry category: exec`. 
Other arguments are kept as same to [RFC: Swappable Signature Verification Protocol Spec](https://talk.nervos.org/t/rfc-swappable-signature-verification-protocol-spec/4802).

A valid dynamic library denoted by `EntryType` should provide `validate_signature`.

### Entry Category: Exec

This category actually shares same arguments and behavior to `Entry Category: Exec`. It uses `exec` instead of `dynamic linking`. When entry category is `exec`, it follows the rules described in [Ideas on chained locks](https://talk.nervos.org/t/ideas-on-chained-locks/5887) with update of arguments format:

```text
<code hash>:<hash type>
:<algorithm id 1>:<pubkey hash 1>:<message 1>:<signature 1>
:<algorithm id 2>:<pubkey hash 2>:<message 2>:<signature 2>
...
:<algorithm id n>:<pubkey hash n>:<message n>:<signature n>
```
Here 2 new fields are added:
- algorithm id
- pubkey hash

We can implement different algorithm in same code binary. For example, the secp256k1 and shcnorr algorithm can be implemented from [bitcoin secp256k1](https://github.com/bitcoin-core/secp256k1). The RSA and ISO_9796_2 can be from [mbedtls](https://github.com/ARMmbed/mbedtls). The `algorithm id` occupies 4 bytes.

### High Level APIs
The following API can combine the low level APIs together:

```C
int validate_signature(EntryType* entry, const uint8_t *signature_buffer,
    size_t signature_size, const uint8_t *message_buffer, size_t message_size,
    uint8_t *pubkey_hash, size_t pubkey_hash_size);
```
 
And much higher level and final API for identity: 
```C
int ckb_validate_identity(EntryType* entry, CkbIdentityType *id, const uint8_t *message32, uint8_t *signature, size_t signature_size)
```
Most of developers only need to use this function without knowing the low level APIs.


### Signature Format

TODO

Some signatures are not standardized. e.g. RSA, ISO-9796-2, BLS12-381. Here we propose a data structure for each.


