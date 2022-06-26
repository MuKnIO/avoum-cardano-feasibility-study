# AVOUM for Cardano Feasibility Study

This document describes the results of a feasibility study for
implementing AVOUM on Cardano.

* [Background](#background)
* [Summary](#Summary)
* [Protocol and Interface Modifications](#Protocol-and-Interface-Modifications)
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
to support "malleable" transactions, which can be submitted to a miner, who is
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
    - In addition to being able to read data from the original
      transaction with which the rebase script was submitted, the
      script may also read any applied transactions which consumed the
      inputs expected by the original transaction.
    - The script's return value is a new, "rebased" transaction, which
      is semantically equivalent to the original transaction but updated
      to apply cleanly to the current state.
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

<a name="Rebase-Script-Execution-Environment"></a>
### Rebase Script Execution Environment

<a name="Proof-of-Concept"></a>
## Proof of Concept

To illustrate how malleable transactions will work, this section
sketches a proof of concept implementation of a simple English first
price auction with open bids.

The example prioritizes simplicity and clarity over flexibility; some patterns
(such as hard-coded indices) are likely inappropriate for a real-world
implementation, but are sufficient to illustrate the design.

We include Haskell-inspired pseudocode for both a validation and rebase
script for the auction, as well as validation scripts suitable for bids
& assets. When it comes time to first implement this proof of concept,
as an incremental step we may hard-code the rebase logic for this particular
contract in the node, and introduce proper support for rebase scripts
as a subsequent task.

# TODO

A (not necessarily exhaustive) list of some things that still need to
be done to this document:

- Go through and make sure we're using proper terminology; some of the
  language is borrowed from a similar study we did for Nervos, so we
  should make sure we're not using terminology that doesn't apply to
  Cardano (e.g, what does Cardano call cells?).
- Sanity check some details:
  - We talk of validation scripts on e.g. escrowed cells, but we may not
    need to use separate ones; we might be able to just get away with
    assigning them to the same credential as the auction itself.
