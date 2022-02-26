![Simple Ledger Protocol](images/SLP-logo-solid-200.png)

# SLP Token Type 1 Burn Specification
### Version: 0.1
### Date published: February 26, 2022

## Purpose

This specification describes an additional transaction type, BURN, for the Simple Ledger Protocol (SLP) Token Type 1. This transaction type explictly specifies that token units have been removed from circulation, "burned," on purpose and not as the result of an error.

## Background

The SLP Token Type 1 specification consists of four different transaction types:
* **GENESIS** creates a brand new token
* **MINT** puts more units into circulation from an already created token
* **SEND** sends units of value
* **COMMIT** *(not implemented)* allows on-chain checksum commitment of previous token transactions

The original specification does not specifically address applications where units of token value are purposely removed from circulation. There are many scenarios where removing value from circulation would be desired, such as with reserve-backed stablecoins or redeemed promotional vouchers/coupons.

This specification extends the [SLP Token Type 1 Protocol Specification](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md), adding a new transaction type, **BURN**.

SLP-enabled node implementations, exchange and payment processor API's, and other recipients of signed token transactions who relay to the wider network, have an incentive to prevent inadvertant "burning" of tokens via malformed transactions. Adding the **BURN** transaction type allows senders to explictly signal to relaying entities that token burns are done on purpose.

### BURN - Burn Transaction
#### (Remove token units from circulation)
The following transaction format is used to remove token units from circulation. It is similar in format to a [SEND transaction](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md#send---spend-transaction) except that it creates no token outputs. Nodes and other applications checking for explicit burns should reject the transaction if the sum of the token quantities in the inputs does not equal `token_burn_quantity`.  Any number of additional BCH-only outputs will be allowed.

**Transaction inputs**: Any number of inputs or content of inputs, in any order, but must include sufficient tokens coming from valid token transactions of matching `token_id`, `token_type` (see [Consensus Rules](https://github.com/simpleledger/slp-specifications/blob/master/slp-token-type-1.md#consensus-rules)).

**Transaction outputs**:
<table>
  <tr>
    <td><b>v<sub>out</sub></b></td>
    <td><b>ScriptPubKey ("Address")</b></td>
    <td><b>BCH<br/>amount</b></td>
    <td><b>Implied token amount<br/>(base units)</b></td>
  </tr>
  <tr>
    <td>0</td>
    <td>OP_RETURN<BR>
&lt;lokad id: 'SLP\x00'&gt; (4 bytes, ascii)<BR/>
&lt;token_type: 1&gt; (1 to 2 byte integer)<BR/>
&lt;transaction_type: 'BURN'&gt; (4 bytes, ascii)<BR/>
&lt;token_id&gt; (32 bytes)<BR/>
&lt;token_burn_quantity&gt; (<b>required</b>, 8 byte integer)
  <td>any</td>
  <td>0</td>
  </tr>
    <td>...</td>
    <td>Any</td>
    <td>any</td>
    <td>0</td>
  </tr>
</table>