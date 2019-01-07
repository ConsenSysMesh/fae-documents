# The auction demo

The auction demo consists of one library, `Auction.hs`, and several transaction
templates that use it.

First, start `faeServer`, instructing it to start with a blank transaction
history.

```
faeServer.sh --new-session
```

## Getting coins

Before starting the auction, you need to get some money.  Use the following
sequence of transactions to give a particular named identity, say "person", some
desired number of coins:

```
postTX.sh -e self=person GetCoin
# Copy the transaction ID `<tx1>`
postTX.sh -e self=person -e coinTX=<tx1> GetMoreCoins
# Copy the transaction ID `<tx2>`
postTX.sh -e self=person -e coinTX=<tx2> GetMoreCoins
...
```

The last transaction ID is the one that creates the "account" containing the
final sum of coins.  The intermediate accounts are all removed.

## The auction

Start the auction:

```
postTX.sh Create
# Copy the transaction ID as <tx1>
```

A party can bid using an account, created as above, where the final transaction
ID is `<acctTX>`.  The entire contents of the account will be bid; if the same
party bids again, the amount is added to the running total of coins as the
actual bid.

```
postTX.sh -e key=<name> -e aucTX=<tx1> -e coinTX=<acctTX> Bid
```

Finally, once the auction is over (the last `Bid` will produce a message to that
effect), the seller (whose key name is `seller`) and the losers can get their
coins back.  The next transaction creates a new account with the party's entire
bid or, for the seller, the winning party's winning bid.

```
postTX.sh -e key=<name> -e aucTX=<tx1> Collect
```

## Running the auction via Faeth

The auction can be run via Faeth, and therefore over the Ethereum blockchain,
with a few modifications.  You will need the Parity Ethereum client:

```
# Remove `--config=dev` to run this over the public Ethereum network.
parity --config=dev --ws-apis=all
```

When starting `faeServer`, add the flag `--faeth`.

When running each `postTX.sh` call, also add `--faeth` (to the end of the line).

This will communicate the Fae transactions exactly as in the sandboxed demo of
the previous section.  For a more realistic experience, you can simulate the
seller demanding the right of refusal for each bid and also charging a fee in
Ether by running a bid in two steps.  You will need to note the public key of
each identity, which is displayed after running any of the transactions with
that identity.  Call it `pubKey`.

```
# Bid part 1
postTX.sh -e key=<pubKey> -e aucTX=<tx1> -e coinTX=<acctTX> Bid --faeth-fee=<amt in wei>
# faeServer will produce an error
# postTX will show the Ethereum transaction ID, <ethTX>, and the Fae ID.

# Bid part 2
postTX.sh <ethTX> --faeth-eth-value=<amt in wei> --faeth-add-signature=<name>:<pubKey>
# faeServer will show that the transaction was added
# postTX will show the new Ethereum ID and the old Fae ID, <faeTX>
```

None of the Faeth transactions will produce visible transaction results, but you
can view them by sending a request to `faeServer`:

```
postTX.sh <faeTX> --view
```

