# RFC: Taproot Lock

## Introduction

The Bitcoin Taproot upgrade will activate later in 2021. The Taproot aims
to improve privacy, efficiency and flexibility of Bitcoin's scripting capabilities
without adding new security assumptions.

In this specification, A taproot like implementation is introduced on CKB which 
shows unlimited flexibility. It follows [BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) and [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) with some minor modifications due to different technology choices. Some design, concepts and code are based on [Regulation Compliance Lock](rc_lock.md).


## Identity

The Taproot lock shares the same concept of `identity` which is used in RC Lock.
It is an identity with 21 byte long data structure containing the following components:
```
<1 byte flag> <20 byte identity content>
```
A new identity flags value (0x6) is allocated for Taproot lock. The identity content is the blake160 
hash of taproot output key. A taproot output key is the 32-byte array which represents a public key according to [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki).


## Taproot Lock Script

The Taproot lock script has the following structure:
```
Code hash: Taproot lock script code hash
Hash type: Taproot lock script hash type
Args: <21 byte identity>
```


## Taproot Witness
When unlocking a Taproot lock,  the corresponding witness must be a 
proper `WitnessArgs` data structure in molecule format: In lock field of the `WitnessArgs`, 
a `TaprootLockWitnessLock` data structure as follows, must be present:
```
table TaprootLockWitnessLock {
    signature: BytesOpt,
    preimage: BytesOpt
    smt_proof: BytesOpt,
    exec_argv: BytesOpt
}
```
When the `signature` is present, it must follow the following data structure:
```
<pubkey, 32 bytes> <signature, 64 bytes>
```
The details of `pubkey` and `signature` are described in BIP340. The `pubkey` can be also called `Taproot output key` in BIP341.

When `preimage` is present, it must contains the following data structure:
```
taproot_output_key: 32 bytes
taproot_internal_key: 32 bytes
smt_root: 32 bytes
y_parity: 1 byte
script_code_hash: 32 bytes
script_hash_type: 1 byte
argv_hash: 20 bytes
```
When `preimage` is present, `smt_proof` and `exec_argv` must be present too. 
These fields will be described in the following sections.

