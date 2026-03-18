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
| `activeCap` | `400_000e6` | 500K × 80% — fits exactly 8 markets |

---

## Test Sections

### Section 1 — Initialization (`BlieverV1Pool_InitTest`)

Tests the `initialize()` function: parameter storage, role grants, ERC-20 metadata, double-init protection, bare-implementation protection, and all 7 input-validation revert paths.

The validation guards in `initialize()` are identical to those in the corresponding setters (`setAlpha`, `setMaxRiskPerMarket`, `setReserveBps`) — same error types, same bounds. Every parameter that can later be changed via a setter is validated at construction time with the same rule.

**Input validation test pattern — `_deployUninitProxy()`**

The 7 input-validation tests use a two-step pattern: deploy an uninitialized proxy first, then call `initialize` with bad params as a direct external call.

`_deployUninitProxy()` deploys a fresh `ERC1967Proxy` with **empty calldata** (`""`). When `data.length == 0`, `ERC1967Utils.upgradeToAndCall` skips the delegatecall path entirely and only sets the implementation slot — the proxy's `Initializable` storage stays at version 0, leaving it ready for a first-time `initialize` call in the test body.

Why not `new ERC1967Proxy(impl, badInitData)` directly? Foundry's `vm.expectRevert` intercepts the innermost external call in a call frame, which inside the `ERC1967Proxy` constructor is the DELEGATECALL to `initialize`. Foundry marks the expectation as fulfilled at that level, swallows the delegatecall revert, and allows the proxy constructor to complete. The outer CREATE never reverts, so the test reports "next call did not revert as expected". Separating deployment from initialization means `vm.expectRevert` intercepts a plain external CALL, which is the opcode it was designed for.

**Cheatcode ordering: prank + expectRevert**

Three distinct prank-ordering hazards apply across this suite:

1. **Role constant evaluated while prank is queued.** `pool.MARKET_ROLE()` is a staticcall. If `vm.prank(attacker)` is set first and `pool.MARKET_ROLE()` is evaluated as an argument (e.g., inside `_accessDenied`), the staticcall consumes the single-call prank. The subsequent target function runs from the test contract's address. Fix: evaluate all role constants before setting any prank.

2. **`vm.startPrank` active when `vm.expectRevert` is called.** In some Foundry versions, calling `vm.expectRevert` while `vm.startPrank` is already active causes the framework to consume the prank context during cheatcode processing. The next real external call then runs from the test contract. Fix: use `vm.expectRevert` first (no prank active), then `vm.prank` as a single-call prank consumed only by the function under test.

3. **`vm.prank` consumed by a staticcall used in `grantRole`.** Caching role constants into named variables before calling `vm.prank` avoids this. See `test_unpause_reverts_byPauserOnly` where `bytes32 pauserRole = pool.PAUSER_ROLE()` is cached before `vm.prank(admin); pool.grantRole(pauserRole, pauserOnly)`.

The universal safe pattern across this suite: evaluate all contract getter calls, then `vm.expectRevert(...)`, then `vm.prank(addr)`, then the target function call.

**Deposit and withdraw pause behavior**

The contract pauses deposits and withdrawals via the ERC-4626 max-override pattern: `maxDeposit()` returns 0 and `maxWithdraw()` returns 0 when paused. The ERC-4626 base `deposit()` and `withdraw()` check these limits and throw `ERC4626ExceededMaxDeposit` / `ERC4626ExceededMaxWithdraw` — not `EnforcedPause()`. The `_update` hook (which carries `whenNotPaused`) blocks bLP token transfers directly via the EVM-level revert; USDC-level ERC-4626 operations go through the max-cap path. Tests in Section 9 assert the correct OZ error selectors for each path.

