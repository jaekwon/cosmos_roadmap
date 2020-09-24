# The Shape of Cosmos

The default bias of decentralized development is for all parties to petition to
add more features to the root chain.  While encouraging experimentation and
permissionless innovation in various fronts such as those projects listed here,
we would also do well to promote hub minimalism, and to leverage the
affordances of IBC and shared security to improve the Cosmos ecosystem and
bring value to the ATOM token.  The resulting system would be more secure, and
the desired end result is not one hub but many that keep each other in check
according to a common protocol of interchain accountability.

The future shape of the interchain is a complex network graph; not a
heirarchical dag, not a winner-take-all single blockchain system; not even a
DAG.  It will be complex interwoven network with interzone IBC links created
(or not created) according to the cost-risk-benefit-analysis and regulatory
considerations at the edges.

The goal of this document is to convince the ecosystem to adopt this philosophy
by expounding upon it in relation to abstract theory as well as present day
innovations.


## Shared Security

The term "shared security" refers to a family of protocols that address
concerns like scaling, security, economic incentives, and usability.  To be
more specific, I'll describe several models of shared security.

### Simple Replicated Security

In "simple replicated security", there is a leader-chain that has a mutable
validator set, and one or more follow-chains that replicate the validator set
of the leader chain.

It's possible to do this without a bi-directional IBC connection, but merely a
one-way channel of state reads.  No "packets" need to get involved at all, but
there is the notion of a peer chain with validator set updates which are
submitted on the follow-chains. We could require that follow-chains follow
validator set updates frequently (but not necessarily all of them) say by
halting if necessary until the changes are applied.

NOTE: This use for a one-way IBC channel (and there are more use-cases too) is
compelling.  Here's an interface design question: What's the most elegant
implementation of bi-directional IBC communication channels that are composed
of two one-way channels?

This form of replicated security requires that all validators on the
leader-chain also validate all the follow-chains.  It's also possible to shard
(sharD, not sharE) the validator set, such that not all validators are required
on follow-chains.  So this would be "sharded replicated security".  If you
don't require the follow-chain validators to be selected from the set of
validators on the leader-chain, then you have "sharded (non-replicated)
security".  

In all cases, evidence of signing faults get recorded in the leader-chain,
consequences happen, and all the follow-chains follow suit.

You can also have interchain-staked replicated security.  A new hub can be
branched and interchain staked, and it can go on and offer replicated security,
sharded replicated security, or sharded non-replicated security to any number
of zones.

TODO: ETHERMINT

### Sunny's Opt-In Security

See: https://docs.google.com/document/d/1bXfqV7lQvWwcwJrmseffqvhpZmh8GxytsTksyqXthrc/edit

In Sunny's shared security model, there is one leader chain that determines the
validator set as in simple replicated security, but validators get to opt into
validating on other zones.  If a validator does something bad on one of these
zones being validated by the leader chain's validator, they get slashed on the
leader chain, the same way that they would with simple replicated security.

The benefit of this approach is that shared security becomes permissionless.  A
validator doesn't need the consent of other validators to begin validating for
any zone that wants to pay for validation services.

Zones and the validators that opt in to validate are registered on the leader
chain for accountability and to formalize open participation among the
validators.

This isn't much more difficult to implement than the simple replicated security
model -- mostly we just need an extra module on the leader chain to keep track
of opt-in participation.

### The Economics of Shared Security

Whether simple replicated security or something more fluid like Sunny's model,
the value of a system that provides secure transaction capability to many
blockchains is could be called "a cash cow".  If the Cosmos Hub leads in
rolling out these features first in a responsible way, the value proposition of
the ATOM token would be secured.  And if we do this while preserving openness
and multi-hub accountability, the crypto defi ecosystem would be secured as
well.

### Interchain Staking

__NOTE: This section on Interchain Staking appears first under "Shared Security"
because I believe it provides framing in terms of the greater ecosystem that we
are building for.  However, it isn't representative of what should be
prioritized first, which will be described later.__

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
committed at the same height, there are enough validators (more than 1/3) to
censor evidence from being submitted to the blockchain, so the previous
construction doesn't work.  *One way to prepare blockchain failure is to
delegate the handling of failure via a "consensus trial" on another blockchain
(the "consensus court" zone) that was previously self-elected by that
blockchain*.  Upon the discovery of Evidence that a chain has halted,
represented by two conflicting Commits, the consensus court and validators
enter into a challenge and response protocol to determine who is at fault for
the fork of that chain.  The protocol is described in the Tendermint wiki, but
is getting formalized and implemented.  (At least it will be on Tendermint
Classic, a fork of Tendermint that I'm working on to refine the original design
principles and to prototype improvements across the stack.  But more about that
later. [XXX])

The consensus court does not need to be too intelligent.  It could just be a
"consensus court yard".  It just needs to be a system that all observers can
observe, and provides some guarantees around liveness and the ability to keep
track of time.  In the very minimum, it just needs to serve as a communication
medium for sharing all evidence, like a shared network folder that faithfully
keeps track of timestamps.  All relevant evidence must be submitted within the
synchronous time window, or your validator may get slashed, or at least not
able to participate in the re-org rebooted chain until governance decides what
should be done.

A more intelligent consensus court could record the result of the "consensus
trial" in the Merkle store.  Most clients or smart contracts will prefer to
rely on a particular single recoverer -- they will just want to delegate the
responsibility of keeping track of consensus failures and consensus trials to a
blockchain that is dedicated to performing that role for many other chains.  I
think this should be a "branch" of the Cosmos Hub, but more on that later
[XXX].

