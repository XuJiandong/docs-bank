# Omni Lock

NOTE: We used to use the term *Regulation Compliance Lock (RC Lock)* instead of *Omni Lock*. These terms are interchangeable.

*Omni Lock* is a new lock script designed to be used together with [Regulation Compliance Extension](https://talk.nervos.org/t/rfc-regulation-compliance-extension/5338). Apart from being a regular lock, it provides one unique feature: certain administrators of the xUDT can revoke tokens held by certain malicious users. In otherwise, a cell guarded by this lock can be unlocked by either of the following parties:


* Owner of the cell
* Administrators of the Omni Lock, typically regulators of a particular xUDT token


One might find this lock against common beliefs in Nervos CKB: the owner of a lock has absolute control on his/her cells. One unique power of CKB, is to provide options, it's perfectly good to use a lock that only one themselves can unlock, we will be continuing supporting this solution to the end of time. Having Omni Lock will not change this fact.

However, it might also make sense to provide option for the other side of spectrum: certain tokens are regulated by authorities, which will need the ability to revoke issued tokens, once owners stop confronting to regulations. While this option certainly sacrifices some security, it trades us a solution to be more connected with the real world for mass adoption.

When combined, Omni Lock and RCE can provide functionalities on par with [ERC-1404](https://erc1404.org/).

## Auth

Omni Lock introduces a new concept `auth` (authentication) to CKB lock scripts: an auth is a 21-byte long data structure containing the following components:

```
<1 byte flag> <20 bytes auth content>
```

Depending on the value of the flag, the auth content has different interpretations:

* 0x0: The auth content that represents the blake160 hash of a secp256k1 public key. The lock script will perform secp256k1 signature verification, the same as the [SECP256K1/blake160](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0024-ckb-system-script-list/0024-ckb-system-script-list.md#secp256k1blake160) lock.
* 0x01~0x05: It follows the same unlocking methods used by [pw-lock](https://github.com/lay2dev/pw-lock/blob/c2b1456bcca06c892e1bb8ec8ac0a64d4fb2b83d/c/pw_lock.h#L190-L223)
* 0x06: It follows the same unlocking method used by [CKB MultiSig](https://github.com/nervosnetwork/ckb-system-scripts/blob/master/c/secp256k1_blake160_multisig_all.c)

* 0xFC: The auth content that represents the blake160 hash of a lock script. The lock script will check if the current transaction contains an input cell with a matching lock script. Otherwise, it would return with an error.

* 0xFD: The auth content that represents the blake160 hash of a preimage. The preimage contains [exec](https://github.com/nervosnetwork/rfcs/pull/237) information that is used to delegate signature verification to another script via exec.

* 0xFE: The auth content that represents the blake160 hash of a preimage. The preimage contains dynamic linking information that is used to delegate signature verification to dynamic linking script. The interface described in [Swappable Signature Verification Protocol Spec](https://talk.nervos.org/t/rfc-swappable-signature-verification-protocol-spec/4802) is used here.


## Omni Lock Script

An Omni Lock script has the following structure:

```
Code hash: Omni Lock script code hash
Hash type: Omni Lock script hash type
Args: <21 byte auth> <Omni Lock args>
```
The `<Omni Lock args>` has the following structure:
```
<1 byte Omni Lock flags> <32 byte RC cell type ID, optional> <2 bytes minimum ckb/udt in ACP, optional> <8 bytes since for time lock, optional> <32 bytes type script hash for supply, optional>
```

When `<Omni Lock flags> & 1` is not zero,  which we call "administrator mode", <32 byte RC cell type ID> is present. The RC cell type ID contains the [type script hash](https://xuejie.space/2020_02_03_introduction_to_ckb_script_programming_type_id/) used by a special cell with the same format as [RCE Cell](https://talk.nervos.org/t/rfc-regulation-compliance-extension/5338), but the RC cell has the following distinctions:

* The cell used here contains identities, not lock script hashes.
* If the cell contains a RCRule structure, the RCRule structure must be in whitelist mode.
* If the cell contains a RCCellVec structure, there must at least be one RCRule structure using whitelists in the RCCellVec.

To make it easier for reference, we call this cell `RC AdminList Cell`. To make this mode more flexible, 
when no `type script hash` is found in [cell_deps](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md), 
it continues searching input cells with the same type script hash. If a cell is found, it will be used as `RC AdminList Cell`.

When an administrator tries to unlock the cell using Omni Lock, the administrator must provide SMT proofs for the auth used for validation. See the following part for more details. 

When it's in "administrator mode", "anyone-can-pay mode" and "time lock mode" below are ignored.

When `<Omni Lock flags> & (1 << 1)` is not zero, which we call "anyone-can-pay mode", `<2 bytes minimum ckb/udt in ACP>` is present. It follows the script: [anyone-can-pay lock](https://talk.nervos.org/t/rfc-anyone-can-pay-lock/4438). The `<1 byte CKByte minimum>` and `<1 byte UDT minimum>` are present at the same time. 

When `<Omni Lock flags> & (1 << 2)` is not zero,  which we call "time lock mode", `<8 bytes since for time lock>` is present. The [check_since](https://github.com/nervosnetwork/ckb-system-scripts/blob/63c63e9c96887395fc6990908bcba95476d8aad1/c/common.h#L91) is used. The input parameter `since` is obtained from ` <8 bytes since for time lock>`.

When `<Omni Lock flags> & (1 << 3)` is not zero,  which we call "supply mode", `<32 bytes type script hash>` is present. The cell data of `info cell` which is specified by `type script hash` has the following data structure:
```
version (1 byte)
current supply (16 bytes, little endian number)
max supply (16 bytes, little endian number)
sUDT script hash (32 bytes, sUDT type script hash)
... (variable length, other data)
```
Currently, the `version` is 0. Only the `currently supply` field can be updated during transaction. The script iterates all input and output cells, accumulating input [amounts](https://talk.nervos.org/t/rfc-simple-udt-draft-spec/4333) and output amounts identified by `sUDT script hash`.  Then verify:
```
<issued amount> = <output amount> - <input amount>
<output current supply> = <issued amount> + <input current supply>
```

All the modes mentioned above can co-exist. 

## Omni Lock Witness

Note: "identity" and "auth" can be used interchangeably in code, due to historical reason.

When unlocking an Omni Lock, the corresponding witness must be a proper `WitnessArgs` data structure in molecule format, in the `lock` field of the `WitnessArgs`, an `RcLockWitnessLock` data structure as follows, must be present:

```
import xudt_rce;

array Identity [byte; 21];

table RcIdentity {
    identity: Identity,
    proofs: SmtProofEntryVec,
}
option RcIdentityOpt (RcIdentity);

// the data structure used in lock field of witness
table RcLockWitnessLock {
    signature: BytesOpt,
    rc_identity: RcIdentityOpt,
    preimage: BytesOpt,
}
```

When `rc_identity` is present, we will validate if the provided auth in `rc_identity` is present in `RC AdminList Cell` associated with current lock script via SMT validation rules. In this case, the auth included in `rc_identity` will be used in further validation.

When `rc_identity` is missing, the auth included in lock script args will then be used in further validation.

Once the auth to be used is confirmed, we will look for the flag in the designated auth for succeeding operations:

* When the auth flag is 0x0, a signature must be present in `RcLockWitnessLock`. We will use the signature for secp256k1 recoverable signature verification. The recovered public key hash using blake160 algorithm, must match the current auth content.

* When the auth flag is 0xFC, we will check against current transaction, and there must be an input cell, whose lock script matches the auth content when hashed via blake160.

When `signature` is present, the signature can be used to unlock the cell in "anyone-can-pay mode".

When `preimage` is present, if flags in auth is:
* 0xFD (exec): the preimage's memory layout will be as follows:
```
exec code hash (32 bytes)
exec hash type (1 bytes)
place (1 bytes)
bounds (8 bytes)
pubkey hash (20 bytes)
```

Firstly, it encodes message, signature, pubkey hash into hex strings suggested by [Ideas on chained locks](https://talk.nervos.org/t/ideas-on-chained-locks/5887). These strings are passed in as arguments in [ckb_exec](https://github.com/nervosnetwork/rfcs/pull/237/files). The final returned code is the same as ckb_exec.

* 0xFE (dynamic linking), the preimage's memory layout will be as follows:
```
dynamic library code hash (32 bytes)
dynamic library hash type (1 bytes)
pubkey hash （20 bytes)
```
It loads the dynamic linking library by "code hash/hash type" and gets the entry function of 
[validate_signature](https://talk.nervos.org/t/rfc-swappable-signature-verification-protocol-spec/4802). 
Then it calls the entry function to validate message and signature. 
The "auth" returned from the entry function is compared with blake160 hash of pubkey. 
If they are same, then the validation succeeds.


## Examples

### Unlock via owner's public key hash

```
CellDeps:
    <vec> Omni Lock Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0x0> <pubkey hash 1> <Omni Lock flags: 0>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <valid secp256k1 signature for pubkey hash 1>
        rc_identity: <MISSING>
        preimage: <MISSING>
      <...>
```

### Unlock via owner's lock script hash

```
CellDeps:
    <vec> Omni Lock Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0xFC> <lock hash: 0x1234> <Omni Lock flags: 0>
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock: blake160 for this lock script must be 0x1234
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <MISSING>
        rc_identity: <MISSING>
        preimage: <MISSING>
      <...>
```

### Unlock via administrator's public key hash

```
CellDeps:
    <vec> Omni Lock Script Cell
    <vec> RC AdminList Cell 1
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0x0> <pubkey hash 1> <Omni Lock flags: 1> <RC AdminList Cell 1's type ID>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <valid secp256k1 signature for pubkey hash 2>
        rc_identity:
           identity: <flag: 0x0> <pubkey hash 2>
           proofs: <SMT proofs for the above identity in RC AdminList Cell 1>
        preimage: <MISSING>
      <...>
```

### Unlock via administrator's lock script hash

```
CellDeps:
    <vec> Omni Lock Script Cell
    <vec> RC AdminList Cell 1
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0> <pubkey hash 1> <Omni Lock flags: 1> <RC AdminList Cell 1's type ID>
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock: blake160 for this lock script must be 0x1234
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <MISSING>
        rc_identity:
           identity: <flag: 0xFC> <lock hash: 0x1234>
           proofs: <SMT proofs for the above identity in RC AdminList Cell 1>
        preimage: <MISSING>
      <...>
```

### Unlock via anyone-can-pay
```
CellDeps:
    <vec> Omni Lock Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0x0> <pubkey hash 1> <Omni Lock flags: 2> <2 bytes minimun ckb/udt in ACP>
    <...>
    follow anyone-can-pay rules
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <MISSING>
        rc_identity: <MISSING>
        preimage: <MISSING>
      <...>
```

### Unlock via dynamic linking
```
CellDeps:
    <vec> Omni Lock Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0xFE> <preimage hash> <Omni Lock flags: 0>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <valid secp256k1 signature for pubkey hash 1>
        rc_identity: <MISSING>
        preimage: <code hash> <hash type> <pubkey hash 1>
      <...>
```


### Unlock via exec
```
CellDeps:
    <vec> Omni Lock Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0xFD> <preimage hash> <Omni Lock flags: 0>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <valid secp256k1 signature for pubkey hash 1>
        rc_identity: <MISSING>
        preimage: <code hash> <hash type> <place> <bounds> <pubkey hash 1>
      <...>
```

### Unlock via owner's public key hash with time lock limit

```
CellDeps:
    <vec> Omni Lock Script Cell
Inputs:
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0x0> <pubkey hash 1> <Omni Lock flags: 4> <since 1>
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <valid secp256k1 signature for pubkey hash 1>
        rc_identity: <MISSING>
        preimage: <MISSING>
      <...>
```

### Unlock via administrator's lock script hash (2)

Note: the location of `RC AdminList Cell 1`

```
CellDeps:
    <vec> Omni Lock Script Cell

Inputs:
    <vec> RC AdminList Cell 1
        Data: <RCData, union of RCCellVec and RCRule>
        Type: <its hash is same to RC AdminList Cell 1's type ID>
        Lock: <...>
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock:
            code_hash: Omni Lock
            args: <flag: 0> <pubkey hash 1> <Omni Lock flags: 1> <RC AdminList Cell 1's type ID>
    <vec> Cell
        Data: <...>
        Type: <...>
        Lock: blake160 for this lock script must be 0x1234
    <...>
Outputs:
    <vec> Any cell
Witnesses:
    WitnessArgs structure:
      Lock:
        signature: <MISSING>
        rc_identity:
           identity: <flag: 0xFC> <lock hash: 0x1234>
           proofs: <SMT proofs for the above identity in RC AdminList Cell 1>
        preimage: <MISSING>
      <...>
```

## Deployment info

- Lina

| parameter   | value                                                                |
| ----------- | -------------------------------------------------------------------- |
| `code_hash` | `0x9f3aeaf2fc439549cbc870c653374943af96a0658bd6b51be8d8983183e6f52f` |
| `hash_type` | `type`                                                               |
| `tx_hash`   | `0xaa8ab7e97ed6a268be5d7e26d63d115fa77230e51ae437fc532988dd0c3ce10a` |
| `index`     | `0x1`                                                                |
| `dep_type`  | `code`                                                               |

- Aggron

| parameter   | value                                                                |
| ----------- | -------------------------------------------------------------------- |
| `code_hash` | `0x79f90bb5e892d80dd213439eeab551120eb417678824f282b4ffb5f21bad2e1e` |
| `hash_type` | `type`                                                               |
| `tx_hash`   | `0x9154df4f7336402114d04495175b37390ce86a4906d2d4001cf02c3e6d97f39c` |
| `index`     | `0x0`                                                                |
| `dep_type`  | `code`                                                               |


<details><summary>reproduction and verify deployment</summary>
    
    
```sh
git clone https://github.com/nervosnetwork/ckb-production-scripts.git
cd ckb-production-scripts
git checkout -b rc_lock 09f37ee525e566a050d252cdb1020e345a57bef7
git submodule update --init --recursive
make all-via-docker
```

```js
const fs = require('fs');
const { utils } = require('@ckb-lumos/base');
const { RPC } = require('@ckb-lumos/rpc');
const { Reader } = require('ckb-js-toolkit');

// const RPC_URL = 'https://testnet.ckb.dev/rpc';
// const TX_HASH = '0xaa8ab7e97ed6a268be5d7e26d63d115fa77230e51ae437fc532988dd0c3ce10a';
// const OUTPUT_INDEX = 0;
const RPC_URL = 'https://mainnet.ckb.dev/rpc';
const TX_HASH = '0xaa8ab7e97ed6a268be5d7e26d63d115fa77230e51ae437fc532988dd0c3ce10a';
const OUTPUT_INDEX = 1;
const BINARY_PATH = '/path/to/build/rc_lock';


function calculateCodeHashByBin(scriptBin /*: Buffer*/) /*: string*/ {
  const bin = scriptBin.valueOf();
  return new utils.CKBHasher().update(bin.buffer.slice(bin.byteOffset, bin.byteLength + bin.byteOffset)).digestHex();
}

async function getDataHash(txHash /*: string*/, index /*: number*/) {
  const rpc = new RPC(RPC_URL);

  const tx = await rpc.get_transaction(txHash);

  if (!tx) throw new Error(`TxHash(${txHash}) is not found`);

  const outputData = tx.transaction.outputs_data[index];
  if (!outputData) throw new Error(`cannot find output data`);

  return new utils.CKBHasher().update(new Reader(outputData)).digestHex();
}

async function validate() {
  const localHash = calculateCodeHashByBin(fs.readFileSync(BINARY_PATH));
  const onChainHash = await getDataHash(TX_HASH, OUTPUT_INDEX);

  console.log('local hash is', localHash);
  console.log('on-chain hash is', onChainHash);
}

validate();
```

```
local hash is 0x1bf147d1c95a5ad51016bd426c33f364257e685af54df26fc69ec40e3ae267d1
on-chain hash is 0x1bf147d1c95a5ad51016bd426c33f364257e685af54df26fc69ec40e3ae267d1
```
    
</details>

