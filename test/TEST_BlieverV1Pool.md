# BlieverV1Pool — Unit & Fuzz Test Documentation

**File:** `test/BlieverV1Pool.t.sol`  
**Contract under test:** `src/BlieverV1Pool.sol`  
**Phase:** Development  
**Run command:** `forge test --match-path "test/BlieverV1Pool.t.sol" -vvv`

---

## Overview

This file contains the complete **unit and fuzz test suite** for `BlieverV1Pool.sol`.

Tests are organised into 13 logical sections, each as a separate Foundry test contract inheriting from `BlieverV1PoolBase`. Every test exercises the contract through a UUPS proxy (matching production deployment) with a fresh `MockUSDC` (6-decimal) and `MockMarket` instances standing in for real prediction-market contracts.

---

## Architecture

```
BlieverV1PoolBase (abstract)
  ├── setUp()              — deploys proxy, seeds 500K USDC liquidity
  ├── _depositLiquidity()  — helper: mint + deposit
  ├── _registerMarket()    — helper: deploy MockMarket + register
  ├── _registerAndTrade()  — helper: register + push one trade
  └── _accessDenied()      — builds OZ-v5 AccessControlUnauthorizedAccount bytes

test/mocks/MockUSDC.sol    — ERC-20, 6 decimals, open mint()
test/mocks/MockMarket.sol  — relays collectTradeCost / settleMarket / claimWinnings
                             as the market contract (holds MARKET_ROLE after register)
```

### Default Test Parameters

| Parameter | Value | Notes |
|---|---|---|
| `ALPHA` | `3e16` (3%) | Valid LS-LMSR α |
| `MAX_RISK` | `50_000e6` | 50 000 USDC per market |
| `RESERVE_BPS` | `2_000` | 20% reserve buffer |
| `LP_DEPOSIT` | `500_000e6` | Seed liquidity deposited in setUp |
| `activeCap` | `400_000e6` | 500K × 80% — fits 8 markets |

---

## Test Sections

### Section 1 — Initialization (`BlieverV1Pool_InitTest`)

Tests the `initialize()` function: parameter storage, role grants, ERC-20 metadata, double-init protection, and all 7 revert paths (zero-address USDC, zero-address admin, alpha out of range, zero maxRisk, bps out of range).

| Test | What it verifies |
|---|---|
| `test_initialize_setsProtocolParams` | alpha / maxRisk / reserveBps / counters start at expected values |
| `test_initialize_grantsAllRolesToAdmin` | All 5 roles granted to `admin` param |
| `test_initialize_setsMarketRoleAdminToMarketManager` | `getRoleAdmin(MARKET_ROLE) == MARKET_MANAGER_ROLE` |
| `test_initialize_setsERC20Metadata` | name / symbol / decimals / asset |
| `test_initialize_reverts_doubleInit` | `InvalidInitialization()` on second call |
| `test_initialize_reverts_ZeroAddress_usdc/admin` | `ZeroAddress` error |
| `test_initialize_reverts_InvalidAlpha_tooLow/High` | `InvalidAlpha` with bad value |
| `test_initialize_reverts_InvalidMaxRisk_zero` | `InvalidMaxRisk` error |
| `test_initialize_reverts_InvalidBps_tooLow/High` | `InvalidBps` error |

---

### Section 2 — Market Registration (`BlieverV1Pool_RegisterMarketTest`)

Tests `registerMarket()`: successful registration, state changes, event emission, and all revert conditions.

| Test | What it verifies |
|---|---|
| `test_registerMarket_succeeds` | Full `MarketInfo` struct populated correctly |
| `test_registerMarket_incrementsActiveCount` | Counter increases by 1 per registration |
| `test_registerMarket_updatesTotalLiability` | `totalLiability += riskBudget` on each register |
| `test_registerMarket_grantsMarketRole` | `hasRole(MARKET_ROLE, market)` is true |
| `test_registerMarket_emitsEvent` | `MarketRegistered(market, nOutcomes, riskBudget)` |
| `test_registerMarket_minOutcomes_2/maxOutcomes_100` | Boundary: 2 and 100 both valid |
| `test_registerMarket_reverts_ZeroAddress` | `ZeroAddress` error |
| `test_registerMarket_reverts_NotAContract_EOA` | `NotAContract` for EOA address |
| `test_registerMarket_reverts_InvalidOutcomeCount_one/101` | `InvalidOutcomeCount` boundary |
| `test_registerMarket_reverts_AlreadyRegistered` | Duplicate registration blocked |
| `test_registerMarket_reverts_CapacityExceeded_noDeposits` | Zero-deposit vault blocks all registration |
| `test_registerMarket_reverts_whenPaused` | `EnforcedPause()` |
| `test_registerMarket_reverts_unauthorized` | `AccessControlUnauthorizedAccount` |

