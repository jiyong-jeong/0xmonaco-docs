# Stack

## Language & toolchain

- **Solidity** 0.8.13 — pinned at the version the game contract uses, so the bot compiles with identical overflow/check semantics
- **Foundry** for build, test, gas profiling — `forge test`, `forge snapshot`
- **Solmate** for utility libraries (`SafeCastLib` for narrowing casts without panicking)

## Math

- **SignedWadMath** — fixed-point arithmetic at 18 decimals (`wadMul`, `wadDiv`, `wadExp`, `wadLn`). Required because the pricing formula uses continuous exponential decay; integer math would diverge after a few turns.

## Code layout (canonical)

```
src/
├── Monaco.sol             — game contract (the "spec")
├── utils/SignedWadMath.sol — fixed-point helpers
└── cars/
    ├── Car.sol            — abstract base; defines takeYourTurn interface
    └── ExampleCar.sol     — reference implementation, useful as opponent

script/
└── Deploy*.s.sol          — Foundry deploy script

test/
└── Monaco.t.sol           — Foundry tests; also doubles as a poor-man's simulator
```

## Foundry configuration

Key bits worth keeping:

```toml
[profile.default]
bytecode_hash = "none"      # deterministic bytecode (matters when judging)
optimizer_runs = 1000000    # max optimization; deploy gas matters less than runtime gas
solc = "0.8.13"             # pinned

[profile.intense]
fuzz_runs = 10000           # for property-based testing edge cases
```

## Why this stack for this problem

- **Foundry over Hardhat:** test loop is 10–100× faster, which matters because every bot iteration involves replaying many simulated games
- **Solmate over OpenZeppelin:** Solmate's libs (especially the fixed-point math) are smaller, gas-leaner, and the wad-math primitive isn't available in OZ
- **No frontend, no off-chain orchestration in the contract repo:** the simulator is a separate project (recommend Python or TypeScript) that *mirrors* the Solidity rules — keeping the on-chain spec dumb and the off-chain optimizer rich

## Deployment notes

For the CTF itself the organizer deploys the game contract and registers each player's car. For local self-play, the included deploy script wires up three cars against one Monaco instance. The bot doesn't need any off-chain infrastructure at runtime — it's pure on-chain reaction.
