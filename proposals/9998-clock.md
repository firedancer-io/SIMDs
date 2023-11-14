---
simd: 'XXXX'
title: Block Timing
authors:
  - Liam Heeger (@CantelopePeel)
category: Standard
type: Core
status: Draft
created: 2023-11-09
feature: TBD
supersedes: 
extends:
---

## Summary

Real-world block times must be measured precisely and decided upon on 
chain to enable consistent and equitable to end-users and maximally profitable
to the validator set. This proposal adds an on-chain high resolution clock 
for recording block times via a new sysvar. It requires voters to post
high resolution time stamps with their votes.

This proposal also introduces a required mechanism for ensuring leaders do not 
exceed their allotted time to produce and broadcast a block. Voters will post a 
high resolution timestamp as a part of each vote. Leader (un-burned) fees for the
block are held by the bank until the block is rooted. Once the block becomes 
rooted, a median of the stake weighted population of these timestamps for each 
vote is computed. A formula for how fees are granted to the leader for this 
rooted block is defined based on the difference between the prior timestamp and
the current rooted timestamp is defined, and apply. Any un-granted fees are
burned.

## Motivation
A constant design problem with blockchains is to how to form and shape the fees
for transactions in such a way to attain desirable properties for the network.

Frequently, and especially at the time of writing this[^1], the proposed 
solutions to this is to engineer a model of the real world into the actual base
fees for transactions that incentivizes validators to take up transactions. 

One issue with this approach is that the model more rigid to evolution (as it is
typically embedded into the runtime, so any changes are consensus changes) than 
the actual population of transactions. That is to say, the population of 
transactions users are sending changes faster than we can update the cost model.

Another issue is that the cost model for transactions is extremely complex, and
highly dependent on the validator. Some variables are hidden, like the cost to
pull accounts from disk, others are visible like the number signatures that need
verification. Nonetheless, validators are incentivized to take on transactions
which are profitable, and maximize fees.

We solve this by proposing a mechanism that incentivizes all validators to 
maximize fees per second of transaction, while not overrunning their allotted
block time. 

<!-- 
- Machines in validator set are heterogeneous
- Cold accounts, reduces info needed to be a block producder.
- Fee market complexity
- Reducing the slop in block timing
- Leaders become competitive, fee markets
- Account state growth
-->

## Alternatives Considered

<!-- TODO: Fee model changes-->

## Detailed Design

The following are the design changes to the protocol which must be made to
implement the proposal:

### Vote Program Changes

#### Vote Timestamp
The various `timestamp` used in `VoteInstruction` variants:

  - `CompactUpdateVoteStateSwitch`
  - `CompactUpdateVoteState`
  - `UpdateVoteState`
  - `UpdateVoteStateSwitch`
  - `Vote`
  - `VoteSwitch`

MUST change from representing some timestamp in seconds since Unix epoch[^2] to 
representing a timestamp in nanoseconds since Unix epoch time. Vote accounts 
states also store this number in the existing timestamp field. Vote account 
transaction submitters SHOULD make a best effort to represent this timestamp as
the time when they finished processing the block which is being voted on.

### Clock Sysvar Changes
The clock sysvar, when calculating the staked weighted timestamp, remains 
unchanged except vote timestamps are interpreted now as:

```math
\mathtt{vote\_timestamp} = \mathtt{vote\_state.timestamp} \div 10^9
```
[^3]

### High Resolution Clock
The high resolution clock (HRC) is a unsigned 64-bit value representing a 
singular nanosecond timestamp computed from the set of timestamps posted in 
votes in a block. This value will not be made available to the program runtime 
in this proposal. The HRC value for a given slot are computed as part of slot 
rooting procedure. The algorithm for computing the HRC value is as follows:

1. The rooted slot has a list of tuples of vote state timestamps (VSTs) and 
stake weights (SWs), $\langle \mathtt{VST}, \mathtt{SW} \rangle$, for each voter
that participated in the slot. The stake weight is the total lamports staked to
the validator. We call this list the timestamp stakes (TS) and is defined as 
$\mathtt{TS} = [ \langle \mathtt{VST}_i, \mathtt{SW}_i \rangle : i \in \mathtt{voters\_at\_slot} ]$.

1. Map each unique VST to the sum of SWs the for VST, sort the result with VST 
descending.

1. Compute the median VST by iterating from highest to lowest VST, adding up
stake weights until you have iterated at or over the floor of half of total 
stake weight. The VST at this value is the HRC.

1. Before rooting, hold the current HRC value ($\mathtt{HRC}_s$) and rooted slot 
HRC value ($\mathtt{HRC}_{s-1}$) for leader fee distribution. If 
$\mathtt{HRC}_s$ is less than $\mathtt{HRC}_{s-1}$, $\mathtt{HRC}_s$ MUST be
set to $\mathtt{HRC}_{s-1}$.


