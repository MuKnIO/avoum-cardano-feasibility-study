# AVOUM for Cardano

This document describes the design of the Account-View-on-UTXO Model
(AVOUM), a technique whereby users may interact with DApps using the
convenient Account/Balance model made popular by Ethereum on top of a
smart-contract-capable Blockchain that uses the more robust but more
rigid UTXO model, like Cardano, Nervos, Bitcoin Cash.

We first supply context and discuss AVOUM in general terms,
then propose a more detailed approach to applying this to the
Cardano blockchain in particular. In the latter portion, we also
outline how a representative example open contract could work in
this model.

## The Problem

### An Auction Gone Wrong

Let’s imagine that Alice wants to sell a painting by her famous deceased
father in an online auction. She chooses to do it through a contract on
a blockchain that uses the UTXO model. The painting is worth about one
million dollars, which is what the highest bidder, Bob, is ready to
offer. Bob signs a transaction to purchase the painting, and waits… but
instead the painting ends up being awarded to Mallory, who only paid
Alice a few cents for it! What happened?

To interact with the auction contract, Bob’s transaction had to refer to
its current state, embodied in a UTXO. But what if some other user also
posted a bid, that appeared before Bob’s? The bid might be less than
Bob’s, but because it appeared before Bob’s, it interacted with the
contract, consuming its previous UTXO and generating a new UTXO that
accounts for that new bid. Bob’s transaction, referring to the old UTXO,
is therefore invalid, and Bob has to sign a new transaction and wait for
it to be either confirmed or rejected. But what if yet another bidder
managed to get their bid in before Bob’s? Then Bob must try again, in an
endlessly repeated cycle, until the end of the auction arrives and his
bid never made it through.

What did Mallory do? She sent to the blockchain nodes a stream of
transactions making bids for one cent, two cents, three cents, etc.,
such that the UTXO constantly changes, and changes faster than Bob or
any other honest but technically unsophisticated bidder can react.
Regular users wait for a couple of blocks each time to determine what
the UTXO to sign a transaction against will be, but Mallory makes
multiple transactions at every block, and even maintains multiple rival
states of the contract so the odds that honest bidders write their
transactions against a valid UTXO are very low. Mallory also watches how
much Bob puts down in transaction fees, and makes sure to always leave a
larger tip to the miners, so she’ll be sure that they prefer her
transaction to Bob’s. In the end, Mallory racks up a very large bill in
transaction fees: thousands of dollars worth of miner tips, maybe much
more, along the duration of the auction. But in the end, she manages to
keep out all the other bidders, and win the auction with a ridiculously
low bid. For a few thousand dollars in fees, she wins an asset worth a
million dollars.

Mallory can similarly run a *Economic Denial-of-Service Attack* (EDoS)
to exclude rivals from any contract where she could benefit from this
exclusion.

# Introduction

The two main issues we solve are:

- “Open” contracts with an unlimited number of participants or
  transactions can be subject to economic DoS attacks whereby
sophisticated attackers can modify the contract’s UTXO faster than the
victims can react, thus blocking them from interacting with it.  By
making these transactions suitably malleable, intermediaries (at
equilibrium, miners) can compete to get the transactions accepted by the
blockchain in exchange for a fee.
- Writing UTXO unlocking scripts wherein transactions are suitably
  malleable enables users to interact with contracts as if the
blockchain were using an Account/Balance model; but applying the design
pattern by hand requires a great discipline and the result may or may
not be recognized by the transaction posting intermediaries (miners).
Our solution is to automate this discipline away using a suitable Nervos
library.

Our solution will enable the safe deployment of “open” contracts on the
Cardano blockchain, which is not currently possible, and will bring its
smart contract capabilities in parity with Ethereum.

-----

This document describes the results of a feasibility study for
implementing AVOUM on Cardano.

