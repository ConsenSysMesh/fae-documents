# Two-party swap

This demo presents a simple but general contract, the “two-party swap”, for mediating a barter interaction.  It assumes that the reader understands the contents of the [tutorials](https://consensys.quip.com/folder/tutorial-2).

## Description of the swap contract

The two-party swap assists with transactions between two equal parties, each of which intends to offer something in exchange for the other's offering.  The quantities exchanged need not be currency, nor need they exist entirely within Fae, and the contract enforces a symmetric treatment of the parties.  The course of the exchange is of three phases:

1. **Offer:** Both parties sign an initial transaction creating the swap contract and endowing it with temporary ownership of each of their offerings.
2. **Evaluation:** In separate transactions, each party must submit a decision to the swap contract, either assent or refusal.
3. **Collection:** Depending on the two decisions, each party may collect one of the offerings:
    1. If either one refused, then both can collect their own offering back.
    2. If both assented, then each one can collect the other's offering.

Due to the logic of phase 3, the offerings are not “earmarked” for the other party during phase 1: if either party refuses, then ownership reverts for both, rather than one or both being locked in.  The parties are therefore separately capable of unilateral veto, and the swap occurs if and only if it is satisfactory to both parties.

## Interactive session

To see the swap contract in action, one can use [`curlTX`](https://consensys.quip.com/kN9MAhiNm8dz/Project-information)with the following transactions (which can all be found in the [git repository](https://github.com/ConsenSys/Fae)'s subdirectory `txs`), after starting the Fae server:

```
> stack exec faeServer
```

### Phase 1

**TwoPartyTX1.hs:**

```
import Blockchain.Fae.Contracts

body :: Transaction Void ()
body _ = twoPartySwap x y where
  x = "Hello from A!" :: String
  y = "Hello from B!" :: String
```

**TwoPartyTX1:**

```
body = TwoPartyTX1
keys
  partyA = key1
  partyB = key2
```

**Command line:**

```
> stack exec curlTX TwoPartyTX1
Transaction a23993f7115b499488dfc6175929055b7b18b655ef19638dfd485bb1b83a66da
  result: ()
  outputs: [0]
  signers:
    partyA: b822a487e1ce529e28ffb2fe77c1e40eed6187ad8f6b3c81406ee5c4874444ee
    partyB: ea8c2940e0f510be44a68e1a82fd2c0ba814a043ee2b5861a7a7f6e97a9ed820
```

(The hashes will be different)

This transaction creates a swap contract with simple messages as offerings; it establishes the two parties as those with keys named `key1` and `key2`.  In the command output, the line `outputs: [0]` indicates that a single contract has been created by the transaction, with index 0.

### Phase 2

**TwoPartyTX2-3.hs:**

```
body :: Transaction Void ()
body _ = return ()
```

This transaction does nothing other than call a contract and ignore its return value.  The entire value of the transaction is in the state update of the swap contract.

**TwoPartyTX2-3:**

```
body = TwoPartyTX2-3
inputs
  TransactionOutput $tx1 0 = Just True
keys
  self = $key  
```

This transaction specification expects environment variables `tx1` and `key`, corresponding to the transaction that created the swap contract (which we saw above had a single output indexed 0), and to the party making the call to the swap contract.  In phase 2, the parties submit decisions to the contract; this transaction makes both of those decisions assent.  As an alternative, the `True` can be changed to `False`, or the `Just True` to `Nothing`, to see the resulting diagnostic messages.

**Command line:** 

```
> key=key1 \
  tx1=a23993f7115b499488dfc6175929055b7b18b655ef19638dfd485bb1b83a66da \
  stack exec curlTX TwoPartyTX2-3  
Transaction d1f326bc1f6ebef19cf33b1d411e90eb574a66fd5699b73af6ca3bdbdf2d11c4
  result: ()
  outputs: []
  signers:
    self: b822a487e1ce529e28ffb2fe77c1e40eed6187ad8f6b3c81406ee5c4874444ee
  input 1347ac817e198a2c595b55d704d84f3bda7ac5dee43d1f40dec8f67031619a54
    nonce: 1
    outputs: []
    versions:
    
> key=key2 \
  tx1=a23993f7115b499488dfc6175929055b7b18b655ef19638dfd485bb1b83a66da \
  stack exec curlTX TwoPartyTX2-3
Transaction f8de5dffc626b2e44988ba03e6681cf3870dc1fcb31d0ec94b9fa0a9558bbb1d
  result: ()
  outputs: []
  signers:
    self: ea8c2940e0f510be44a68e1a82fd2c0ba814a043ee2b5861a7a7f6e97a9ed820
  input 1347ac817e198a2c595b55d704d84f3bda7ac5dee43d1f40dec8f67031619a54
    nonce: 2
    outputs: []
    versions:
```

In these command outputs, the lines beginning with `input` reflect the outcome of calling the various input contracts, in this case just the swap contract.  They show that the nonce (number of times successfully called) increments each time, and that no new contracts are created by either call.  The `versions` field is not important in this example.

Although this is not displayed, the fact that the transactions completed without error indicates that the state was indeed updated to include two assents.  A further attempt to vote by either party will cause an error.

### Phase 3

**TwoPartyTX4-5.hs:**

```
body :: Transaction (Maybe (Either (Versioned String) (Versioned String))) String
body (Just (Left (Versioned s))) = return s
body (Just (Right (Versioned s))) = return s
body _ = return ""
```

Unlike transactions 2-3, here we use the return value of the swap contract call, so the transaction's argument type is the full return type of the swap contract rather than the black-hole type `Void`.

**TwoPartyTX4-5:**

```
body = TwoPartyTX4-5
inputs
  TransactionOutput $tx1 0 = Nothing
keys
  self = $key
```

Identical to transactions 2-3 except that the argument to the swap contract is `Nothing`, reflecting the absence of an attempt to provide a decision.  One can run it with `Just True` or `Just False` arguments instead, or run it twice with the same key, to see the errors.

**Command line:**

```
> key=key1 \
  tx1=a23993f7115b499488dfc6175929055b7b18b655ef19638dfd485bb1b83a66da \
  stack exec curlTX TwoPartyTX4-5
Transaction 21d8955730be3d85deb893db90658c89ff94ddbc648f8c7420cd2829a64608d4
  result: "Hello from B!"
  outputs: []
  signers:
    self: b822a487e1ce529e28ffb2fe77c1e40eed6187ad8f6b3c81406ee5c4874444ee
  input 1347ac817e198a2c595b55d704d84f3bda7ac5dee43d1f40dec8f67031619a54
    nonce: 3
    outputs: []
    versions:
    
> key=key2 \
  tx1=a23993f7115b499488dfc6175929055b7b18b655ef19638dfd485bb1b83a66da \
  stack exec curlTX TwoPartyTX4-5
Transaction 5ced804503d3d93ffe5250520a1544e762caf2df431de0fd6aee9f812478de51
  result: "Hello from A!"
  outputs: []
  signers:
    self: ea8c2940e0f510be44a68e1a82fd2c0ba814a043ee2b5861a7a7f6e97a9ed820
  input 1347ac817e198a2c595b55d704d84f3bda7ac5dee43d1f40dec8f67031619a54
    nonce: -1
    outputs: []
    versions:    
```

Each of these transactions correctly obtains the other party's offering.  Note that the final nonce of the swap contract is -1, indicating that it has been closed.

## Motivation and discussion

The structure of the swap transaction allows the parties to make an off-Fae evaluation of each other's offerings.  For instance, the swap could be a real estate transaction, with the offerings being a digital deed and a mortgage application; then the buyer would need to examine the property and the seller would need to examine the amount of the intended payment and the validity of the loan.

One could object that, since both parties need to sign off on the swap contract's creation in phase 1, there is no need for phases 2 and 3 at all, and instead the parties could simply write a single transaction directly transferring ownership of their offerings if both agreed, and not write the transaction at all otherwise.  Although this is true for an exchange of “pure” quantities, the example of the real estate transaction suggests the possibility that the real-world component of the transaction may be more complex than simply transferring ownership.  The seller may require an application process, leaving them the opportunity to check the buyer's credit, and the buyer, of course, will want to inspect the property.  The existence of the unfulfilled swap contract then serves as a commitment to proceed in good faith, as in an ordinary home-buying transaction, where the seller accepts one and only one of many possible offers, but the finalization of the contract is deferred for inspection, with the understanding that if all is well, the buyer will, in fact, pay and the seller will, in fact, deliver.  The Fae contract replaces this trust-based understanding with automated enforcement.

Continuing with the real estate example, we see that the swap contract serves as a neutral intermediary, say the title company that normally presides over the contract's closing.  Although Fae eliminates much of the need for intermediaries, the title company's role doubles as enforcement of a reporting requirement, creating a document trail of standard forms.  Off-Fae, the swap contract serves as a powerless and ambiguous third party that may not have an opinion on the actual transaction, but which documents that it occurred properly, possibly recording additional secure information about the parties that must not be placed in a public ledger such as Fae.  This replaces the role of local government and the judiciary in settling ownership disputes; in this role, it is important that the transaction occurred via a standard mechanism so that it can be located on Fae in the event of, say, an exploit permitting the seller access to two deeds to the same property and selling it twice.  A dispute arising from such a failure cannot, by definition, be settled on Fae, and the requirement that the transactions at least be mediated by the standard two-party swap makes them transparent to real-world lawyers and judges.

## The swap contract as a Fae contract

The two-party swap contract is part of the Fae standard library, `Blockchain.Fae.Contracts`.  We reproduce it here for discussion:

### Creating a Swap Contract

```
twoPartySwap ::
  (
    HasEscrowIDs a, HasEscrowIDs b, 
    Versionable a, Versionable b,
    Typeable a, Typeable b,
    NFData a, NFData b,
    MonadTX m
  ) =>
  a -> b -> m ()
twoPartySwap x = do
  partyA <- signer "partyA"
  partyB <- signer "partyB"
  newContract [bearer x, bearer y] $ twoPartySwapC partyA partyB x y
```

This is the function called in phase 1 to create the swap contract.  It requires that the calling transaction have two named signatories `partyA` and `partyB`, and accepts the offerings as its two arguments.  The fact that the transaction is signed by both parties means that each one can, in the same transaction, perform whatever contract calls are necessary to obtain their offering.  Because the offerings are valuable, they must be instances of `HasEscrowIDs` and `Versionable`, to contain and protect those values.  The actual `Contract` function is then the evaluation of the following function on the parties' identities and offerings:

### Contract type signature

```
twoPartySwapC :: 
  (HasEscrowIDs a, HasEscrowIDs b, Versionable a, Versionable b) =>
  PublicKey -> PublicKey -> 
  a -> b ->
  Contract (Maybe Bool) (Maybe (Either (Versioned a) (Versioned b)))
```

The return value of this function is a `Contract` accepting `Maybe Bool` arguments and returning, possibly, either one of the offering types.  The boolean represents each party's decision, with `True` meaning assent;the fact that it is wrapped in `Maybe` reflects the difference between phase 2, where decisions are allowed, and phase 3, where they are not.  In the return type, the  `Maybe`  represents the fact that, until phase 3, the swap contract does not actually deliver anything, but simply records decisions.  Once phase 3 arrives, each party will get one of the offerings from calling the contract, though whether it is offering `a` or offering `b` depends, of course, on the results of phase 2.

### The two parts

Phases 2 and 3 of the swap each have two stages, corresponding to the actions in that phase by each of the parties.  For convenience, we have an enumeration type:

```
data Stages = Stage1 | Stage2
```

The high-level description of the contract is then:

```
twoPartySwapC partyA partyB x y choice = do
  values <- part1 Stage1 choice
  noChoice <- release Nothing
  part2 Stage1 noChoice values
  
  where
```

 `part1`, when finished, determines which offering each party can receive in `part2`, after which the first of the collection requests is awaited.

### Safe signatures

In each part, we use the following function to safely get the sender of a contract call:

```
getPartySigner = do
  who <- signer "self"
  unless (who == partyA || who == partyB) $ throw NotAParty
  return who
```

This simply checks that the signer called `self` actually is one of the parties.

### Exceptions

As shown in the last code block, this contract uses custom named exceptions to flag contract-level errors.  The exception type is:

```
data TwoPartyException = NotAParty | MustVote | AlreadyVoted | CantVote | AlreadyGot
  deriving (Show)
    
instance Exception TwoPartyException
```

The latter four exceptions will arise below.

### Part 1

```
part1 _ Nothing = throw MustVote
part1 _ (Just False) = return (xRet, yRet)
part1 Stage1 (Just True) = do
  sender1 <- getPartySigner
  choice2 <- release Nothing
  sender2 <- getPartySigner
  when (sender1 == sender2) $ throw AlreadyVoted
  part1 Stage2 choice2
part1 Stage2 (Just True) = return (yRet, xRet)

-- Convenient abbreviations
xRet = Left $ Versioned x
yRet = Right $ Versioned y
```

The operation of `part1` is simple: it accepts two decisions (that is, `Just` booleans; a `Nothing` at this point is an error) and, if either is a refusal, immediately terminates the swap and authorizes the return of the offerings to their original owners.  Otherwise, it authorizes the exchange of offerings.  Neither party is allowed to change their decision; the reason for this is that if `sender1` at first voted `True` with the expectation of possibly later changing to `False`, they set up a “race” with the other party to get the next vote.  Therefore the option to change one's decision is not a contractually guaranteed right, and should be forbidden.

### Part 2

```
part2 _ (Just _) _ = throw CantVote
part2 Stage1 Nothing (forA, forB) = do
  receiver1 <- getPartySigner
  let 
    orderedValues
      | receiver1 == partyA = (forA, forB)
      | otherwise = (forB, forA)
  noChoice <- release $ Just $ fst orderedValues
  receiver2 <- getPartySigner
  when (receiver1 == receiver2) $ throw AlreadyGot 
  part2 Stage2 noChoice orderedValues
part2 Stage2 Nothing (_, ret) = spend $ Just ret
```

`part2` establishes the convention that `values`, as a pair, records in its first component the value given to `partyA` and in its second component the value given to `partyB`.  This explains how the result of `part1` is formatted.  Only `Nothing` decisions are allowed at this point, and each party is entitled to call the swap contract once (successfully) to get their value.  Once both do, the contract is closed.
