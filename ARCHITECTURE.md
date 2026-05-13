# Architecture

## High-level shape

```
┌────────────────────────────────────────────────────────┐
│                  Monaco (game contract)                │
│                                                        │
│  state:                                                │
│    cars[3] = (balance, speed, position, contractAddr)  │
│    turn counter, last-purchased prices                 │
│                                                        │
│  actions (callable by the active car):                 │
│    buyAcceleration(n)   →  +n speed for caller         │
│    buyShell(n)          →  set next car's speed to 1   │
│                                                        │
│  per-turn loop:                                        │
│    for car in cars:                                    │
│        car.takeYourTurn(carsView, myIndex)             │
│        car.position += car.speed                       │
│    if any car.position >= FINISH_DISTANCE: emit Dub    │
└────────────────────────────────────────────────────────┘
              ▲                          ▲
              │                          │
       ┌──────┴──────┐            ┌──────┴──────┐
       │  Bot car A  │            │  Bot car B  │   (player contracts)
       │ takeYourTurn│            │ takeYourTurn│
       └─────────────┘            └─────────────┘
```

Each player ships a `Car` contract. The game contract calls `takeYourTurn` on each car in turn, passing the full visible state. The car decides whether to buy speed or shells, then returns.

## The dynamic pricing layer

Both `ACCELERATE` and `SHELL` are priced by a **continuous decay schedule** — price drops the longer nobody buys, jumps back up after each purchase. Concretely, the price function:

```
price(turn, soldSoFar) = TARGET_PRICE *
                        exp( -PER_TURN_DECREASE * turn )      // time-decay
                      * exp(  SELL_PER_TURN     * soldSoFar ) // surge on buys
```

(Implemented in fixed-point with `wadExp` / `wadMul` from a signed wad-math helper.)

**Why this matters for the bot:** the price you'll pay is a *deterministic function of public state*. You can compute, on-chain, exactly what each next action would cost — including how much the price would jump for the *opponent* if they bought right after you. That's the whole game.

## The bot's decision surface

Each turn the bot sees `(myBalance, mySpeed, myPosition)` × all three cars. It can choose:

1. Do nothing (free)
2. Buy `k` units of acceleration → costs `getAccelerateCost(k)` from balance
3. Buy `k` units of shell → costs `getShellCost(k)`, sets the car directly ahead to speed 1

The decision is sequential and local — no off-chain compute, no persistent storage between turns. **Every turn is a pure function of the current state.**

### Sketch of a strong strategy (anonymized)

```
on takeYourTurn:
  if I'm in lead by a comfortable margin:
    buy minimum acceleration to maintain lead
  else if a shell on the car ahead would close the gap and is cheap:
    buy 1 shell
  else if acceleration price is well below the time-averaged price:
    buy acceleration aggressively
  else:
    do nothing (let the price decay one more turn)
```

The hard part is "comfortable margin" and "well below time-averaged price" — both depend on **how many turns are left** and **what the opponents are likely to do**.

## Design choices worth flagging

### 1. State is global, decisions are local
The game contract holds canonical state; the bot is a stateless decision function. This means **the bot can never assume anything between turns** — opponents may have changed everything. Stateless design forces clean reasoning.

### 2. Fixed-point everywhere
Prices use 18-decimal signed wad math via a helper library (`SignedWadMath`). Naive integer arithmetic would compound rounding errors across hundreds of turns.

### 3. The "shell" mechanic creates a non-linear payoff
A single shell instantly drops the target's speed to 1, regardless of how many accelerations they bought. This is a *kill switch* — but it's expensive and gets more expensive as more shells get bought. The bot has to weigh: is the opponent's current speed advantage worth the shell price, or will they slow down anyway?

### 4. Three players, not two
With three cars, you can sometimes shell the leader and let the third car eat the cost of overtaking *you* — second-place is often better than first-place under threat. The bot needs to reason about *which* opponent to target, not just whether to attack.