---

### Section 3 — Market Deregistration (`BlieverV1Pool_DeregisterMarketTest`)

Tests `deregisterMarket()`: only allowed for registered, unsettled, trade-free markets.

| Test | What it verifies |
|---|---|
| `test_deregisterMarket_succeeds` | `registered` flag cleared after delete |
| `test_deregisterMarket_decrementActiveCount` | Counter decremented |
| `test_deregisterMarket_releasesLiabilityFromTotal` | `totalLiability -= riskBudget` |
| `test_deregisterMarket_revokesMarketRole` | `MARKET_ROLE` removed |
| `test_deregisterMarket_emitsEvent` | `MarketDeregistered(market)` |
| `test_deregisterMarket_reverts_NotRegistered` | Unregistered market blocked |
| `test_deregisterMarket_reverts_AlreadySettled` | Cannot deregister settled market |
| `test_deregisterMarket_reverts_MarketHasTrades` | `MarketHasTrades` once a trade occurred |
| `test_deregisterMarket_reverts_whenPaused/unauthorized` | Pause + access control |

---

### Section 4 — Collect Trade Cost (`BlieverV1Pool_CollectTradeCostTest`)

Tests `collectTradeCost()`: the primary vault interaction from market contracts. Covers USDC transfer, liability accounting, the riskBudget cap, and event emission.

**Key design point tested:** Trader approves the **vault** (not the market) for `cost` USDC. This is a common integration mistake.

| Test | What it verifies |
|---|---|
| `test_collectTradeCost_withCost_transfersUSDC` | Vault receives `cost`; trader balance decremented |
| `test_collectTradeCost_zeroCost_noUSDCTransfer` | Zero-cost trades do not move USDC |
| `test_collectTradeCost_setsHasTrades` | `hasTrades` flag set on first call |
| `test_collectTradeCost_decreasesLiability_normalPath` | `totalLiability` reflects decreased `newLiability` |
| `test_collectTradeCost_updatesTotalLiabilityCorrectly` | Multi-trade delta tracking |
| `test_collectTradeCost_capsLiabilityAtRiskBudget` | Vault silently caps overbudget reports |
| `test_collectTradeCost_emitsTradeCostCollected` | Event with all fields |
| `test_collectTradeCost_emitsMarketLiabilityUpdated_whenChanged` | Emitted only when liability changes |
| `test_collectTradeCost_noLiabilityEvent_whenSameLiability` | No redundant event when unchanged |
| `test_collectTradeCost_reverts_*` | Settled market, zero trader address, paused, unauthorised |

---

### Section 5 — Settle Market (`BlieverV1Pool_SettleMarketTest`)

Tests `settleMarket()`: the normal resolution path. Covers payout accounting, liability release, active count, and the three event branches.

**Critical design points tested:**
- `settleMarket` is **not pause-gated** — markets must be settleable during emergencies.
- Zero-payout path revokes `MARKET_ROLE` immediately (no claim call can ever arrive).
- `currentLiability` (live, post-trade) is released — not the original `riskBudget`.

| Test | What it verifies |
|---|---|
| `test_settleMarket_withPayout_setsSettledState` | `settled=true`, `settledPayout`, `currentLiability=0` |
| `test_settleMarket_releasesCurrentLiabilityFromTotal` | `totalLiability` decreases by live `currentLiability` |
| `test_settleMarket_decrementsActiveMarketCount` | Counter decremented |
| `test_settleMarket_emitsMarketSettled` | Event with `profit = riskBudget - totalPayout` |
| `test_settleMarket_zeroPayout_revokesRoleImmediately` | Role gone at settlement, not at claim time |
| `test_settleMarket_zeroPayout_emitsMarketFullyClaimed` | `MarketFullyClaimed(market, 0)` |
| `test_settleMarket_untraded_emitsMarketExpiredUntraded` | `MarketExpiredUntraded(market, riskBudget)` |
| `test_settleMarket_worksWhileVaultIsPaused` | **Must** succeed even when paused |
| `test_settleMarket_reverts_PayoutExceedsRiskBudget` | LS-LMSR bound enforcement |
| `test_settleMarket_reverts_AlreadySettled/unauthorized` | Guards |

