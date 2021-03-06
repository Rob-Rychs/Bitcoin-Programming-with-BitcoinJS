# 8.2: Multi-signature Native Segwit 2 of 4

> To follow along this tutorial and enter the commands step-by-step
> * Type `node` in a terminal after `cd` into `./code` for a Javascript prompt
> * Open the Bitcoin Core GUI console or use `bitcoin-cli` for the Bitcoin Core commands
> * Use `bx` aka `Libbitcoin-explorer` as a handy complement 

Let's create a 2 of 4 multi-signature with a native Segwit P2WSH transaction.


## Creating and Funding the P2SH 
 
Import libraries, test wallets and set the network
```javascript
const bitcoin = require('bitcoinjs-lib')
const { alice, bob, carol, dave } = require('./wallets.json')
const network = bitcoin.networks.regtest
```

Prepare four key pairs.
```javascript
const keyPairAlice0 = bitcoin.ECPair.fromWIF(alice[0].wif, network)
const keyPairBob0 = bitcoin.ECPair.fromWIF(bob[0].wif, network)
const keyPairCarol0 = bitcoin.ECPair.fromWIF(carol[0].wif, network)
const keyPairDave0 = bitcoin.ECPair.fromWIF(dave[0].wif, network)
```

And an other one for Alice that will redeem the multi-signature funds.
```javascript
const keyPairAlice1 = bitcoin.ECPair.fromWIF(alice[1].wif, network)
const p2wpkhAlice1 = bitcoin.payments.p2wpkh({pubkey: keyPairAlice1.publicKey, network})
```

Create the locking script with the special `p2ms` payment method.
```javascript
const p2ms = bitcoin.payments.p2ms({
  m: 2, pubkeys: [
    keyPairAlice0.publicKey,
    keyPairBob0.publicKey,
    keyPairCarol0.publicKey,
    keyPairDave0.publicKey], network})
```

Check the locking script.
```
$ decodescript [p2ms.output.toString('hex')]
``` 
`"2 03745c9aceb84dcdeddf2c3cdc1edb0b0b5af2f9bf85612d73fa6394758eaee35d 
027efbabf425077cdbceb73f6681c7ebe2ade74a65ea57ebcf0c42364d3822c590 
023a11cfcedb993ff2e7523f92e359c4454072a66d42e8b74b4b27a8a1258abddd 
02e9d617f38f8c3ab9a6bde36ce991bafb295d7adba457699f8620c8160ec9e87a 4 OP_CHECKMULTISIG"`

Feed the `p2wsh` method with the special BitcoinJS `p2ms` object.
The `p2wsh` method generates an object that contains the P2WSH address.
```javascript
const p2wsh = bitcoin.payments.p2wsh({redeem: p2ms, network})
```

Send 1 BTC to this P2WSH address. 
```
$ sendtoaddress bcrt1qtvraes6lc2efmwqtupv7f9wg3adhvzwpu0vg3s2zgpnc7qpp0v7sj6dkmu 1
```

Get the output index so that we have the outpoint (txid / vout).
> Find the output index (or vout) under `details > vout`.
```
$ gettransaction "txid"
```


## Preparing the spending transaction

Now let's prepare the spending transaction by setting input and output and having two people (private keys) to sign the 
transaction. 
Here Alice_0 ad Bob_0 will redeem the P2WSH multi-signature and send the funds to Alice_1 P2WPKH address.

Create a BitcoinJS transaction builder object.
```javascript
const txb = new bitcoin.TransactionBuilder(network)
```

Create the input by referencing the outpoint of our P2SH funding transaction.
Create the output that will send the funds to Alice_1 P2WPKH address, leaving 100 000 satoshis as mining fees.
```javascript
txb.addInput('TX_ID', TX_VOUT)
txb.addOutput(p2wpkhAlice1.address, 999e5)
```

Alice_0 and Bob_0 now sign the transaction.
Note that, because we are doing a P2WSH, we need to provide the locking script as witnessScript sixth parameter of the 
`sign` method, as well as the input value.
```javascript
// txb.sign(index, keyPair, redeemScript, sign.hashType, value, witnessScript)
txb.sign(0, keyPairAlice0, null, null, 1e8, p2wsh.redeem.output)
txb.sign(0, keyPairBob0, null, null, 1e8, p2wsh.redeem.output)
```

Build the transaction and get the raw hex serialization.
```javascript
const tx = txb.build()
console.log('tx.toHex()  ', tx.toHex())
```

Inspect the raw transaction with Bitcoin Core CLI, check that everything is correct.
```
$ decoderawtransaction "hexstring"
```


## Broadcasting the transaction

It's time to broadcast the transaction via Bitcoin Core CLI.
```
$ sendrawtransaction "hexstring"
```

Inspect the transaction.
```
$ getrawtransaction "txid" true
```


## Observations

We can see that the scriptSig unlocking script is empty and instead all the data are located in the `txinwitness` field
  * an empty string that will convert to a useless but mandatory `00` value due to a bug in `OP_CHECKMULTISIG`
  * Alice_0 and Bob_0 signatures
  * and our redeem script


## What's Next?

Continue "PART THREE: PAY TO SCRIPT HASH" with [8.3: Multi-signature Embedded Segwit 2 of 4](08_3_Multisig_P2SH_P2WSH_P2MS_2_4.md).
