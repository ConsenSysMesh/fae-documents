# Project information

## Fae overview and goals

Fae is a smart contract system with a functional angle.  It has several goals:

* It aims to be scalable (parallelizable) to the maximum extent possible given the necessity of executing complete transaction histories.  
* It is minimal in its features, containing no integrated currency, no preferred contract state structure, no domain-specific contract language, no inherent limits on computational resource usage, no system-controlled accounts, no integrated or preferred consensus mechanism, and only one kind of transaction, which is syntactically identical to contracts.
* Adding transactions from new blocks should be very fast, requiring only a constant small amount of computing per transaction.
* Contracts should be pure functions of their inputs, their behavior entirely predictable from their source code, and therefore safe from sabotage by untrusted code.

Additional features are provided by the default choice of Haskell as the contract implementation language:

* Formal verification assistance from Haskell's famously strong and expressive type system, approaching the goal of “if it compiles, then it works”.
* Complete flexibility in contract state management via Haskell's library of monad transformers.
* Fluently declarative contract logic is simplified by Haskell's support for custom control structures.

## Project structure

Fae is currently a work in progress, with a basic implementation in Haskell of the smart contract system completed but no networking integration or blockchain management yet written.  The Github repository is available at <https://github.com/ConsenSys/Fae> and updates are pushed to the conversation thread in this document.  Documentation of the Haskell API is at <https://consensys.github.io/Fae/>.  In this Drive, the Tutorial directory, still in progress, contains a walkthrough of Fae's features and usage.

## Presentations

I've talked about Fae in various venues:

