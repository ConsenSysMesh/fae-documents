# Tutorial 1: Transactions and contracts

This first tutorial will introduce the most basic concepts of Fae, namely transactions and contracts.  The sample code will use the conventions of [the postTX utility](https://consensys.quip.com/kN9MAhiNm8dz/Project-information#GTTACAArd2o) and the sample command lines use [the docker images](https://consensys.quip.com/kN9MAhiNm8dz/Project-information#GTTACAfd57C).  After following the instructions there, you should execute this first to have a local server running:

```
./faeServer.sh
```

## Hello, world!

**Transaction source file: HelloWorld.hs**

```
body :: Transaction Void String
body _ = return "Hello, world!"
```

**Transaction message: HelloWorld**

```
body = HelloWorld
```

**Command line**

```
./postTX.sh HelloWorld
```

This transaction does only one thing: declare its “results” to be the string `Hello, world!`.  This result is saved within the Fae storage in association with the transaction's unique identifier (the hash of this source file plus some identity data), which can be examined by anyone running a Fae client by linking an executable to this storage and reading its contents.  `faeServer` and `postTX` together form such a client, and will print an informational message containing this result and other data.

### Transaction type

The transaction must declare its type, which is always `Transaction inputType resultType`.  The input type, here, is `Void`, the type with no values, because the list of inputs is empty; later, we will see how to provide nontrivial input lists.  The result type is `String` because the `return` value is a string.  The transaction body itself is always a function called `body`.

A `Transaction` is actually a function of the form `inputType -> FaeTX resultType`, where `FaeTX` is an environment providing the Fae API for a transaction, none of which is actually used in this transaction.  (Formally, this environment is a *monad*, which is an intimidating word with an intimidating reputation and which is often poorly explained.  A monad is what is described here: an execution environment providing an API.)  The input type being `Void`, we cannot use the actual value of the argument, so it is elided with the wildcard `_`.

## Creating and calling a contract

**Transaction source file: HelloWorld2.hs**

```
body :: Transaction Void ()
body _ = newContract [] c where
  c :: Contract () String 
  c _ = spend "Hello, world!"
```

**Transaction message: HelloWorld2**

```
body = HelloWorld2
```

**Command line**

```
./postTX.sh HelloWorld2
# Output including a line:
#   Transaction cab24f...
# The hex string is the transaction ID
```

This transaction also does only one thing, which is entirely useless in isolation: it creates a new contract that, when called with the empty argument, “spends” the value `Hello, world!`.  This contract is, however, not actually called, nor does the `newContract` function return any value itself.  We must therefore provide a second transaction:

**Transaction source file: CallHelloWorld2.hs**

```
body :: Transaction String String
body = return
```

**Transaction message: CallHelloWorld2**

```
body = CallHelloWorld2
inputs
  TransactionOutput $txID 0 = ()  
```

**Command line**

```
./postTX.sh -e txID=<transaction ID from HelloWorld2> CallHelloWorld2
```

This transaction *calls* the contract denoted by its ID with an argument given as a string literal. The return value of that transaction is then provided to `body` as an input, which it passes through to `return` (pass-through is denoted by assigning the `return` function directly to `body`, not even writing an argument).

### The `()` type

The type `()` appears both in the new contract as an argument and the first transaction as a return value.  This type, unlike `Void`, has only one value, which is also called `()`, and is used to denote a “value” that is actually absent.  Conceptually, the new contract takes no arguments, but it is required to take an argument, so we give it `()`.  Likewise, the `newContract` function returns no value, and therefore the transaction `body` also returns no value, so we call the return type `()`.

### Contract creation

A new contract is created with the first of the two API functions seen here: the `newContract` function, which takes two arguments: a list, whose contents will be described in a later tutorial, and a function whose type is of the form `Contract argType valType` that forms the body of the contract.  When called, the contract takes an argument of type `argType` and “spends” a value of type `valType`.  In the definition of `c`, we elide the (unused, empty) argument with a `_` argument and return a `String`.

### Contract Type

The type `Contract argType valType` is (nearly) a synonym for the function type `argType -> Fae argType valType valType`, where the three-part type `Fae argType valType` is an environment (monad) for a contract taking an `argType` and returning a `valType`.  It differs from `FaeTX` in two ways: first, by specifying the argument and return types; and second, by possessing a slightly larger set of API functions that we will describe in part here and in part in a later tutorial.  The function type therefore denotes a function taking an `argType` and returning a `valType` in the `Fae argType valType` monad; although the parameters to `Fae` seem redundant here, the `Fae` monad can contain any type, not just `valType`, and not all API functions deal with the `argType`, while some of them need to know the actual argument and return types declared for the contract, so the parameters are present.

### Spending values

The other API function we see here is `spend`, which is much like `return`, and in fact in this context is no different, but in general deals with value-bearing return types appropriately.  We will see what this means in a later tutorial; suffice it to say, here, that a contract *must* end with `spend`.  This API function is one of two that are only available in `Fae` but not in `FaeTX`; it can only be used by contract code and not in a transaction.  After a contract spends a value, it is deleted from Fae storage and cannot be called again.

### Input list

The second transaction employs an *input list* to call the contract created in the first one.  Contract calls are denoted by a *contract ID* and an *argument*.  The contract ID in this case is `TransactionOutput $txID 0`, meaning the first (indexed-0) contract created by the transaction with ID `$txID`.  Other forms of the contract ID are possible and will be introduced at the appropriate times.  The `newContract` function does *not* return the contract ID because it is impossible to use it meaningfully within Fae code; the only way a contract ID can be used is in the list of transaction inputs, which is part of the transaction message, not the source code.

### Input arguments

The input contract is called with a string-literal argument `()`.  All arguments to input contracts are string literals but are parsed behind the scenes into the expected argument type for the contract as it was created.  This scheme is the first instance of Fae's contract security guarantees: it ensures that the caller of a contract cannot provide it with a computationally expensive argument, thus performing a denial-of-service attack on that contract.  The parser is declared along with the contract's argument type (the type `()` has its parser declared in the language definition) and, therefore, is completely under the control of the contract's *author*, who can design it to handle input strings in a safe way or at least know to expect the possible negative consequences of parsing.

## Repeatable contracts

**Transaction source file: HelloWorld3.hs**

```
inputs = []

body :: Transaction Void ()
body _ = newContract [] c where
  c :: Contract () String
  c _ = do
    release "Hello, world!"
    spend "Goodbye!"
```

**Transaction message: HelloWorld3**

```
body = HelloWorld3
```

**Command line**

```
./postTX.sh HelloWorld3
```

To call this contract, the following transaction may be run repeatedly.

**Transaction source file: CallHelloWorld3.sh**

```
body :: Transaction String String
body = return
```

**Transaction message: CallHelloWorld3**

```
body = CallHelloWorld3
inputs
  TransactionOutput $txID 0 = ()
```

**Command line:**

```
./postTX.sh -e txID=<transaction ID from HelloWorld3> CallHelloWorld3
```

The first transaction creates another new contract that does a little more than before; the second transaction calls it.

### Extended contract bodies

The contract here contains more than one line and, therefore, employs a syntactical construct called the *do block*.  A do block is simply an expression in an environment, here the `Fae () String` monad; each line is a separate action, which are all executed sequentially.

### Releasing values

In this contract we encounter the `release` API function, which is similar to `spend` but with two important differences: first, it does not *terminate* the contract but merely *suspends* it, i.e. ends the current contract call but allows it to be called again; and second, it returns the argument of the subsequent call.  Here, the argument type is `()`, which can be ignored in a do block statement, so we do not reference it.  This API function is the other one that is available in `Fae` but not in `FaeTX`.

### Calling a contract repeatedly

This contract is called twice.  The first time, it will return `Hello, world!`, which is therefore the result of Transaction 2.  After this call, the contract is updated in storage to begin at the second line, so on the second call, it returns `Goodbye!`, the result of Transaction 3.  After *this* call, the contract is “spent” and removed from Fae storage, so a third call will fail with an error that the contract does not exist.

## Endless contracts

**Transaction source code: HelloWorld4.hs**

```
body :: Transaction Void ()
body _ = newContract [] c where
  c :: Contract () String
  c _ = forever (release "Hello, world!")
```

**Transaction message: HelloWorld4**

```
body = HelloWorld4
```

**Command line:**

```
./postTX.sh HelloWorld4
```

To call this new contract, run the following transaction repeatedly:

**Transaction source code: CallHelloWorld4.hs**

```
body :: Transaction String String
body = return
```

**Transaction message: CallHelloWorld4**

```
body = CallHelloWorld4
inputs
  TransactionOutput $txID 0 = ()
```

**Command line:**

```
./postTX.sh -e txID=<transaction ID from HelloWorld4> CallHelloWorld4
```

### Running forever

The `forever` function is an infinite loop that performs its argument on each iteration.  By placing a `release` inside it, we construct a contract that can be called indefinitely, each time returning the same string `Hello, world!`.  This contract need not “end with `spend`" because it never ends.

## Summary

This first tutorial demonstrated the basic features of Fae:

* Transactions: input and body
* Contracts: creation, spending, suspending, and calling as inputs
* Do blocks and the two Fae monads `FaeTX` and `Fae`.

Although this set of features is enough to employ the *language* of smart contracts, it is not enough to write contracts that have serious content.  In the next tutorial, we will explore the mechanism for denoting and handling value in contracts, which allows authors to create and use such essentials as currencies, identity tokens, and abstract deals.
