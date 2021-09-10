In this document, we will introduce a new concept, `auth` (authentication).
Previously it's called `identity` which was firstly introduced in [RFC: Regulation Compliance Lock](https://talk.nervos.org/t/rfc-regulation-compliance-lock/5788).
It's used for authentication by validating signature.

### Definition

```C
typedef struct CkbAuthType {
  uint8_t flags;
  uint8_t content[20];
} CkbAuthType;

```
It takes 21 bytes memory on chain. The content can be blake160 hash of pubic key or preimage. 

### Flags
TODO:

- ckb = 0,
- ethereum = 1,
- eos = 2,
- tron = 3,
- bitcoin = 4,
- dogecoin = 5,
- owner_lock = 0xFC,

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
The first argument denotes the algorithm id described above.

The names of last 2 arguments are changed into input arguments instead of output arguments: make them same to `Entry category: exec`. 
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
int validate_signature(EntryType* entry, const uint8_t *signature,
    size_t signature_size, const uint8_t *message, size_t message_size,
    uint8_t *pubkey_hash, size_t pubkey_hash_size);
```
 
And much higher level and final API for auth: 
```C
int ckb_auth(EntryType* entry, CkbAuthType *id, const uint8_t *message32, uint8_t *signature, size_t signature_size)
```
Most of developers only need to use this function without knowing the low level APIs.


### Signature Format

TODO

Some signatures are not standardized. e.g. RSA, ISO-9796-2, BLS12-381. Here we propose a data structure for each.