* [Fae seminar: “Faeth” integration with Ethereum (Apr 13, 2018)](https://consensys.zoom.us/recording/share/IuzENhfTosd65OZkM8UpTUMg9D1oD7mP-bz_xDAKW5-wIumekTziMw)
* [Fae seminar #4 (Mar 2, 2018)](https://consensys.zoom.us/recording/play/CuTZvoUt-RwfgzMLYQIxnK7UMqIYTk2fRzD27bv8F1exFqf-fB7VWUPaH-yxRs8V)
* [Discussion with David Roon (OpenLaw) (Mar 1, 2018)](https://consensys.zoom.us/recording/play/XvmXLylw5IzOCCV8mJ_Zxw5EF3YnpNaXiDsIh4rt2vNyRdBgy0_E3TM0ecQo65r8)
* [Fae seminar #3 (Feb 23, 2018)](https://consensys.zoom.us/recording/play/FryJ7Hp6YHgALi3wYr6Oy1yVvzMDOQRZ_sk7OdSo_bMaHy-IMSt083SandwIyMj2)
* [Fae seminar #1 (Jan 26, 2018)](https://zoom.us/recording/play/oieO1CpV0bWQkqsdr8pN0ZOOyHL_QNIWw_7vmN13G1GuMydfyeKc_ysQ0pPMkkad)
* [PegaSys Lunch&Learn (Jan 25, 2018)](https://consensys.zoom.us/recording/play/moOuj2xqvK4aeTVkRflqdGq1-Oum5k0MpzIrggE-QNpaJT_C4W1KPg6tFXRv84JK)
* [Fae, functional object-oriented contracts (Dec 22, 2017)](https://consensys.zoom.us/recording/play/ipq-kHCVgPmqSQiElgP8fxarRHzviN0ojCLPYkntWRucwDpYabSgxmAsIGX01Uw6)
* [Functional smart contracts (Oct 11, 2017)](https://zoom.us/recording/play/LojPnRwdTLyv4PmdDfCWT-1MpJjijYyCCX7K5ihFLmcO9wFMawlVAP2t3dcPP2lx)

## Compilation and execution

There are two ways to get Fae on your computer: compiling from source, or using the Docker images.

### Compiling from Source

Fae must be compiled from the source found in the Github repository in a properly installed Haskell system, because the server uses the compiler's own API and the system libraries to power the Haskell interpreter.  The only environment I have tested this in is Ubuntu Linux 17.10, using [stack v1.6.3](https://docs.haskellstack.org/en/stable/README/)for Haskell, in which installation is simply

```
stack build
```

inside the Fae project directory.  This builds the library and two executables, `faeServer` and `postTX`, which respectively run Fae as a simple HTTP server accepting individual transactions in a certain format, and deliver transactions in that format from a configuration file.

### Docker images

As of Fae v0.9.9.9, I have built Docker images providing `faeServer` and `postTX` respectively.  They are currently tagged (with those names as tags) in a private Docker repository `teamfae/faeserver` and `teamfae/posttx` respectively on `hub.docker.com`; please contact me if you wish to obtain access.  Once you have logged in (with `docker login`), run `docker pull teamfae/faeserver; docker pull teamfae/posttx` to get the images.

The exact operation of the images is a bit picky, so they contain scripts to automate it.  Get them like so:

```
docker create --name faeServer teamfae/faeserver
docker cp faeServer:/etc/faeServer.sh .
docker rm faeServer
```

and likewise for `postTX`.  I am not providing any other form of shell scripting language right now, as I have no way of testing it and no familiarity with the syntax.

### Operation of FaeServer

If compiling from source, run `stack exec faeServer` in a subdirectory of the source repository.  The server generates a directory tree `Blockchain/` and any number of files corresponding to keys, so it is best to create a dedicated home for it.

The Docker image is run simply as `faeServer.sh` (the actual `docker` invocation can be inferred from the contents of the script).  Nothing is created on the local machine.

Either way, the server binds port 27182 and listens for incoming transactions via http.  Any uncaught exceptions while running a transaction will be dumped to the terminal.  There is a small delay when it processes the first transaction; this is the initialization of the interpreter, and only happens once: subsequent transactions should be near-instantaneous.

### Operation of `PostTX`

The `postTX` utility submits single transactions to `faeServer` over http.  Each transaction has a *config file*, a *body source file*, and (optionally) a number of *other source files*. Run it as one of:

```
stack exec postTX -- <config file name> [<host>] [--fake]
```

```
postTX.sh <config file name> [<host>] [--fake]
```

The `--fake` flag optionally indicates that the transaction is *run* but not *committed*; this allows one to see its effect, most importantly the version IDs produced by various input contract calls, so that they can be entered in the config file.  The `host` argument, which is optional, overrides the default of `localhost:27182`, which is also the default for a local `faeServer`.

The nature of the source files is described in the tutorials and will not be covered here.  The config file is a simple yaml-like specification describing the elements of a transaction.  It accepts fields with the following forms:

* `body = <body module name>`: This *required* field names the Haskell module containing the `body` function for the transaction; i.e. if the body source file is `Body.hs` then the module name is `Body`.  The source file is looked up in the current directory.
* `reward = (True|False)`: This *optional* field indicates whether the transaction is to receive a reward token.  In normal blockchain operation, this would be determined by the block creator, not the transaction author; it is present in `postTX` for testing purposes.  The default is `False`.
* `others`: an *optional* section delimiter initiating a list of additional module names to be included in the transaction.  These modules will be available unqualified from within the `body` source file or each other, and from any source file in a later transaction under the namespace `Blockchain.Fae.Transactions.TX<this transaction ID>`.  The format of the list is as in yaml: each line is initiated by the exact string `  -` (two spaces and a dash), followed by the module name.
* `inputs`: an *optional* (but usually included) section delimiter initiating a list of contract calls to be used as inputs to the transaction.  A contract call is present on a line beginning with exactly two spaces and has the form `<contract ID> = <argument>`, where the contract ID is as described in the tutorials and the argument is an unquoted literal value of the appropriate type.  Both the contract ID and the argument may contain (delimited by spaces) environment variable references of the form `$envvar`.
* `keys`: an *optional* section delimiter initiating a list of signer names and their corresponding keys.  In this testing scenario, the keys are managed by `faeServer` internally and available by textual names, say `key1`, `seller`, and so on.  If this list is empty, it defaults to `self = key1`.  The format of the list is as for `inputs`, with each item of the form `<signer name> = <key name>`, and each side may contain environment variable references.

Note that in the above, we allow environment variables in several of the specs.  This works slightly differently with `stack exec postTX` versus `postTX.sh` (i.e. `docker run postTX`), because of how Docker passes environment variables to containers:

```
var1=val1 ... varN=valN stack exec postTX [args]
./postTX.sh -e var1=val2 ... -e varN=valN [args]
```

Note that there can't be spaces around the equals signs in either form.

Here is a sample transaction config file, say called `MyTX`:

```
body = MyTXBody
reward = True
others
  - TXModule1
  - TXModule2  
inputs
  TransactionOutput $tx1 1 :# 4 = 1234
  TransactionOutput $tx2 0 = Just $tx1Version
  InputOutput $id 1 = "Hello!"
keys
  self = ryan
  buyer = alice
  seller = bob
```

It would be run, for example:

```
id=18df3f tx1=222333 tx2=deadbeef tx1Version=fffee0 stack exec postTX MyTX
./postTX.sh -e id=18df3f -e tx1=222333 -e tx2=deadbeef -e tx1Version=fffee0 MyTX
```


