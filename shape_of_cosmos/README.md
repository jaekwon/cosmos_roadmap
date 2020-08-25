# Table of Contents

## State of the Ecosystem

This post assumes some background information about the state of the Cosmos
ecosystem, as detailed in [XXX include ECOSYSTEM-URL].  If you aren't aware of all the
work going on below, please check it out, or jump to the details of any
particulr project by clicking on the link below:

- IBC [XXX make hyperlink to portion in above ECOSYSTEM-URL, and for all items below].
- Stargate
- Cosmwasm
- Liquid Staking
- DEX
- Ethermint
- Peggy
- Shared Security
- Stargate
- Starport
- Tendermint/Classic

[NOTE: Keep above list in sync]

## Shape of Cosmos

* Shared Security
  - Interchain Staking
  - Replicated Security
  - Other models

- Exchanging
  - Simple Orderbook
  - Simple Uniswap
  - Standardization

- Hubs and ATOMs
  - The Function of the Hub
  - The Function of ATOM
  - Branching for Innovation
  - The Function of the ATOM
  - Dealing with Hub and Zone Failures
  - Some Math

## Short Term Plan

- Technical
  - Chain Upgrading
  - Hub Staking
  - Liquid Staking
    - interchain accounts
  - Smart Contracts
  - AminoX & Object Store

## Potential Pitfalls

- Peg Risk
  - vs immutable PoW vs PoS
- Reducing Surface Area
- Liquid Staking

## Credits and References

If you can read this, this essay is a work in progress. If you want to
contribute to this essay, just make some insightful comments or sections in the
form of a PR or comment, and I'll add them here.

<!-- ######################################## -->

# Shape of Cosmos

## Shared Security

Any chain can act as a hub.  For decentralized crypto security, ideally we have
many interoperable hubs that compete for security.

Any two chains can connect directly exchange tokens without going through a DEX
zone, thereby eliminating any external third party risk.

The future shape of the interchain is a complex network graph; not a
heirarchical dag, not a winner-take-all single blockchain system; not even a
DAG.  It will be complex interwoven network with interzone IBC links created
(or not created) according to the cost-risk-benefit-analysis and regulatory
considerations at the edges.

This section will attempt to explain why I believe this to be true, and show
what role the Cosmos Hub and the ATOM token can maintain, for the best
product-market fit and for the good of the ecosystem.

### Interchain Staking

The basic construction of staking *on a single chain* is simple.  For example
on the Cosmos Hub, there are ATOM holders, and any ATOM holders can "stake"
their ATOMs by delegating them to an active validator.  There is an unbonding
period of 3 weeks to get your ATOMs back.  The top N validators at any moment
according to total delegated ATOMs participate in a BFT algorithm that involves
cryptographic signatures of blockhash votes.  If a Cosmos Hub validator signs
maliciously, that evidence can get submitted to the Cosmos Hub. So far so good,
sovereign staking on a single chain, at least if you only want to consider
minor faults like a single validator double-signing.

If there is a chain-fork, aka double-spend attack, where say two blocks get
committe at the same height, there are enough validators (more than 1/3) to
censor evidence from being submitted to the blockchain, so the previous
construction doesn't work.  *One way to prepare blockchain failure is to
delegate the handling of failure via a trial on another blockchain (the
"designated recoverer") that was previously self-elected by that blockchain*.
Upon the discovery of Evidence that a chain has halted, represented by two
conflicting Commits, the designated recoverer and validators enter into a
challenge and response protocol to determine who is at fault for the fork of
that chain.  The protocol is described in the Tendermint wiki, but is getting
formalized and implemented.  (At least it will be on Tendermint Classic, a fork
of Tendermint that I'm working on to refine the original design principles and
to prototype improvements across the stack.  But more about that later. [XXX])

The designated recoverer does not need to be too intelligent.  It just needs to
be a system that all observers can observe, and provides some guarantees around
liveness and the ability to keep track of time.  In the very minimum, it just
needs to serve as a communication medium for sharing all evidence, like a
shared network folder that faithfully keeps track of timestamps.  All relevant
evidence must be submitted within the synchronous time window, or your
validator may get slashed, or at least not able to participate in the re-org
rebooted chain until governance decides what should be done.

A more intelligent designated recoverer could record the result of the
"consensus trial" in the Merkle store.  Most clients or smart contracts will
prefer to rely on a particular single recoverer -- they will just want to
delegate the responsibility of keeping track of consensus failures and
consensus trials to a blockchain that is dedicated to performing that role for
many other chains.  I think this should be a "branch" of the Cosmos Hub, but
more on that later [XXX].


The single chain designated recoverer system doesn't always work though.  For
example, there may be no designated recoverer, or the designated recoverer may
also have failed.  I suppose you could create a linked list of recovery, but
that may not make much of an improvement in practice, compared to just
defaulting to let governance deal with this rare fault.  It is probably
possible to make the single-recoverer system even safer with multiple
(non-intelligent, store-only) designated recoverers, assuming that at least one
of these recovery mediums can store all relevant information.

[XXX add more]

### Interchain Staking Economics

