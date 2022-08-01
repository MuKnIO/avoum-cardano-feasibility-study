# AVOUM for Cardano

This document describes the design of Account-View-on-UTXO-Model
(AVOUM), a technique whereby users may interact with DApps using the
convenient Account/Balance model made popular by Ethereum on top of a
smart-contract-capable Blockchain that uses the more robust but more
rigid UTXO model, like Cardano, Nervos, Bitcoin Cash.

We first supply context and discuss AVOUM in general terms,
then propose a more detailed approach to applying this to the
Cardano blockchain in particular. In the latter portion, we also
outline how a representative example open contract could work in
this model.

* [The Problem](#problem)
  * [An Auction Gone Wrong](#An-Auction-Gone-Wrong)
  * [A Market for Open-Contract Transactions is Inevitable](#A-Market-for-Open-Contract-Transactions-is-Inevitable)
* [Background Concepts](#Background-Concepts)
  * [UTXO Model vs Account/Balance Model](#UTXO-Model-vs-Account/Balance-Model)
  * [FOMO3D and the Dark Forest](#FOMO3D-and-the-Dark-Forest)
  * [The Implicit Auction for Blockchain Space](#The-Implicit-Auction-for-Blockchain-Space)
  * [Miner Extractable Value](#Miner-Extractable-Value)
* [Solution Overview](#Solution-Overview)
* [Feasibility Study](#Feasibility-Study)
  * [Summary](#Summary)
  * [Protocol and Interface Modifications](#Protocol-and-Interface-Modifications)
    * [Malleable Transaction Submission API](#Malleable-Transaction-Submission-API)
    * [Rebase Script Execution Environment](#Rebase-Script-Execution-Environment)
  * [Proof of Concept](#Proof-of-Concept)
* [Bibliography](#Bibliography)
* [Appendix](#Appendix)
* [Ineffective alternatives to AVOUM](#Ineffective-alternatives-to-AVOUM)

<a name="problem"></a>
## The Problem

<a name="An-Auction-Gone-Wrong"></a>
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

<a name="A-Market-for-Open-Contract-Transactions-is-Inevitable"></a>
### A Market for Open-Contract Transactions is Inevitable

Any “open” contract on a UTXO blockchain can be subject to the same
Economic DoS: if anyone can interact with the contract, or if an
existing participant can interact with it an indefinite number of times
before others may react, then a sophisticated attacker can use the same
technique as described above to effectively prevent any other
unsophisticated participant from interacting with the contract until
after it is too late.

Even without an intentional attacker, the UTXO for heavily-used
contracts will be subject to heavy contention and frequent change, such
that regular non-technical users will have trouble interacting with it.
They have to watch the blockchain, determine the UTXO for the contract,
sign a transaction with this UTXO, wait for this transaction to either
make it through or be invalidated by a rival transaction, then sign a
new transaction and try again, repeating for hours on, hoping that at
some point there will be a lull in the activity of that contract and
their transaction will finally make it through.

Does that mean that open contracts on UTXO blockchains are forever
reserved for use by sophisticated participants who can run a full node,
watch the blockchain at high-speed, and race rival transactions in all
of multiple possible worlds at the same time, in servers close enough to
the miners to matter?

Actually, no: someone sophisticated has to do this job. But if the open
contract is properly structured, then this someone can be any
participant in a transaction posting auction market. Then, the
economically competent but technically unsophisticated user posts only
have to pay a sufficient transaction fee, and the sophisticated
professionals will compete with each other to be the one who effectively
posts the transaction.

A single sophisticated professional — offering his services to ensure
that users’ transactions get through in a timely fashion to interact
with frequently changing contracts — would likely be enough to start
this market. Posting a transaction then becomes fire-and-forget for
regular users, and the professional will earn high fees for consistently
winning the race to the UTXO. High fees will likely attract competition,
until a market emerges where sophisticated users compete to earn fees
from users, driving those fees down, and flooding the network with rival
transactions.

Eventually, miners would realize that they could cut out the middleman
between end-users and the blockchain by running those servers directly
on their mining rigs, saving a lot in fees and network traffic for
everyone.

In the end, when the market finally reached its equilibrium, users would
interact with open contracts by posting suitably malleable transactions
to the blockchain, and trust that miners would automatically substitute
the most recent contract UTXO for the one they posted, thereby having
the blockchain behave exactly as if it had an Account/Balance model.

<a name="Background-Concepts"></a>
## Background Concepts

<a name="UTXO-Model-vs-Account/Balance-Model"></a>
### UTXO Model vs Account/Balance Model

Cardano, like the first blockchain, Bitcoin, represents available assets
as UTXO, which stands for Unspent Transaction Outputs: each transaction
may take as inputs some UTXOs of previous transactions (which are
therefrom spent and no longer UTXOs) and itself produces UTXOs, thereby
transferring the assets from previous owners to new ones. Ownership is
enforced by the “lock script” associated with each UTXO, that ensures
that only the possessors of some cryptographic keys may spend the UTXO,
either freely or according to some “covenant” that limits their ability.
(Cardano's eUTXO model extends this, but in ways that are orthogonal to
this issue).

By contrast, the first blockchain with the ability to write smart
contracts, Ethereum, uses an Account/Balance model, transactions
transfer assets between accounts that persist across transactions, by
modifying their balances. Some accounts are simply controlled by
cryptographic keys, while more elaborate accounts are controlled by
“smart contracts” which are scripts written in the blockchain’s virtual
machine.

UTXOs make it easy for participating network nodes to verify
transactions, in parallel—whereas the Account/Balance model, at least in
its naive implementation, makes playing or replaying past transactions
an essentially sequential activity. Joining the network and validating
its state can therefore be done faster with UTXOs than with
Account/Balance, for the same reason. UTXOs provide an all over more
robust data model.

On the other hand, the Account/Balance model makes it much easier to
interact with “open” contracts involving a lot of participants in
parallel: a user can “just” sign a transaction that describes the action
they want to take, and this transaction will be valid irrespective of
the state of the contract when the action is taken. By contrast, with
the UTXO model, the user would have to track down exactly the state of
the contract (its UTXO) at the time the action is to be performed to
even prepare the correct transaction; but this state could change very
quickly, which opens the contract users to the Economic
Denial-of-Service Attack described above.

<a name="FOMO3D-and-the-Dark-Forest"></a>
### FOMO3D and the Dark Forest

Economic Denial-of-Service Attacks are not mere speculation. And they
are not just for UTXO blockchains. Indeed, they already happen, all the
time, on the Ethereum blockchain, that uses the Account/Balance model!
The most famous stories about it are the winning of the FOMO3D tontine,
and the Dark Forest of front-runners for swap contracts.

The FOMO3D was a joke of a contract, a blockchain variant of a tontine.
In a historical tontine, many participants, usually young people, put
money in a pot, or left it to a careful money manager—until all the
participants died but one, who then got all the money in the pot. In
times and places when life expectancy was shorter than the modern world,
the lone survivor might even still be young enough to actually enjoy his
fortune. To adapt the concept of tontine to the blockchain, a simple
mechanism was used to detect the last man standing (or then again last
robot operating): anyone could add tokens to the pot at any time (above
some floor contribution) and thereby become a participant; to take the
tokens out, you just had to be the last one putting tokens in, with no
one else having put in any tokens for the last 24 hours. Of course, some
rival people were running robots to prevent anyone else from winning, by
making sure to chip in a few tokens after 23 hours and 55 minutes or so,
if needed. And yet, some clever person managed to steal the then
million-dollar pot while rival robots were still operating. How did he
do? By running an Economic Denial-of-Service Attack: he simply put money
in the pot, waited for about 23 hours and 55 minutes, and then bought
the entire Ethereum blockchain for 5 minutes until his rivals had been
excluded long enough for him to take the money out. How did he buy the
entire blockchain?  With repeat copies of a transaction that paid for
the entire block gas limit at a gas price advantageous to the miners,
until the 5 minutes had passed.

In the Dark Forest story, also on Ethereum, a person noticed a way that
anyone can take tokens out of a misconstrued ERC20 token contract with a
clever swap transaction. A good actor, he decided to take the tokens out
before they disappear, and safeguard them for their original owners. But
there are robots out there that watch all ERC20 contracts for any
advantageous swap transaction and will front-run that transaction with a
rival transaction executing the same swap, but to the advantage of the
robot operator rather than the original arbitrageur. To make sure the
rival transaction makes it in, the robot simply puts a larger gas price,
as long as there is any profit in the swap large enough to justify the
gas cost. And our good actor was thereby scooped by a robot, and
concluded that Ethereum is a Dark Forest where you’ll easily get
mugged.

<a name="The-Implicit-Auction-for-Blockchain-Space"></a>
### The Implicit Auction for Blockchain Space

Thus, Economic Denial-of-Service Attacks already exist, and so even on
Account/Balance blockchains. At the bottom of the issue, the underlying
reality is that space on the blockchain is scarce. Whether limited
explicitly through a conventional block size limit, a conventional gas
limit, or just implicitly via the network bandwidth of the winning
miners, there is a limit on how many transactions can make it to any
given block. From an economic point of view, that means that getting a
transaction into a block is winning an auction.

Therefore, the first issue, demonstrated by the FOMO3D story, is that of
technical sophistication. When one participant knows how to play the
transaction-posting auction correctly, when the other participants are
too inept to even realize the game they’re playing, more technically
sophisticated than others, the sophisticated participant can run circles
around the other ones and prevent them from ever getting their
transactions to the blockchain, then cause them to timeout and extract
any resulting value from the smart contract.

In the case of racing a UTXO, our solution above makes the same
sophistication available to every participant, for a small fee.
Sophisticated professionals, typically miners, handle the complexity for
regular users, and the technical problem has been reduced down to the
underlying economic problem. In the case of the FOMO3D story above, the
economic problem was hidden under a very thin layer of technical
sophistication. In both cases, the auction for blockchain space,
previously left implicit, was made explicit: to get your transaction in,
you need to bid a higher fee than your rivals—which shouldn’t be a big
problem if you’re playing a positive sum game while they’re playing a
zero-sum game.

Now, the auction for blockchain space is a first price auction, since a
second price auction would be gameable by miners. Thus, to win an
auction and get their transactions in, the participants must remain
active, monitor the blockchain, and carefully increase their bid with
market conditions, alongside their rivals; otherwise they’ll pay too
much (if they reveal their actual maximum bid too early), or they’ll
fail to get in (if they fail to raise the price to their maximum bid).
There is still some sophistication required on the side of the
participant: not technical sophistication, but economic sophistication.
Posting transactions is still not fire-and-forget: users must keep
watching the blockchain, check whether their transaction went through,
and if not, sign a new variant of the transaction with a higher fee and
watch again (and be ready to be surprised in case their old transaction
made it through after all). This matters a lot whenever there is a
deadline to sending transactions, which is almost always the case when
writing robots that automate participation to some DApps.

To go back to our illustrating example, the FOMO3D story, “all” the
tontine-participating robots had to do to keep playing their chicken
game was to match and slightly outbid the fees posted by the attacker
for one block. Indeed, to win the pot, the attacker had to win the
auction for each and every block for the entire duration of the attack,
whereas the defenders would only have had to win one single block. Since
there were about 20 blocks in those five minutes, they could have let
him buy fifteen entire blocks before they outbid him, making it a dear
lesson no would-be attacker would forget. Defending successfully would
have been an order of magnitude less costly than the attack. It would
have been yet another order of magnitude less costly if they had started
their transactions an hour in advance rather than waited until just five
minutes before the deadline. Or if the blockchain had had ten times as
much throughput. In general, defenders against Economic DoS attacks are
at an advantage and attackers are at a disadvantage—but only if they
play the game correctly. With economically rational and technically
correct robots, the attacker would stand no chance. But that in itself
is the topic for a separate project.

<a name="Miner-Extractable-Value"></a>
### Miner Extractable Value

The second issue, demonstrated by the “Dark Forest” story, is that in a
sophisticated-enough market, the miners will extract the value that is
theirs. And this value comprises the fees that a transaction is worth
paying for, as well as any profit that can be made by rewriting
malleable transactions to their profit. This includes changing the name
of the beneficiary of a single-transaction arbitrage—a zero-sum game
where the user loses—as well as inserting the latest UTXO in a
transaction to extract a fee—a positive-sum game where the user wins. In
doing so, they are not being evil in one case and good in the other,
they are just playing the game competently, as designed.

Super-competent miners might even loosely collude with each other to
squeeze higher transaction fees from users who fail to tip their
transactions sufficiently when they have a short deadline. This
collusion in practice will be counterbalanced by the interest of the
miner to defect from the cartel and accept the lower fee for themselves,
rather than help another miner collect the higher fee. But the more
concentrated the mining power, the more the miners should cooperate to
raise fees. However, this is speculation for a distant future.

Our present project is to build the engine for miners to play the
positive-sum game where miners and users win as the users’ transactions
get posted.


<a name="Solution-Overview"></a>
## Solution Overview

The Account-View-On-UTXO-Model (AVOUM) approach solves the EDoS problem
described above by providing a way to support "malleable" transactions.
Malleable transactions can be submitted to a node, who is then able to
update the transaction in order to solve conflicts with intervening
transactions, rather than having to reject the original transaction
outright. This eases the burden on participants to deal with contention,
while still allowing the benefits of the UTXO model.

Our solution will enable the safe deployment of “open” contracts on the
Cardano blockchain, which is not currently possible, and will bring its
smart contract capabilities in parity with Ethereum.

It is worth observing that the AVOUM service could, in principle, be
provided entirely as an added layer atop the current Cardano blockchain;
it does not *require* changes to the core Cardano software (such as
`cardano-node`). However, as discussed above, once the market reaches
equilibrium, it would be most natural for miners/node operators to offer
this service themselves, so we propose integrating the necessary
functionality directly into the core Cardano software.

<a name="Feasibility-Study"></a>
## Feasibility Study

We have conducted a feasibility study to determine the practicality of
implementing AVOUM on top of the Cardano blockchain; this section
describes the results of that study, which include a detailed sketch
of a concrete design, and a sketch of a proof-of-concept contract
built atop that design.

<a name="Summary"></a>
### Summary

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
  by design, and inputs which require a signature also defeat
  malleability.  For users holding assets protected by one of these
  conventional mechanisms, using malleable transactions thus requires an extra
  step: first, cells compatible with malleability are created in a regular
  non-malleable transaction. Only then can these cells be assembled into
  malleable transactions.

<a name="Protocol-and-Interface-Modifications"></a>
### Protocol and Interface Modifications

This section describes necessary modifications to Cardano protocols and
interfaces.

<a name="Malleable-Transaction-Submission-API"></a>
### Malleable Transaction Submission API

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
#### Rebase Script Execution Environment

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
-- field is treated as opaque by the miner, and available for use by the
-- contract itself.
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

...and the return value is the raw bytes of the new transaction.

<a name="Proof-of-Concept"></a>
### Proof of Concept

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

<a name="Auction-State"></a>
#### Auction State

The auction contract tracks its state in an account, whose datum is
an encoded value of type `AvoumCell AuctionState`, where `AuctionState`
is defined as:

```haskell
data AuctionState = AuctionState
    { asDeadline :: POSIXTime
      -- ^ Deadline for bids; after this the auction is closed.
    , asCurrentBidder :: Credential
      -- ^ Credential of the current highest submitted bid; if the
      -- current bid wins the auction, this credential should be
      -- attached to the purchased assets.
    , asSeller :: Credential
      -- ^ Credential of the seller; when the auction closes,
      -- this credential should be attached to the winning bid.
    , asCurrentBid :: Value
      -- ^ The current highest bid.
    , asAssets :: Value
      -- ^ The assets being sold.
    , asMintSymbol :: CurrencySymbol
      -- ^ Symbol used by our minting policy. We store this field
      -- here to break a cyclic dependency; otherwise the validator
      -- would have to know the minting policy's currency symbol
      -- a-priori.
    }
```

For an `AvoumCell AuctionState` to be valid, its cell must hold
tokens equal to the sum of:

- the current bid (`asCurrentBid`),
- the assets being sold (`asAssets`),
- ...and a balance of 1 token of the currency identified in the cell's
  `aidSymbol` field of its account id.

This invariant is enforced by the minting policy when issuing the token
in `aidSymbol`, and by the validator when updating the state. We assume
helpers:


```haskell
-- | Check that the value of the output agrees with its state.
valueStateConsistent :: TxOut -> AuctionState -> Bool
valueStateConsistent txOut state =
  txOutValue txOut == sum
    [ asAssets state
    , asCurrentBid state
    , auctionMarkerValue (asMintSymbol state)
    ]

-- | Value whose presence indicates taht this is an auction cell; the
-- miniting policy for the passed CurrencySymbol must enforce that only
-- auction state cells can hold this.
auctionMarkerValue :: CurrencySymbol -> Value
auctionMarkerValue mintSym =
    -- XXX/TODO: It is unclear to me why the map is nested here, and
    -- what the role of TokenName is. Maybe it's a namespacing mechanism
    -- so that a single minting policy can mint different tokens? Need
    -- to investigate further.
    Value $ Map.singleton mintSym (Map.singleton "auction" 1)
```

<a name="Transaction-Format"></a>
#### Transaction Format

There are three types of transactions that can involve an auction:

- Opening the auction
- Placing a bid
- Closing the auction (and distributing assets/bids).

Opening the auction is handled by the minting policy; the transaction
should have one input, which is the assets being sold, and one output
which is the auction cell.

The other two transaction types are handled by the validator. The
validator can distinguish them based on their valid time range;
bids must be valid only before the deadline, and closing transactions
must be valid only after the deadline.

A transaction issuing a new bid comprises:

- Two input cells
  - The first of which is the initial auction state
  - The second of which is a cell constituting a new bid
- Two output cells
  - The first of which is the new auction state; it should
    reflect the new highest bid (which of course must be greater
    than the old one).
  - The second of which is a refund for the previous highest bidder;
    it should have the value and credential specified in the input
    auction state.

A transaction closing the auction comprises:

- One input cell, the auction state.
- Two output cells
  - The first of which is the assets being sold, which should have the
    credential specified for the current bidder in the input state.
  - The second of which is the highest bid, and should have the
    credentials for the seller.

<a name="Minting-Policy"></a>
#### Minting Policy

The minting policy is responsible for ensuring:

- A newly minted token's datum is a valid `AvoumCell AuctionState`, as
  described above.
- All auctions have distinct `AvoumId`s.

Below we sketch the implementation of the minting policy:

```haskell
auctionPolicy :: ScriptContext -> Bool
auctionPolicy ctx =
  let ownSymbol = ownCurrencySymbol ctx
      txInfo    = scriptContextTxInfo ctx

      -- State assigned to the newly minted cell.
      outputState :: AvoumCell AuctionState
      outputState = decode $ snd (txInfoData txInfo !! 0)

      txOut = txInfoOutputs txInfo !! 0
  in
  and
    [ length (txInfoInputs txInfo) == 1
    , length (txInfoOutputs txInfo) == 1

    -- Make sure we're minting exactly one of the expected token type
    , txInfoMint txInfo == auctionMarkerValue ownSymbol

    , aidSymbol (acId outputState) == ownSymbol
    , valueStateConsistent txOut (acState outputState)

    -- We use the reference to this transaction's first input as our
    -- unique value; since this transaction spends that output, it
    -- cannot be used again.
    , decode (aidUnique (acId outputState)) ==
        txInInfoOutRef (txInfoInputs txInfo !! 0)

    -- Make sure the output is guarded by our expected validator script.
    -- We assume the hash of the validator script is known and can just
    -- be hard-coded in the minting policy's code:
    , addressCredential (txOutAddress txOut) == ScriptCredential validatorHash

    -- Because we know the validator's hash, it can't know ours without
    -- creating a cyclic dependency. So instead we store our symbol in
    -- the auction state, and assert its correctness here:
    , asMintSymbol (acState outputState) == ownSymbol
    ]
```

<a name="Validator"></a>
#### Validator

The validator is responsible for enforcing the rules of the auction once
it has begun.

```haskell

txValid :: () -> AvoumCell AuctionState -> ScriptContext -> Bool
txValid () input ctx =
    let deadline = asDeadline (acState cell)
        scriptContextTxInfo ctx
        validRange = txInfoValidRange (scriptContextTxInfo ctx)
    in
    if ivTo validRange < deadline then
        -- Before the deadline; check for a valid bid.
        txValidBid
    else if ivFrom validRange > deadline then
        -- After the deadline; check for a valid auction-closing
        -- transaction.
        txValidClose
    else
        -- Ambiguous as to whether we are before or after the
        -- deadline; reject.
        False
 where
    txValidBid =
        let txInfo = scriptContextTxInfo ctx
            output :: AvoumCell AuctionState
            output = decode $ snd (txInfoData txInfo !! 0)
            inS = acState input
            outS = acState ouptut

            auctionOut = txInfoOutputs txInfo !! 0
            refundOut  = txInfoOutputs txInfo !! 1
            auctionIn  = txInfoInputs txInfo !! 0
            bidIn      = txInfoInputs txInfo !! 1
        in
        and
            [ -- Two outputs: new state and refunded old bid:
              length (txInfoOutputs txInfo) == 2
            -- Two inputs: old state and new bid:
            , length (txInfoInputs txInfo) == 2

            -- This script should be invoked in order to spend input
            -- cell 0. This is important when we check for consistency
            -- between that cell and the new one:
            , scriptContextPurpose ctx == Spending (txInInfoOutRef auctionIn)

            -- Validity checks for new auction state:
            , acId input == acId output
            , valueStateConsistent auctionOut outS
            ,    addressCredential (txOutAddress (txInInfoResolved auctionIn))
              == addressCredential (txOutAddress                   auctionOut)

            -- New bid must actually beat the old one:
            , asCurrentBid sOut > asCurrentBid sIn

            -- Refunded old bid must have the right value & ownership:
            , txOutValue refundOut == asCurrentBid sIn
            , addressCredential (txOutAddress refundOut) == asCurrentBidder sIn

            -- Make sure the recorded bid matches the input:
            , txOutValue (txInInfoResolved bidIn) == asCurrentBid outS

            -- TODO: check that new auction state has the correct bidder?
            -- Need a convention for denoting this then.
            -- Or maybe just leave that to the validator for the input
            -- bid, since this allows more flexible policy?
            ]
    txValidClose =
      let s         = acState input
          txInfo    = scriptContextTxInfo ctx
          assetsOut = txInfoOutputs txInfo !! 0
          bidOut    = txInfoOutputs txInfo !! 1
      in
      and
        [ -- Should output two cells, one for the assets being
          -- sold, and one for the bid being accepted:
          length (txInfoOutputs txInfo) == 2

        -- Make sure the ouptuts have the values and credentials they
        -- should:
        , txOutValue assetsOut == asAssets s
        , txOutValue bidOut == asCurrentBid s
        , addressCredential (txOutAddress assetsOut) == asSeller s
        , addressCredential (txOutAddress bidOut) == asCurrentBidder s
        ]

```

<a name="Rebase-Script"></a>
#### Rebase Script

```haskell
rebase :: ScriptContext -> Map AvoumId ScriptContext -> ByteString
rebase origCtx ctxById =
    let origTxInfo = scriptContextTxInfo origCtx

        origOutput :: AvoumCell AuctionState
        origOutput = decode $ snd (txInfoData origTxInfo !! 0)
        origState = acState origOutput

        interveningCtx = ctxById Map.! acId origOutput
        interveningTxInfo = scriptContextTxInfo interveningCtx

        deadline = asDeadline origState
        interveningValidRange = txInfoValidRange interveningTxInfo

        origValidRange = txInfoValidRange origTxInfo
    in
    if ivTo interveningValidRange >= deadline then
        error "intervening transaction closed the auction; can't rebase."
    else
        -- Placing a bid.
        let interveningOuptut :: AvoumCell AuctionState
            interveningOuptut = decode $ snd (txInfoData  interveningTxInfo !! 0)
            interveningState = acState interveningOutput
        in
        if ivTo origValidRange < deadline then
            let refundValue = asCurrentBid interveningState
                refundCredential = asCurrentBidder interveningState

                newState = interveningState
                    { asCurrentBid = asCurrentBid origState
                    , asCurrentBidder = asCurrentBidder origState
                    }

                newOutput = AvoumCell
                    { acId = acId origOutput
                    , acState = newState
                    }
            in
            makeTx
                -- inputs:
                [ copyOutputToInput interveningTxInfo 0
                , copyInputCell origTxInfo 1
                ]
                -- outputs:
                [ makeAuctionCell newOutput
                , makeDistributeValueToCell refundValue refundCredential
                ]
        else
            -- Closing the auction.
            makeTx
                -- inputs:
                [ copyOutputToInput interveningTxInfo 0
                ]
                -- outputs:
                [ makeDistributeValueToCell
                    (asCurrentBid interveningState)
                    (asSeller interveningState)
                , makeDistributeValueToCell
                    (asAssets interveningState)
                    (asCurrentBidder interveningState)
                ]
```

<a name="Bibliography"></a>
## Bibliography

- “Ethereum is a Dark Forest”, Dan Robinson and Georgios
  Konstantopoulos, August 2020,
  <https://medium.com/@danrobinson/ethereum-is-a-dark-forest-ecc5f0505dff>
- “Flash Boys 2.0: Frontrunning, Transaction Reordering, and Consensus
  Instability in Decentralized Exchanges”, Daian et al., April 2019,
  <https://arxiv.org/pdf/1904.05234.pdf>
- “Why Developing for the Blockchain is Hard — Part 1: Posting
  Transactions”, François-René Rideau, December 2018,
  <https://hackernoon.com/why-developing-for-the-blockchain-is-hard-part-1-posting-transactions-dde21c025c65>
- “How the winner got Fomo3D prize — A Detailed Explanation”, SECBIT Labs, August 2018,
  <https://medium.com/coinmonks/how-the-winner-got-fomo3d-prize-a-detailed-explanation-b30a69b7813f>
- “Chimeric Ledgers: Translating and Unifying UTXO-based and
  Account-based Cryptocurrencies”, Joachim Zahnentferner, March 2018,
  <https://iohk.io/en/research/library/papers/chimeric-ledgerstranslating-and-unifying-utxo-based-and-account-based-cryptocurrencies/>
- “Flashbots Transparency Report”
  <https://medium.com/flashbots/flashbots-transparency-report-february-2021-8ac45b467d0a>

<a name="Appendix"></a>
## Appendix

<a name="Ineffective-alternatives-to-AVOUM"></a>
### Ineffective alternatives to AVOUM

Can the race condition that AVOUM solves be solved without AVOUM? One
tempting alternative approach is to split interactions in two phases,
one in which orders are issued in parallel without contended access to
shared state, while the other sees the orders being combined with shared
state. For instance, in an auction, bids would be posted in parallel
without touching the contract state, then in a second phase, they would
be collected with the top bid winning.

With this approach, no attack is possible during the first phase, and
the attack surface in the second phase is bounded by the number of
orders issued during the first phase. Unhappily, this alternative
approach does not actually solve the problem, and only results in making
life harder for honest users.

Indeed, the attacker can publish plenty of UTXOs in the first phase,
under as many sybil identities; then in the second phase, the attacker
can still race the honest users out of being able to access the shared
state, by issuing transactions to process his UTXOs in many different
orders, in parallel. The phasing forced him to be more prepared, reveal
his attack intention and pay for these UTXOs. But by hypothesis he is
more technically savvy and prepared than honest participants, can reveal
at the last minute, and was already willing to pay for said UTXOs. The
approach forces him to only have bounded attack capacity during the
second phase, but he also only needed bounded attack capacity all along,
and by hypothesis already possesses it.

Honest participants can detect the attack at the end of the first phase,
but inasmuch as they can do anything to defend against it, the
infrastructure required for this defense would be no simpler than the
one required to sustain the original attack. And inasmuch as miner
intelligence could help honest participants by letting them pay higher
fees to have their requests prioritized over those of the attacker,
adding this intelligence to the miners’ software would not be simpler
than adding AVOUM support to it.

In the end, no actual defense is achieved by this approach, whereas the
interaction flow is made longer and more complex for users, who now have
to issue two transactions instead of one, during each of two phases,
with an additional delay for the second phase.

Moreover, it is not clear that this approach is applicable to arbitrary
contracts. Auctions and any interaction with a bounded number of
transaction phases can use this approach and at most double the number
of transaction phases to implement this dubious attack mitigation. But
continuous interactions in which new participants can join with no
deadline limitation cannot be modified this way: for instance, tokens
with shared state as needed for yield farming, the books of exchanges,
and other common continuous interactions, cannot be transformed this
way. Arbitrarily dividing these interactions into phases is conceivable,
but would only add latency and complexity without solving the underlying
issue.

Another “solution” is to vastly increase the cost of interacting with
the shared state, via a fee that discourages posting of UTXOs. But this
fee only discourages attacks inasmuch as it discourages honest
interactions even more so, and in the end doesn’t actually prevent the
attack in the case when honest interactions themselves weren’t
discouraged.

# TODO

A (not necessarily exhaustive) list of some things that still need to
be done to this document:

- Finish the proof of concept section.
  - Fill in TODOs in the comments
  - Do the rebase script
  - ...?
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
- Make sure the TOC is consistent, after the body has all been written.
