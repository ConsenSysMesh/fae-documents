# Auction

This demo presents a simple auction contract.  Unlike the [two-party swap](https://consensys.quip.com/VQoyAdJvBiaY/Two-party-swap) this is not part of `Blockchain.Fae.Contracts`, as it is just one of many conceivable formats for an auction and makes some simplifying assumptions that may not be realistic.  It demonstrates a more advanced use of contract state and the use of versioned values.

This demo assumes the use of the [docker images](https://consensys.quip.com/kN9MAhiNm8dz/Project-information#GTTACAfd57C) to run `postTX` and (especially) `faeServer`.

## Description of the auction contract

The auction has three phases: 

* **Creation:** an auction contract is created, holding the item for auction, its initial bid amount, and the number of bids to accept.
* **Bidding:** Anyone may submit a bid to the auction contract, which must be larger than the previous bid.
* **Remittance:** When the last bid is accepted, the bidder is awarded the item, and their bid is earmarked for the seller.  Subsequently, any of the other bidders or the seller may call the contract to retrieve their bid.

A more elaborate auction would have different rules for bidding and different termination conditions; it is probably not reasonable from a game-theoretic perspective to establish a well-known maximum number of bids, but suffices for this demo.

## Interactive session

To see the auction contract in action, one can use [`postTX`](https://consensys.quip.com/kN9MAhiNm8dz/Project-information)with the following transactions (which can all be found in the [git repository](https://github.com/ConsenSys/Fae)'s subdirectory `demos`), after starting the Fae server:

```
> ./faeServer.sh
```

### Phase 1

**Create.hs:**

```
import Auction
import Blockchain.Fae.Currency

body :: Transaction Void ()
body _ = do
  let price = 1 :: Valuation Coin
  let numBids = 3
  auction ("You won!" :: String) price numBids
```

We use the naive currency `Coin` from `Blockchain.Fae.Currency` for the auction currency.

**Create:**

```
body = Create
keys
  self = seller
others
  - Auction
```

The signer of the creation transaction is, by definition, the seller; we use a key of that name.  The `others` module `Auction` contains the definition of the auction contract and is described below.

**Command line:**

```
> ./postTX.sh Create
Transaction a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e
  result: ()
  outputs: [0]
  signers:
    self: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
```

One output, the auction contract itself, is created.

### Getting coins

**GetCoin.hs:**

```
import Blockchain.Fae.Contracts
import Blockchain.Fae.Currency

body :: Transaction RewardEscrowID ()
body rID = do
  coin <- reward rID
  deposit coin "self"
```

This abstraction of a money-making transaction awards a single coin to the signer; we will need to call it many times to get the funds to bid.

**GetCoin:**

```
body = GetCoin
reward = True
keys
  self = $key
```

This establishes the `GetCoin` transaction as receiving a reward, which is necessary to create a `Coin`.  The environment variable `key` allows us to change the party receiving the coin.

**GetMoreCoins.hs:**

```
import Blockchain.Fae.Contracts
import Blockchain.Fae.Currency

body :: Transaction (RewardEscrowID, Coin) ()
body (rID, oldCoin) = do
  coin <- reward rID
  newCoin <- add oldCoin coin
  deposit newCoin "self"
```

This transaction expands the block of `Coin`s owned by the signer; it withdraws a previous cache and adds a new coin to it.  The auction contract unsophisticatedly requires the bid to consist of a single `Coin` value, and this neatly provides it.

**GetMoreCoins:**

```
body = GetMoreCoins
reward = True
keys
  self = $key
inputs
  TransactionOutput $txID 0 = ()
```

This transaction, which is also a reward, calls a contract (of type `Contract () Coin`) so that the newly awarded coin can be added to its contents.

**Command line:**

We intend to make the following bids:

* `bidder1`: 2 (since the initial bid is 1, and the first bid must beat it)
* `bidder2:` 3
* `bidder1`: 2 more (total of 4)

We will create caches of these sizes all at once now.

```
> ./postTX.sh -e key=bidder1 GetCoin
Transaction a4aa63205405f312b291854f09a7147aac0a1f7583284d0323f45cfb73bc93dd
  result: ()
  outputs: [0]
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
        
> ./postTX.sh \
  -e key=bidder1 \
  -e txID=a4aa63205405f312b291854f09a7147aac0a1f7583284d0323f45cfb73bc93dd \
  GetMoreCoins
Transaction 42b008573bafdca193dd54ab1ea2e67550c5e4e2e16a0135a7a0a2c7d5f170f0
  result: ()
  outputs: [0]
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
  input 3a96abe35014cd41b4e9062094e1ac9a23623093fe620b1aeaed6f55314cb347
    nonce: -1
    outputs: []
    versions:        
```

Now `bidder1` has 2 `Coin`s.  Note that in the second transaction, the input ended up with nonce -1: this means that the first coin account, created by the first transaction, was closed, and replaced by the output of the second transaction.

```
> ./postTX.sh -e key=bidder2 GetCoin
Transaction d39dbffd8e5a372526c764f890f4d6dc757af4240675815289fe32f2b6f3d084
  result: ()
  outputs: [0]
  signers:
    self: 206ccd26bc061386969af7a77800b66577718caf0cbe37c68e93b68ec5606b51
    
> ./postTX.sh \
  -e key=bidder2 \
  -e txID=d39dbffd8e5a372526c764f890f4d6dc757af4240675815289fe32f2b6f3d084 \
  GetMoreCoins
Transaction 589c63d012e938e5c62d79bb99800fcf9bc9fc1c7c033f641351457d5a9fafd5
  result: ()
  outputs: [0]
  signers:
    self: 206ccd26bc061386969af7a77800b66577718caf0cbe37c68e93b68ec5606b51
  input 20e07c0eb3848954711716616ed702e203af7216bb3c72e5169460f4ebc05fed
    nonce: -1
    outputs: []
    versions:

> ./postTX.sh \
  -e key=bidder2 \
  -e txID=589c63d012e938e5c62d79bb99800fcf9bc9fc1c7c033f641351457d5a9fafd5 \
  GetMoreCoins
Transaction 5b7213a06e404fe8b21df05ca396b9b87e1b63698193c56ceaa1f952adf3de77
  result: ()
  outputs: [0]
  signers:
    self: 206ccd26bc061386969af7a77800b66577718caf0cbe37c68e93b68ec5606b51
  input c40516f94a602fa50d8ee2a826b29348df0ce3e7238111fcfb03ecfd9e7302bd
    nonce: -1
    outputs: []
    versions:
```

Now `bidder2` has 3 `Coin`s.

```
> ./postTX.sh -e key=bidder1 GetCoin
Transaction cccd576c002b497f42b4cc3f06bccec9786416b0fec399034575078166be67c7
  result: ()
  outputs: [0]
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc

> ./postTX.sh \
  -e key=bidder1 \
  -e txID=cccd576c002b497f42b4cc3f06bccec9786416b0fec399034575078166be67c7 \
  GetMoreCoins
Transaction 08d651e78077e480596def97b5cd01b8a4332d9d61fb86ea0ee8281c6dc78621
  result: ()
  outputs: [0]
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
  input 6b9c9171a46c0bc2f74cdad796418812dd89338fc28105873c50d2fa20a32f71
    nonce: -1
    outputs: []
    versions:
```

And finally, `bidder1` has 2 more `Coin`s, in a separate account from their first 2.

### Bidding

**Bid.hs:**

```
import Blockchain.Fae.Contracts
import Blockchain.Fae.Currency

body :: Transaction (Coin, Maybe (Either (Versioned Coin) (Versioned String))) String
body (_, Nothing) = return ""
body (_, Just (Right (Versioned s))) = return s
```

This transaction withdraws a `Coin` cache (which is provided to the auction contract as a bid and not used in the `body`), then echos the returned string.

**Bid:**

```
body = Bid
keys
  self = $key
  seller = seller
inputs
  TransactionOutput $coinTX 0 :# 0 = ()
  TransactionOutput $aucTX 0 = Just ($coinSCID ::: $coinVersion)
```

A bid has two signers, as explained below, one of whom is the seller and the other, the bidder.  The first input call gets the bid payment as a `Versioned` value, indicated by the explicit nonce reference in the contract ID.  The second call provides that versioned value to the auction contract.

**Command line:**

Now we make the bids, which are, again:

* `bidder1`: 2 (since the initial bid is 1, and the first bid must beat it)
* `bidder2:` 3
* `bidder1`: 2 more (total of 4)

We will use the three `Coin` accounts created in the previous sequence of transactions.

```
> ./postTX.sh \
  -e key=bidder1 \
  -e coinVersion="" \
  -e coinSCID="" \
  -e coinTX=42b008573bafdca193dd54ab1ea2e67550c5e4e2e16a0135a7a0a2c7d5f170f0 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Bid --fake
Transaction 4b7037f6151488a68009f145230692a5c2204a846d5613e70755fe4c2a41bda2
  result: <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
  outputs: <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
    seller: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input 86ff79fb0e5c2451f21203338e80403875ed361be417659a058915f2d035c4df
    nonce: -1
    outputs: []
    versions:
      8030687d9ce3f9a82dfe5b250fc10baa568c3b92db5c527cf951e422aec8dc1d: EscrowID Token CoinVal
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
    
> ./postTX.sh \
  -e key=bidder1 \
  -e coinVersion=8030687d9ce3f9a82dfe5b250fc10baa568c3b92db5c527cf951e422aec8dc1d \
  -e coinSCID=86ff79fb0e5c2451f21203338e80403875ed361be417659a058915f2d035c4df
  -e coinTX=42b008573bafdca193dd54ab1ea2e67550c5e4e2e16a0135a7a0a2c7d5f170f0 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Bid
Transaction a1411c29488857c72184dc3268c44a4b5f2577a24f0fe53076de921a2cead688
  result: ""
  outputs: []
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
    seller: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input 86ff79fb0e5c2451f21203338e80403875ed361be417659a058915f2d035c4df
    nonce: -1
    outputs: []
    versions:
      8030687d9ce3f9a82dfe5b250fc10baa568c3b92db5c527cf951e422aec8dc1d: EscrowID Token CoinVal
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    nonce: 1
    outputs: []
    versions:
```

We use `bidder1` to make first a fake bid, then a real one; from the fake bid, we learn the version ID of the `Coin` value for the payment, which is displayed under `versions:` in the first `input` display (the `Coin` name is an alias for `EscrowID Token CoinValues`).  The errors in the transaction result, outputs, and second input reflect the fact that the coin version provided and the “short contract ID” of the contract that returned the coin, which are both the empty string, make no sense, which is intentional since the fake transaction exists solely to determine the version and short contract ID.  In the second, real transaction, there are no errors, indicating that the bid was accepted.

```
> ./postTX.sh \
  -e key=bidder2 \
  -e coinVersion="" \
  -e coinSCID="" \
  -e coinTX=5b7213a06e404fe8b21df05ca396b9b87e1b63698193c56ceaa1f952adf3de77 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Bid --fake
Transaction 5a015d56c56d59dbcd9d686a3e2dc22d5f4d1bfebb30d9c0fea23388c4c9a218
  result: <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
  outputs: <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
  signers:
    self: 206ccd26bc061386969af7a77800b66577718caf0cbe37c68e93b68ec5606b51
    seller: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input f35adaf19e5d03f2fc6ae3647aa1269a8da45d454f23a81a173da6b527b06558
    nonce: -1
    outputs: []
    versions:
      2eb80ae2d5ee3fa336fec46cf76238d3b625e110018e2e7601569420c22f22b6: EscrowID Token CoinVal
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'

> ./postTX.sh \
  -e key=bidder2 \
  -e coinVersion=2eb80ae2d5ee3fa336fec46cf76238d3b625e110018e2e7601569420c22f22b6 \
  -e coinSCID=f35adaf19e5d03f2fc6ae3647aa1269a8da45d454f23a81a173da6b527b06558 \
  -e coinTX=5b7213a06e404fe8b21df05ca396b9b87e1b63698193c56ceaa1f952adf3de77 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Bid
Transaction 365bd002168bc7431814d4e79d530fbfcfc7176a91d02e3ce65ed28b65d1e58d
  result: ""
  outputs: []
  signers:
    self: 206ccd26bc061386969af7a77800b66577718caf0cbe37c68e93b68ec5606b51
    seller: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input f35adaf19e5d03f2fc6ae3647aa1269a8da45d454f23a81a173da6b527b06558
    nonce: -1
    outputs: []
    versions:
      2eb80ae2d5ee3fa336fec46cf76238d3b625e110018e2e7601569420c22f22b6: EscrowID Token CoinVal
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    nonce: 2
    outputs: []
    versions:
```

Now `bidder2` makes first a fake, then a real bid, which is accepted.

```
> ./postTX.sh \
  -e key=bidder1 \
  -e coinVersion="" \
  -e coinSCID="" \
  -e coinTX=08d651e78077e480596def97b5cd01b8a4332d9d61fb86ea0ee8281c6dc78621 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Bid --fake
Transaction c260176a4bbbcd73d01c675ecdcf5a3691dcaf5a122c3aefa2bf4b9fa7dd2c3c
  result: <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
  outputs: <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
    seller: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input c1bbf5a4d7e0df9808fa28fb1063354d6a477af3478ce905788f44cfa6911d8e
    nonce: -1
    outputs: []
    versions:
      85e906f8f13be8570c1d01e8d46996461395034f8898faf528112450d6a04045: EscrowID Token CoinVal
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    <exception> Unable to parse '"Just ( ::: ) as type: Maybe (Versioned (EscrowID Token CoinVal))"'

> ./postTX.sh \
  -e key=bidder1 \
  -e coinVersion=85e906f8f13be8570c1d01e8d46996461395034f8898faf528112450d6a04045 \
  -e coinSCID=c1bbf5a4d7e0df9808fa28fb1063354d6a477af3478ce905788f44cfa6911d8e \
  -e coinTX=08d651e78077e480596def97b5cd01b8a4332d9d61fb86ea0ee8281c6dc78621 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Bid
Transaction 80a2e90fd2fedee5959c5c9581f37eb0d87b215df889e23ac4bced11a2b75209
  result: "You won!"
  outputs: []
  signers:
    self: 3b344cb8b4e9b06df96054f1c63465504cc46f5a19752d41b2bea47cdbad33cc
    seller: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input c1bbf5a4d7e0df9808fa28fb1063354d6a477af3478ce905788f44cfa6911d8e
    nonce: -1
    outputs: []
    versions:
      85e906f8f13be8570c1d01e8d46996461395034f8898faf528112450d6a04045: EscrowID Token CoinVal
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    nonce: 3
    outputs: []
    versions:
```

Finally, `bidder1` makes the final bid and wins the auction.

### Remittance

**Withdraw.hs:**

```
import Blockchain.Fae.Contracts
import Blockchain.Fae.Currency

body :: Transaction (Maybe (Either (Versioned Coin) (Versioned String))) String
body (Just (Left (Versioned c))) = do
  deposit c "self"
  return "Withdrew"
body (Just (Right (Versioned s))) = return s
body Nothing = return ""
```

This transaction deposits the withdrawn funds in an account owned by the signer.  In this phase, the auction contract will not return the item, but for completeness, we include a pattern for a `Right` return value anyway.

**Withdraw:**

```
body = Withdraw
keys
  self = $key
inputs
  TransactionOutput $aucTX 0 = Nothing
```

We call the auction contract with `Nothing` to indicate intent to withdraw, rather than bid.

**Command line:**

```
> ./postTX.sh \
  -e key=bidder2 \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Withdraw
Transaction 0b04b2f9780109c62efb9828aae4ebf9778a43499675e4c974c1af582c8cd92f
  result: "Withdrew"
  outputs: [0]
  signers:
    self: 206ccd26bc061386969af7a77800b66577718caf0cbe37c68e93b68ec5606b51
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    nonce: 4
    outputs: []
    versions:
```

We see that `bidder2` gets their bid back.

```
> ./postTX.sh \
  -e key=seller \
  -e aucTX=a82b661c0e6a662947bf9d7e271ce23af4980bd25ec04161305baec5e77a349e \
  Withdraw
Transaction 0cde2682e4a88b4d46d37f4a081e5c33469f92808c8b4f515c66459b0a647d2b
  result: "Withdrew"
  outputs: [0]
  signers:
    self: 0331775a097e2b85c1f3cb17a802009616dc220b24cb1a2b53168147fe5cf2ca
  input f4c77feab60e43394a9245e4999c440e321675f7caefd5e84a27b0855e85f0df
    nonce: 5
    outputs: []
    versions:
```

The seller also gets to collect the winning bid.  The auction contract does not close, though it is possible to write it to do so.

## The auction contract

This contract is an example of the use of an explicit state object and returning versioned values.

### Creating an auction

```
data AuctionState coin = 
  BidState
  {
    bids :: Map PublicKey coin,
    highBid :: Valuation coin,
    seller :: PublicKey,
    bidsLeft :: Natural
  } |
  RemitState (Map PublicKey (Maybe coin))

auction :: 
  (NFData a, Versionable a, HasEscrowIDs a, Currency coin, MonadTX m) =>
  a -> Valuation coin -> Natural -> m ()
auction _ _ 0 = throw NoBids
auction x bid0 maxBids = do
  seller <- signer "self" 
  let state0 = BidState Map.empty bid0 seller maxBids
  newContract [bearer x] $ flip evalStateT state0 . auctionC x
```

The auction state records the progress of the auction.  During bidding, various pieces of information about the bids and their relationships need to be retained; once bidding ends, the only thing to remember is who gets what amount back, and whether they have already done so.  When the auction is created, the state is initialized with no bids, an initial high bid and number of bids given as arguments, and the signer of the transaction as the seller.

The somewhat elliptical expression `flip evalStateT state0 . auctionC x` means that the `auctionC` function returns not a `Contract` but rather, a contract with state, in this case state managed by the `StateT` monad transformer on top of a `Fae` monad.  By providing the initial state to this monad, we reduce it back down to\`Contract`, which is required by `newContract`.

### Auction outline

```
auctionC ::
  (HasEscrowIDs a, Currency coin) =>
  a -> 
  ContractM (StateT (AuctionState coin)) 
    (Maybe (Versioned coin))
    (Maybe (Either (Versioned coin) (Versioned a)))
auctionC x bidM = do
  bidStage bidM 
  -- The last bidder was the one with the highest (winning) bid, so they
  -- get the prize
  next <- release (Just $ Right $ Versioned x) 
  remitStage next
```

The type signature of `auctionC` confirms the description above; given an item to auction, it returns a `ContractM (StateT (AuctionState coin))`, which means a `Contract` whose monad is augmented with a `StateT (AuctionState coin)` transformer; i.e. a contract tracking a state of type `AuctionState coin`.

The argument type of this contract is `Maybe (Versioned coin)`, which indicates two things: first, that it may be called either with no bid, or with a bid provided by version.  This is the necessary structure for a contract expecting payment, which cannot be provided as a literal value but must be retrieved as the result of another contract call, for which it must be referenced by version for safety.

The return type is `Maybe (Either (Versioned coin) (Versioned a))`, meaning that if it returns anything, that return value may be either a monetary value (corresponding to remittance of a losing bid or collection of the winning bid) or the item up for bid.  Both of these are protected by versioning, which is actually not explicitly necessary (Fae will produce versions for all subobjects of the return value by default) but does limit the number of versions produced to those that are somehow meaningful.

The progress of the auction is in two stages, corresponding to the bidding and remittance phases of the description given at the top.  The `bidStage` subcontract starts off with the argument to the first contract call, and returns nothing, indicating that its only effect is to update the contract state.  Once it is complete, the current signer is the winner, and the item is released.  The subsequent argument is awaited and then passed to the remittance stage.

### Auction errors

```
data AuctionError =
  NoBids | MustBid | CantBid | CantRemit | AlreadyGot | 
  MustBeat Natural | UnauthorizedSeller PublicKey
  deriving (Show)

instance Exception AuctionError
```

### The bidding stage

```
bidStage :: 
  (Currency coin, HasEscrowIDs a) => 
  Maybe (Versioned coin) ->
  StateT 
    (AuctionState coin) 
    (Fae (Maybe (Versioned coin)) (Maybe (Either (Versioned coin) (Versioned a))))
    ()
```

The bid stage accepts payments in the form of `coin` values, returning a null action in the same `Fae` monad as `auctionC`

```
bidStage Nothing = throw MustBid
bidStage (Just (Versioned inc)) = do
```

There are two patterns: no bid (which is an error at this stage) or a versioned `coin` bid.  All the logic is in the latter.

```
  BidState{..} <- get
  claimedSeller <- signer "seller"
  unless (claimedSeller == seller) $ throw (UnauthorizedSeller claimedSeller)
```

First the state is retrieved and its fields unpacked into variables named after them (this is the `{..}` syntax).  The signer named `seller` is compared to the stored seller, because for safety, payments should be signed onto by the recipient to combat potentially dangerous “counterfeit” values.

```
  bidder <- signer "self"
  let oldBidM = Map.lookup bidder bids
  newBidCoin <- maybe (return inc) (add inc) oldBidM
  newBid <- value newBidCoin
  -- Make sure they are actually raising
  unless (newBid > highBid) $ throw (MustBeat $ fromIntegral highBid)
  let bidsLeft2 = bidsLeft - 1
```

The bidder may have made a previous bid, which would be recorded in `bids`; if so, the amount provided to the contract needs to be added to it to determine the total bid by this bidder.  If that total does not exceed the previous high bid, it is not accepted.

```
  if (bidsLeft2 > 0) 
  -- Loop
  then do
    let bids2 = Map.insert bidder newBidCoin bids
    put $ BidState bids2 newBid seller bidsLeft2
    next <- release Nothing 
    bidStage next
```

If any bids remain, the state is updated with the results of the current bid, and `bidStage` loops with the next call's argument.

```
  else put $ RemitState $ fmap Just $
    Map.insert seller newBidCoin $
    Map.delete bidder bids
```

If not, then the state transitions to remittance, with all of the current bidders marked as being owed their bids back, except that the winning bidder is replaced by the seller.

### The remittance stage

```
remitStage ::
  (Currency coin, HasEscrowIDs a) => 
  ContractM 
    (StateT (AuctionState coin)) 
    (Maybe (Versioned coin)) 
    (Maybe (Either (Versioned coin) (Versioned a)))
```

The signature for this stage is the same as the overall contract type.

```
remitStage Nothing = do
  RemitState remits <- get
  sender <- signer "self"
  let
    bid =  
      fromMaybe (throw AlreadyGot) $
      fromMaybe (throw CantRemit) $
      Map.lookup sender remits
```

Before paying out, the contract verifies that the caller a) is actually one of the parties, and b) hasn't already been paid.

```
  put $ RemitState $ Map.insert sender Nothing remits
  next <- release (Just $ Left $ Versioned bid)
  remitStage next
```

If all is well, the caller is marked as being owed nothing and their bid is released back to them.  The stage then loops with the next call's argument; this particular contract never closes, even after all the bids are retrieved.

```
remitStage _ = throw CantBid
```

Finally, it is an error to request a payout while at the same time submitting what appears to be a bid.
