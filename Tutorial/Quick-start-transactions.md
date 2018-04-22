# Quick start transactions

You can experiment with Fae using [the Docker images](https://consensys.quip.com/kN9MAhiNm8dz#GTTACAfd57C)once you gain access to the repository.  Here are a few simple transactions to illustrate its workings.

## Proof of acquaintance

This transaction simply records, in immutable Fae storage, the fact that two people affirm that they have met.

**Acquaintance.hs:**

```
import Control.Monad

body :: Transaction Void ()
body _ = do
  key1 <- signer "person1"
  key2 <- signer "person2"
  newContract [] $ \() -> forever $ release (key1, key2)
```

**Aquaintance:**

```
body = Acquaintance
keys
  person1 = key1
  person2 = key2
```

In the playground environment, Fae provides the cryptography, so you only need to name the keys (`key1` and `key2`) you want to use for each person; the names can be anything and you don't have to worry about whether they already exist.  Each time you run the transaction, you may replace the key names with others to get different “people” acquainted.

Each run of `postTX` to submit a transaction will return a dump of the results of that transaction's execution. It will look something like:

```
Transaction <TX ID hex string>
  result: ()
  outputs: [0]
  signers:
    person1: <pubkey hex string>
    person2: <pubkey hex string>    
```

This identifies the transaction by its immutable ID, indicates that one contract (indexed 0) was created, and displays the public keys of the signers.  Using this information you can query the records these transactions create to obtain the proofs of acquaintance:

**SeeAcquaintance.hs:**

```
import Data.Functor.Identity

body :: Transaction (Identity (PublicKey, PublicKey)) (PublicKey, PublicKey)
body keyPair = return keyPair
```

**SeeAcquaintance:**

```
body = SeeAcquaintance
inputs
  TransactionOutput $txID 0 = ()
```

This transaction should be sent with the environment variable `txID` set to the transaction ID shown in the above display.  You will see a similar display whose `result` field shows the two public keys, which should match the ones shown when the corresponding transaction was run.

For the curious, the `Identity` wrapping the input pair signals to Fae that the transaction is intended to accept *one* contract output that is a pair of public keys, rather than *two* contract outputs each of which is a public key.

## Secret keeping

This transaction saves a bundle of data in such a way that only the owner can retrieve it.  When they do, they can (optionally) choose to release that data.

**SecretTX.hs:**

```
import Blockchain.Fae.Contracts

import Secret

body :: Transaction Void PublicKey
body _ = do
  deposit Secret "self"
  signer "self"
```

**Secret.hs:**

```
module Secret (Secret, SecretID) where

import Blockchain.Fae

data Secret = Secret deriving (Generic, Show)

type SecretID = EscrowID () Secret
```

**Secret:**

```
body = SecretTX
keys
  self = $key
others
  - Secret
```

When you run the transaction `Secret` with the environment variable `key` set to some key name (say, `key1`), it will create a contract containing the abstract secret type `Secret`.  The `Secret` module is available to `SecretTX` under its own name, and to subsequent transactions under a qualified name, as shown below.  

Run the next transaction to retrieve and return the secret.

**GetSecret.hs:**

```
import Blockchain.Fae.Transactions.TX<TX ID hex string>.Secret

body :: Transaction Secret Secret
body = return
```

**GetSecret:**

```
body = GetSecret
inputs
  TransactionOutput $txID 0 = ()
keys
  self = $key
```

Before running this transaction, you will need to paste the transaction ID (found as before in the output from the first transaction) into the `GetSecret.hs` file, so that it can locate the same `Secret` module that the first transaction also imported.  Set the environment variable `txID` to this value as well.  You can experiment with `key`: if you set it to a different key, say `key2`, you will see an error indicating that the “secret” value is, indeed, access-restricted.  When set to the same key (say, `key1`) again, the display will show “Secret” as the output, indicating that the secret value has been revealed.