| Test | What it verifies |
|---|---|
| `test_initialize_setsProtocolParams` | alpha / maxRisk / reserveBps / counters start at expected values |
| `test_initialize_grantsAllRolesToAdmin` | All 5 roles granted to `admin` param |
| `test_initialize_setsMarketRoleAdminToMarketManager` | `getRoleAdmin(MARKET_ROLE) == MARKET_MANAGER_ROLE` |
| `test_initialize_setsERC20Metadata` | name / symbol / decimals / asset |
| `test_initialize_reverts_doubleInit` | `InvalidInitialization()` on second proxy call |
| `test_initialize_reverts_onBareImplementation` | `_disableInitializers()` in constructor blocks direct `initialize` on the logic contract |
| `test_initialize_reverts_ZeroAddress_usdc/admin` | `ZeroAddress` — via `_deployUninitProxy` + direct CALL pattern |
| `test_initialize_reverts_InvalidAlpha_tooLow/High` | `InvalidAlpha` — via `_deployUninitProxy` + direct CALL pattern |
| `test_initialize_reverts_InvalidMaxRisk_zero` | `InvalidMaxRisk` — via `_deployUninitProxy` + direct CALL pattern |
| `test_initialize_reverts_InvalidBps_tooLow/High` | `InvalidBps` — via `_deployUninitProxy` + direct CALL pattern |

---

### Section 2 — Market Registration (`BlieverV1Pool_RegisterMarketTest`)

Tests `registerMarket()`: successful registration, state changes, event emission, and all revert conditions including two distinct capacity-exceeded scenarios.

All `CapacityExceeded` reverts assert the specific `CapacityExceeded(projected, activeCap)` selector with exact arguments — bare `vm.expectRevert()` is not used anywhere in this suite.

| Test | What it verifies |
|---|---|
| `test_registerMarket_succeeds` | Full `MarketInfo` struct populated; 2 outcomes is the valid minimum |
| `test_registerMarket_incrementsActiveCount` | Counter increases by 1 per registration |
| `test_registerMarket_updatesTotalLiability` | `totalLiability += riskBudget` on each register |
| `test_registerMarket_grantsMarketRole` | `hasRole(MARKET_ROLE, market)` is true |
| `test_registerMarket_emitsEvent` | `MarketRegistered(market, nOutcomes, riskBudget)` |
| `test_registerMarket_maxOutcomes_100_succeeds` | Boundary: 100 outcomes valid |
| `test_registerMarket_reverts_ZeroAddress` | `ZeroAddress` error |
| `test_registerMarket_reverts_NotAContract_EOA` | `NotAContract` for EOA address |
| `test_registerMarket_reverts_InvalidOutcomeCount_one/101` | `InvalidOutcomeCount` at both boundaries |
| `test_registerMarket_reverts_AlreadyRegistered` | Duplicate registration blocked |
| `test_registerMarket_reverts_CapacityExceeded_noDeposits` | `CapacityExceeded(MAX_RISK, 0)` — empty vault, activeCap = 0 |
| `test_registerMarket_reverts_CapacityExceeded_nthMarketExceedsCap` | `CapacityExceeded(450K, 400K)` — 9th market over-fills activeCap |
| `test_registerMarket_reverts_whenPaused` | `EnforcedPause()` |
| `test_registerMarket_reverts_unauthorized` | `AccessControlUnauthorizedAccount` |

---

### Section 3 — Market Deregistration (`BlieverV1Pool_DeregisterMarketTest`)

Tests `deregisterMarket()`: only allowed for registered, unsettled, trade-free markets.

**`test_deregisterMarket_allowsReregistration_sameAddress`** verifies that `delete markets[market]` fully resets the entry (`registered = false`), allowing the same contract address to be cleanly registered again. `activeMarketCount` returns to 1 after re-registration.

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
| `test_deregisterMarket_allowsReregistration_sameAddress` | Same address cleanly re-registered after delete |

---

### Section 4 — Collect Trade Cost (`BlieverV1Pool_CollectTradeCostTest`)

Tests `collectTradeCost()`: the primary vault interaction from market contracts. Covers USDC transfer, liability accounting, all three liability-delta branches, the riskBudget cap, and event emission.

**`test_collectTradeCost_reverts_MarketAlreadySettled`:** A zero-payout settle (`doSettle(0)`) immediately revokes `MARKET_ROLE`. Any subsequent call then hits `onlyRole(MARKET_ROLE)` before the `MarketAlreadySettled` guard at line 506. The test uses a non-zero-payout settle to keep the role alive, so `MarketAlreadySettled` is the actual revert.

