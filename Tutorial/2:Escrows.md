# Tutorial 2: Escrows

In this second tutorial, we will introduce the remaining feature of Fae: *escrows*, which are the means by which you may create value and perform complex transactions.  As in the first tutorial, the sample code will use the conventions of [the postTX utility](https://consensys.quip.com/kN9MAhiNm8dz/Project-information#GTTACAArd2o) and the sample command lines use [the docker images](https://consensys.quip.com/kN9MAhiNm8dz/Project-information#GTTACAfd57C).  After following the instructions there, you should execute this first to have a local server running:

```
./faeServer.sh
```

## Creating escrows

To present the concepts surrounding escrows accessibly, we will start with not a full transaction, but simply a contract that uses them.

### Contract source code

```
c :: Contract String (EscrowID () String)
c name = newEscrow [] e where
  e :: Contract () String
  e _ = spend ("Property of: " ++ name)
```

This is a contract that creates an escrow personalized according to the caller's request.

### Escrow creation

The escrow creation function, `newEscrow`, has exactly the same arguments as `newContract`, including the second one having type `Contract argType valType`: escrows are the same as contracts but have a different role.  As we will see later, they can be called as well, with different restrictions; this involves the *escrow ID*, which is returned by `newEscrow`.  This particular escrow can be used once, returning a declaration of who created it, and is then deleted, just like contracts that end with `spend` are deleted when that line is executed.

### Escrow ID

An escrow is created with a unique identifier whose type is parallel to the contract type from which the escrow was created.  The reason the argument and value types are included is so that contracts that accept escrow IDs can guarantee that they get the right kind of escrow, and users of contracts that return escrow IDs can guarantee that *they* got the right kind of escrow.  As we will see, escrows can represent valuable quantities in Fae, so these guarantees amount to an assurance that the contracts deal in a particular kind of valuable (say, a specific currency).

## Using escrows

An escrow is of no use in isolation: it must be used, if only to verify it.  The transaction below will demonstrate calling it.  First, we should commit the contract above to Fae via a transaction.

**Transaction source file: Escrow1.hs**

```
body :: Transaction Void ()
body _ = newContract [] c where
  -- The same as the previous code sample
  c :: Contract String (EscrowID () String)
  c name = newEscrow [] e where
    e :: Contract () String
    e _ = spend ("Property of: " ++ name)
```

**Transaction message: Escrow1**

```
body = Escrow1
```

**Command line:**

```
./postTX.sh Escrow1
```

This should produce a transaction summary that includes the transaction ID, say `cab24f...`. This example also demonstrates how additional code beyond the transaction executable can be included in a transaction source file.

**Transaction source file: CallEscrow1.hs**

```
body :: Transaction (EscrowID () String) String
body eID = useEscrow eID ()
```

**Transaction message: CallEscrow1**

```
body = CallEscrow1
inputs
  TransactionOutput $txID 0 = Ryan Reich
```

**Command line:**

```
./postTX.sh -e txID=cab24f... CallEscrow1
```

The contract ID format `TransactionOutput $txID 0` should be familiar from the first tutorial, signifying that the contract to be called was the first one created by transaction `$txID`.  This transaction calls the contract above, passing it my name as an argument, and then uses it with the ID that was returned from that call.

### Calling an escrow

Escrows can be called like functions with `useEscrow`, which takes two arguments: the escrow ID and the argument to the escrow itself; here, the escrow accepts `()`.  This returns a string, as indicated by the second type argument to `EscrowID () String`, which is returned from the transaction.  The transaction result is therefore `Property of: Ryan Reich`.

### Escrow ownership

One subtle but extremely important point that comes out in this example is the *ownership* of this escrow.  Originally, it was owned by its creator, contract `TransactionOutput cab24f... 0`.  However, by returning the escrow ID from its call, that contract *transfers* ownership to the transaction that called it.  It is only because of this transfer that the `useEscrow eID` is valid; if an escrow is called by a contract or transaction that does not own it, the call will fail with an error indicating that the escrow does not exist.  Escrows only exist for their owners.

## Saving escrows

The transaction above does not use escrows to their full potential, but merely as glorified functions.  In Fae, escrows are the only means of creating persistent value, and in order to persist, they must be held by a contract rather than expended by a transaction.

**Transaction source file: Escrow2.hs**

```
body :: Transaction (EscrowID () String) ()
body eID = newContract [bearer eID] (\() -> spend eID)
```

**Transaction message: Escrow2**

```
body = Escrow2
inputs
  TransactionOutput `cab24f... 0 = Ryan Reich`
```

**Command line:**

```
./postTX.sh Escrow2
```

This transaction creates a contract to hold the escrow rather than calling it.  This contract, in turn, will pass the escrow back to its next caller, then be deleted (leaving the escrow in possession of the caller).  We'll leave it as a voluntary activity to formulate the transaction that does so.

### Bearers of value

In this contract creation, we see for the first time a use of the formerly empty brackets that always appear as the first argument to `newContract` (this use has the same effect for the first argument of `newEscrow`).  In general, this argument is a list of *bearers of value* that will be transferred to the new contract.  Here we see an escrow ID being used as a bearer (and which must be marked as such); other instances will be given later.  In general, a bearer of value is anything that is given its value by being “backed” by some escrows, which must be present in order for that value to exist.  Including such a thing in the list, marked by `bearer`, causes its backing escrows to be transferred to the new contract or escrow, so that the same *program value* (variable) that represents that bearer is also a *Fae value* in the new contract.  In other words, just like an escrow ID can't be called except by the owner of the escrow itself, a bearer of value is invalid if its backing escrows are not owned by the contract that makes use of it.

The upshot is that in the body of the new contract, it is meaningful to use `eID`, as it is used in the code.  This contract owns the escrow and is free to, in its turn, return it, in this case via `spend`.

### Lambda expressions

This example also contains another example of new syntax, the *lambda expression*.  We use it for convenience rather than writing a full declaration and definition of the contract body; it denotes *just* the code and omits the type and function name.  Lambda expressions are introduced with a backslash `\` (which is supposed to resemble the letter lambda), followed by arguments to the function; the symbol `->` marks the beginning of the function body, which looks the same as a function defined in the formal way.

Although this lambda expression does not declare its type as a function, it nonetheless has a specific type that is inferred from the contents of the expression:

* The argument `()` is a *pattern *indicating a particular value of the input type.  We have seen that `()` is a value of the type `()`, so we deduce that the lambda expression takes the type `()` as its argument.
* The body expression `spend eID`* *is familiar from contracts we have written before; it is a contract expression, and therefore the lambda expression denotes a value of some type of the form `Contract argType valType`, where `argType` and `valType` are the argument and value types of the contract.  We have already determined that `argType = ()`, and we can see from the fact that we `spend` an `EscrowID () String` that `valType = EscrowID () String`.

Therefore the lambda expression is a `Contract () (EscrowID () String)`.  Because of the convenience of lambda expressions for “spontaneous” functions, we will be using them often.

## Restricted-access contracts

The example above is highly flawed if the escrow is to be a bearer of value, since there are no access restrictions on the contract in which it is deposited.  Here we show how to rectify this.

**Transaction source file: Escrow3.hs**

```
body :: Transaction (EscrowID () String) ()
body eID = do
 us <- signer "self"
 let
   c :: Contract () (EscrowID () String)
   c _ = do
     them <- signer "self"
     unless (us == them) (error "Wrong sender")
     spend eID
 newContract [bearer eID] c
```

**Transaction message: Escrow3**

```
body = Escrow3
inputs
  TransactionOutput `cab24f... 0 = Ryan Reich`
```

**Command line:**

```
./postTX.sh Escrow3
```

This example is more complex than usual and introduces several new syntax features.  The result is a contract that only returns the escrow ID to the same person (literally person, as in “someone who can send a transaction”) that first deposited it.

### Binding the Fae API

This example has two more instances of the *do block *for creating contract bodies longer than a single line.  This block allows more than simple imperative expressions like `newContract [] c`; it also allows *bindings* of values obtained from the Fae API.  We have seen the monad `FaeTX` for transaction bodies; bindings in transaction bodies can assign any name (like `us`) to any value of some type `FaeTX a` (like `signer "self"`), “extracting” the content of the monadic value at the present time.

### Let declarations

The indented block kicked off by `let` is similar to the blocks we have seen previously that are kicked off by `where`.  The difference is that `let`, which occurs in a `do` block, can make reference to names introduced earlier in the block, whereas `where` can only use the names of the function arguments.  Since we use `us` in the definition of the contract, we must place that definition in a `let` block between the definition of `us` and the use of `c`.

### Signer

The `signer` API function is the interface to an associative array containing the public keys (values of type `PublicKey`) of the various parties that signed the transaction currently being processed when it is bound.  Multiple signatures are possible and this enables some intricate ownership schemes, as well as defending against the misuse of one's contracts.  Therefore, in the code above, the binding `us <- signer "self"` binds `us` to the public key of the signer named `self` of the transaction written in this code; however, the binding `them <- signer "self"` that appears in the contract it creates binds the `self` signer of the transaction that *calls* the contract, since the line is not actually executed until that time.  Note that the contract itself doesn't need to do any cryptography to use these public keys, since they are validated when the transaction is received by Fae.

Fae often uses `self` as the default name for a transaction signer (`faeServer` actually supplies it if it is not present in the message, but this is a demo behavior).  Signers can be declared in the transaction message similarly to inputs:

```
signers
  self = ryan
  other = dave
```

The names on the right indicate named public keys to be used to sign the transaction (again, `faeServer` simulates this and correspondingly manages the keys internally).

### Decisions and exceptions

The line `unless (us == them) (error "Wrong sender")` is quite novel for us, as it contains the first appearance of one of the staples of programming, the conditional.  The `unless` method takes a boolean value and, if it is false, executes the second argument; if it is true, it does nothing.  Here, the boolean is `us == them`, which exploits the fact that `PublicKey` supports the `==` operator; in formal terms, it is an instance of the `Eq` class of types (or *typeclass*).  We will encounter typeclasses more in the future.

The action `error "Wrong sender"` raises an *exception*, which causes the entire transaction to end and its return value to be replaced by this exception.  In addition, none of the transaction's actions are saved: new contracts are not created, though contract calls are still made normally.  An exception in a contract call will roll back just the one call, preventing its update or deletion.  Aside from the exceptional return value, it is as though the transaction (respectively, contract call) did not happen.  Exceptions may be thrown using `error` or `throw` (which takes a different kind of argument) but may not be caught, and so should not be used for control flow.  The appropriate time to throw an exception is when an error occurs that renders the contract meaningless.

Exceptions are raised not only by explicit errors but also by errors of programming such as type mismatches, evaluating an undefined value (for example, `1/0`), calling a contract by an invalid contract ID or with a literal argument that doesn't parse correctly, or using an escrow that you do not own.

The inability to catch exceptions has one exception in that it is possible to attach recovery routines to a contract in case it does exit prematurely.  This feature will be discussed elsewhere.

### Library alternative

This example is instructive but unnecessary, since Fae provides a contract library that includes a method to save a value in a contract owned by a particular public key.  Here is a shorter way of writing the transaction source file above.

```
import Blockchain.Fae.Contracts

body :: Transaction (EscrowID () String) ()
body eID = deposit eID "self"
```

The `import` statement brings the named *module* into scope; all of Fae's modules are in the namespace `Blockchain.Fae`.  This module contains a method `deposit` accepting a value-bearing argument and a name, and creates a contract that releases the value to the caller if that caller is the same as the one referred to by the name *in the original transasction*.  Its implementation is as shown above.

## Restricted-access escrows

This ongoing example still contains a serious issue: the escrow can easily be obtained without calling contract `TransactionOutput `cab24f... 0``.  Indeed, *anyone* is free to write the code in [the contract definition](https://consensys.quip.com/2wGTAw6Fgm87/Tutorial-2-Escrows#ZRIACABrgQl) to create their own escrow that works the same way.  This means that the escrows issued by that contract have no value: the access control placed on the contract that saves them does not actually keep the escrow private.  Here, we at last present the method that makes escrows actually valuable.

### Module source code

```
module Nametag (Nametag, getNametag, checkNametag) where

import Blockchain.Fae

data Private = P String
data Token = T
type Nametag = EscrowID Token Private

getNametag :: String -> FaeTX Nametag
getNametag name = newEscrow [] $ \T -> forever $ release (P $ "Property of: " ++ name)

checkNametag :: Nametag -> String
checkNametag eID = s where P s = useEscrow eID T
```

This code defines not a transaction or contract, but a new module like `Blockchain.Fae.Contracts`.  This module provides the interface to escrows like the ones previously created by contract `TransactionOutput `cab24f... 0``.  

### Module definitions

A module is just a source file that can be imported by others, including transactions.  It *exports* some or all of the names defined in it, which are available to any other source file that imports it, whereas the ones that are not exported, are not available outside the module.  Modules occupy a place very similar to that of classes in languages like `C++`: indeed, you can make an easy analogy between the above module and the definition of a class called `Nametag` having a private type `Private` with a constructor `P`, a public type name `Nametag`, and two public functions `getNametag` and `checkNametag` defining its interface.  The analogy ends there, though, since one does not instantiate the `Nametag` type as a programmatic value `x` that has members such as `x.checkNametag`.

### Module imports

This module imports `Blockchain.Fae`, the main Fae module that contains the Fae API.  This module is provided implicitly to all transactions, but must be imported explicitly by modules (because, conceivably, a module could provide just a general source code library intended to be used in contracts but not actually using them itself).

### New datatypes

The definition `data Private = P String` is new to us; it defines a new type called `Private` whose values follow the *pattern* (in exactly the same sense as patterns in lambda expression arguments discussed previously) `P String`.  Here, `P` is called the *constructor*, and the pattern `P String` can fit any value of the form `P s`, where `s :: String`.  So values of `Private` are the same as `P`-tagged strings; while this may seem redundant, it has the feature that the nature of `Private` as a string is hidden, so none of the string interface can be used on it directly, but only after extracting the internal string from its pattern.

We also define a type `Token` with only one value, the constructor `T` itself.  Its purpose is simply to be present in a pattern.

### New Type names

The definition `type Nametag = EscrowID () Private` is a *type synonym*.  It exists entirely for convenience: any appearance of `Nametag` can be replaced by `EscrowID () Private` without affecting anything.  By defining it, we can both give a meaningful name to this particular type of escrow and avoid exporting `Private`, which is, after all, private.

### The dollar sign operator

The code for `getNametag` has an odd syntax, separating functions from their arguments with a dollar sign `$`.  This useful operator allows us to avoid writing parentheses: it collects its entire right-hand side and provides it as a single value to its left-hand side.  So if `f` is a function, `f $ x = f (x)` for any expression `x`, which is visually cleaner particularly when there are nested parentheses.

### How it works

There are no other new syntactical elements in this file, but its construction exploits the semantics of module exports and patterns in a new way.  Since the type `Private`, and in particular its constructor `P` is not exported, it is impossible for anyone importing this module to write the lambda expression in `getNametag`; therefore, we have achieved the goal of limiting access to the creation of nametag escrows.  Because the absence of `P` makes it impossible to use the escrow, we must provide `checkNametag` as a smart getter that makes the `useEscrow` call for us, using the constructor that it, as a module function, has access to.  So an importer of `Nametag` can request a new nametag and see inside it, but cannot perform these operations at a “low level”; they must use the provided interface.

The `checkNametag` function has another purpose: validation.  Although no one can use the same lambda expression to create a nametag, it is still possible to write contract code like

```
newNametag :: String -> FaeTX Nametag
newNametag name = newEscrow [] (\_ -> undefined)
```

which is always type-correct but can never be evaluated.  Without the constructor, you can't make a concrete value of `Private`, but you can still use that type indirectly if it is not evaluated.  However, pattern matching forces evaluation because it needs to see the constructor, so calling `checkNametag` on the result of `newNametag` will raise an exception for having evaluated `undefined`, whereas on a genuine nametag escrow, it will just return the string inside.  So by “checking” your `Nametag` you can be sure that it is real.  This is also why the nametag escrows have been redefined to `release` rather than `spend` their contents, so that they can be checked repeatedly without being deleted.

The same concern motivates the use of the `Token` type as the escrow argument.  If the escrow allowed anyone to call it, they could obtain a value of type `Private` which, even though the constructor is not visible, could be placed instead of `undefined` in `newNametag`.  Making the argument another private type completely obscures the internals.

In summary, the `Nametag` module provides an opaque escrow ID type `Nametag` and two interface functions `getNametag` and `checkNametag` that, together, are the only way that a valid nametag can be created and used.

## Rewards and payment

Using the module concept, we can lock down access to a particular type of escrow to its public interface.  As written, the `Nametag` interface does not restrict the use of `getNametag`, so that although the value of nametags is protected by the module, it does not really exist at all because nametags are free.  The solution to this is, of course, to charge for them.  The only form of value built in to Fae is the reward, which may be awarded as a participation incentive.  To use it, we just alter `getNametag` slightly.

```
getNametag :: RewardEscrowID -> String -> FaeTX Nametag
getNametag rID name = do
  claimReward rID
  newEscrow [] $ \T -> forever $ release (P $ "Property of: " ++ name)
```

Then, a special *reward transaction* can call this function, but no other kind of transaction.

**Transaction source file: RewardNametag.hs**

```
import Blockchain.Fae.Contracts
import Nametag

body :: Transaction RewardEscrowID ()
body rID = do
  ntID <- getNametag rID
  signOver ntID "self"
```

**Transaction message: RewardNametag**

```
body = RewardNametag
reward = True
signers
  self = ryan
```

**Command line:**

```
./postTX.sh RewardNametag
```

This cashes in the reward to get a nametag, then saves that nametag in a signature-protected contract.

### Rewards and reward transactions

Rewards in Fae have the type `Reward`, which is provided by the module `Blockchain.Fae` (and therefore available implicitly to any transaction).  This is a private type like `Private` for nametags, and its corresponding escrow type is the `RewardEscrowID`.  This kind of escrow is only created in reward transactions, which are designated as such in blocks and may be issued, for instance, to miners (the `reward` field in the message indicates to `faeServer` to simulate this designation; obviously, in reality a transaction should not be able to declare itself a reward recipient).  A reward transaction gets an extra input value, before the the list of explicit contract calls, of type `RewardEscrowID`, which is guaranteed to match an escrow that exists in the transaction.

### Claiming rewards

As we saw for nametags, the private type must be accompanied by a public interface that can validate an escrow of that type.  For rewards, that interface has only one function, `claimReward`, which accepts a valid reward escrow ID and deletes the escrow (that is, a reward escrow `spend`s its value).  Therefore, rewards cannot be reused or accumulated (though they can be saved), so are not really a currency; they are just an entrypoint to self-contained Fae value.

Because rewards are scarce and can be validated, they are one way to meaningfully limit access to the `Nametag` type.  Essentially, the `getNametag` function is a reward-to-nametag conversion engine, though this does not make nametags the same as rewards, because (for example) they cannot be used to pay `getNametag`.

## Summary

This lengthy tutorial gradually developed the machinery of escrows, which are contracts-within-contracts that can be used to represent value in Fae.  We saw:

* Escrows can be created like contracts and called like functions using their escrow ID, but only by their owner.
* Escrows can be transferred to new contracts or escrows, or returned by ID from contract or escrow calls.
* To make an escrow type valuable, it should be encapsulated in a module defining a public interface to a private type.

We also learned about some more programming features:

* Lambda expressions, patterns, binding in do-blocks, decision control methods, and the dollar sign operator.
* The `signer` API method.
* The `Blockchain.Fae.Contracts` module and, in particular, `deposit` for saving things in a new contract.  

