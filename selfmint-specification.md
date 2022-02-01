![Simple Ledger Protocol](images/SLP-logo-solid-200.png)

# Simple Ledger Protocol (SLP) Self Mint Token

#### Version: 0.1
#### Date published: January 30, 2022

## Purpose

This specification describes a protocol for decentralized, distributed issuance of tokens on networks, such as Bitcoin Cash (BCH) and eCash (XEC), that implement the Simple Ledger Protocol (SLP). By using this protocol, users individual users can issue ("mint") new value, from an already created token, into existance from inside their own non-custodial wallet.

## Background

In the traditional model, SLP tokens are brought into existence, via GENESIS and MINT transactions that are effectuated by the original issuer of the token and whose value immediately is placed under the control of said issuer. From there, tokens are distributed to other users as standard SEND transactions. There are some exceptions to this model, such as the [MIST](https://read.cash/@kasumi/mist-mineable-slp-token-a166a114) mineable SLP token.

The Simple Ledger Protocol Self Mint Token operates on a model where all token issuance is distributed and decentralized. In the canonical implementation, the original creator of the token - the entity that performed the original GENESIS transaction - never takes possession of any token value whatsoever. Instead, each new value issuance (MINT) is accomplished via a special [covenant-based transaction](https://cashscript.org/docs/guides/covenants/) with the value assigned to the individual user ("transaction signer") creating, signing, and broadcasting the transaction. The covenant consists of several key spending constraints that allow the original creator of the token to have limited control over new token issuance, without ever transmitting token value.

## Specification

### Initial GENESIS Transaction

Initial creation of the token is done via a standard SLP Token Type 1 [GENESIS](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md#genesis---token-genesis-transaction) with the following required values:
* `<mint_baton_vout>` 0x02-0xff
* `<initial_token_mint_quantity>` 0x0000000000000000

The value of tokens created during GENESIS must be 0.

#### Example GENESIS Transaction

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | *funding input*  | **OP_RETURN SLP GENESIS** |
| 1 | | **(0 tokens)** 546 satoshis |
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

While the covenant locking script may take many forms and can be extended with more [spending constraints](#self-mint-baton-covenant) than are listed in this specification, the elements of the unlocking script (scriptPubKey), used to spend Self Mint covenant UTXOs, are standardized. This standardization allows maximum interoperability between wallets and reduces the number of different implementations required to support a robust ecosystem of Self Mint tokens.

The standard Self Mint scriptPubKey consists of the following elements (in the order processed):

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


## Reference Implementations

### Clients
[CashTab](https://github.com/badger-cash/cashtab) ([wallet.badger.cash](https://wallet.badger.cash) fork) - ReactJS

### Libraries
None currently

### Code

Reference code for creating covenant scripts using [bcash](https://github.com/badger-cash/bcash) (Badger.cash fork) JavaScript library

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
