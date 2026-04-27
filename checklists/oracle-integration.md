# Oracle Integration Checklist

Oracles are where the big money bugs live. Every protocol that reads an external price is a target.

## Chainlink

**Staleness** — the #1 most common oracle bug in the wild:

- [ ] `updatedAt` checked against a max staleness threshold
- [ ] Threshold matches the actual feed's heartbeat (not a hardcoded guess — check [the docs](https://docs.chain.link/data-feeds))
- [ ] `answeredInRound >= roundId`
- [ ] `price > 0`

```solidity
// This is the wrong way (but you'll see it everywhere)
(, int256 price, , , ) = priceFeed.latestRoundData();

// This is the right way
(uint80 roundId, int256 price, , uint256 updatedAt, uint80 answeredInRound) =
    priceFeed.latestRoundData();
require(price > 0, "Invalid price");
require(block.timestamp - updatedAt <= MAX_STALENESS, "Stale price");
require(answeredInRound >= roundId, "Incomplete round");
```

**L2-specific** — people forget this one:

- [ ] Sequencer uptime feed checked *before* reading price
- [ ] Grace period after sequencer comes back up (prices are stale right after restart)

```solidity
(, int256 answer, uint256 startedAt, , ) = sequencerUptimeFeed.latestRoundData();
require(answer == 0, "Sequencer down");              // 0 = up
require(block.timestamp - startedAt > GRACE_PERIOD, "Grace period");
```

**Easy to miss:**

- [ ] Feed deprecation — Chainlink can turn off feeds. Is there a fallback?
- [ ] Multiple feeds with different heartbeats sharing one staleness constant

## TWAP (Uniswap V3, etc.)

- [ ] Window is long enough — 30min minimum, 1h+ preferred
- [ ] Pool has enough liquidity to make manipulation expensive
- [ ] Not using `slot0` (spot price) for anything financial
- [ ] Pool observation cardinality is set correctly

## Decimals

This is where the "boring" bugs are:

- [ ] Oracle returns 8 decimals, token has 18, protocol assumes they match
- [ ] Derived prices (A/B = A/USD / B/USD) — denominator can be zero
- [ ] Price pair direction — is it TOKEN/USD or USD/TOKEN?

## What oracle bugs enable

When you find one, check if it leads to:

1. **Undercollateralized borrowing** — attacker borrows more than collateral is worth
2. **Unfair liquidations** — stale price triggers liquidations that shouldn't happen
3. **Vault share inflation** — wrong price feeds into share calculation, distorting deposits/withdrawals

## Real incidents

| Protocol | Year | What happened | Damage |
|----------|------|---------------|--------|
| Mango Markets | 2022 | Oracle price manipulation in one tx | $115M |
| BonqDAO | 2023 | Custom oracle could be moved freely | $120M |
| Euler | 2023 | Donation attack exploiting price calc | $197M |
