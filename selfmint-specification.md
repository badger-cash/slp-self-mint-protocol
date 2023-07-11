![Simple Ledger Protocol](images/SLP-logo-solid-200.png)

# Simple Ledger Protocol (SLP) Self Mint Token

#### Version: 0.2
#### Date published: July 11, 2023

## Purpose

This specification describes a protocol for decentralized, distributed issuance of tokens on networks, such as Bitcoin Cash (BCH) and eCash (XEC), that implement the Simple Ledger Protocol (SLP). By using this protocol, individual users can issue ("mint") new value, from an already created token, into existance from inside their own non-custodial wallet.

## Background

In the traditional model, SLP tokens are brought into existence via GENESIS and MINT transactions that are effectuated by the original issuer of the token and whose value immediately is placed under the control of said issuer. From there, tokens are distributed to other users as standard SEND transactions. There are some exceptions to this model, such as the [MIST](https://read.cash/@kasumi/mist-mineable-slp-token-a166a114) mineable SLP token.

The Simple Ledger Protocol Self Mint Token operates on a model where all token issuance is distributed and decentralized. In the canonical implementation, the original creator of the token - the entity that performed the original GENESIS transaction - never takes possession of any token value whatsoever. Instead, each new value issuance (MINT) is accomplished via a special [covenant-based transaction](https://cashscript.org/docs/guides/covenants/) with the value assigned to the individual user ("transaction signer") creating, signing, and broadcasting the transaction. The covenant consists of several key spending constraints that allow the original creator of the token to have limited control over new token issuance, without ever transmitting token value.

## Specification (Token Type 1)

### Initial GENESIS Transaction

Initial creation of the token is done via a standard SLP Token Type 1 [GENESIS](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md#genesis---token-genesis-transaction) with the following required values:
* `<mint_baton_vout>` 0x02
* `<initial_token_mint_quantity>` 0x0000000000000000

The value of tokens created during GENESIS must be 0.

#### Creator Public Key

The funding input at index 0 must spend a UTXO that originates from a P2PKH address associated with the public key used in the [self mint covenants](#self-mint-baton-covenant). In other words, the public key in the scriptSig of `vin 0` can be used to construct the covenants from a template or [reference code](#code).

#### Initial Self Mint Stamp

The output at index 1 is a stamp assigned to the P2SH address for the [self mint postage covenant](#self-mint-postage-covenant) associated with the token. This output represents a value of 0 tokens created in GENESIS. This should be the first output ever sent to that address. In the case of the transaction standard provided in this specification, the stamp amount is 2300 satoshis.

With a GENESIS transaction constructed in this manner, a user can derive all needed information to construct a self mint transaction given only the data provided in the [authorizaton code](#authorization-code), as a query for the first transaction associated with the same address as the stamp to be spent in any given transaction will return the token GENESIS transaction.

#### Example GENESIS Transaction

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | *funding input*  | **OP_RETURN SLP GENESIS** |
| 1 | | **(0 tokens)** *self mint stamp* 2300 satoshis |
| 2 | | **(mint baton)** 546 satoshis |
| 3 | | *change* |

### Self Mint Baton Covenant

The mint baton must be assigned, in the GENESIS transaction to a Self Mint covenant Pay-To-Script-Hash address. The script itself may take several forms, but the standard implementation will have the following spending constraints:
* `Input at index 0 (vin 0)` *stamp*
   * `hash`
   * `index`
* `Output at index 0 (vout 0)` must be a NULL DATA output representing a SLP Token Type 1 [MINT](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md#mint---extended-minting-transaction)
   * `<mint_baton_vout>` 0x02
   * `<additional_token_quantity>`
* `vout 1`
   * `scriptPubKey` Pay-To-Public-Key-Hash (P2PKH) where public key hash is HASH160 of public key of transaction signer
* `vout 2`
   * `scriptPubKey` returns baton to same P2SH address as it is being spent from (mint baton address assigned in original GENESIS transaction)

All constraints are accomplished via an oracle-like [authorization signature](#authorization-signature) provided by the original creator of the token, whose public key is made part of the covenant script. 

#### Self Mint Postage Covenant

An unspent transaction output (UTXO) with sufficient native value (eg. BCH satoshis) required to make the transaction valid is used as the first input (`vout 0`) of the MINT transaction. We refer to this UTXO as a *self mint stamp*. Such UTXOs are created, in standard units, and assigned to a covenant that shares all spending constraints with the [Self Mint Baton Covenant](#self-mint-baton-covenant) and contains one additional constraint:
* `Input at index 0 (vin 0)` must be the current input being signed

The use of the Self Mint Baton and Self Mint Postage covenants means there are two distinct but correlated P2SH addresses, from which UTXOs are used in every MINT transaction. In the canonical application, all MINT transactions will use only two inputs, one UTXO from each address - a stamp and the baton.

Because a UTXO can only be spent once, and since the hash and index of the *self mint stamp* are part of the spending constraints, each [authorization signature](#authorization-signature) is only valid for one transaction.

#### Example MINT Transaction

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | *self mint stamp*  | **OP_RETURN SLP MINT** |
| 1 | *mint baton* | **(n tokens)** 546 satoshis |
| 2 | | **(mint baton)** 546 satoshis *same address as vin 1* |


### Authorization Code

In order to perform a Self Mint transaction, the end user must receive certain information from the original creator, whose public key is present in the Self Mint covenants described above. This information is delivered as a serialization of
* `bytes8 additional_token_quantity` *(big endian)* - as it appears in the [MINT OP_RETURN](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md#mint---extended-minting-transaction)
* `bytes36 stamp_outpoint` - [Bitcoincash-BIP143](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md#digest-algorithm) outpoint (`bytes32 hash` + `bytes4 vout`) of the stamp UTXO to be spent.
* `bytes authorization_signature` - [the signature](#authorization-signature) that will be used along with OP_CHECKDATASIG to execute the covenants

The code is parsed upon receipt by the end user and the three elements are used independently in the construction of the Self Mint transaction.

#### Authorization Signature

All [covenant spending constraints](#self-mint-baton-covenant) are secured by a signature. The private key used to create this signature corresponds to the public key placed into the [baton](](#self-mint-baton-covenant)[) and [postage](#self-mint-postage-covenant) covenants. The message signed is a serialization of
* `bytes36 stamp_outpoint`
* `tx_outputs` - the serialization of all output amounts (`bytes8` little endian) paired up with their scriptPubKey (serialized as scripts inside CTxOuts). This is the data on which is performed a double SHA256 to create hashOutputs in [Bitcoincash-BIP143](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md#digest-algorithm) transaction digest. Cosists of the following outputs:
   * `vout 0`
      * `bytes51 scriptPubKey` MINT OP_RETURN
      * `bytes8 satoshis` - 0
   * `vout 1`
      * `bytes26 scriptPubKey` - P2PKH with public key hash corresponding the the key being used to perform the final OP_CHECKSIG on both inputs
      * `bytes8 satoshis` - 546
   * `vout 2`
      * `bytes24 scriptPubKey` - P2SH for Self Mint Baton Covenant
      * `bytes8 satoshis` - 546 

### Covenant Unlocking

While the covenant locking script may take many forms and can be extended with more [spending constraints](#self-mint-baton-covenant) than are listed in this specification, the elements of the unlocking script (scriptSig), used to spend Self Mint covenant UTXOs, are standardized. This standardization allows maximum interoperability between wallets and reduces the number of different implementations required to support a robust ecosystem of Self Mint tokens.

The standard Self Mint scriptSig consists of the following elements (in the order processed):

* `p2sh_subscript`
* `prevout_sequence` - serialization of all input outpoints (`bytes32 hash` + `bytes4 vout`)
* `tx_outputs` - the serialization of all output amounts (`bytes8 satoshis` little endian) paired up with their scriptPubKey  (as described in [Authorization Signature](#authorization-signature) section)
* `authorization_signature` - [Authorization Signature](#authorization-signature)
* `preimage` - [Bitcoincash-BIP143](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md#digest-algorithm) transaction digest
* `spender_public_key`
* `spender_signature`


#### Example Covenant Unlock Process

The following is an example of operations followed by a Self Mint covenant script during the unlock procedure

1. Standard P2SH process, leaving subscript to execute stack
2. *(Self Mint Postage Input Only)* Check that this input is at index 0
3. Check that `scriptPubKey` at `vout 1` corresponds to `spender_public_key`
4. Check that spending constraints correspond to message signed by `authorization_signature`
5. Check that double SHA256 of `prevout_sequence` corresponds to `hashPrevouts` in `preimage`
6. Check that double SHA256 of `outputs_sequence` corresponds to `hashOutputs` in `preimage`
7. Validate `preimage` against `spender_public_key` and `spender_signature` using OP_CHECKDATASIG
8.  OP_CHECKSIG using `spender_public_key` and `spender_signature`

## Specification (Token Type 2)

### Initial GENESIS Transaction

Initial creation of the token is done via a standard SLP Token Type 2 [GENESIS](https://github.com/badger-cash/slp-specifications/blob/master/slp-token-type-2.md#genesis---token-genesis-transaction) with the following required values:
* `<mint_baton_vout>` 0x02
* `<initial_token_mint_quantity>` 0x0000000000000000

The value of tokens created during GENESIS must be 0.

#### Merkle Proof Public Key Rotation

While the public key scheme described in the previous section of this document may be used, in this section is provided a means of doing a form of key rotation by including a Merkle root in the script vault locking script and then providing a public key and Merkle proof - demonstrating that the provided public key exists in an accepted set of public keys - in the unlocking script att he time of minting. This allows an authorizing entity to effectively rotate keys at intervals and represents an ostensibly more secure implementation of the specification.

HASH256 (double SHA256 hash) is the algorithm used to construct the Merkle tree used in this specification.

The Merkle proof, as used in the unlocking script, is a serialization of Merkle branches, in the order that the hashing operations are performed. In order to ease in computation at the time of script validation, the order of the values to be hashed is specified in the proof data provided in the unlocking script. Thus, an individual branch is the serialization of:

* `bytes1 hashing_order` - `0x00` indicates that the accumulated result hash precedes the current branch level hash. `0x01` indicates that the current branch level hash precedes the accumulated result hash.
* `bytes32 branch_level_hash`


#### Example GENESIS Transaction

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | *funding input*  | **OP_RETURN SLP GENESIS** |
| 1 | | **(0 tokens)** *self mint stamp* 2300 satoshis |
| 2 | | *change* |

### Self Mint Vault Covenant

The mint_vault_scripthash field in the GENESIS OP_RETURN must be assigned to a Self Mint covenant. The script itself may take several forms, but the standard implementation will have the following spending constraints:
* `Input at index 0 (vin 0)` *valid UTXO originating from the MINT vault*
   * `hash`
   * `index`
* `Input at index 0 final OP_CHECKSIG` associated public key is defined by authorizer in authorization code message/signature
* `Outputs` must precisely match those defined in the authorization code

All constraints are accomplished via an oracle-like [authorization signature](#authorization-signature-type-2) provided by the original creator of the token, whose public key (via [Merkle proof method](#merkle-proof-public-key-rotation) is made part of the covenant script. 
Because a UTXO can only be spent once, and since the hash and index of the *self mint vault UTXO* are part of the spending constraints, each [authorization signature](#authorization-signature-type-2) is only valid for one transaction.

#### Example MINT Transaction

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | *self mint vault UTXO*  | **OP_RETURN SLP MINT** |
| 1 | | **(n tokens)** 546 satoshis |
| 2 | | **(n tokens)** 546 satoshis |
| 3 | | **(n tokens)** 546 satoshis |


### Authorization Code (Type 2)

In order to perform a Self Mint transaction, the end user must receive certain information from the original creator, whose public key (or Merkle root) is present in the Self Mint covenant described above. This information is delivered as a serialization of
* `bytes32 authorizer_public_key`
* `bytes merkle_proof` - the [Merkle proof](#merkle-proof-public-key-rotation)` that validates the provided public key against the Merkle root in the locking script
* `bytes1 authorization_signature_length`
* `bytes authorization_signature` - [the signature](#authorization-signature-type-2) that will be used along with OP_CHECKDATASIG to execute the covenants
* `bytes authorization_message` - the message signed by `authorization_signature`

The code is parsed upon receipt by the end user and the elements are used in the construction of the Self Mint transaction.

#### Authorization Signature (Type 2)

All [covenant spending constraints](#self-mint-baton-covenant) are secured by a signature. The private key used to create this signature corresponds to the public key provided in the [authorization code](#authorization-code-type-2). The message signed is a serialization of
* `bytes6 mint_id` - a required field used to identify and disambiguate individual authorization codes, as many codes may be issued that specify the exact same output data
* `bytes20 minter_pubkeyhash` - public key hash (HASH160) corresponding the the key being used to perform the final OP_CHECKSIG on the input at index 0. This restricts the authorization code to only be valid when used by an entity in control of a specific private key
* `bytes36 mint_vault_UTXO_outpoint`
* `tx_outputs` - the serialization of all output amounts (`bytes8` little endian) paired up with their scriptPubKey (serialized as scripts inside CTxOuts). This is the data on which is performed a double SHA256 to create hashOutputs in [Bitcoincash-BIP143](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md#digest-algorithm) transaction digest. An example with 3 MINT outputs would consist of the following outputs:
   * `vout 0`
      * `bytes65 scriptPubKey` MINT OP_RETURN specifying 3 output amounts
      * `bytes8 satoshis` - 0
   * `vout 1`
      * `bytes26 scriptPubKey` - Token output 1
      * `bytes8 satoshis` - 546
   * `vout 2`
      * `bytes26 scriptPubKey` - Token output 2
      * `bytes8 satoshis` - 546
   * `vout 3`
      * `bytes26 scriptPubKey` - Token output 3
      * `bytes8 satoshis` - 546

### Covenant Unlocking

While the covenant locking script may take many forms and can be extended with more [spending constraints](#self-mint-baton-covenant) than are listed in this specification, the elements of the unlocking script (scriptSig), used to spend Self Mint covenant UTXOs, are standardized. This standardization allows maximum interoperability between wallets and reduces the number of different implementations required to support a robust ecosystem of Self Mint tokens.

The standard Self Mint scriptSig consists of the following elements (in the order processed):

* `p2sh_subscript`
* `merkle_data` - serialization of:
*`authorizer_public_key`
*`merkle_proof`
* `authorization_data` - serialization of:
*‘authorization_signature_length`
*`authorization_signature`
*`authorization_message`
* `preimage` - [Bitcoincash-BIP143](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/replay-protected-sighash.md#digest-algorithm) transaction digest
* `spender_public_key`
* `spender_signature`


#### Example Covenant Unlock Process

The following is an example of operations followed by a Self Mint covenant script during the unlock procedure

1. Standard P2SH process, leaving subscript to execute stack
2. Perform Merkle proof and verify that authorizer public key provided is valid
3. Validate `authorization_message` and `authorization_signature` using OP_CHECKDATASIG
4. Check that spending constraints correspond to message signed by `authorization_signature` using preimage for transaction introspection
5. Validate `preimage` against `spender_public_key` and `spender_signature` using OP_CHECKDATASIG
6.  OP_CHECKSIG using `spender_public_key` and `spender_signature`


## Reference Implementations

### Clients
[CashTab](https://github.com/badger-cash/cashtab) ([wallet.badger.cash](https://wallet.badger.cash) fork) - ReactJS

### Libraries
None currently

### Code

Reference code for creating covenant scripts using [bcash](https://github.com/badger-cash/bcash) (Badger.cash fork) JavaScript library

#### Token Type 1

```
const buildOutScript = (authPubKey, checkIsFirstInput = false) => {
   
   const script = new bcash.Script()
      .pushSym('2dup')
      .pushInt(36)
      .pushSym('split')
      .pushSym('drop');

   if (checkIsFirstInput) {
      script.pushSym('dup')
      script.pushInt(6)
      script.pushSym('pick')
      script.pushInt(104)
      script.pushSym('split')
      script.pushSym('drop')
      script.pushInt(68)
      script.pushSym('split')
      script.pushSym('nip')
      script.pushSym('equalverify');
   }

   script.pushSym('swap')
      .pushSym('dup')
      .pushInt(78)
      .pushSym('split')
      .pushSym('nip')
      .pushInt(20)
      .pushSym('split')
      .pushSym('drop')
      .pushInt(7)
      .pushSym('pick')
      .pushSym('hash160')
      .pushSym('equalverify')

      .pushInt(132)
      .pushSym('split')
      .pushSym('drop')
      .pushSym('cat')
      .pushInt(3)
      .pushSym('roll')
      .pushSym('swap')
      .pushData(authPubKey)
      .pushSym('checkdatasigverify')
      .pushInt(2)
      .pushSym('roll')
      .pushSym('dup')
      .pushSym('size')
      .pushInt(40)
      .pushSym('sub')
      .pushSym('split')
      .pushSym('swap')
      .pushInt(4)
      .pushSym('split')
      .pushSym('nip')
      .pushInt(32)
      .pushSym('split')
      .pushSym('drop')
      .pushInt(3)
      .pushSym('roll')
      .pushSym('hash256')
      .pushSym('equalverify')
      .pushInt(32)
      .pushSym('split')
      .pushSym('drop')
      .pushSym('rot')
      .pushSym('hash256')
      .pushSym('equalverify')
      .pushSym('sha256')
      .pushSym('3dup')
      .pushSym('rot')
      .pushSym('size')
      .pushSym('1sub')
      .pushSym('split')
      .pushSym('drop')
      .pushSym('swap')
      .pushSym('rot')
      .pushSym('checkdatasigverify')
      .pushSym('drop')
      .pushSym('checksig')
      .compile();

   return script;
}
```

#### Token Type 2

##### (Merkle tree consists of 128 hashed public keys)

```
const type2Outscript = (merkleRootBuf) => {

   const script = new Script()
      .pushInt(33)
      .pushSym('split')
      .pushSym('over')
      .pushSym('hash256')
      .pushSym('swap');

   // Iterate through merkle proof
   for (let i = 0; i < 6; i++) {
      script.pushInt(1)
         .pushSym('split')
         .pushInt(32)
         .pushSym('split')
         .pushSym('toaltstack')
         .pushSym('swap')
         .pushSym('bin2num')
         .pushSym('if')
         .pushSym('swap')
         .pushSym('endif')
         .pushSym('cat')
         .pushSym('hash256')
         .pushSym('fromaltstack');
   }

   script.pushInt(1)
      .pushSym('split')
      .pushSym('swap')
      .pushSym('bin2num')
      .pushSym('if')
      .pushSym('swap')
      .pushSym('endif')
      .pushSym('cat')
      .pushSym('hash256')
      .pushData(merkleRootBuf)
      .pushSym('equalverify')
      .pushSym('swap')
      .pushInt(1)
      .pushSym('split')
      .pushSym('swap')
      .pushSym('bin2num')
      .pushSym('split')
      .pushSym('dup')
      .pushSym('toaltstack')
      .pushSym('rot')
      .pushSym('checkdatasigverify')
      .pushSym('fromaltstack')
      .pushInt(26)
      .pushSym('split')
      .pushSym('swap')
      .pushInt(6)
      .pushSym('split')
      .pushInt(4)
      .pushSym('pick')
      .pushSym('hash160')
      .pushSym('equalverify')
      .pushSym('drop')
      .pushInt(36)
      .pushSym('split')
      .pushSym('hash256')
      .pushInt(2)
      .pushSym('pick')
      .pushSym('size')
      .pushInt(40)
      .pushSym('sub')
      .pushSym('split')
      .pushInt(32)
      .pushSym('split')
      .pushSym('drop')
      .pushSym('rot')
      .pushSym('equalverify')
      .pushInt(68)
      .pushSym('split')
      .pushSym('nip')
      .pushInt(36)
      .pushSym('split')
      .pushSym('drop')
      .pushSym('equalverify')
      .pushSym('sha256')
      .pushSym('3dup')
      .pushSym('rot')
      .pushSym('size')
      .pushSym('1sub')
      .pushSym('split')
      .pushSym('drop')
      .pushSym('swap')
      .pushSym('rot')
      .pushSym('checkdatasigverify')
      .pushSym('drop')
      .pushSym('checksig')
      .compile();
      
   return script;
}
```