**`test_collectTradeCost_increasesLiability_whenBetweenOldAndBudget`** covers the `capped > old` branch (lines 519–520): when a market's running worst-case re-expands (still within budget), `totalLiability` must increase.

**`test_collectTradeCost_newLiabilityZero_totalLiabilityDropsToZero`** verifies the most economically important LS-LMSR outcome: a fully-hedged market (`newLiability == 0`) must drive `totalLiability` to zero.

**Key design point tested:** Trader approves the **vault** (not the market) for `cost` USDC.

| Test | What it verifies |
|---|---|
| `test_collectTradeCost_withCost_transfersUSDC` | Vault receives `cost`; trader balance decremented |
| `test_collectTradeCost_zeroCost_noUSDCTransfer` | Zero-cost trades do not move USDC |
| `test_collectTradeCost_setsHasTrades` | `hasTrades` flag set on first call |
| `test_collectTradeCost_decreasesLiability_normalPath` | `totalLiability` reflects decreased `newLiability` |
| `test_collectTradeCost_updatesTotalLiabilityCorrectly` | Multi-trade delta tracking |
| `test_collectTradeCost_capsLiabilityAtRiskBudget` | Vault silently caps overbudget reports; `LiabilityCapApplied` emitted |
| `test_collectTradeCost_increasesLiability_whenBetweenOldAndBudget` | `capped > old` branch: `totalLiability` rises when liability re-expands within budget |
| `test_collectTradeCost_newLiabilityZero_totalLiabilityDropsToZero` | `newLiability == 0` drives `totalLiability` to zero |
| `test_collectTradeCost_emitsTradeCostCollected` | Event with all fields |
| `test_collectTradeCost_emitsMarketLiabilityUpdated_whenChanged` | Emitted only when liability changes |
| `test_collectTradeCost_noLiabilityEvent_whenSameLiability` | No redundant event when unchanged |
| `test_collectTradeCost_reverts_MarketAlreadySettled` | Non-zero settle keeps role; `MarketAlreadySettled` is the revert |
| `test_collectTradeCost_reverts_ZeroAddress_trader` | `ZeroAddress` guard |
| `test_collectTradeCost_reverts_whenPaused` | `EnforcedPause()` |
| `test_collectTradeCost_reverts_unauthorized_nonMarketRole` | `AccessControlUnauthorizedAccount` |

---

### Section 4B — Distribute Refund (`BlieverV1Pool_DistributeRefundTest`)

Tests `distributeRefund()`: the vault's sell-side counterpart to `collectTradeCost()`.

**Structural differences from `collectTradeCost` — all tested explicitly:**

| Dimension | `collectTradeCost` | `distributeRefund` |
|---|---|---|
| USDC direction | trader → vault (`safeTransferFrom`) | vault → trader (`safeTransfer`) |
| `hasTrades` | Set to `true` | Not touched |
| Solvency check | Not called | `_assertSolvent()` at end |
| Zero-amount guard | `if (cost > 0)` before transfer | `if (refundAmount > 0)` before transfer |

**`test_distributeRefund_doesNotSetHasTrades`** is the most critical behavioural distinction. If `distributeRefund` ever set `hasTrades`, a sell-only market would be permanently blocked from deregistration — `deregisterMarket` guards on `hasTrades`. The test calls `distributeRefund`, then asserts `hasTrades == false` and confirms `deregisterMarket` completes without revert.

**`test_distributeRefund_reverts_VaultInsolvent`** verifies `_assertSolvent()` fires on the outgoing-transfer path. `deal` drains the vault to exactly `MAX_RISK` (equal to `totalLiability`), then a 1-wei refund pushes the balance below liability.

**`test_distributeRefund_reverts_MarketNotRegistered`** manually grants `MARKET_ROLE` to an unregistered contract to bypass `onlyRole` and reach the body-level guard — the only state where a caller has the role without being in the `markets` mapping.

