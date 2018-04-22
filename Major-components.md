# Major components to create

* A means to input transactions conveniently.  Currently they can only be executed by invoking some internal functions and providing the transaction ID and public key separately — fine for a mock-up but not so good for actual use.
* An interface for accessing created contract IDs in a semantically meaningful way.  This is absolutely necessary for writing transactions, which need to call contracts by ID.  Ideally, this system would link with the transaction input mechanism.
* An interface for accessing transaction outputs from outside.  Otherwise, no data can ever get off the chain!
* An API for contracts to expose their internal state, at their discretion, to be read from the outside (from the inside is impossible given laziness).  This is also important for getting information off the chain, though ideally, transactions should report the important information in their results.
* A supervisor to execute transactions sequentially but also to jump back to execute transactions that branch (in a separate thread, of course).
* A transaction message format and a client for exchanging messages over the network.
* A block format and a client for exchanging blocks independently of transactions.

