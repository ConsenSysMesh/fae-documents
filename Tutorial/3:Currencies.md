# Tutorial 3: Currencies

The first two tutorials introduced all the elements of Fae contracts and transactions; this one will focus on a significant economic application, currencies.  With hindsight, it is clear that the elements discussed in the first two tutorials do not include an integrated currency, and this is because Fae's transaction model does not require charging for computational complexity, and therefore does not require a coin to charge.  With the exception of the reward token from the end of the second tutorial, it in fact does not provide any valuable medium of exchange, and the reward token is intentionally limited in its usefulness to encourage participants in Fae to build their own economies and operate them on equal footing.

## Basic currency operations

A currency is a valuable numeric quantity that has a certain minimal set of operations.  Fae understands those operations to be:

* `zero`: a currency token with value zero.  This is actually a token *factory* in that it produces a new zero token each time.
* `value`: a function on currency tokens returning their numeric value.  This value should retain the type information of *which* currency it denotes.
* `add`: spends two currency tokens and creates a new one whose value is the sum of the originals.
* `change`: attempts to “make change” for a currency token into a smaller amount and a remainder.  The attempt fails if the amount is not actually smaller than the value of the token.

Each of these operations is *intended* to work on some escrow-backed value.  The module provides a toy currency called `Coin` whose tokens are defined to be `EscrowID Token CoinValue`, where both `Token` and `CoinValue` are opaque private types (as discussed in the second tutorial) restricting access to the escrow and the value it contains.

This list is notable for the operations it does *not* contain:

* Subtraction.  With unrestricted subtraction, tokens of negative value can be produced and then subtracted from zero to create value from nothing.  The `change` function is the restricted version of subtraction that, explicitly, fails if this is attempted.
* Multiplication by a number.  This has obvious potential for abuse; although it makes sense to talk about multiples of a currency *value* it does not make sense to actually scale a token of that value.
* Division.  Nothing is wrong with this in principle, but it can be reduced to `change` and division of values, so it is not part of the interface.
* Token creation.  Of course, every currency needs a mechanism to create its tokens.  This mechanism *must not* be available unrestricted to general users in order to protect the integrity of the currency. Restricted creation functions are potentially safe, but also most likely currency-specific, so are not part of the interface.

However the actual currency type is defined, we can use this interface to formulate some common functions dealing with currency.  The following sections are different potential scenarios exploited to demonstrate the currency interface and further code constructs.

## **Accepting payment**

*Scenario:* A merchant selling an item for money. 

We will begin with a very specific implementation and then make it generic.  The item for sale will be [the `Nametag`](https://consensys.quip.com/2wGTAw6Fgm87/Tutorial-2-Escrows#ZRIACAcNynv) from tutorial 2 and the implementation will be simply a new function `buyNametag` in the `Nametag` module, as follows:

```
buyNametag :: Coin -> String -> FaeTX (Nametag, Maybe Coin)
buyNametag coin name = do
  maybeChange <- change coin 5
  case maybeChange of
    Nothing -> error "A nametag costs 5 Coins"
    Just (payment, maybeLeftover) -> do
      deposit payment "Ryan Reich"
      tag <- newEscrow [] $ \T -> forever $ release (P name)
      return (tag, maybeLeftover)
```

Many new concepts and details are present in this code, which is also the most complex function (by far) that has yet appeared in the tutorials.

### Maybe and Tuple types

The first unfamiliar expression is the return value, `(Nametag, Maybe Coin)`.  This contains two new types:

* The *tuple*, or more specifically the *pair*.
* The `Maybe` type

The pair is exactly what it seems: a single value (written like coordinates for a point in plane geometry) consisting of two other values of simpler types.  Here, we have the pair of a `Nametag` and a `Maybe Coin`, whatever that is.  Glancing at the `return` statement, we see that a value of this pair type is a similar-looking coordinate pair whose first element is a `Nametag` and whose second is a `Maybe Coin` (or at least, seems to be, given the descriptive name).

More interesting is `Maybe`.  Like the pair, we have a `Maybe` for all possible base types, all of which are different `Maybe` types.  A `Maybe Coin` represents either a `Coin` that is present or a sentinel value indicating its absence.  As for the types `Private` and `Token` from the `Nametag` module, `Maybe` has constructors, analogous to their constructors `P` and `T`.  These constructors are present in pattern matches, such as the one that appears in the third and fourth lines of the `do` block.  They are `Nothing`, the sentinel value, and `Just`, which is a tag attached to (in this case) the `Coin` that the `Maybe Coin` may be.

### The change function

This is the sole appearance of the `Currency` module's methods in this function.  The `change` function, as described above, “makes change”, so in this example, tries to extract 5 `Coin`s from the sumitted `coin`.   Its type signature expresses the result of such an attempt:

```
change :: Coin -> Coin -> FaeTX (Maybe (Coin, Maybe Coin))
```

The fact that the return value is in the monad `FaeTX `means that `change` both invokes internal methods of Fae, and is permissible in the body of any transaction.  In fact, the real type signature is more permissive and allows it to be used in the body of any contract as well.

The kind of `FaeTX` value that is returned is a `Maybe`: either it is possible to make change (the `Just` branch), or it is not possible (the `Nothing` branch).  If it is possible, the contents of the change themselves have two variants: just the value requested if there is no change, or both the value and the remainder if there is.  This pair of `Maybe`s covers all possibilities: either the total coin value is less than, equal to, or greater than the amount required.

### The Case statement

We see for the first time the expression `case ... of ...`, which is used to branch on the presence of various constructors.  It is, in other words, an inline pattern-match, like the ones that are part of function arguments, and the cases have the same form.  In this statement, the outer `Maybe` of the return value of `change coin 5` is examined.  The first case, as discussed above, is that it is `Nothing`, in which case an error message is thrown.  The second case is that change is made, with a possible remainder; this corresponds to the value of `coin` being at least 5.

### The Return statement

The rest of the code operates in a familiar way: `deposit` leaves the desired payment (which is always 5 `Coin`s) with Ryan Reich, the seller in this scenario.  A new `Nametag` is created and returned with the possible remainder of the payment submitted.  Unlike in a full contract, this code does not end with `spend` or `release`, and can therefore be embedded as a modular part of some larger contract (if only one that does nothing but call it and then `spend` the results).  The `return` statement is the generic way of introducing a value in a monad such as `FaeTX` but does not handle escrows properly, which is why the actual transfer of control from a contract to its caller needs to use one of the two API functions.

### Generalization to other currencies

It is clear that this function can work the same with any type that implements `change`, so rather than specializing it to `Coin`