---

### Section 6 — Force Settle Market (`BlieverV1Pool_ForceSettleTest`)

Tests `forceSettleMarket()`: emergency path for stuck/broken markets.

**Key difference from normal settlement:** `settledPayout = 0` and `riskBudget = 0` — no claims possible. `MARKET_ROLE` revoked immediately.

| Test | What it verifies |
|---|---|
| `test_forceSettleMarket_setsSettledState` | `settled=true`, payout=0, liability=0, riskBudget=0 |
| `test_forceSettleMarket_releasesLiability` | `totalLiability` decreases by `currentLiability` |
| `test_forceSettleMarket_revokesMarketRole` | Role removed immediately |
| `test_forceSettleMarket_emitsMarketForceSettled` | Event with `lossAbsorbed` |
| `test_forceSettleMarket_worksWhileVaultIsPaused` | Emergency path ignores pause |
| `test_forceSettleMarket_reverts_*` | Not registered, already settled, non-EMERGENCY_ROLE |

---

### Section 7 — Claim Winnings (`BlieverV1Pool_ClaimWinningsTest`)

Tests `claimWinnings()`: USDC payout to winners after settlement.

**Critical design points tested:**
- `claimWinnings` is **not pause-gated** — winners must always be paid.
- Last claim (when `claimedPayout == settledPayout`) revokes `MARKET_ROLE`.

| Test | What it verifies |
|---|---|
| `test_claimWinnings_transfersUSDCToWinner` | Vault sends exact `amount` to winner |
| `test_claimWinnings_partialClaims_trackAccumulation` | `claimedPayout` accumulates correctly |
| `test_claimWinnings_lastClaim_revokesMarketRole` | Role revoked after full payout |
| `test_claimWinnings_lastClaim_emitsMarketFullyClaimed` | Final event fired |
| `test_claimWinnings_emitsWinningsClaimed` | Per-claim event |
| `test_claimWinnings_worksWhileVaultIsPaused` | **Must** succeed even when paused |
| `test_claimWinnings_reverts_MarketNotSettled` | Cannot claim on active market |
| `test_claimWinnings_reverts_ZeroAddress/ZeroAmount` | Input validation |
| `test_claimWinnings_reverts_PayoutExceedsSettlement` | Over-claim blocked |
| `test_claimWinnings_reverts_unauthorized` | Non-MARKET_ROLE blocked |

---

### Section 8 — Admin Parameters (`BlieverV1Pool_AdminParamsTest`)

Tests `setAlpha`, `setMaxRiskPerMarket`, and `setReserveBps`. All require `DEFAULT_ADMIN_ROLE`.

Each parameter is tested for: successful update, event emission, valid boundary values (min/max), rejection below min, rejection above max, and rejection by non-admin.

---

### Section 9 — Pause Control (`BlieverV1Pool_PauseTest`)

Tests asymmetric pause design:
- `PAUSER_ROLE` can pause (fast circuit breaker)
- Only `DEFAULT_ADMIN_ROLE` can unpause (prevents compromised pauser key defeating pause)

Also verifies that `settleMarket`, `forceSettleMarket`, and `claimWinnings` work during pause (tested in sections 5, 6, 7).

| Test | What it verifies |
|---|---|
| `test_pause_succeeds_byPauser` | pause() works with PAUSER_ROLE |
| `test_unpause_succeeds_byAdmin` | unpause() works with DEFAULT_ADMIN_ROLE |
| `test_unpause_reverts_byPauserOnly` | PAUSER_ROLE cannot unpause — asymmetry enforced |
| `test_pause_reverts_byNonPauser` | Non-pauser blocked |
| `test_pause_blocksDeposit/Withdraw/Register/TokenTransfer` | All affected operations blocked |
| `test_pause_maxDeposit/Mint/Withdraw/Redeem_returns0` | ERC-4626 caps return 0 when paused |

---

### Section 10 — ERC-4626 LP Vault (`BlieverV1Pool_ERC4626Test`)

Tests the LP deposit/withdraw/redeem cycle and the constrained max functions.