The consensus court system could still fail. I suppose you could create a
linked list of recovery, but that may not make much of an improvement in
practice, compared to just defaulting to let governance deal with this rare
fault.  It is probably possible to make the single-recoverer system even safer
with multiple (non-intelligent, store-only) courts, assuming that at least one
of these recovery mediums can store all relevant information.  Arguably
multiple smart court (zones) could run a consensus protocol through IBC, but
this nested consensus approach in general seems like unnecessary
complexification.

For interchain staking to be supported generally, would require the handling of
evidence to be made from within the ABCI application.  Currently, the
TendermintCore implementation has an EvidenceReactor, which started off as a
copy of the mempool reactor for general transactions, but made separate in
order to provide some guarantees on liveness and availability of the slashing
data.  Here, I am advocating to remove Evidence related messages from ABCI, to
remove the evidence reactor, and implement all slashing logic from within the
ABCI application logic.  And later as a matter of secondary priority, to
improve the mempool reactor with features for prioritization of general []byte
transactions (and this is the kind of protocol prototyping that
Tendermint/Classic is for).

Once we have interchain BFT accountability, we can add additional features for
the Cosmos Hub, such as data availability challenges, block header challenges,
and so on.  More on that later.

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

*A key principle of staking is that there should generally be agency and
intelligence involved in deciding which validators to stake to or how to secure
the system* -- as Taleb says, there needs to be skin in the game, and it
shouldn't be outsourced.  This agency is exercised by delegating staking tokens
to your choice of validator.  So, here I assume a model where you somehow
receive some number of BarChain's native staking token in return for
interchain-staking for BarChain ATOMs on the hub.

#### Interchain Staking Inflationary ATOMs

There's one question here that pops up, for readers aware of the economics of
the Cosmos Hub: why would you "stake" ATOMs on BarChain if you aren't going to
earn inflationary rewards of ATOMs? 

It wouldn't be fair to bypass the Cosmos Hub ATOM inflation tax simply for
staking those ATOMs on another chain.  If that were the case, everyone rational
would stake their ATOM tokens on a fake zone that did nothing but the bare
minimum, and would therefore yeild higher returns because it would be easier to
secure (by virtue of it not doing anything).

Let's say that you could stake to all the validators in proportion to their
current voting power, or in other words, you could stake on the hub itself.
I'll call this "auto-staking". For example, if a validator that had 10% of the
voting power were to get slashed 30% on the Cosmos Hub, everyone who
auto-staked ATOMs on the Cosmos Hub would get slashed 3%.

So, one solution is to let ATOM holders x-stake, but require them to
also double-stake on the Cosmos Hub itself via auto-staking.  Now, there is no
"free lunch" to interchain stakers, so they can also earn inflationary ATOMs.
Triple-staking would not be allowed -- it's not needed for interchain staking,
and it would make security guarantees worse.

The down-side of auto-staking is that it's a form of "blind staking" -- a loss
of agency on behalf of the staker to the manual-stakers.  If there were 90% of
ATOMs that were auto-staked, then delegating manually brings 9x the extra
voting power with it, and so the system becomes overall unstable, and bribing
with side-channel paybacks would become the norm -- something we want to avoid.

Various forms of interchain-staking with ATOMs can work, but it invites
yield-seeking capital to hold atoms to stake on various zones, and eventually
defeats the purpose of having a separate staking token for the hub in the first
place.  The amount or ratio of ATOMs thus x-staked could be limited,
but this is still a slippery slope.

*In short, interchain-staking with ATOMs is not a good idea in general*.

#### Interchain Staking Non-inflationary Tokens

*Interchain-staking should be performed with non-inflationary tokens like PHOTONS
which don't suffer from the above problem/feature.*

It's possible to have significantly more value at stake with non-ATOM tokens at
stake via interchain-staking from the hub.  In short, the idea here is to
respect the sovereignty of a token according to well defined contracts
explicitly made beforehand that govern that token, such that the zones
connected to a hub can keep the hub accountable. [XXX more on this, and how
this is not the case for say, pegged ETH].

Any PHOTONs or native currency tokens that don't inflate
exponentially like the ATOM token by design can be inflated on the hub and
rewarded to ATOM stakers in the form of inflationary block rewards.  So there
is a clear link between the market of interchain-staking and value accrual to
the ATOM token, even if the ATOM token can not be x-staked.

#### Sovereignty Preserving Bonding Curves

So far we've established the following:

 * Dealing with major blockchain faults automatically requires interchain staking.
 * Interchain-staking with ATOMs or staking tokens is self defeating.
 * Interchain-staking with other tokens is sufficient.
 * Interchain-staking should involve agency and intelligence.

Here is another principle of interchain-staking:

 * *The amount of voting power or influence you receive on an interchain staked
zone should be sublinear to the total interchain staked amount*.

Otherwise, that zone has a "sovereignty issue", in that outsized (say) PHOTON
holders could fairly easily conduct a hostile takeover of the zone. In other
words, if I had 100 PHOTONs on the Cosmos Hub, I could x-stake them on
BarChain, but I shouldn't have the same influence or voting power as the last
person who x-staked 100 PHOTONs.  Otherwise, someone with 1 billion PHOTONs
could easily take over BarChain.

There are an infinite number of designs that can try to preserve this property,
but maybe the simplest construction is to have either a Bancor-like bonding
curve and continuous inflation model for the zone's staking tokens (e.g.
BarChain's BAR tokens), or an AMM/uniswap exchange with no continuous
inflation.  Let's consider both of them.

In the Bancor model, it's possible to limit the amount of BAR token inflation
by adjusting the reserve ratio parameter.  It's also a system that incentivizes
early participation, as the first BAR token minted via a Bancor protocol would
be the cheapest.  Setting the reserve ratio at 100% is like giving no
preference to the earlier BAR token minters.

