# Trace-Break

Breaking things so you don't have to.

Exploit PoCs, audit methodology, and the bugs your auditor missed.

## What's here

**[`exploits/`](exploits/)** — Vulnerability patterns stripped to minimal Solidity + Foundry tests. Each one has a vulnerable contract, a fixed version, and a test that proves the exploit. No fluff.

| # | Pattern | Category |
|---|---------|----------|
| [001](exploits/001-stale-oracle/) | Stale Oracle Price | Oracle |
| [002](exploits/002-total-vs-available/) | Total vs Available Balance | Accounting |
| [003](exploits/003-same-asset-liquidation/) | Same-Asset Liquidation Blocked | Lending |

**[`methodology/`](methodology/)** — How I actually audit. Not a textbook process — a workflow that came from screwing up and learning what matters.

**[`checklists/`](checklists/)** — Per-category security checklists. Open during review, check things off, don't rely on memory.

| Checklist | |
|-----------|---|
| [Oracle Integration](checklists/oracle-integration.md) | Chainlink, TWAP, custom oracles |

## Run a PoC

```bash
git clone https://github.com/Kematens/Trace-break.git
cd Trace-break/exploits/001-stale-oracle
forge install
forge test -vvv
```

## Contributing

PRs welcome. Each exploit needs:
1. Minimal vulnerable contract (not a 2000-line fork)
2. Fixed version
3. Foundry test that breaks the vulnerable one

## License

MIT
