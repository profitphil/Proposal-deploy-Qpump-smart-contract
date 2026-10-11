# Proposal: Deploy Qpump Smart Contract

Deploy the Qpump smart contract on Qubic — a new, non-custodial launchpad for community tokens. A creator launches a token for 25,000,000 QU instead of the 1,000,000,000 QU an asset normally costs; balances live inside the contract while the token trades along a fixed bonding curve. When the curve sells out, the contract pays QX the full issuance fee out of the money that token raised, issues the real asset, opens a Qswap pool, locks the liquidity forever, and delivers every holder automatically.

## Available Options

- **Option 0:** No, do not approve the Qpump deployment.
- **Option 1:** Yes, approve the Qpump deployment.

## Context

Qpump is a **new** smart contract — an initial deployment, not an upgrade. It does not exist on-chain yet and changes nothing about any existing contract. It calls QX and Qswap through their published interfaces only, and sends a share of its fees to the QX and QTREAT contracts.

## What The Yard Does

[The Yard](https://theyard.meme) is the launch platform built on Qpump, part of the QDoge Protocol. Anyone can launch a token and anyone can trade it, with no listing, no approval and no team allocation.

A token's life runs in five stages:

- **Create** — a ticker, a name and an image, for 25,000,000 QU.
- **Enter The Yard** — a 1,200-tick opening window (~5 minutes) where every buyer settles at the same price, so being first by a millisecond is worth nothing.
- **Train** — the price rises along a fixed line from 1 QU to 10 QU per token as people buy. Holders can sell back into the curve at any time.
- **Escape** — all 710,000,000 curve tokens are sold. The contract issues the real asset and seeds the pool itself.
- **Unleashed** — the token trades on Qswap and QX with no ceiling, its liquidity locked for good.

### Live today

The platform is fully built at [theyard.meme](https://theyard.meme) — explore, screener, charts, token pages, trading and the create flow are all ready, running on simulated data. The only piece missing is the Qpump smart contract: trading goes live once it is deployed.

### Where the fee comes from

Every launch, buy, sell and transfer settles through the contract, where the fee is collected. It is pooled and paid out once per epoch, so the flat costs of external calls are paid once rather than per trade. Real usage turns directly into dividends, QDOGE buybacks, and funding for the contract's own execution reserve.

## Summary of Features

### Launching

- Any identity can call `CreateCoin` with a ticker, a name and a metadata digest, attaching 25,000,000 QU.
- Nothing is issued at launch. Balances are internal records until the curve sells out, which is what makes a launch 40 times cheaper than an asset.
- A creator may hold **10 live tokens at once**; a slot frees when one escapes or is refunded, so there is no lifetime cap.
- Token metadata lives off-chain; only its SHA-256 digest is on-chain, so anyone can verify that a token page shows what its creator committed to.

### Trading

- `Buy`, `BuyWithQu` and `Sell` trade against a linear curve priced from 1 QU to 10 QU per token across 710,000,000 tokens, raising **3,905,000,000 QU** on a full curve.
- All curve maths is integer, computed in 128-bit, rounded up on buys and down on sells, so rounding never favours the trader over the contract.
- **No minimum trade, no minimum balance, no per-holder fee.** Any amount can be bought, sold or transferred.
- Every caller passes a slippage limit, and the opening window is capped at a quarter of the curve so no single order can take it.

### Fees

- Launch: **25,000,000 QU** — 20,000,000 to shareholders, 5,000,000 burned into the execution reserve.
- Trade: **1%** of each trade, or **1,000 QU**, whichever is greater, charged on top of the price and never skimmed out of the curve.
- Transfer: **100 QU** flat, burned.
- The trade fee splits **34% to the QX contract**, which pays it out to QX shareholders, **33% into a QDOGE buyback**, and **33% to the QTREAT contract's dividend fund**. All three destinations are compile-time constants, so no key can redirect them.
- The QX share is taken as the remainder, so rounding dust accrues to QX shareholders rather than being lost.
- Trade fees do not pay Qpump's own shareholders. Qpump shareholders are paid from the 20,000,000 QU launch-fee share.

### Dividends and Buyback

- Each epoch the contract transfers the QX share to the QX contract and the QTREAT share to the QTREAT contract, pays its launch-fee shareholder pot with `distributeDividends`, and burns its burn pot into the execution reserve.
- Neither transfer needs a cross-contract call: QX books an incoming `qpiTransfer` into its earned amount and QTREAT into its dividend fund, and each pays its own holders.
- The QDOGE share is spent in a **single Qswap purchase once per epoch**, capped at 2% of the pool's QU, so the flat swap fee is paid once. The QDOGE bought stays in the contract, which has no function to move it.
- Burns are not a cost to anyone: they fund the contract's own execution reserve, so Qubic is paid for running it without external top-ups.

### Graduation: No Help Required

When the last curve token sells, the contract runs the whole escape itself, resuming from the last finished step because nothing rolls back:

- **1,000,000,000 QU to QX**, which issues the real asset, 1,000,000,000 supply, this contract as issuer.
- **A Qswap pool is created** and funded at exactly 10 QU per token with ~2,644,000,000 QU and ~264,000,000 tokens — locked **forever**, because there is no withdraw path.
- **Every holder is delivered automatically**, a few per tick, funded from the raise. Nobody claims and nobody pays; `Claim` exists only as a backup if that budget runs out.
- **50,000,000 QU to the creator** and **10,000,000 QU donated to a fixed charity identity**, both from the raise.
- Every external spend is measured by the contract's own balance before and after, so a fee kept on a failed call, or a refund of an overpayment, is always accounted exactly.
- If graduation stalls for 2 epochs, the coin either returns to trading or holders are delivered and the unspent pool QU is returned pro-rata, so nothing stays locked.

### Inactivity, Not a Deadline

- A token is only ever removed for **inactivity: 9 epochs (about two months) with no trade**. There is no age limit, so a slow burn is never killed by a clock.
- Transfers do not count as activity, so a dead token cannot be kept alive with 100 QU pokes.
- On removal, holders are refunded **pro-rata by tokens held** and the ticker is freed.
- A **graduated token is never removed**. It keeps a permanent on-chain record and its ticker can never be launched again.

### Design: No Admin

- Qpump has no admin and no privileged role. There is no pause, no fee setter, no withdrawal and no upgrade path. Every parameter is a compile-time constant and every function is open to everyone on the same terms.
- `Process` is permissionless: anyone can settle an opening batch, retry a stalled graduation, push payouts or expire an idle token. The contract also does this itself every tick.
- There is no platform wallet. Fees only ever reach the QX contract, the QDOGE buyback, the QTREAT contract, Qpump shareholders, or the contract's own execution reserve.

### Interface

9 view functions (`GetCoin`, `GetHolder`, `QuoteBuy`, `QuoteSell`, `QuoteBudget`, `GetFees`, `ListCoins`, `GetStats`, `ListGraduated`) and 7 procedures (`CreateCoin`, `Buy`, `BuyWithQu`, `Sell`, `Transfer`, `Claim`, `Process`).

Every state change emits an event carrying the coin name and id, the actor, the QU and tokens moved, the coin's raised QU and tokens sold, the holder count and the trade fee, so an indexer can be built entirely on the event stream.

### State

238,502,104 bytes, about 24% of Qubic's 1 GB contract limit, sized up front so no migration is needed as usage grows: 32,768 live tokens, 65,536 permanent graduated records, 4,194,304 holder positions and 1,048,576 wallets. Each token's holders form a linked list, so a payout visits only that token's holders however large the tables grow.

### Tests

The full lifecycle — launch, opening batch settlement, curve buys and sells, transfers, a real graduation into a real Qswap pool, automatic holder delivery, epoch-end dividends, the QDOGE buyback, and inactivity refunds — was verified end-to-end on a live Qubic devnet driving the real contract with real transactions: **45 of 45 lifecycle assertions and 32 of 32 stress assertions passed, zero failures**, the stress run with 30 funded wallets trading 5 tokens at once. The graduation seeded a real pool with 2,644,898,805 QU and 264,489,880 tokens at 10 QU per token.

Every figure was checked against an [independent integer model](https://github.com/profitphil/qpump/blob/main/sim/qpump_model.py) rather than against the contract's own maths, and the contract passes Qubic's official `contract-verify` tool. A GoogleTest suite runs in the core harness, and full devnet logs are published in the [Qpump repository](https://github.com/profitphil/qpump).

## Technical Implementation

**Developer:** [double-k-3033](https://github.com/double-k-3033)

Contract source and testnet evidence: [https://github.com/profitphil/qpump](https://github.com/profitphil/qpump)

Core implementation PR: [https://github.com/qubic/core/pull/1010](https://github.com/qubic/core/pull/1010)