In the Uniswap AMM model the amount of BAR tokens or pegged PHOTON tokens in
the liquidity pool is fixed or provided by existing holders, and there is no
inflation.

Also see https://yos.io/2018/11/10/bonding-curves/.

The AMM could be complemented by an orderbook exchange module -- see the
[[Exchanging]] section.

Some combination with a bit of parameterization should be suffiicent for most
new staking tokens.

#### Putting it together

Putting these things together, a new zones's staking token could be created via
an initial deposit to the new zone much like a blockchain fundraiser, then once
the fundraising is complete using a parameterized Bancor+AMM-like
inflation/exchange model to bring new stakers onto the system over time.

The x-staked PHOTON tokens on the Hub would be at risk of slashing if the
x-staked validator gets slashed on the new zone.  Evidence of consensus failure
is submitted to the Hub which starts a consensus trial process on the Hub to
determine how many PHOTONs should be slashed.  This is straightforward if the
exchange rate between PHOTONs and BAR staking tokens were fixed, but we've
determined that the rate can't be fixed.  If I x-stake 100 PHOTONs for 100 BAR
worth of voting power, but later the price of PHOTONs rise in comparison to
BAR, and I wanted to unbond my PHOTONs, should I get 100 PHOTONs back (
assuming no slashing and no rewards), or should I get less?

If I am to get all 100 PHOTONs back, that either means my PHOTONs have become
devalued while staked (as I only got 100 BARs for x-staking 100 PHOTONs), or
that the voting power of the validator I x-staked to changes depending on the
exchange rate.  We definitely don't want the latter, as it creates complex
distortions in relative ownership that depends on an exchange rate.

The simpler system is one where x-stakers assume that they will trade their
exposure for their x-staked tokens (say PHOTONs) for the staking tokens (say
BARs).  When I x-stake my PHOTONs to validator A to receive BAR staking tokens,
I am now exposed to the price of BAR tokens.  When I unbond my BAR tokens to
get back PHOTONs, I may get more or less depending on market conditions.

Arguably, *all new public zones should start with a reserve pool of x-staked
tokens for zone and ecosystem safety*.  I'm not aware of any other way to tell
without any prior information, whether a blockchain has any stake behind it or
not.  A lot of us have been on a gardening kick, so lets call this "branching".

### Addendum: On-Chain Staking Derivatives

(C note: not sure this goes in this section, but will write up the 
derivative idea we discussed).

One challenge for holders of ATOM in deciding to bond or not is their perception
of expected future asset flows (rewards from bonding and taking the risk of
exposure to slashing, as well as penalties of the bonded capital due to 
slash). Furthermore, for a participant to consider staking ATOM, they also
lose the liquidity and optionality that comes along with holding the ATOM
outright (i.e. being able to transact or allocate ATOM in a different way,
perhaps through a different validator).

A way to potentially address this challenge may be in permitting the 
tokenizing of staked ATOM. Specifically, given a validator, XYZ, a
given participant may generally wish to delegate ATOM to XYZ for XYZ 
to bond on their behalf because they believe XYZ is a good validator. However,
once that delegation is done on-chain, the participant that has delegated
their ATOM to XYZ loses the ability to transact with the ATOM, borrow
against their future expected asset flows, or contribute to price
discovery of said future asset flows. Suppose we create, on-chain,
the following conduit to tokenize the future asset flows of XYZ: issue
the VAL_XYZ token (the denominator is arbitrary, but suppose for now it 
is in the base unit of ATOM), where VAL_XYZ deterministically receives
100% of the staking rewards (from the bonded ATOM) that XYZ has contributed.

Ignoring implementation details for a moment, the benefit of VAL_XYZ to 
the ecosystem is clear. First, it reduces the loss of liquidity and optionality
that a given participant may have faced by choosing to stake their ATOM with XYZ.
Instead of having no assets to hold onto (and having to wait for XYZ to decide to pay
out staking rewards), the participant receives, in approximately real-time, the economic
upside immediately of staking rewards. Furthermore, the participant now has
the additional benefit of not only holding a claim to their delegated ATOM,
but also a *productive* asset that holds a claim to their delegated ATOM,
which means that they can also deposit their tokenized staked ATOM in another
system to generate additional productive yield. This effectively allows them
not only to borrow against their ATOM, which can be done through any collateralized
lending system that supports ATOM, but also borrow against their *future expected
asset flows* from staking with XYZ.