* [Background](#background)
* [Summary](#Summary)
* [Protocol and Interface Modifications](#Protocol-and-Interface-Modifications)
  * [Malleable Transaction Submission API](#Malleable-Transaction-Submission-API)
  * [Rebase Script Execution Environment](#Rebase-Script-Execution-Environment)
* [Proof of Concept](#Proof-of-Concept)

<a name="background"></a>
## Background

Cardano is a UTXO blockchain, like Bitcoin but unlike the
account/balance model used by Ethereum. This has advantages, such as
parallel verification of transactions, but it means that contracts have
no built-in notion of a stable identity, and transactions are only valid
if they operate on the current state of a contract. In contrast, with
the account/balance model, transactions cannot be verified in parallel,
but a transaction can be applied to a contract even if there have been
intervening transactions since it was constructed.

This causes problems if there is contention on a contract, as the author of a
transaction must continually monitor the blockchain and retry submitting the
transaction after each intervening change. This can result in denial of service
vulnerabilities if the participants lack a fair amount of technical
sophistication, and this has shown itself to be a problem in practice.

The Account-View-On-UTXO-Model (AVOUM) approach solves this by providing a way
to support "malleable" transactions, which can be submitted to a node, who is
then able to update the transaction in order to solve conflicts with intervening
transactions, rather than having to reject the original transaction outright,
thus easing the burden on participants to deal with contention, while still
allowing the benefits of the UTXO model.

For more background information see the [AVOUM for
Nervos](https://j.mp/AVOUMforNervos) document, which outlines the same
problem in the context of the Nervos blockchain, and the corresponding
[AVOUM for Nervos Feasibility
Study](https://gitlab.com/mukn/nervos-avoum-feasibility-study/-/blob/main/study.md)

<a name="summary"></a>
## Summary

To summarize the study's findings:

- No modifications are needed to the transaction format or to the
  validation process to support malleable transactions.
- To support "account view" semantics, we will need a convention
  for assigning a persistent identifier to "the same" cell across
  transactions; we describe a scheme to use for this purpose. We call
  these persistent identifiers "account ids," and the chains of cells
  that they identify are "accounts."
- To support malleable transactions, we will need to supply the Cardano
  node  with an additional script, beyond the usual validation scripts,
  which we call the "rebase" script, by analogy to the rebase operation in
  version control systems such as Git.
  - Like validation & minting policy scripts, the rebase script is a
    plutus function. Compared to the existing scripts, the rebase script:
    - Is able to read data from both the original transaction with which
      the rebase script was submitted, and the last transaction to
      affect each of the accounts that were inputs to the original
      transaction. This effectively allows the rebase script to view the
      "latest state" of these accounts.
    - Returns a new, "rebased" transaction, which is semantically equivalent
      to the original transaction but updated to apply cleanly to the current
      state. This is in contrast with the existing scripts, which either
      return unit on success, or fail.
  - Malleable transactions are submitted to the node with two
    new pieces of information:
    - The rebase script to use to rebase the transaction if should
      other transactions consume its inputs before it is applied to
      the blockchain.
    - A list of transaction inputs that should be treated as accounts.
    We define a new API endpoint for submitting malleable transactions;
    non-malleable transactions can continue to use the existing endpoint.
  - Importantly, the rebase script itself does *not* appear in a
    transaction; it only needs to be recognized and used by rebasing
    nodes. Validation depends only on the validation scripts, as
    is currently the case.
- In general, inputs to malleable transactions will require specialized
  validation scripts, as common validation scripts disallow malleability
  by design (e.g. by verifying a signature). For regular users with regular
  validation scripts to use malleable transactions thus requires an extra
  step: first, cells compatible with malleability are created in a regular
  non-malleable transaction. Only then can these cells be assembled into
  malleable transactions.

<a name="Protocol-and-Interface-Modifications"></a>
## Protocol and Interface Modifications

This section describes necessary modifications to Cardano protocols and
interfaces.

<a name="Malleable-Transaction-Submission-API"></a>
## Malleable Transaction Submission API

We propose adding a new API, along the lines of the existing submission
API; to submit a malleable transaction, rather than posting the transaction
data to `/api/submit/tx`, a user should post a CBOR encoded value of the
following datatype to `/api/submit/malleable-tx`:

```haskell
data MalleableTx = MalleableTx
  { rebaseScript :: ByteString
    -- ^ The compiled plutus rebase script
  , accountIndicies :: [Int]
    -- ^ The indices of transaction inputs that should be treated
    -- as accounts.
  , transaction :: ByteString
    -- ^ The encoded original transaction
  }
```

<a name="Rebase-Script-Execution-Environment"></a>
### Rebase Script Execution Environment

The rebase script is run when a malleable transaction specifies input
cells which:

- Have been consumed by another transaction, and
- Were marked as an account when the transaction was submitted to the
  miners.

We call the transaction with stale inputs the "original transaction,"
and the committed transactions which consumed those inputs
"intervening transactions."

We describe the rebase script's interface as a Haskell function; we assume
arguments and return values have a suitable serialization as `BuiltinData`.
First, we define data types used for identifying an account:

```haskell

-- | An account id. This consists of a currency symbol and an arbitrary
-- ByteString, which the minting policy for the currency must ensure
-- is unique for all accounts which hold that currency (perhaps with the
-- help of trusted validator scripts on issued cells).
--
-- Rebasing miners must ensure that any cells they treat as accounts hold
-- a non-zero balance of the specified currency. Rebasing miners may
-- assume that this uniquely identifies an account for indexing
-- purposes; if the currency's minting policy fails to ensure this
-- the node will not suffer negative consequences, but cells whose ids
-- collide may spurriously replace one another in the index, possibly
-- causing rebase scripts to fail.
newtype AvoumId = AvoumId
    { aidSymbol :: CurrencySymbol
    , aidUnique :: ByteString
    }

-- | The state of a cell that represents an account. The @acId@ field is
-- understood by rebasing miners and used for indexing purposes. The @acState@
-- field is available for use by the contract itself.
data AvoumCell a = AvoumCell
  { acId :: AvoumId
  , acState :: a
  }

```

A rebase script is logically a function of type:


```haskell
rebase :: ScriptContext -> Map AvoumId ScriptContext -> ByteString
```

...where arguments are:

- The `ScriptContext` passed to the original transaction's validator.
- A map from account ids in the original transaction to the `ScriptContext`
  for the last intervening transaction to modify the corresponding account.

...and the return value is the raw bytes of the new transaction (TODO: should
we expose this as `BuiltinData` instead? It's not like we need to preserve
signatures).

<a name="Proof-of-Concept"></a>
## Proof of Concept

To illustrate how malleable transactions will work, this section
sketches a proof of concept implementation of a simple English first
price auction with open bids.

The example prioritizes simplicity and clarity over flexibility; some patterns
(such as hard-coded indices) are likely inappropriate for a real-world
implementation, but are sufficient to illustrate the design.

We include Haskell-inspired pseudocode for a validator, minting policy
and rebase script for the auction, as well as validation scripts
suitable for bids & assets. The auction uses a minting policy to ensure
that cells in invalid states cannot be constructed.

When it comes time to first implement this proof of concept,
as an incremental step we may hard-code the rebase logic for this particular
contract in the node, and introduce proper support for rebase scripts
as a subsequent task.

<a name="Transaction-Format"></a>
### Transaction Format


```haskell

```

# TODO

A (not necessarily exhaustive) list of some things that still need to
be done to this document:

- Finish the proof of concept section.
- Go through and make sure we're using proper terminology; some of the
  language is borrowed from a similar study we did for Nervos, so we
  should make sure we're not using terminology that doesn't apply to
  Cardano (e.g, what does Cardano call cells?).
- Sanity check some details:
  - We talk of validation scripts on e.g. escrowed cells, but we may not
    need to use separate ones; we might be able to just get away with
    assigning them to the same credential as the auction itself.
  - Some of the places we use `ScriptContext` in the interfaces for
    rebase scripts probably need to use a transaction instead (TxInfo?).
    ScriptContext includes a purpose field, which iiuc is different for
    each script in run as part of the transaction.
