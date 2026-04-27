# How I Audit

Not a framework. Not a "comprehensive methodology." This is what I actually do when I sit down with a new codebase.

## Before touching code

Spend 30 minutes understanding what the protocol does in plain language. Read the docs. If there are no docs, that's already a finding.

Figure out: **where is the money, and who can touch it?**

Map the privileged roles (owner, admin, keeper, oracle updater). Check what they can do. Half the critical bugs I've seen come from "admin can do X, and X breaks invariant Y."

Read prior audit reports if they exist. Not to see what was found — to see what wasn't looked at.

## Architecture pass

Before hunting bugs, understand the system:

- Draw the contract dependency graph (even on paper)
- List every `external`/`public` function — that's your attack surface
- Identify external integrations: oracles, DEX routers, bridges, other protocols
- Check upgrade patterns: who owns the proxy? Is there a timelock?

## Hunting

I work through categories, not line-by-line. Reading sequentially is a trap — you'll spend 3 hours on the token contract and miss the oracle bug in the vault.

**Access Control** — Missing role checks, front-runnable initialization, renounceOwnership footguns.

**Reentrancy** — Not just the classic CEI violation. Cross-function, cross-contract, and the sneaky one: read-only reentrancy through view functions that return stale state mid-callback. This one still gets people.

**Oracle** — Stale prices, missing sequencer checks on L2, spot price used where TWAP should be, decimal mismatches between feeds. Oracles are where the big money bugs live.

**Token weirdness** — Fee-on-transfer breaking accounting, rebasing tokens breaking share math, ERC-777 callbacks. If the protocol says "works with any ERC-20," it probably doesn't.

**Economic logic** — Rounding direction (must favor the protocol, not the user), first-depositor inflation attacks on ERC4626, liquidation edge cases where positions become unliquidatable. The math bugs are the hardest to spot and the easiest to exploit.

**Flash loans** — Can governance be hijacked in one tx? Can oracle prices be moved? Can collateral ratios be temporarily inflated?

**DoS** — Unbounded loops, gas griefing, dust deposits that brick withdrawal functions.

## Writing the PoC

When something looks wrong:

1. Write the attack in words first. "Attacker does X, then Y, result is Z, protocol loses N."
2. If you can't write that sentence clearly, you probably don't have a real bug yet.
3. Build a Foundry test. Fork mainnet if needed.
4. If the test passes, write the fix and verify it blocks the attack.

## Report format

Every platform has its own template, but the structure is always:

- **Title**: one line, describes the impact (not the mechanism)
- **Root cause**: the specific code that's wrong, with line numbers
- **Impact**: who loses what, how much, under what conditions
- **PoC**: runnable test
- **Fix**: actual code, not "consider adding a check"

## Things I've learned the hard way

- Gas optimizations are not security findings. Stop submitting them as medium.
- "Admin can rug" is only valid if admin is supposed to be trustless. Read the trust model.
- The boring parts (deployment scripts, config, migrations) hide real bugs. Nobody audits them.
- If the protocol has been audited 3 times and you're looking at the same files everyone else looked at, go look at the parts they skipped.
- Write the PoC before the report. Half the bugs that "look right" fall apart when you try to actually exploit them.

## Tools

| Tool | What I use it for |
|------|-------------------|
| Foundry | Everything. Testing, fuzzing, fork testing, deployment verification |
| Slither | First pass. Catches the obvious stuff so I can focus on logic |
| Aderyn | Second opinion on static analysis |
| cast | Quick on-chain queries during review |
| Tenderly | When I need to trace a live transaction |