The settlement of VAL_XYZ is also simple -- by allowing creation and destruction
of VAL_XYZ token in direct 1:1 proportion of the underlying staked ATOM, it is
simple to see how if the market price of VAL_XYZ were to deviate substantially from
the actual expected future gains and losses of XYZ as a validator, an arbitrageur could
step in to keep the market in line. (Precisely speaking, if the VAL_XYZ token were to trade
too high, an arbitrageur could purchase ATOM at market, short the perpetual,
then mint additional VAL_XYZ by staking some ATOM with XYZ, and then sell it at market,
in which case if the arbitrageur is correct, the yield from the sold VAL_XYZ will
exceed the return from short ATOM perpetual and actual asset flows from VAL_XYZ.
Similarly, if VAL_XYZ is too cheap relative to the actual estimated future asset
flows of XYZ, the arbitrageur can purchase VAL_XYZ at market, and simply wait and
collect the asset flows that they are entitled to as share of XYZ's rewards.)

In the case that XYZ acts incorrectly or is penalized during the course of being
a validator (i.e. is slashed), the VAL_XYZ may lose perceived value (due to the
overall market thinking that XYZ may be a less trustworthy or competent validator),
but the initial bonded ATOM that was entrusted to XYZ is the only amount of capital
at risk (and not any of the proceeds from holding VAL_XYZ, unless the holder had
chosen to re-stake the emitted ATOM from holding VAL_XYZ with XYZ, thus increasing
the units of VAL_XYZ that the participant holds, and also increase the basis of 
their exposure to XYZ, which is, again, at their option to do so on-chain).

Implementation wise, one way of treating the issuance of VAL_XYZ is thinking of
a "wrapper pool" (non-custodial) that XYZ is itself a participant of, and in
some sense a partial controller of (not in a custodial sense, since the ability for 
XYZ to arbitrarily withdraw ATOM that a participant has entrusted to XYZ for the
purposes of validating is not desired) the VAL_XYZ "ATOM Pool," and VAL_XYZ is simply
a liquidity provider (LP) token that represents cryptographic proof of partial
rights to extracting basis capital (initially staked ATOM, less pro-rata slashing 
penalties) and resultant rewards (from XYZ performing as a skilled validator). 
This greatly simplifies the mechanics of VAL_XYZ, as then no "push"-based issuance
mechanism is needed for distributing the asset flows. (One downside of this style of
implementation, however, is that it requires actions on the part of the holder of
VAL_XYZ to extract ATOM from the pool periodicially lest there is a flaw in the
implementation of the VAL_XYZ pool, not unlike the periodic responsible
"harvesting" of yields that rewards-based pools often encourage.)

One ecosystem benefit for introducing and popularizing the practice of tokenizing the
asset flows from staked ATOM is that across the top 100 or so validators, the implicit
yields (of staking ATOM with one of those validators) can be taken as a COSMOS reference
rate for opportunity cost of ATOM, which itself, as an on-chain primitive building block,
greatly reduces the friction for architecting additional financial instruments, as the
effective "validator yield rate" (VYR) can become a standard for pricing in the 
opportunity cost of ATOM. It is easy to imagine e.g. centralized exchanges also offering
futures or swaps on VYR, as well as VYR being composed with other instruments to produce
a healthier, more transparent ecosystem for COSMOS.

(C note: can link to my chain-agnostic CYBOR idea write-up when appropriate -- 
crypto yield baseline offered rate, that is validator yield agnostic.)


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

In the simplest construction, an orderbook exchange just needs to implement
1 transaction type: the "limit order".  A limit order roughly has the following
format:

```go
type LimitOrder struct {
    Time            time.Time
    Market          string // e.g. "USDT/ATOM:AMM" or "PETH/ATOM:ORDERBOOK"
    Type            OrderType // buy or sell
    Status          OrderStatus
    Account         crypto.Address
    Target          sdk.Coin
    TargetFilled    sdk.Coin
    Base            sdk.Coin
    BaseFilled      sdk.Coin
}
```

Instad of market orders, we only allow limit orders.  In order to prevent
front-running and re-ordering of transactions by the proposer, the transactions
of a block get batched into a batch transaction at the end of every block.
While it is possible to simplify this even further and not have batching at
all, we believe the tradeoff is worthwhile to aim for per-block batch order
execution for the initial iteration of the simple orderbook DEX module.

List of known projects for simple orderbook implementation.

* ...

### Simple AMM

Order books are a necessity for institutional traders, but not all coins
will have an active market with market makers.  For this reason, the
complementary half of the simple orderbook exchange module is the simple
automated market maker module -- think configurable Uniswap.

See B-Harvest's proposed AMM module for more information.

* https://github.com/b-harvest/Liquidity-Module-For-the-Hub

### General API for Pluggable Exchanges

It might be possible to have a common minimal interface and market state and
order transaction data.  Perhaps the same LimitOrder structure can be used for
all kinds of markets, allowing the same client to interact with any kind of
pluggable exchange module (where the simple orderbook and AMM modules come
standard).

```go
// See https://github.com/jaekwon/ftnox-backend/blob/master/exchange/exchange.go
// for inspiration.
// TODO: merge w/ other DEX models.
type Market interface {
    Type() MarketType
    AddOrder(order LimitOrder) error
	Deposit(sdk.Coins) error
	Withdraw(sdk.Coins) (sdk.Coins, error)
}

type OrderbookMarketState struct {
    Target       sdk.Coin // denom and total in pool
    Base         sdk.Coin // denom and total in pool
    Shares       sdk.Coin // denom and total shares issued
    Bids         sdk.Tree // [XXX object store]
    Asks         sdk.Tree // [XXX object store]
    HasMoreBids  bool
    HasMoreAsks  bool
}

type AMMMarketState struct {
	[XXX todo]
}
```

### DEX Zones

Any two chains can connect directly exchange tokens without going through a DEX
zone, thereby eliminating any external third party risk.

The disadvantage to direct interchain exchanging is that it's potentially
expensive -- the IBC connection and trade transactions may be prohibitively
expensive on some chains where the transaction fee is high due to high volume.
For this reason (and more, TODO) there will also be DEX zones that specialize
in exchanging.  The disadvantage of trading in a DEX zones is that it incurs
additional risk (namely the failure of the DEX zone and its dependencies).

## Function of the Hub and ATOM Token

If there is to be many hubs and many staking tokens, what is the purpose of the
Cosmos Hub?

The Cosmos Hub should be the standard interchain staking hub. It becomes the
standard by doing one thing really, really well.  Everything that is 

### Economics of Replicated Security and Sharded Security

I was there when Amazon EC2 started, and grew from nothing to the geological
phenomena that it is today.  The same will happen in the public blockchain
sector, and the pendulum will once again swing from time-shared mainframes
(EC2) to decentralized personal computers (sovereign zones).

I don't know to what degree we will have homogenous distributed computing --
e.g. anyone can become a validator and making meaningful contributions.  What I
do know is that economy of scale applies to the industry of validating, and
those who invest in their local datacenters with fault-tolerant peer
incentivized and people owned internet connectivity, those validators will have
the most stake in processing people's transactions.

The infrastructure of secure internet communication, computation, and
validation should all be owned by their respective communities, if they want,
and many will and rightfully should.

Like Amazon EC2 was able to dominate by innovating early and doing the job
well, the same will be true of the validation-as-a-service industry, and Cosmos
is first and will do it better than anyone, for as long as we don't lose focus,
or make critical mistakes that undermine trust in our competence as observed by
the users.

There's an infinite universe of possible VaaS models that we could explore, and
the exact details don't matter so much as this: if we make it easy to branch
and maintain functional zones, and to interchain-stake them and thus to support
them in cases of failure, such that the stake behind all zones could be
quantified, then we have an infinitely scalable, decentralized token network.

In short, ATOMs and whatever PHOTONs it decides on, fundamentally derive their
value from the following:

 * Interchain staking, aka "consensus court"
 * PHOTON inflations
 * Transaction fees across replicated zones

The transaction fees alone in a massively easily scalable blockchain network
that solves interchain staking and is producing the next generation token
ecosystem.  ATOM and PHOTONs don't need to be and should not be the "world
dominant" staking and currency tokens for the tide to left all of us -- if
anything, ensuring a rich and well connected ecosystem of independent sovereign
competing and cooperative zones is what we need to prevent the untimely
resurgence of Moloch.

### Function of the Hub

The ultimate responsibility of the hub is to secure the ecosystem.  The
ecosystem is supported by a handful of primitives, such as interchain-token
pegging, and staking, running the DPoS algorithm for the ATOM token, running
the governance function (and soon peer communication could be on-chain too),
and providing the IBC and basic DEX modules as described here.

Experimental features should first be tested on a zone that branches off of the
hub, or on a replicated follow-chain that follows the hub, rather than be added
to the hub's leader-chain itself.  If it isn't criticial for the ultimate
responsibility of the hub, and it isn't making a sigificant improvement to the
performance of the hub, then it doesn't belong in the hub, if anything it is
detrimental to the hub, as it adds more complexity and surface area for bugs
and exploits.

The software running on the hub, whether leader-chain or follow-chains, needs
to be of utmost quality.  It's likely that initially, validators in the Cosmos
ecosystem will rely on existing virtualization technology to run the many zones
that they validate or peer on, and they all seem to be in a state of
development with the occasional critical bug.  Cosmos Hub (and any hub) should
not approve of new software to be run under replicated security without first
auditing and understanding what it does -- and this takes time.

New financial and technical experiments should best be rolled out slowly,
rather than all out, to protect the users instead of inadvertantly or
maliciously channeling their tokens to the hands of exploiters.

The hub via its supporters of the ecosystem should incentivize competition
among different designs for the critical components that it needs.  We're not
going to figure out the final design of DEXes, IBC, consensus, native and
interchain staking, governance, or anything else, in the first try.  There will
be failures, sometimes massive, theft, and even sabotage and theft from within
the ecosystem (the same way it happened to Bitcoin and Ethereum).

The ultimate principle for a hub is safety.  Safety over liveness, safety over
features.  Provably quantifiable security.  It proves its worth over time by
not making big existential mistakes.  It doesn't lose its optionality, and it
isn't swayed by contributors eager to contribute for recognition, or
competition with different shiny features.  It seeds competition around it for
new innovations, and takes a stake in it.

## Chain Upgrading

It's true that the hub will need to occasionally upgrade to support different
IBC implementations or plugins.  This is the most compelling reason for
frequent upgrades to a conservative hub.  Even Peggy is being proposed as a
module to be implemented directly on the hub.

Given the arguments made in this paper, and the following choices for chain
upgrading, I argue that the hub's lead-chain should not adopt WASM at this
time, and instead propose an alternative roadmap.

There are two generally good ways of supporting chain upgrades that I am aware
of, and there may be more, that don't require WASM.  One is to consider the app
to be a separate container or process, and to create an ABCI middleware-app
that switches between applications appropriately, as directed by ABCI.  This
can pretty easily be done via network or Unix sockets, but it can also be done
via Go plugins.  I'm not advocating for the whole application to implement a Go
plugin interface, but rather for the IBC module to allow the expansion of
functionality by compiling a Go dynamic plugin.

NOTE: The main limitation of Go plugins at this time is that unloading them
is not supported, so this doesn't work for general smart contracts, but it's
perfect for uploading new IBC components.

The first method is a specification and formalization of ABCI and application
versions that all blockchains will occasionally need.  The second method is
complementary, and allows for upgrading within the same container or process,
without any application version upgrades.  These processes or containers could
also be executed in the form of a WASM image, so we have several valid options
for process isolation and upgrading as conducted by ABCI middleware.

The second method using Go plugins is basically smart contracts for
long-running smart contracts, written in Go. It just needs a bit of AST
introspection and sanitizing (the same stuff that powers gofmt), and can be
implemented easily.  You could actually build a Go-based smart contracting
system with just AST validation/manipulation and the occasional Go plugin
refresh, if the creation of new smart contracts is sufficiently throttled.

One of my goals for the end of 2020 for Tendermint/Classic is to demonstrate
both upgrading mechanisms.  From there it should be easy to integrate those
protocols or similar ones to TendermintCore and the CosmosSDK.

Moving on, to smart contracts.

## Smart Contracts

While I support WASM, and projects like CosmWASM, I am strongly opposed to
running WASM on the Cosmos Hub's lead-chain.  Primarily, because it increases
the surface area for attack by an order of magnitude.  WASM as a spec and its
implementations are still maturing, and though available on browsers, it hasn't
gone through the gauntlet of time.  All new complex technologies like WASM,
like Java, Linux, and even Go, in hindsight have numerous bugs that could have
or were used maliciously.  The same will be true of any WASM integration with
the hub, and this potential for exploits combined with the massive potential
rewards (especially of pegged PoW tokens) makes such exploits an inevitability.

Given the history of the evolution of these kinds of technologies, and the
principle of hub minimalism and conservatism, I feel that I must voice my
strong opposition to mandating the adoption of WASM for the lead-chain, even in
the form of governance-based permissioned code injection.

What WASM is good at is native speed execution with memory bounds and sandbox
isolation. By virtue of the problem that it is trying to solve, the primitives
are more like machine instructions than a language abstract syntax tree.  Both
Ethereum and Polkadot have taken this low-level machine approach to smart
contract development, but I believe the opportunity will be in offering
something closer to procedural programming as in Javascript, rather than the
specifics of the machine environment, of which there are many choices.

It is Solidity that ultimately made the EVM easy to use, though it's not
without its drawbacks.  It is OCaml that lets Tezos create formal proofs on
functions.  And in the context of massive multiuser smart contracts, native
speed of execution doesn't matter as much as Merkle store latency, so you might
as well run an AST interpreter.

### Notes on CosmWASM

WASM itself is a near-machine-level virtual machine that is designed for
restricted memory allocation and isolation.  What it doesn't do is prescribe
much more about how the programming language works at a higher level, because
it's designed to accommodate any language.  So there is still the work that
must be done to make this WASM machine accessible to the programmer.  CosmWASM
provides this glue initially making it possible to program in Rust, but other
languages with support for the WASM build target can be used eventually.

Here are what I see are the primary design philosophies of CosmWASM (as would
be differentiated from alternative smart contracting system designs in our
emerging ecosystem):

 * Based on the Actor Model
 * Atomic Execution
 * Dynamically Linking Host Modules

I like the actor model.  It makes it easier to model, as the idea is that all
state is local to the actor, and all state changes happen via a sequence of
simple data messages.  The design of Tendermint itself is based on a modified
actor model where peer connections have multiplexed channels (e.g. fairness)
which helps with liveness in a Byzantine setting.  An ABCI app itself is a kind
of actor, and here I discussed ABCI middleware for upgrading, and in the future
there will be more discussions around extending ABCI for dealing with
concurrent ABCI sub-applications.

I'm in favor of both the actor model, and would like to see this built in Go,
built on top of primitives proposed in the CosmosSDK.  These are things like
the "SubKVStore" decorator for stores which allow you to provide access to only
part of a store, which allow for the construction of Actors in the likeness of
Keepers that each have access to their local substore.  In short, we could
build an actor model system on the SDK without WASM, if we wanted to.

A good use-case for this would be for say, parallelizing an ABCI application.
Each parallel instance may best be modeled as an Actor; though there will also
be needs for alternatives, such as something based on a versioned/shared-memory
model which may help with performance in some applications.

I don't agree with CosmWASM's justification for promoting this model in terms
of security.

> Secondly, each Actor can effectively run on its own thread, with its own queue.
> This both enables concurrency (which we don't make use of in CosmWasm... yet),
> and serialized execution within each actor (which we do rely upon). This means
> that it is impossible for the Handle method above to be executed in the middle
> of a previously executed Handle call. Handle is a synchronous call and returns
> before the Actor can process the next message. *This feature is what protects us
> from reentrancy by design*. - https://docs.cosmwasm.com/architecture/actor.html

I think what is being implied here is that programming the traditional way with
type-checked languages like Go or Solidity isn't good because of potential
security issues, and that CosmWASM/WASM can solve this by virtue of supporting
the Actor model.  Here, I want to show how to solve this in Go by doubling down
on the original premise that this is a good language for building secure
applications out of.  If you've read the technical discussions about Ethereum's
DAO hack, the analysis doesn't recommend the following type of remedy, which
was maybe not available to Solidity at the time (and maybe still doesn't, I
don't know) but is afforded by Go. 

```go
func (_ BankKeeper) Withdraw(ctx sdk.Context, addr sdk.Address, amount sdk.Coins) *HotCoins {
    ... 
}

func (_ BankKeeper) Deposit(ctx sdk.Context, addr sdk.Address, amount *HotCoins) {
    ...
}

...
hotcoins := bk.Withdraw(ctx, addr1, amount)
bk.Deposit(ctx, addr2, hotcoins)
...
```

The idea is that HotCoins is something that only be constructed by withdrawing,
and so for any receipt of tokens to happen, it must be withdrawn from an
account that has a positive balance.  The logic of the bank keeper and HotCoins
and the state of Context and HotCoins would ensure that everything is accounted
for, even that all withdrawn coins are deposited somewhere.  The BankKeeper
could check this invariance for all transactions at the end of every
transaction.

This works because the Go language is Object Oriented.  Deposit requires a kind
of object that can only be created by Withdrawing, and thereby solving the
re-entrancy problem.  You don't need to sacrifice recursion; you can write it
the way that TheDAO did, but you just need a proper OOP langauge to express it.
The Actor model works really well when you're dealing with many independent
threads of execution, such as in a telephone network, for which Erlang is
designed.  But when you use it in leu of (potentially recursive) function calls,
you're forced to either deal with the complexity of asynchrony (e.g. persist state
and deal with either success or failure of an IBC call later), or you have to
abort the entire transaction somehow.  The CosmosSDK does this very well,
by letting the programmer simply panic, leveraging the native feature of Go.
How would it work in an Actor-based system?

> Before executing a Msg that came from an external transaction, we create a
> SavePoint of the global data store, and pass in a subset to the first contract.
> We then execute all returned messages inside the same sub-transaction. If all
> messages succeed, then we can commit the sub-transaction. If any fails (or we
> run out of gas), we abort execution and rollback the state to before the first
> contract was executed.
> 
> This allows us to optimistically update code, relying on rollback for error
> handling. For example if an exchange matches a trade between two "ERC20"
> tokens, it can make the offer as fulfilled and return two messages to move
> token A to the buyer and token B to the seller. (ERC20 tokens use a concept of
> allowance, so the owner "allows" the exchange to move up to X tokens from their
> account). When executing the returned messages, it turns out the the buyer
> doesn't have sufficient token B (or provided an insufficient allowance). This
> message will fail, causing the entire sequence to be reverted. Transaction
> failed, the offer was not marked as fulfilled, and no tokens changed hands.
> - https://docs.cosmwasm.com/architecture/actor.html

But this kind of rollback model goes against the point of adopting the Actor
model, for parallelism and concurrency.  You want actors to be independent and
to run at full capacity, not having to roll back here because of an error
there.  Another way to say this is that Go already got concurrency right, when
it made the primitives of cheap Go-routines and channels.  An Actor can be
represented by a Go-routine, and there it's clear to see that you won't get
parallelism by rolling back many go-routine states upon a local error.

I've shown how it would work here with "keepers" which is a Cosmos SDK
construct, which are stateless objects with methods that take stateful context.
Imagine if you didn't have to use these abstractions for keeping persistent
state, but rather persistence was as simple as using a map, so you don't have
to pass the context around.  We can enable this with AST sanitization and
transformation, or we can adapt a Go interpreter or create our own.  (I'm
slightly leaning toward doing the latter.)

Given that in a massively multi-user smart contracting system, the persisted
state is so much greater than what can be held in memory, and assuming that
most smart contracts only run occasionally, the traditional garbage collection
debate tips in favor of garbage collection for blockchain smart contract
programming.  So while any implementation can eventually get optimized in Rust,
as GoEthereum was later dominated by Parity, I bet that smart contract
innovation and initial implementation should first happen in other languages
like Solidity, its descendants, or else be derived from modern langauges like
Go.

It's not that I beieve that Go is the perfect language, but rather that it's a
fairly good base.  The ideal language might have some additional conventions
(perhaps in the form of comments) that let us write assertions, invariants, or
other statements on a piece of code.  This is building upon an intuition or
premise that I hold, which is that we want our languages to have procedural
statements, logic and data encapsulation, concurrency (as actors or
goroutines), and other tags or decorators; and that from upon some minimal
language that supports these features, we should build upwards and outwards the
intelligence in code to analyze them, fuzz them, prove them, simulate them, and
more; and that the elements of this system as represented by the AST should not
just be Turing complete, but also intuitive, such that the formal proofs about
our code can be likewise intuitive.  You don't get intuition by going through
machine-level instructions, wasm, the EVM, any more than by converting proofs
to set-theory.

I dream of a sufficiently good language with a secure implementation, improving
upon something like Go, and writing intuitive and simple but competitively
performant blockchain/consesnsus software as well as smart contracts, all in
this one minimal universal procedural language, such that the whole stack from
the consensus engine to the smart contract, such that all the layers can
benefit from all the tooling built in a multiplicative way.  That's why I'm
working on Tendermint/Classic and committing to Go and Go-Amino-X.  The
objective is to reach [zero
Kelvin](https://urbit.org/blog/toward-a-frozen-operating-system/) with a
minimal common language.  It's the defense we need against institutional
subversion.

## Photons
TODO

## Token-Pegging to PoW

Much of this section is a response to a proposal in the works regarding Althea
Peggy.

> Due to this consideration it is a requirement that this version of Peggy is
> run on a Cosmos zone where the total value of the staking token is greater
> than the total value locked in the Peggy Ethereum bridge at all times.
>
> *This makes the Cosmos Hub the ideal zone to deploy Peggy*.
> - https://github.com/UniFiDAO/unifidao-proposals/blob/master/UP-101.md

In this section, I argue that we should not peg to PoW chains on the hub.

Two-way-pegging is when you lock up tokens on one chain, materialize these
tokens in another chain such that they can be used there, with the option to
withdraw back on the original chain in the future.

*Pegging to PoW is just as dangerous as centralized custody of Bitcoins*.  You
can't remove the risk that the custodians of the PoW tokens won't just steal
the underlying collateral.  This is for a variety of reasons, but the most
salient one is that PoW chains generally aim to be "immutable" and
"governless".  Bitcoin and Ethereum have both forked in the past, but both
strongly prefer to stay immutable and governless.

Between two Cosmos well-governed BFT-PoS chains connected by IBC, it's possible
for the two chains to come to agreement about the protocols in case of zone or
hub failure.  But there is probably no negotiating with Bitcoin or Ethereum if
a Bitcoin address or Ethereum smart contract loses its tokens due to malicious
theft or software bugs.

Furthermore, with pseudonymous or anonymous key holders of a peg to a PoW
chain, if what is at stake is less in value than what is locked in the peg, the
expected outcome should be nothing other than theft.  Don't expect an anonymous
stranger to keep your ETH safe if they have nothing to lose and something to
gain by absconding with it.  This picture changes with real-world identity and
reputation working in legal jurisdictions with cross-jurisdictional cooperation
and the rule of law and the threat of guns, but it isn't always clear how to
value such things, and they often fail anyways. In short, in general, *pegs to
PoW chains or any governless chains require overcollateralization*.

*Consensus-based peg collateralization requires the safe handling of +1/3
Byzantine faults, which means that some form of interchain staking is
necessary*.  It isn't sufficient to slap a peg to Ethereum on the Cosmos Hub
and require that all ATOM takers also stake on the Ethereum peg, because
nothing is keeping the hub accountable in the case of +1/3 failure.  In theory,
the entire ecosystem (including centralized exchanges and users) could keep the
ATOM stakers accountable even in the case of +2/3 failure -- by forking the hub
-- but there's no guarantee that such coordination will happen.

NOTE: In previous sections, I argued that x-staking ATOMs is not a good idea.
Arguably, a portion of ATOMs could be staked on the hub only for pegging, and
the hub could make an exception for pegging to Bitcoin and Ethereum; but the
fungibility between ETH pegging and Hub staking makes both more susceptible to
a hostile takover, reduces the legitimacy of the self-sovereignty of the hub,
and introduces a requirement to try to balance incentivization between hub
staking and peg staking, with may fail.  

This isn't to say that IBC pegs to BFT chains can't fail. *Pegs to
well-governed PoS chains still require controls to prevent catastrophic
failure*.  For example, IBC token pegging could be made more safe by throttling
the amount of tokens that can be (peg) transferred via IBC, to ensure that even
in the case of zone failures due to bugs, hacks, malice, etc, massive amounts
of tokens can't be instantly transferred and dispersed, or smurfed.  Maybe not
for IBCv1, but we should have some options with IBCv2.

Pegging to PoW is like operating a gold depository.  You can have a centralized
depository (one owned by a single owner) or a depository with decentralized
ownership (one owned by shareholders), but either way the gold may disappear.
Overcollateralization helps, but isn't perfect, as the exchange rate may
fluctuate wildly.  Another disincentive against theft (or negligence or
incompetence) is the rule of law, as enforced by various jurisdictions.  The
most secure pegging system would leverage both on-chain collateralization as
well as the law of various legal jurisdictions.  Since each jurisdiction is
different and people want to derisk for the potential for centralized failure,
*the logical steady state solution to PoW pegging to Bitcoin or Ethereum
involves many independent pegs, and a way to combine these tokens on the Cosmos
side e.g.  by creating a basket token*.  This gives the ecosystem the ability
to innovate peg designs permissionlessly, and gives the users the ability to
pick the solutions that work for them.

Regarding pegging to PoW chains in general, we still have a lot of questions
left, and some we can't even answer ourselves.

* What should be the procedure for recovering peg-locked ERC20 tokens if +1/3
  of the voting power disappears forever? (This also implies a chain halt). I
think a form of time-based annealing would be best, so that the funds can
eventually be recovered.
* In case of consensus failure among peg signers (disagreement where none get
  +2/3 votes), should one party receive all of the tokens or should the tokens
be split among the forks?
* How do we deal with drifting exchange rates to ensure overcollateralization?
* What are the incentive economics of pegging?  It seems that peg token
  providers should earn something (with the possible risk of damages), while
pegged-token users should pay something over time.
* What sort of legal agreement would best represent the responsibilities and
  liabilities for peg signers/custodians/validators?
* If +2/3 can move pegged tokens, only +2/3 of peg stakers may get slashed,
  which means naive BFT-based pegs require 33% extra collateralization.  Should
non-BFT (e.g. single-custodian) pegs also be supported?
* What is the best way (or what are the ways) to integrate pegging with
  regulatory frameworks and legal jurisdictions?

Given all of the above, we should test various Peggy implementations on a zone,
x-staked from the hub, and disclaim its risks.  Recently there was a proposal
in the making to put the Althea peggy logic on the hub.  One of the stated
reasons for this was to provide maximal security for the peg, but as explained,
we shouldn't do this.  Not just because of the reasons made prior, but also
because *maximizing security via large ATOM staking participation sends the
wrong message regarding the security of the peg*. Software takes time to
mature, and that large amounts of money should not be put at risk in the
beginning.  Also, we should be careful about signaling and the risk of failure
and damage to the reputation of the ecosystem.  If a zone on Cosmos fails,
that's one thing... if an official peg of Cosmos fails, it could cause
existential harm to the entire Cosmos ecosystem.  And with pegs to PoW, we
won't be able to roll back like Ethereum did with the DAO hack.

The risk of subversion (internal and external) or opportunistic malice is too
great when we combine the radically increased surface area introduced by
CosmWASM and WASM and Peggy on the Hub.  Not only would we be better off by
enriching the ecosystem with new zones that prove these new technologies
outside of the hub; these technologies are also not mature enough to guarantee
any security on the scale of a billion dollars.  Should the hub evolve in a way
such that both of these new technolgies (as well as numerous updates to
Tendermint and the SDK) get compiled into the hub any time soon, the incentive
for failure is so great, that massive failure is inevitable. It would be
unethical for me to support such a system, and I would feel compelled to fork
it or to try to halt it.

TODO : Althea's Peggy should be called Althea Peggy, and we should incentivize
for more Peggy experiments.

## Liquid Staking
TODO

 * Interchain accounts
 * Proportional slashing

## Proposal X

The proposal is pretty simple:

 * Keep Cosmos-Hub minimal and type-checked, no wasm or peggy.
 * Run Peggy and CosmWASM as independent sovereign zones "branched" from the
   hub, with their own staking tokens.

The following are implied:

 * Implement PHOTONs.
 * Implement interchain-staking and replicated security and replicated sharded,
   and offer validation as a service for many zones.

Finally, should this proposal fail, as a last resort, hard-spoon a new minimal
zone with tokens hard-spooned to aligned voters, and exit the old chain.

## Proposal Y

Immediate Critical Roadmap.

 * IBC followed by Sunny's model.
 * Photons.

Also:

 * Interchain staking.
 * CosmWASM and Peggy as new zones.
 * More competition and experimentation.