| Test | What it verifies |
|---|---|
| `test_deposit_mintsShares` | Shares minted non-zero; balance matches |
| `test_deposit_increasesTotalAssets` | totalAssets grows by deposit |
| `test_withdraw_burnsSharesAndReturnsUSDC` | LP receives USDC on withdrawal |
| `test_redeem_burnsSharesAndReturnsUSDC` | Redeem path works |
| `test_maxWithdraw_cappedByFreeLiquidity` | maxWithdraw ≤ availableLiquidity |
| `test_maxWithdraw_zeroWhenFullyUtilised` | 8 markets = zero withdrawable |
| `test_withdraw_reverts_beyondFreeLiquidity` | Cannot withdraw locked capital |
| `test_decimals_returns18` | 6 USDC + 12 DECIMALS_OFFSET |
| `test_totalAssets_equalsRawUSDCBalance` | totalAssets = `usdc.balanceOf(pool)` |

---

### Section 11 — View Functions (`BlieverV1Pool_ViewFunctionsTest`)

Tests all public view functions: `availableLiquidity`, `utilizationBps`, `nav`, `isSolvent`, `isActiveMarket`, `getMarketInfo`.

Key formula verified:  
`free = totalAssets − totalLiability − (totalAssets × reserveBps / BPS_BASE)`

---

### Section 12 — UUPS Upgrade (`BlieverV1Pool_UpgradeTest`)

Tests that:
- Non-`UPGRADER_ROLE` callers cannot upgrade (reverts with `AccessControlUnauthorizedAccount`)
- `UPGRADER_ROLE` can upgrade successfully
- State (registered markets, totalLiability) is preserved across an upgrade

Uses ERC-1967 implementation slot (`0x3608...`) to verify the proxy's stored implementation address.

---

### Section 13 — Fuzz Tests (`BlieverV1Pool_FuzzTest`)

Property-based tests with randomised inputs.

| Fuzz Test | Property |
|---|---|
| `testFuzz_deposit_sharesAlwaysGtZero` | Deposit of any valid USDC amount mints non-zero shares |
| `testFuzz_collectTradeCost_liabilityTrackedCorrectly` | totalLiability == newLiability after single-market trade |
| `testFuzz_collectTradeCost_capNeverExceedsRiskBudget` | Overbudget reports are capped; stored value ≤ riskBudget |
| `testFuzz_freeLiquidity_neverNegative` | availableLiquidity never underflows; solvency holds |
| `testFuzz_settleMarket_profitNonNegative` | Settlement always solvent after any valid payout |
| `testFuzz_setReserveBps_boundsEnforced` | Any uint16: valid range accepted, outside rejected |
| `testFuzz_setAlpha_boundsEnforced` | Any uint256 in [0,1e18]: valid range accepted, outside rejected |
| `testFuzz_multipleDeposits_totalAssetsConsistent` | totalAssets == pre-balance + sum of deposits |

---

## How to Run

```bash
# All unit tests
forge test --match-path "test/BlieverV1Pool.t.sol" -vvv

# Specific section (e.g. settlement)
forge test --match-contract "BlieverV1Pool_SettleMarketTest" -vvv

# Only fuzz tests
forge test --match-contract "BlieverV1Pool_FuzzTest" -vvv

# Coverage report
forge coverage --match-path "test/BlieverV1Pool.t.sol" --report summary

# Gas snapshot for hot paths
forge snapshot --match-path "test/BlieverV1Pool.t.sol"
```

---

## Extending These Tests

- **New admin parameter:** Add a `test_set<Param>_*` block in `BlieverV1Pool_AdminParamsTest` following the existing pattern (happy path, boundary min/max, tooLow, tooHigh, unauthorized).
- **New market exit path:** Add coverage in `BlieverV1Pool_SettleMarketTest` and verify `MARKET_ROLE` is revoked via the appropriate invariant.
- **New ERC-4626 operation:** Add to `BlieverV1Pool_ERC4626Test`; always verify `totalAssets == usdc.balanceOf(pool)` post-operation.
- **LSMath integration:** Once market contracts are available, replace `MockMarket.doCollectTrade` calls with real market logic to test end-to-end trade cost calculations.

---

## Known Scope Boundaries

| Out of scope | Reason |
|---|---|
| LSMath library correctness | Separate contract; tested independently |
| Market contract logic | Not yet developed (dev phase) |
| Oracle resolution | External dependency; mocked via MockMarket |
| Fork tests against Base mainnet | Deferred until contracts are deployed |