| Test | What it verifies |
|---|---|
| `test_distributeRefund_transfersUSDCToTrader` | Vault balance decremented; trader balance incremented by exact refund |
| `test_distributeRefund_zeroRefund_noUSDCTransfer` | No USDC moves; liability update still applies |
| `test_distributeRefund_decreasesLiability` | Normal sell: `totalLiability` falls when `newLiab < currentLiability` |
| `test_distributeRefund_increasesLiability_whenBetweenOldAndBudget` | `capped > old` branch: liability re-expands within budget |
| `test_distributeRefund_newLiabilityZero_dropsToZero` | Fully closed position drives `totalLiability` to zero |
| `test_distributeRefund_capsLiabilityAtRiskBudget` | Overbudget report capped; `LiabilityCapApplied` emitted |
| `test_distributeRefund_emitsRefundDistributed` | All 4 fields: market, trader, refundAmount, newCurrentLiability |
| `test_distributeRefund_emitsMarketLiabilityUpdated_whenChanged` | Conditional event fires on any delta |
| `test_distributeRefund_noLiabilityEvent_whenUnchanged` | No redundant event when liability is unchanged |
| `test_distributeRefund_doesNotSetHasTrades` | `hasTrades` stays false; deregister succeeds after sell-only usage |
| `test_distributeRefund_reverts_VaultInsolvent` | `_assertSolvent()` catches any outflow that breaks solvency |
| `test_distributeRefund_reverts_MarketNotRegistered` | Manually granted role; body guard reachable |
| `test_distributeRefund_reverts_MarketAlreadySettled` | Non-zero first settle keeps role; guard reached |
| `test_distributeRefund_reverts_ZeroAddress_trader` | `ZeroAddress` input guard |
| `test_distributeRefund_reverts_whenPaused` | `whenNotPaused` blocks sell path during pause |
| `test_distributeRefund_reverts_unauthorized` | Non-`MARKET_ROLE` caller blocked |

### Section 5 — Settle Market (`BlieverV1Pool_SettleMarketTest`)

Tests `settleMarket()`: the normal resolution path. Covers payout accounting, liability release, active count, and three event branches.

**Critical design points tested:**
- `settleMarket` is **not pause-gated** — markets must be settleable during emergencies.
- Zero-payout path revokes `MARKET_ROLE` immediately (no valid claim can ever arrive).
- `currentLiability` (live, post-trade) is released — not the original `riskBudget`.

**`test_settleMarket_reverts_AlreadySettled`:** Uses a non-zero first payout to keep `MARKET_ROLE` alive; `MarketAlreadySettled` is then the revert on the second call, not `onlyRole`.

**`test_settleMarket_totalPayout_equalsRiskBudget_succeeds`:** The guard is `> not >=` (line 574). `totalPayout == riskBudget` is the "LP absorbs maximum loss" case and must succeed.

**`test_settleMarket_profitIncreasesLPShareValue`:** After trade costs enter the vault without new share issuance, `convertToAssets(shares)` is checked before and after to confirm LP share value grows.

**`test_assertSolvent_reverts_VaultInsolvent`:** Uses `deal` to drain vault USDC below the liability that remains after one of two active markets settles, confirming `_assertSolvent()` (called at settlement end) fires `VaultInsolvent`.

| Test | What it verifies |
|---|---|
| `test_settleMarket_withPayout_setsSettledState` | `settled=true`, `settledPayout`, `currentLiability=0` |
| `test_settleMarket_releasesCurrentLiabilityFromTotal` | `totalLiability` decreases by live `currentLiability`, not `riskBudget` |
| `test_settleMarket_decrementsActiveMarketCount` | Counter decremented |
| `test_settleMarket_emitsMarketSettled` | Event with `profit = riskBudget - totalPayout` |
| `test_settleMarket_zeroPayout_revokesRoleImmediately` | Role gone at settlement, not at claim time |
| `test_settleMarket_zeroPayout_emitsMarketFullyClaimed` | `MarketFullyClaimed(market, 0)` |
| `test_settleMarket_untraded_emitsMarketExpiredUntraded` | `MarketExpiredUntraded(market, riskBudget)` |
| `test_settleMarket_worksWhileVaultIsPaused` | **Must** succeed even when paused |
| `test_settleMarket_totalPayout_equalsRiskBudget_succeeds` | Exact riskBudget payout accepted (`>` not `>=`) |
| `test_settleMarket_profitIncreasesLPShareValue` | `convertToAssets` increases after trade costs enter vault |
| `test_assertSolvent_reverts_VaultInsolvent` | `_assertSolvent` fires when balance < remaining liability |
| `test_settleMarket_reverts_PayoutExceedsRiskBudget` | LS-LMSR bound enforcement |
| `test_settleMarket_reverts_AlreadySettled` | Non-zero first settle keeps role; second settle hits guard |
| `test_settleMarket_reverts_unauthorized` | `AccessControlUnauthorizedAccount` |

