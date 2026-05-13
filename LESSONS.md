# Lessons

## What surprised me

### Pricing math is the easy part — opponent modeling is the hard part
The continuous-decay pricing schedule is fully deterministic and the contract exposes helpers (`getAccelerateCost`, `getShellCost`). You can compute exactly what every action costs. What you can't compute: what your two opponents will do in the same turn. Spending compute on opponent heuristics paid off more than spending it on cost optimization.

### A naive "always accelerate when cheap" loses
It feels like the right move: prices oscillate, buy at the trough. But the same trough is visible to everyone, so all three cars buy simultaneously, the surge eats your savings, and the leader who *didn't* buy now has a runway. **Cheap doesn't mean affordable — it means crowded.**

### Shells are weaker than they look
A shell drops one car to speed 1 for one turn. That car then accelerates next turn and is back. Meanwhile you spent the shell-price *and* the opponent ahead of you (if you didn't shell them) gained a turn. Shells are only powerful very late game, when the target has no balance left to recover.

### Speed compounds, balance doesn't
A speed-10 car that has 1000 balance left will out-finish a speed-2 car with 5000 balance, every time. Once you have a clear lead, **don't conserve money — convert it to speed before the opponents can shell you out of it.**

## What was tricky

### Fixed-point traps
Multiplying two 18-decimal wads gives a 36-decimal number; you have to divide back by 1e18 (or use `wadMul`). Forgetting this once silently misprices the entire decision. The helper library's `wadExp` saturates rather than throwing — that's a footgun if you don't expect it.

### `takeYourTurn` runs in a `try/catch` on the host side
If your car reverts (out-of-gas, division by zero, asserting an invariant), you're *punished* — your turn is forfeit and you may lose balance. The bot has to be defensive: every external view call should be bounded, every arithmetic op should account for overflow at the wad-math boundary.

### Gas is a hard cap on cleverness
Each turn has a per-call gas budget. A 200-loop simulation of opponent moves over the remaining race won't fit. You have to do "shallow reasoning per turn, called many times" — Monte-Carlo-lite, not deep tree search.

## What I'd do differently

### Build a proper off-chain simulator first
The contract is the spec; a faithful Solidity-mirror in Python/TypeScript lets you self-play tens of thousands of races overnight against different opponent archetypes. I tried to iterate on the bot directly in Foundry tests and it was painfully slow.

### Encode the strategy as a small table, not a long if-else
Branching logic on `(my_position_rank, gap_to_lead, balance_ratio, turns_remaining)` quickly turns into spaghetti. A bucketed lookup table (turn-into-race × position-rank → action policy) is easier to tune and easier to debug against simulator output.

### Don't pre-commit to a shell budget
I split the initial 15k balance into "acceleration budget" and "shell budget" in early versions. Bad idea. Shells should be **opportunistic**, triggered by opponent state, not budgeted. Treating the whole balance as fungible and asking "what's the best action right now?" beat any pre-allocated scheme.

## Generalizable takeaways

If you're building on-chain agents for any auction-style market:

1. **The price formula is public, so price discovery isn't the moat.** The moat is *timing relative to other agents*.
2. **Resource conservation is overrated.** Most agents lose by hoarding, not by spending. If you have a lead, lock it in.
3. **Defensive code matters more than clever code.** A merely-okay bot that never reverts beats a brilliant one that occasionally OOGs.
4. **Self-play in a faithful off-chain mirror is the only way to iterate fast.** On-chain tests are too slow and have too few sample points.