IBC is just two blockchains commmunicating with each over by leveraging BFT
consensus proofs (lots of cryptographic signatures), and also the magic of
Merkle trees.  For example, "Alice on the Cosmos Hub wants to send 100ATOMs to
Bob on BarChain" is information that can be proven to BarChain.

Using IBC, you could actually stake across chains.  Here's a simple
construction: Alice has 100ATOMs on the Cosmos Hub.  Alice wants to participate
in BarChain, which has its own staking token (and thus an independent set of
validators), but doesn't have any BAR tokens.  No problem, the Cosmos Hub has
an IBC connection with BarChain, and both understand the "Interchain Staking
Protocol".  Alice stakes 100ATOM with a transaction on the Cosmos Hub, and this
increases the real security of BarChain, because the Cosmos Hub ensures that in
the case of consensus failure on BarChain, staked ATOMs will get slashed.

There's one question here that pops up, for readers aware of the economics of
the Cosmos Hub: why would you "stake" ATOMs on BarChain if you aren't going to
earn inflationary rewards of ATOMs?  I'll address that, but first lets consider
a simpler construction by assuming that you can stake other tokens, like the
hypothetical PHOTON token, which avoids the question.

In the simplest construction, interchain staking is not like delegating to
one's choice of validators on the Cosmos Hub -- it would be complicated to deal
with the security implications of staking to only a particular validator on
that zone, because of a variety of complicating factors -- the chain's total
interchain security may change drastically depending on the activity status of
that validator.  The amount staked in this way would not be representative of
the "true" security that is being afforded via interchain staking -- for
example, imagine if $1M worth of PHOTONS were staked on the least validator on
BarChain, when the total market cap of the BAR staking token were only $100K.
The validators to whom the PHOTONs were interchain staked to need not
participate in any attack at all, so potentially nothing is "at stake".

Instead, you could stake to all the validators in proportion to their current
voting power, or in other words, you could stake on the zone itself.  I'll call
this "auto-staking". For example, if a validator that had 10% of the voting
power were to get slashed 30% on BarChain, everyone who interchain-staked
PHOTONs from the Cosmos Hub to BarChain would get slashed 3%.

Now let's discuss interchain staking the ATOM token. It wouldn't be fair to
bypass the Cosmos Hub ATOM inflation tax simply for staking those ATOMs on
another chain.  If that were the case, everyone rational would stake their ATOM
tokens on a fake zone that did nothing but the bare minimum, and would
therefore yeild higher returns because it would be easier to secure (by virtue
of it not doing anything).

You could try to design a system wherein there is some relative penalty or tax
for interchain staking as opposed to native staking.  I supposed the thing to
balance here is the relative amount of staking on the hub vs other zones, but
the more ATOMs there are, whether free or interchain staked, the more the Hub
is vulnerable to hostile takeovers.

The simple workaround is the following: let ATOM holders interchain-stake, but
require them to also double-stake on the Cosmos Hub itself via auto-staking.
Now, there is no "free lunch" to interchain stakers, so they can also earn
inflationary ATOMs.  Triple-staking would not be allowed -- it's not needed for
interchain staking, and it would make security guarantees worse.

Auto-staking may also help mitigate validator delegation centralization.
Auto-staking has less of a centralizing effect than staking only to the top
validators.  If we also tax auto-staking a bit, and use the proceeds to
subsidize a base-income for all validators, then the common reserve pool would
have more tokens to pay for equal validator subsidies, which I generally
support (to a limit).  Interchain staking makes a case to implement
auto-staking soon.

In this construction, it is the responsibility of the staked zone to reward the
ATOM or PHOTON holder who interchain staked the zone.  There could be many ways
to implement this, but a simple way would be to allocate some % of inflationary
rewards to interchain stakers from a particular zone, like the Cosmos Hub.
Market dynamics would ensure at least some participation.  The remaining
challenge would be to tune the %'s, which could initially be done by governance
experimentations, but later automatically using IBC price feeds.

## Exchanging

Assuming IBC and token peg transfers over IBC is implemented on chain A and
chain B, B tokens that are pegged and tranfered to chain A can be exchanged
with other tokens on chain A (native or pegged) assuming that chain A
implements an exchange module.  This direct exchange between two chains is
suffient for most use-cases and most token pair markets, and doesn't come with
the drawback of third-party (e.g. third-party blockchain) custodial risk.

A simple orderbook-based exchange module could be implemented on chain A, which
implements limit buy and sell orders without margin.  To accomodate tail
markets without sufficient active liquidity, a parallel module should be
implemented which implements a uniswap-like automated market making (AMM)
exchange module.  The combination of an orderbook-based exchange and an AMM on
top of IBC token pegging should be provided as a standard pair of modules that
should be available to most blockchains on the Cosmos Network.

Market making participants can smooth out the discrepancy in price between the
order-book system and the AMM system.  In a later, 2.0 release, perhaps the two
market systems can be integrated more tightly, but for 1.0 we should basically
implement two independent exchange modules.

### Simple Orderbook

TODO

### Simple AMM

TODO

### Standard API

TODO