---

### Section 6 — Force Settle Market (`BlieverV1Pool_ForceSettleTest`)

Tests `forceSettleMarket()`: emergency path for stuck or broken markets.

**Key difference from normal settlement:** `settledPayout = 0` and `riskBudget = 0` — no claims possible. `MARKET_ROLE` revoked immediately.

| Test | What it verifies |
|---|---|
| `test_forceSettleMarket_setsSettledState` | `settled=true`, payout=0, liability=0, riskBudget=0 |
| `test_forceSettleMarket_releasesLiability` | `totalLiability` decreases by `currentLiability` |
| `test_forceSettleMarket_revokesMarketRole` | Role removed immediately |
| `test_forceSettleMarket_emitsMarketForceSettled` | Event with `lossAbsorbed` |
| `test_forceSettleMarket_worksWhileVaultIsPaused` | Emergency path ignores pause |
| `test_forceSettleMarket_reverts_NotRegistered` | Unregistered market blocked |
| `test_forceSettleMarket_reverts_AlreadySettled` | Cannot force-settle twice |
| `test_forceSettleMarket_reverts_unauthorized` | Non-`EMERGENCY_ROLE` blocked |

---

### Section 7 — Claim Winnings (`BlieverV1Pool_ClaimWinningsTest`)

Tests `claimWinnings()`: USDC payout to winners after settlement.

**Critical design points tested:**
- `claimWinnings` is **not pause-gated** — winners must always be paid.
- Last claim (when `claimedPayout == settledPayout`) revokes `MARKET_ROLE`.

**`test_claimWinnings_afterForceSettle_reverts_unauthorized`:** `forceSettleMarket` revokes `MARKET_ROLE` immediately and sets `settledPayout = 0`. A subsequent `doClaim` call hits `onlyRole(MARKET_ROLE)` — not `MarketNotSettled` — because the role is already gone. The revert reason distinction matters for integrators.

| Test | What it verifies |
|---|---|
| `test_claimWinnings_transfersUSDCToWinner` | Vault sends exact `amount` to winner |
| `test_claimWinnings_partialClaims_trackAccumulation` | `claimedPayout` accumulates correctly across calls |
| `test_claimWinnings_lastClaim_revokesMarketRole` | Role revoked when `claimedPayout == settledPayout` |
| `test_claimWinnings_lastClaim_emitsMarketFullyClaimed` | Final event fired |
| `test_claimWinnings_emitsWinningsClaimed` | Per-claim event |
| `test_claimWinnings_worksWhileVaultIsPaused` | **Must** succeed even when paused |
| `test_claimWinnings_reverts_MarketNotSettled` | Cannot claim on active market |
| `test_claimWinnings_reverts_ZeroAddress_winner` | `ZeroAddress` guard |
| `test_claimWinnings_reverts_ZeroAmount` | `ZeroAmount` guard |
| `test_claimWinnings_reverts_PayoutExceedsSettlement` | Over-claim blocked with exact remaining shown |
| `test_claimWinnings_reverts_unauthorized` | Non-`MARKET_ROLE` blocked |
| `test_claimWinnings_afterForceSettle_reverts_unauthorized` | Role revoked by force-settle; revert is access control, not MarketNotSettled |

---

### Section 8 — Admin Parameters (`BlieverV1Pool_AdminParamsTest`)

Tests `setAlpha`, `setMaxRiskPerMarket`, and `setReserveBps`. All require `DEFAULT_ADMIN_ROLE`.

