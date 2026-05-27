## Competitive Audit Archive

This directory contains my complete competitive smart contract audit history across Sherlock, Code4rena, and Cantina.

Each markdown file corresponds to a single audit contest and contains every submission made during that engagement.

This archive is intentionally comprehensive and includes both validated and non-validated findings to accurately represent research process, depth, and iteration.

## Classification Legend

Each finding is tagged using the following labels:

| Label | Meaning |
|------|---------|
| `H` | Validated High Severity Finding |
| `M` | Validated Medium Severity Finding |
| `L` | Validated Low Severity Finding |
| `S` | submitted Finding |

These are being progressively standardized.

## What This Archive Demonstrates

This archive reflects practical smart contract security research across production-grade DeFi systems audited through competitive platforms.

### Protocol Experience
Security review experience across:

- lending and borrowing markets
- stablecoin systems
- AMMs / DEX infrastructure
- staking and reward distribution protocols
- ERC-4626 vault architectures
- cross-chain and async settlement systems
- payment abstraction and liquidity protocols

### Security Review Focus
Common analysis areas include:

- accounting and fund mis-accounting issues
- precision / rounding vulnerabilities
- insolvency and liquidation edge cases
- oracle dependency risks
- invariant violations and state inconsistencies
- access control and upgradeability weaknesses
- denial-of-service vectors
- race conditions and execution ordering flaws
- protocol economic attack surfaces

## Validated Findings Index

| Contest Name | H | M | L | Submission Archive |
|-------------|---|---|---|-------------------|
| USG-Tangent | 0 | 1 | 0 | [View Report](./2025-08-usg-tangent-submissions.md) |
| Ammplify | 0 | 1 | 0 | [View Report](./2025-09-ammplify-submissions.md) |
| Super DCA Liquidity Network | 1 | 0 | 0 | [View Report](./2025-09-super-dca-submissions.md) |
| Hybra Finance | 1 | 0 | 5 | [View Report](./2025-10-hybra-finance-submissions.md) |
| VII Finance Contracts | 0 | 1 | 0 | [View Report](./2026-01-VII-Finance-submissions.md) |
| Chainlink Payment Abstraction V2 | 0 | 0 | 1 | [View Report](./2026-03-chainlink-submissions.md) |
| **Total** | **2** | **3** | **6** | **6 contests** |

---

### File Naming Convention

```text
YYYY-MM-protocol-submissions.md
```