# Accounting Invariants Checklist

Most DeFi hacks aren't flashy exploits. They're accounting bugs. A counter that's off by one field, a sum that double-counts, a state variable that drifts from reality. The protocol works fine for weeks, then someone withdraws and the math implodes.

## The Golden Rule

**`sum(all user balances) == global total` at all times, in all states, after all operations.**

If you can break this invariant, you have a bug. Everything below is a way to break it.

## Global vs Per-User State

When a protocol tracks both a global aggregate and per-user records, they can drift apart. See [004](../exploits/004-aggregate-vs-user-accounting/).

- [ ] Every function that modifies a user balance also modifies the global counter by the same amount
- [ ] Addition and subtraction happen in the **same function** (or same tx). If split across functions, verify the round-trip
- [ ] Epoch/cycle close operations restore everything that was subtracted at entry
- [ ] Interest/yield is added to **both** the user record and the global counter (not just one)
- [ ] Emergency/fallback withdrawal paths use the same accounting logic as normal paths

**Quick test**: after any state transition, does `sum(user[i].amount for all i) == globalTotal`? Write a fuzz test for this.

## Total vs Available Balance

Using the wrong balance field for solvency checks. See [002](../exploits/002-total-vs-available/).

- [ ] `total` vs `available` vs `free` — which one is used for transfer feasibility?
- [ ] If the system has holds/locks/reservations, are they subtracted before the "can I send this?" check?
- [ ] State is decremented **after** transfer succeeds, not before (or use checks-effects-interactions properly)
- [ ] Silent transfer failures: does the external call return `false` without reverting? Is that checked?

**Red flag**: any line that says `require(balance >= amount)` — immediately ask "which balance?"

## Mode-Dependent Semantics

External data sources that change what they return based on system configuration. See [005](../exploits/005-mode-dependent-double-count/).

- [ ] Does the protocol read from an external oracle/precompile/API?
- [ ] Does that source have **modes** (cross-margin vs portfolio, isolated vs shared, etc.)?
- [ ] When the mode changes, does the return value's **scope** change? (e.g., starts including collateral that was previously tracked separately)
- [ ] Is the protocol adding values from the oracle AND a separate loop over the same underlying data?
- [ ] Is there a mode-aware branch (`if (portfolioMode)`) that adjusts the calculation?

**Quick test**: flip every boolean config, rerun your accounting invariant checks.

## Cross-Function Consistency

Bugs that live in the gap between two functions that should be mirrors of each other.

- [ ] Deposit decrements counter X. Does the corresponding withdrawal increment X by the same logic?
- [ ] If there are multiple deposit types (regular, mid-epoch, emergency), do they all have matching withdrawal paths?
- [ ] Copy-pasted sibling paths: if path A does `+= withdrawn` and path B does `+= interest`, that's a bug

**Audit technique**: for every `-=` in the codebase, find its matching `+=`. If you can't, you found something.

## What accounting bugs enable

1. **Permanent fund lock** — underflow reverts block all withdrawals
2. **Phantom surplus** — inflated backing enables excess yield distribution, draining the vault
3. **Insolvency spiral** — last redeemers find nothing left (bank run)
4. **Griefing** — attacker can't steal, but can brick the protocol for everyone

## Patterns that indicate risk

| Signal | Why it matters |
|--------|---------------|
| Epoch/cycle-based architecture | State split across enter/exit operations |
| Multiple balance types (total, free, locked) | Wrong field in wrong context |
| External oracle with config modes | Semantic shift breaks assumptions |
| Parallel paths for different user types | One path correct, sibling path wrong |
| "Restore" or "reconcile" functions | Manual fixups suggest the system drifts |