Each parameter is tested for: successful update, event emission, valid boundary values (min/max), rejection below min, rejection above max, and rejection by non-admin.

**`test_setMaxRiskPerMarket_doesNotAffectExistingMarkets`** verifies the contract's explicit intent: parameter changes only apply to markets registered *after* the call. A market registered before the change keeps its original `riskBudget`; a market registered after uses the new value.

| Test group | Tests |
|---|---|
| `setAlpha` | updatesValue, emitsEvent, boundary_minAndMax, reverts_tooLow, reverts_tooHigh, reverts_unauthorized |
| `setMaxRiskPerMarket` | updatesValue, emitsEvent, reverts_zero, reverts_unauthorized, doesNotAffectExistingMarkets |
| `setReserveBps` | updatesValue, emitsEvent, boundary_minAndMax, reverts_tooLow, reverts_tooHigh, reverts_unauthorized |

---

### Section 9 — Pause Control (`BlieverV1Pool_PauseTest`)

Tests asymmetric pause design:
- `PAUSER_ROLE` can pause (fast circuit breaker)
- Only `DEFAULT_ADMIN_ROLE` can unpause (prevents a compromised pauser key from defeating its own pause)

`settleMarket`, `forceSettleMarket`, and `claimWinnings` are verified as pause-exempt in sections 5, 6, and 7 respectively.

**Pause enforcement mechanism — two distinct patterns:**

Functions with an explicit `whenNotPaused` modifier (`registerMarket`, `deregisterMarket`, `collectTradeCost`, `transfer`) revert with `EnforcedPause()` directly.

`deposit` and `withdraw` are paused via the ERC-4626 max-override pattern: `maxDeposit()` and `maxWithdraw()` return 0 when paused. The ERC-4626 base implementations check these limits before executing and revert with `ERC4626ExceededMaxDeposit(owner, assets, max)` and `ERC4626ExceededMaxWithdraw(owner, assets, max)` respectively — not with `EnforcedPause()`. Tests assert the correct typed selector with exact arguments derived from `pool.maxDeposit`/`pool.maxWithdraw`.

| Test | What it verifies |
|---|---|
| `test_pause_succeeds_byPauser` | `pause()` works with PAUSER_ROLE |
| `test_unpause_succeeds_byAdmin` | `unpause()` works with DEFAULT_ADMIN_ROLE |
| `test_unpause_reverts_byPauserOnly` | PAUSER_ROLE cannot unpause — asymmetry enforced. Uses `vm.expectRevert` then `vm.prank(pauserOnly)` (not `startPrank`). In some Foundry versions, calling `vm.expectRevert` while `vm.startPrank` is already active causes the framework to consume the prank context during cheatcode processing, so `pool.unpause()` runs from the test contract rather than `pauserOnly`. Correct ordering: `vm.expectRevert` first (no prank active), then `vm.prank` queued as a single-call prank consumed only by `pool.unpause()` |
| `test_pause_reverts_byNonPauser` | Non-pauser blocked |
| `test_pause_blocksDeposit` | `ERC4626ExceededMaxDeposit(lp2, 1000e6, 0)` — max-override path |
| `test_pause_blocksWithdraw` | `ERC4626ExceededMaxWithdraw(lp, 1000e6, 0)` — max-override path |
| `test_pause_blocksRegisterMarket` | `EnforcedPause()` — explicit modifier |
| `test_pause_blocksTokenTransfer` | bLP transfer blocked via `_update` override |
| `test_pause_blocksCollectTrade` | `EnforcedPause()` — explicit `whenNotPaused` on `collectTradeCost` |
| `test_pause_maxDeposit_returns0` | `maxDeposit` returns 0 when paused |
| `test_pause_maxMint_returns0` | `maxMint` returns 0 when paused |
| `test_pause_maxWithdraw_returns0` | `maxWithdraw` returns 0 when paused |
| `test_pause_maxRedeem_returns0` | `maxRedeem` returns 0 when paused |

---

### Section 10 — ERC-4626 LP Vault (`BlieverV1Pool_ERC4626Test`)

Tests the LP deposit/withdraw/redeem cycle and the constrained max functions.