### Leader Fee Distribution
Until, a slot is rooted, the leader will not receive the collected fees for the 
slot. After the HRC value is derived prior to rooting the slot, the difference
of $\mathtt{HRC}_s$ and $\mathtt{HRC}_{s-1}$ is computed as 
$\Delta\mathtt{HRC}_s$. Based on the value of $\Delta\mathtt{HRC}_s$ the fees
for the slot $s$ are granted in the currently executing slot based on the
following formula:

```math
\mathtt{LeaderFee}(\Delta\mathtt{HRC}_s, \mathtt{fc}) =
\begin{cases}
\mathtt{fc}, & \Delta\mathtt{HRC}_s \leq 400ms \\
\mathtt{fc} * \frac{\Delta\mathtt{HRC}_s - 400ms}{400ms} , & \Delta\mathtt{HRC}_s \gt 400ms
\end{cases}
```

Where $\mathtt{fc}$ is the fees collected in lamports during execution of slot 
$s$. All fees whih are not granted to the leader are burned.

<!-- 
TODO: how to deal with skipped slots
TODO: fork choice rule
TODO: what to do with 50% burn
 -->

## Impact

### Current Perception of Fee Model
Currently it is believed that because validators are including expensive
transactions at the same price as less resource intensive ones, that this is a
failing of the underlying protocol. It is not. 

It means that validators are currently elastic to the price of transactions.
This is not a problem of the protocol, it is a problem of block producers. 

### Solidifying the Fee Model
A function of this proposal is that we solidify the fee model for transactions.
Validators need to evolve the prices they are willing to accept in order to
include a transaction. As a function of this proposal, we propose freezing
changes to the base cost of transactions and instead ask users to set priority
fees based on what they expect the cost of the transaction to be. We ask 
block producers to decide if they want to include transactions which are priced
appropriately.

This means that users can set priority fees to get transactions included and
block producers can decide based on their cost model if the transaction is
includeable.

This eliminates the necessity for consensus-level changes to the fee model:
- Write lock fees: The block producer can decide if the write locks issued 
on a transaction will induce contention as they are building the block. In many
cases there is no contention on transactions which access the same account, but
only the leader knows that at block production time and so only they can
actually price that in.
- Account loading fees: These fees would require payment for the number of
bytes loading by a transaction as a function of the accounts accessed. The 
leader knows best as to the cost of loading accounts (which is not homogenous)
across accounts. Similarly, in order to collect these fees the leader would need
to know the account size, breaking bankless leader efforts.
- Hot-cold account fees: These fees require users to pay more to use an account
which has not been used in some time. This fee is hard to model against the
real world where validators have heterogeneous hardware and cache properties.
- Spam fees: If the validator is being spammed on particular accounts, they are
the only one who knows about this as much of the spam over the network is being
being dropped. Nonetheless, they can interally reprice access to those accounts
or from those transaction senders to require them to pay more for their rude
spammy behavior.

All of these fees boil down to the simple fact that it costs more to the
validator to include transactions which require more work, but each validator
has a different cost model that informs what their transaction inclusion
strategy is.

### Pricing Estimates for Transaction Senders
This means that pricing a transaction becomes a market, which it already is 
after priority fees. Ecosystem participants will have to provide services to
price transactions based on the current going rate. Wallets may have to offer
more diverse options to users when submitting transactions.

### Timeline
We do not believe there are currerntly serious issues with the fee model for
Solana which require major engineering solutions right now. This change will be
necessary but for an orderly validator set, but can be done socially while we
develop this solution. This proposal does not need to come into affect
immediately.

### Future Shorter Block Times
Validators are disincentivized to overrun their block times. We can change the 
shape and parameters of the leader fee distribution function to increase or
decrease the desired block time.

### Compute Unit Limits
This proposal opens up an oppurtunity to remove limits on compute unit limits on
acccounts and blocks. This is due to the soft timing deadline on block 
production allowing for more diverse transaction set to be allowed. 

## Security Considerations
<!-- TODO: what do we need to do to ensure egregious timestamps are not posted? -->

## Drawbacks
If block times come down, it may require validators to have good synchronization
of their internal clocks. Good algorithms exist for this but for now validators
can be effective at the millisecond level by maintaining clock sync via NTP or
similar internet time service.

In the future if this clock sync becomes an issue or their is a desire to 
decentralize the clock synchronization, we can devise a peer-to-peer, secure
algorithm to ensure the Solana network maintains good clock sync.

## Backwards Compatibility

This feature will affect consensus but will not break backwards compatibility.

---

[^1]: Here we are referencing to specific discussions to pricing in write-lock
fees, spam fees, dApp rebates, hot-cold fees, native program fees, and account 
loading fees.

[^2]: Unix epoch time is defined as the time since 1970-01-01T00:00:00+00:00.

[^3]: The symbol $\div$ represents integer division.