# On-Chain Racing Bot — Technical Notes

> Anonymized technical notes from a Paradigm CTF 2022 entry (0xMonaco). The challenge itself is public; these notes focus on the **strategy + engineering lessons** of building an AI bot for an on-chain auction-based racing game.

**Category:** AI strategy bot for an on-chain racing game with dynamic pricing auctions
**Stack:** Solidity 0.8.13, Foundry, Solmate, fixed-point math (SignedWadMath)
**What's interesting:**
- Two-sided market (`ACCELERATE` vs `SHELL` actions) priced by **VRGDA**-style continuous decay → the bot has to time purchases against curves it can compute on-chain
- All bot logic lives in a `takeYourTurn(carsState[], myIndex)` function — pure state-machine reasoning, no persistent storage
- Adversarial: two other opponent bots in every round → opponent modeling matters more than raw math

## Contents

- [Architecture](./ARCHITECTURE.md) — Game lifecycle, action pricing, bot decision flow
- [Lessons](./LESSONS.md) — What surprised, what was tricky, what to do differently
- [Stack](./STACK.md) — Toolchain, libraries, conventions

## Why this is worth reading

If you're building an on-chain agent that has to:
- Reason about **continuous-price auctions** (`x*` token sales, NFT mints with VRGDA, gas-limited resource markets)
- Decide actions per-turn under a **gas budget** and a fixed-state contract API
- Compete against other agents whose behavior you can only infer from on-chain telemetry

…the patterns here generalize.