**`test_withdraw_reverts_beyondFreeLiquidity`** asserts the specific OZ v5 `ERC4626ExceededMaxWithdraw(address owner, uint256 assets, uint256 max)` selector with exact arguments derived from `pool.maxWithdraw(lp)` — bare `vm.expectRevert()` is not used.

| Test | What it verifies |
|---|---|
| `test_deposit_mintsShares` | Shares minted non-zero; balance matches |
| `test_deposit_increasesTotalAssets` | `totalAssets` grows by deposit |
| `test_withdraw_burnsSharesAndReturnsUSDC` | LP receives USDC on withdrawal |
| `test_redeem_burnsSharesAndReturnsUSDC` | Redeem path works |
| `test_maxWithdraw_cappedByFreeLiquidity` | `maxWithdraw ≤ availableLiquidity` |
| `test_maxWithdraw_zeroWhenFullyUtilised` | 8 markets fills `activeCap`; maxWithdraw = 0 |
| `test_withdraw_reverts_beyondFreeLiquidity` | `ERC4626ExceededMaxWithdraw` with exact selector and args |
| `test_decimals_returns18` | 6 USDC + 12 DECIMALS_OFFSET |
| `test_totalAssets_equalsRawUSDCBalance` | `totalAssets() == usdc.balanceOf(pool)` |

---

### Section 11 — View Functions (`BlieverV1Pool_ViewFunctionsTest`)

Tests all public view functions: `availableLiquidity`, `utilizationBps`, `nav`, `isSolvent`, `isActiveMarket`, `getMarketInfo`.

**`utilizationBps` edge cases:** Both explicit contract branches (lines 793–795) are covered:
- `test_utilizationBps_zeroWhenNoAssets` exercises the `if (assets == 0) return 0` path with a fresh pool that has no deposits.
- `test_utilizationBps_fullUtilization` registers 8 markets (400K = 400K activeCap) → 10 000 bps.

**`nav()` floor:** `test_nav_returnsZero_whenLiabilityExceedsAssets` uses `deal` to drain vault USDC below `totalLiability`, confirming `nav()` returns 0 rather than underflowing. `test_nav_remainsPositive_whenFullyUtilised` separately verifies that at full market utilisation, the reserve buffer keeps NAV > 0.

Key formula verified:  
`free = totalAssets − totalLiability − (totalAssets × reserveBps / BPS_BASE)`

| Test | What it verifies |
|---|---|
| `test_availableLiquidity_noMarkets` | Correct formula with zero liability |
| `test_availableLiquidity_decreasesAfterRegister` | Decreases by exactly `riskBudget` |
| `test_availableLiquidity_increasesAfterTrade` | Rises when `currentLiability` decreases |
| `test_utilizationBps_zero_whenNoLiability` | Zero liability → 0 bps |
| `test_utilizationBps_correct_afterRegister` | 50K / 400K × 10000 = 1250 bps |
| `test_utilizationBps_zeroWhenNoAssets` | `assets == 0` branch returns 0 |
| `test_utilizationBps_fullUtilization` | 8 markets → 10 000 bps |
| `test_nav_correct` | NAV = totalAssets − totalLiability |
| `test_nav_decreasesAfterRegister` | NAV = LP_DEPOSIT − MAX_RISK |
| `test_nav_remainsPositive_whenFullyUtilised` | NAV > 0 at full utilisation (reserve buffer) |
| `test_nav_returnsZero_whenLiabilityExceedsAssets` | `deal` drains pool; `nav()` returns 0 |
| `test_isSolvent_trueAlways` | `isSolvent()` true with active markets |
| `test_isActiveMarket_trueAfterRegister/falseAfterSettle/falseForUnregistered` | State machine |
| `test_getMarketInfo_returnsFullStruct` | All 7 `MarketInfo` fields verified |

---

### Section 12 — UUPS Upgrade (`BlieverV1Pool_UpgradeTest`)

Tests that:
- Non-`UPGRADER_ROLE` callers cannot upgrade (reverts with `AccessControlUnauthorizedAccount`)
- `UPGRADER_ROLE` can upgrade successfully
- State (registered markets, `totalLiability`) is preserved across an upgrade

