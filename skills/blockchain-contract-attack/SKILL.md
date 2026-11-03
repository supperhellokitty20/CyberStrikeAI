---
name: blockchain-contract-attack
description: >-
  Blockchain/smart contracts: Etherscan, slither/mythril, reentrancy/access control/oracle/flash loan, cross-chain bridges, RPC exposure. Use when auditing smart contracts, DeFi, or blockchain attack surfaces.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## Blockchain / Smart Contracts

```
=== Blockchain/Smart Contracts ===
Source: Etherscan getsourcecode API | Audit slither/mythril/manticore
Manual: reentrancy (.call{value} transfers before state change, violating C-E-I) | access control (missing onlyOwner) | integer overflow (<0.8 without SafeMath)
  oracle manipulation (flash loan instantly manipulates AMM price) | randomness (block.timestamp controllable) | approval abuse (infinite approve/permit replay) | delegatecall proxy storage collision
DeFi: flash loan attacks/sandwiching (front-running)/governance attacks/signature replay | Cross-chain bridges: signature threshold bypass/replay | RPC: exposed 8545 allows direct eth_sendTransaction
```
