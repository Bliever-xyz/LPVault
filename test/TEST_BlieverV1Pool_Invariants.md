# BlieverV1Pool — Invariant Test Documentation

**File:** `test/BlieverV1Pool_Invariants.t.sol`  
**Contract under test:** `src/BlieverV1Pool.sol`  
**Phase:** Development  
**Run command:** `forge test --match-path "test/BlieverV1Pool_Invariants.t.sol" -vvv`

---

## Overview

This file contains **stateful fuzz (invariant) tests** for `BlieverV1Pool.sol`.

The Foundry invariant fuzzer repeatedly calls random handler functions with random bounded inputs, then checks all `invariant_*` functions after every call. Any violation surfaces an accounting bug that cannot be found with unit tests alone (which test predetermined sequences).

---

## Architecture

```
BlieverV1Pool_InvariantTest  (StdInvariant, Test)
    └── PoolHandler  (CommonBase, StdCheats, StdUtils)
            ├── depositLiquidity()
            ├── withdrawLiquidity()
            ├── registerMarket()
            ├── deregisterMarket()
            ├── collectTrade()
            ├── settleMarket()
            ├── forceSettle()
            └── claimWinnings()
```

### Ghost Variables

The handler maintains two **ghost variables** that shadow the vault's expected on-chain state:

| Ghost Variable | Mirrors | Updated by |
|---|---|---|
| `ghost_expectedTotalLiability` | `pool.totalLiability()` | register, deregister, collectTrade, settle, forceSettle |
| `ghost_activeMarketCount` | `pool.activeMarketCount()` | register, deregister, settle, forceSettle |

Ghost variables are compared to on-chain values in the invariants. Any divergence indicates a missed update path.

### Handler Bounds

All handler inputs are bounded to keep calls within valid business-logic ranges:

| Handler Function | Key Bounds |
|---|---|
| `depositLiquidity` | 1 USDC – 200K USDC |
| `registerMarket` | Max 4 active markets (soft cap, fits within vault's activeCap) |
| `collectTrade` | newLiability as 0–100% fraction of riskBudget; cost 0–2K USDC |
| `settleMarket` | payout as 0–100% fraction of riskBudget |
| All index-based | Bounded to `allMarkets.length - 1` |

---

## Invariants

### Invariant 1 — Solvency

```
usdc.balanceOf(pool) >= pool.totalLiability()
```

**What it catches:** Any accounting path that incorrectly increases `totalLiability` beyond the vault's actual USDC balance. This is the core protocol safety guarantee: the vault can always cover worst-case LP losses.

**Checked by:** `invariant_solvency()`

---

### Invariant 2 — Liability Sum Consistency

```
pool.totalLiability() == Σ currentLiability_i  (all active markets)
```

**What it catches:** Delta-update bugs in `collectTradeCost`, missed liability releases in `settleMarket`/`forceSettleMarket`, or off-by-one errors in multi-market scenarios.

The handler computes the on-chain sum directly via `computeOnChainLiabilitySum()` by iterating all tracked markets and reading `currentLiability` from the vault's `markets` mapping.

**Checked by:** `invariant_totalLiabilityEqualsSum()`

---

### Invariant 3 — Active Market Count

```
pool.activeMarketCount() == count of {market : registered && !settled}
```

**What it catches:** Any path that increments or decrements `activeMarketCount` incorrectly. Includes the underflow revert guard (decrement is intentionally checked, not unchecked).

**Checked by:** `invariant_activeMarketCountConsistent()`

---

### Invariant 4 — Payout Bounds

For every settled market in the handler's set:
```
claimedPayout ≤ settledPayout ≤ riskBudget
```

**What it catches:**
- `claimedPayout > settledPayout`: double-claiming or accounting corruption in `claimWinnings`.
- `settledPayout > riskBudget`: LS-LMSR Proposition 4.9 violation — vault paid out more than its loss guarantee.

Note: after `forceSettleMarket`, both `settledPayout = 0` and `riskBudget = 0`, so `0 ≤ 0 ≤ 0` trivially holds.

**Checked by:** `invariant_payoutBounds()`

---

### Invariant 5 — Current Liability Bound

For every active (registered, unsettled) market:
```
currentLiability ≤ riskBudget
```

**What it catches:** Failure of the vault's cap logic in `collectTradeCost`. If a misbehaving market contract reports `newLiability > riskBudget`, the vault must silently cap it. A violation here means the cap was not applied.

**Checked by:** `invariant_currentLiabilityBounded()`

---

## How to Run

```bash
# Run all invariant tests with default config (256 runs × 128 depth)
forge test --match-path "test/BlieverV1Pool_Invariants.t.sol" -vvv

# Use the CI profile (faster)
FOUNDRY_PROFILE=ci forge test --match-path "test/BlieverV1Pool_Invariants.t.sol"

# Deep run (more fuzz sequences — slow)
FOUNDRY_PROFILE=verbose forge test --match-path "test/BlieverV1Pool_Invariants.t.sol"

# Show counterexample on failure (very verbose)
forge test --match-path "test/BlieverV1Pool_Invariants.t.sol" -vvvv
```

### Recommended `foundry.toml` Settings for Invariants

```toml
[invariant]
runs             = 256     # sequences of handler calls
depth            = 128     # calls per sequence
fail_on_revert   = false   # handler reverts are expected (invalid states skipped)
```

For deeper local testing, increase `runs` to `1000` and `depth` to `256`.

---

## Extending These Tests

### Adding a New Invariant

1. Add a new `invariant_<description>()` function to `BlieverV1Pool_InvariantTest`.
2. If the invariant needs to iterate markets, use `handler.allMarketsLength()` and `handler.allMarkets(i)`.
3. If the invariant needs a new ghost variable, add it to `PoolHandler` and update it in every relevant handler function.

### Adding a New Handler Function

When a new vault function is added (e.g. a fee-collection path):

1. Add a handler function in `PoolHandler` that calls the vault with bounded inputs.
2. Update any relevant ghost variables synchronously.
3. Verify the `fail_on_revert = false` setting so expected reverts (invalid state transitions) do not abort the fuzzing sequence.

### Adding New Markets / LPs

The handler currently uses a soft cap of 4 active markets and 2 LP addresses to keep the state space tractable. For broader coverage, raise `MAX_HANDLER_MARKETS` and add more LP addresses — but expect longer run times.

---

## Known Limitations

| Limitation | Notes |
|---|---|
| Markets tracked by handler only | Markets registered outside the handler (e.g. by unit tests) are not tracked. Invariant tests are fully isolated from unit tests via separate `setUp`. |
| Ghost variable precision | Ghost variables use `try/catch` around vault calls — if a call reverts, no ghost update occurs. This can cause divergence if the handler skips a valid update. The on-chain cross-check (`computeOnChainLiabilitySum`) catches this. |
| No `ExceedsMaxMarkets` path | Registering 10 000 markets is impractical in an invariant test. The `activeMarketCount >= MAX_ACTIVE_MARKETS` revert path is not exercised here. |