There are 2 unlock methods: key path spending and script path spending. They are almost identical as [Script validation rules in BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#script-validation-rules), except some data structures.

### Tagged Hash
In BIP340, the tagged hash is defined as:
```
tagged_hash(tag, x) = SHA256(SHA256(tag) || SHA256(tag) || x)
```
We can defined it as following:
```
ckb_tagged_hash(tag, x) = blake2b(blake2b(tag) || blake2b(tag) || x)
```
It will be used in later section.


### Key Path Spending

When `signature`  is not none and `preimage`, `smt_proof`, `exec_argv` are all none:
the key path spending is used.  It follows the CKB secp256k1 unlocking method, replacing secp256k1 with schnorr algorithm. The identity content in script args is the blake160 hash of `pubkey` (Taproot output key) in `signature`.

Because schnorr signature  is not standardized, we here use all the definitions and scheme in [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki), including:

> * Signature encoding: Instead of using DER-encoding for signatures (which are variable size, and up to 72 bytes), we can use a simple fixed 64-byte format.
> * Public key encoding: Instead of using compressed 33-byte encodings of elliptic curve points which are common in Bitcoin today, public keys in this proposal are encoded as 32 bytes.

> Implicit Y coordinates In order to support efficient verification and batch verification, the Y coordinate of P and of R cannot be ambiguous (every valid X coordinate has two possible Y coordinates).
> Implicitly choosing the Y coordinate that is even[6].

> Final scheme As a result, our final scheme ends up using public key pk which is the X coordinate of a point P on the curve 
> whose Y coordinate is even and signatures (r,s) where r is the X coordinate of a point R whose Y coordinate is even. 
> The signature satisfies s⋅G = R + tagged_hash(r || pk || m)⋅P.



### Script Path Spending

When `signature` is none and `preimage`, `smt_proof`, `exec_argv` are not none:
the script path spending is used. 
To simplify the process, we will use some python code here. The referenced python code can be  found on [bip340 reference.py](https://github.com/bitcoin/bips/blob/master/bip-0340/reference.py).

* tweaked key is calculated from `taproot_internal_key` and `smt_root`

    ```Python
    def taproot_tweak_pubkey(pubkey: bytes, h: bytes) -> Tuple[int, bytes]:
        t = int_from_bytes(tagged_hash("TapTweak", pubkey + h))
        # bip-0341: If t ≥ (order of secp256k1), fail.
        if t >= SECP256K1_ORDER:
            raise ValueError
        # bip-0341: Let p = c[1:33] and let P = lift_x(int(p)) where lift_x and [:] are defined as in BIP340.
        # Fail if this point is not on the curve.
        P = lift_x(pubkey)
        if P is None:
            raise ValueError        
        # bip-0341: Let Q = P + int(t)G.
        Q = point_add(P, point_mul(G, t))
        return 0 if has_even_y(Q) else 1, bytes_from_int(x(Q))


    (returned_y_parity, returned_tweaked_key) = taproot_tweak_pubkey(taproot_internal_key, smt_root)
    ```   
    y_parity is also returned.

* if the returned tweaked key is not same as `taproot_output_key`, then failed
* if the returned y_parity is not same as `y_parity`, then failed
* SMT verification

    When [SMT Verification](https://github.com/jjyr/sparse-merkle-tree/blob/2dce546eab6f7eaaab3a0886247fd12ac798ad28/c/ckb_smt.h#L705) is used, the
key is `script_code_hash`. The `script_hash_type`, `argv_hash`  and 1-byte content with 1 are combined as value. 
The `smt_root` and `smt_proof` are also passed in as arguments.  If the `smt_verify` return failed status, then failed.

    The 1-byte content with 1 in value is to make sure the value can't be  all zero.


* if `argv_hash` is not all zero and blake160 hash of `exec_argv` is not same as `argv_hash`, then failed.

    if `argv_hash` is all zero, it means the `exec_argv` is not used and can be ignored.

* call syscall `exec` with arguments of `script_code_hash`, `script_hash_type` and `exec_argv`

    The syscall is described in [exec](https://github.com/nervosnetwork/rfcs/pull/237). The wrapping version is as following:

    ```C
    int ckb_exec_cell(const uint8_t* code_hash, uint8_t hash_type, uint32_t offset,
                    uint32_t length, int argc, const char* argv[]);
    ```

    The `exec_argv` is converted to hex format. For example, an array of four bytes
    ```
    [1,2,15,16]
    ```
    is converted to string: "01020F10". if `argv_hash` is all zero, the `exec_argv` is not used here: set `argc` to 0.

* the return code of `exec` is the final result of the unlocking process


## Examples

### Unlock via key path spending

```
CellDeps:
    <vec> Taproot Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Taproot Lock
            args: <flag: 0x6> <taproot output key 1 hash>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <valid schnorr signature for taproot output key 1>
        preimage:  <MISSING>
        smt_proof: <MISSING>
        exec_argv: <MISSING>
      <...>
```

---

### Unlock via script path spending

```
CellDeps:
    <vec> Taproot Script Cell
    <vec> Exec Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Taproot Lock
            args: <flag: 0x6> <taproot output key 1 hash>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <MISSING>
        preimage:  
            taproot_output_key: <taproot output key 1>
            taproot_internal_key: <taproot internal key>
            smt_root: <SMT root>
            y_parity: <parity of y in taproot output key position on elliptic curve>
            script_code_hash: <code hash of script>
            script_hash_type: <hash type of script>
            argv_hash: <blake160 hash of exec_argv>
        smt_proof: <proofs>
        exec_argv: <exec arguments>
      <...>
```

---
