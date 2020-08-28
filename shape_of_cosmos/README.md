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

- Shared Security
  - Interchain Staking
  - Replicated Security
  - [XXX]

- Exchanging
  - Simple Orderbook
  - Simple AMM
  - General API
  - DEX Zones

- Function of the Hub

- Pegging to PoW vs PoS

## Short Term Priorities

## Potential Pitfalls

- Peg Risk
  - vs immutable PoW vs PoS
- Reducing Surface Area
- Liquid Staking
- State Accumulation

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

### Replicated Security

When I say "shared security", I mean one of two very different things.  Maybe I
should stop saying "shared security" altogether.  Besides interchain staking,
like having a lifeline in a peer when one loses one's mind, there is also
replicated validator sets, which is more like having a leader and followers,
where the followers have no agency at all.  These follow-chains follow the
validator set changes as they happen on the single leader-chain.

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

## Short Term Priorities

Here are short term priorties beyond IBC.

### DEX modules

Already written about above.
[XXX start mailing list].

### Chain Upgrading

### Photons

### Peggy & Alternatives

### Interchain Staking

### Liquid Staking
 - Interchain accounts
 - Proportional slashing

### Contracts
 - Ethermint
 - CosmWasm
 - More

### Tendermint/Classic

### Merkle Store Improvements