Uses ERC-1967 implementation slot (`0x3608...`) to verify the proxy's stored implementation address after upgrade.

---

### Section 13 — Fuzz Tests (`BlieverV1Pool_FuzzTest`)

Property-based tests with randomised inputs covering the core mathematical and accounting properties.

| Fuzz Test | Property |
|---|---|
| `testFuzz_deposit_sharesAlwaysGtZero` | Deposit of any valid USDC amount mints non-zero shares |
| `testFuzz_collectTradeCost_liabilityTrackedCorrectly` | `totalLiability == newLiability` after single-market trade across full [0, MAX_RISK] range (covers zero-liability boundary) |
| `testFuzz_collectTradeCost_capNeverExceedsRiskBudget` | Overbudget reports are capped; stored value ≤ `riskBudget` |
| `testFuzz_freeLiquidity_neverNegative` | `availableLiquidity` never underflows; `totalLiability ≤ totalAssets` |
| `testFuzz_settleMarket_profitNonNegative` | Settlement always leaves vault solvent for any valid payout in [0, MAX_RISK] |
| `testFuzz_setReserveBps_boundsEnforced` | Any `uint16`: valid range accepted, outside range rejected |
| `testFuzz_setAlpha_boundsEnforced` | Any value in [0, 1e18]: valid range accepted, outside range rejected |

---

## How to Run

```bash
# All unit + fuzz tests
forge test --match-path "test/BlieverV1Pool.t.sol" -vvv

# Specific section
forge test --match-contract "BlieverV1Pool_SettleMarketTest" -vvv

# Only fuzz tests
forge test --match-contract "BlieverV1Pool_FuzzTest" -vvv

# Coverage report
forge coverage --match-path "test/BlieverV1Pool*.t.sol" --report summary --ir-minimum

# Gas snapshot for hot paths
forge test --match-path "test/BlieverV1Pool*.t.sol" --gas-report
```

---

## Extending These Tests

**`vm.prank` ordering rule — required for all access-control revert tests:**

`vm.prank(caller)` applies to the very next external call, including `staticcall` (view functions). When `pool.ROLE()` is evaluated as an argument to `vm.expectRevert(...)` while a prank is active, that staticcall consumes the prank. The function under test then executes from the test contract, not `caller`, producing the wrong address in the `AccessControlUnauthorizedAccount` error.

Always use this order:
```solidity
// Correct: role() resolved before prank is set
vm.expectRevert(_accessDenied(attacker, pool.SOME_ROLE()));
vm.prank(attacker);
pool.guardedFunction();
```

When a role value is needed as an argument to a function (not just to `vm.expectRevert`), cache it before setting the prank:
```solidity
bytes32 role = pool.SOME_ROLE();   // resolve while no prank is active
vm.prank(admin);
pool.grantRole(role, recipient);   // safe — no staticcall in args
```

- **New admin parameter:** Add a `test_set<Param>_*` block in `BlieverV1Pool_AdminParamsTest` following the existing pattern (happy path, boundary min/max, tooLow, tooHigh, unauthorized, isolation from existing markets).
- **New market exit path:** Add coverage in `BlieverV1Pool_SettleMarketTest` and verify `MARKET_ROLE` revocation. Identify whether zero or non-zero payout is needed to keep the role alive for the revert-path test.
- **New ERC-4626 operation:** Add to `BlieverV1Pool_ERC4626Test`; always assert the specific OZ error selector with exact arguments, not bare `vm.expectRevert()`.
- **LSMath integration:** Once market contracts are available, replace `MockMarket.doCollectTrade` calls with real market logic to test end-to-end trade cost calculations.

---

## Known Scope Boundaries

| Out of scope | Reason |
|---|---|
| LSMath library correctness | Separate contract; tested independently |
| Market contract logic | Not yet developed (dev phase) |
| Oracle resolution | External dependency; mocked via MockMarket |
| `AccountingInvariantViolated` error trigger | Requires direct storage manipulation — no normal execution path corrupts `claimedPayout > settledPayout` by design |
| Fork tests against Base mainnet | Deferred until contracts are deployed |
