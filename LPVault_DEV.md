# BlieverV1Pool — Developer Reference

> **Audience:** Developers who want to integrate with, audit, debug, improve, or write tests for `BlieverV1Pool.sol`.

---

## Table of Contents

1. [File & Dependency Map](#1-file--dependency-map)
2. [Inheritance Chain](#2-inheritance-chain)
3. [Storage Layout](#3-storage-layout)
4. [Roles & Access Control](#4-roles--access-control)
5. [Constants & Configuration](#5-constants--configuration)
6. [Data Types](#6-data-types)
7. [Errors](#7-errors)
8. [Events](#8-events)
9. [Function Reference](#9-function-reference)
   - [Initializer](#initializer)
   - [Market Management](#market-management)
   - [Trade Flow](#trade-flow)
   - [Settlement Flow](#settlement-flow)
   - [Parameter Administration](#parameter-administration)
   - [Pause Control](#pause-control)
   - [View Functions](#view-functions)
   - [ERC-4626 Overrides](#erc-4626-overrides)
   - [ERC-20 Overrides](#erc-20-overrides)
   - [UUPS Override](#uups-override)
   - [Internal Helpers](#internal-helpers)
10. [Upgrade Guide](#10-upgrade-guide)
11. [Integration Patterns](#11-integration-patterns)
12. [Invariants for Testing](#12-invariants-for-testing)
13. [Potential Gotchas](#13-potential-gotchas)

---

## 1. File & Dependency Map

```
contracts/
├── BlieverV1Pool.sol          ← this contract
└── LSMath.sol                 ← pure LS-LMSR math library (used by market contracts)

OpenZeppelin Upgradeable (^5.x):
├── proxy/utils/Initializable.sol
├── proxy/utils/UUPSUpgradeable.sol
├── token/ERC20/ERC20Upgradeable.sol
├── token/ERC20/extensions/ERC4626Upgradeable.sol
├── token/ERC20/extensions/ERC20PermitUpgradeable.sol
├── access/extensions/AccessControlDefaultAdminRulesUpgradeable.sol
└── utils/PausableUpgradeable.sol

OpenZeppelin Standard:
├── token/ERC20/IERC20.sol
├── token/ERC20/utils/SafeERC20.sol
├── utils/math/Math.sol
└── utils/ReentrancyGuard.sol        ← stateless in OZ 5.5+; no upgradeable variant needed
```

**Note:** `LSMath.sol` is used by market contracts to compute trade costs and liabilities. `BlieverV1Pool` receives the *results* of those calculations — it does not perform LS-LMSR math internally.

---

## 2. Inheritance Chain

```
BlieverV1Pool
 ├─ Initializable
 ├─ ERC4626Upgradeable          extends ERC20Upgradeable
 │                                      └─ IERC4626, IERC20Metadata
 ├─ ERC20PermitUpgradeable      extends ERC20Upgradeable + EIP712Upgradeable
 ├─ PausableUpgradeable
 ├─ AccessControlDefaultAdminRulesUpgradeable   implements IERC165 + two-step admin transfer
 ├─ ReentrancyGuard             (standard; stateless in OZ 5.5+ via transient storage)
 └─ UUPSUpgradeable
```

**C3 resolution rules for overridden functions:**

| Function | Override Specification | Reason |
|---|---|---|
| `decimals()` | `override(ERC20Upgradeable, ERC4626Upgradeable)` | Both declare it; explicit 18 returned |
| `_update()` | `override(ERC20Upgradeable)` + `whenNotPaused` | Adds pause check to all transfers |
| `totalAssets()` | `override(ERC4626Upgradeable)` | Returns raw USDC balance as full LP NAV |
| `maxWithdraw()` | `override(ERC4626Upgradeable)` | Adds liquidity lock + reserve constraint |
| `maxRedeem()` | `override(ERC4626Upgradeable)` | Adds liquidity lock + reserve constraint |
| `maxDeposit()` | `override(ERC4626Upgradeable)` | Returns 0 when paused |
| `maxMint()` | `override(ERC4626Upgradeable)` | Returns 0 when paused |
| `_decimalsOffset()` | `override(ERC4626Upgradeable)` | Returns 12 (bLP = 18 dec) |
| `_authorizeUpgrade()` | `override(UUPSUpgradeable)` | Requires UPGRADER_ROLE |
| `supportsInterface()` | `override(AccessControlDefaultAdminRulesUpgradeable)` | Merges ERC-165 support |

---

## 3. Storage Layout

> ⚠️ **UPGRADE CRITICAL:** Never remove, rename, or reorder any storage variable. Upgrades must only append.

### 3.1 Inherited Storage (handled by OZ namespaced storage in v5)

OZ 5.x uses ERC-7201 namespaced storage for all inherited contracts, so inherited state does NOT occupy sequential slots in the implementation's storage. Each OZ module stores data in a deterministic keccak-derived slot. This prevents collisions between modules.

### 3.2 Custom Storage (sequential, starts after proxy overhead)

| Slot (approx.) | Variable | Type | Description |
|---|---|---|---|
| 0 | `alpha` | `uint256` | LS-LMSR α parameter (18-dec) |
| 1 | `maxRiskPerMarket` | `uint256` | Max USDC loss per market (6-dec) |
| 2 | `reserveBps` | `uint16` | Reserve buffer BPS — packed in 32-byte slot (2 of 32 bytes) |
| 3 | `totalLiability` | `uint256` | Live sum of all active currentLiability values |
| 4 | `activeMarketCount` | `uint256` | Non-settled market count |
| 5 | `markets` | `mapping(address=>MarketInfo)` | Per-market accounting (pointer) |
| 6–50 | `__gap` | `uint256[45]` | Future variable reserve |

### 3.3 MarketInfo Struct Storage (per mapping entry)

Each `markets[addr]` occupies 5 storage slots:

| Struct Slot | Field(s) | Packed Bytes |
|---|---|---|
| 0 | `registered` (bool,1) + `settled` (bool,1) + `hasTrades` (bool,1) | 3/32 bytes (29 free) |
| 1 | `riskBudget` | 32 bytes |
| 2 | `currentLiability` | 32 bytes |
| 3 | `settledPayout` | 32 bytes |
| 4 | `claimedPayout` | 32 bytes |

---

## 4. Roles & Access Control

```solidity
bytes32 DEFAULT_ADMIN_ROLE     = 0x00 (OZ default — two-step transfer via AccessControlDefaultAdminRules)
bytes32 MARKET_MANAGER_ROLE    = keccak256("MARKET_MANAGER_ROLE")
bytes32 MARKET_ROLE            = keccak256("MARKET_ROLE")
bytes32 PAUSER_ROLE            = keccak256("PAUSER_ROLE")
bytes32 UPGRADER_ROLE          = keccak256("UPGRADER_ROLE")
bytes32 EMERGENCY_ROLE         = keccak256("EMERGENCY_ROLE")
```

**Who holds what after `initialize`:**

| Role | Initial Holder | Purpose |
|---|---|---|
| DEFAULT_ADMIN_ROLE | `admin` param (two-step transfer enforced by OZ DefaultAdminRules) | Grant/revoke any role; set params; unpause; force-admin actions |
| MARKET_MANAGER_ROLE | `admin` param | Register/deregister markets |
| PAUSER_ROLE | `admin` param | Pause vault (low-latency ops multisig in production) |
| UPGRADER_ROLE | `admin` param | Authorize UUPS upgrades |
| EMERGENCY_ROLE | `admin` param | Force-settle broken/stuck markets (faster-response multisig in production) |
| MARKET_ROLE | *auto-granted per market by registerMarket; auto-revoked by forceSettleMarket or when claimedPayout == settledPayout* | Call collectTradeCost / distributeRefund / settleMarket / claimWinnings |

**Role admin hierarchy (`getRoleAdmin`):**

OZ `AccessControl` sets `DEFAULT_ADMIN_ROLE` as the implicit admin of every role unless overridden. The initializer explicitly overrides this for `MARKET_ROLE`:

```solidity
_setRoleAdmin(MARKET_ROLE, MARKET_MANAGER_ROLE);
```

| Role | getRoleAdmin(role) | Meaning |
|---|---|---|
| DEFAULT_ADMIN_ROLE | itself (OZ DefaultAdminRules special case) | Two-step transfer only |
| MARKET_MANAGER_ROLE | DEFAULT_ADMIN_ROLE | Governance controls who can manage markets |
| PAUSER_ROLE | DEFAULT_ADMIN_ROLE | Governance controls who can pause |
| UPGRADER_ROLE | DEFAULT_ADMIN_ROLE | Governance controls who can upgrade |
| EMERGENCY_ROLE | DEFAULT_ADMIN_ROLE | Governance controls who can emergency-settle |
| MARKET_ROLE | **MARKET_MANAGER_ROLE** | Market managers control market role grants — consistent with registerMarket/deregisterMarket |

Without this override, `getRoleAdmin(MARKET_ROLE)` would return `DEFAULT_ADMIN_ROLE`, causing audit tools (OpenZeppelin Defender, Tenderly, custom dashboards) to display the wrong owner. Any V2 contract using `grantRole(MARKET_ROLE, x)` directly would also revert unless called from `DEFAULT_ADMIN_ROLE`, creating a hidden footgun.

**Function → Role requirement:**

| Function | Required Role |
|---|---|
| `registerMarket` | MARKET_MANAGER_ROLE |
| `deregisterMarket` | MARKET_MANAGER_ROLE |
| `collectTradeCost` | MARKET_ROLE (msg.sender = market contract) |
| `distributeRefund` | MARKET_ROLE (msg.sender = market contract) |
| `settleMarket` | MARKET_ROLE (msg.sender = market contract) |
| `forceSettleMarket` | EMERGENCY_ROLE |
| `claimWinnings` | MARKET_ROLE (msg.sender = market contract) |
| `setAlpha` | DEFAULT_ADMIN_ROLE |
| `setMaxRiskPerMarket` | DEFAULT_ADMIN_ROLE |
| `setReserveBps` | DEFAULT_ADMIN_ROLE |
| `pause` | PAUSER_ROLE |
| `unpause` | DEFAULT_ADMIN_ROLE |
| `_authorizeUpgrade` | UPGRADER_ROLE |

---

## 5. Constants & Configuration

| Constant | Value | Description |
|---|---|---|
| `BPS_BASE` | `10_000` | 100% denominator for basis points |
| `MIN_RESERVE_BPS` | `500` | 5% absolute floor on `reserveBps` — never lower |
| `MAX_RESERVE_BPS` | `5_000` | 50% absolute ceiling on `reserveBps` — conservative mode cap |
| `MAX_ACTIVE_MARKETS` | `10_000` | Max simultaneously active markets |
| `MIN_ALPHA` | `1e12` | Minimum valid α (prevents LS-LMSR division by zero) |
| `MAX_ALPHA` | `2e17` | Maximum valid α (20% spread ceiling) |
| `DECIMALS_OFFSET` | `12` | bLP(18) − USDC(6) = 12 decimal offset |

**Recommended deployment parameters:**

| Parameter | Suggested Value | Notes |
|---|---|---|
| `alpha` | `3e16` (3%) | Moderate spread, typical prediction market |
| `maxRiskPerMarket` | `1e6` ($1 USDC) | Low risk budget for scalable 10K markets |
| `reserveBps` | `2000` (20%) | 80% active capital, 20% reserve buffer |

---

## 6. Data Types

### `MarketInfo`

```solidity
struct MarketInfo {
    bool     registered;      // Is the market active in the vault?
    bool     settled;         // Has settleMarket() been called?
    bool     hasTrades;       // Has collectTradeCost() been called at least once?
    uint256  riskBudget;      // = maxRiskPerMarket at registration (≡ C(q⁰) = R)
    uint256  currentLiability;// Live worst-case loss = LSMath.calculateWorstCaseLoss(...)
    uint256  settledPayout;   // Total USDC authorized for winners (set by settleMarket)
    uint256  claimedPayout;   // USDC already paid to winners (cumulative)
}
```

**State machine:**

```
registered=F settled=F  →  (does not exist)
registered=T settled=F  →  ACTIVE (hasTrades=F: no trades yet; hasTrades=T: trading)
registered=T settled=T  →  SETTLED (payout may be partially or fully claimed)
```

---

## 7. Errors

| Error | Signature | When Thrown |
|---|---|---|
| `ZeroAddress` | `ZeroAddress()` | address param is address(0) |
| `ZeroAmount` | `ZeroAmount()` | amount param is 0 where prohibited |
| `InvalidAlpha` | `InvalidAlpha(uint256 value)` | α not in [MIN_ALPHA, MAX_ALPHA] |
| `InvalidMaxRisk` | `InvalidMaxRisk(uint256 value)` | maxRiskPerMarket = 0 |
| `InvalidBps` | `InvalidBps(uint16 bps)` | BPS param out of range |
| `InvalidOutcomeCount` | `InvalidOutcomeCount(uint32 count)` | nOutcomes not in [2,100] |
| `MarketAlreadyRegistered` | `MarketAlreadyRegistered(address market)` | registerMarket called twice |
| `MarketNotRegistered` | `MarketNotRegistered(address market)` | market not in registry |
| `MarketAlreadySettled` | `MarketAlreadySettled(address market)` | settleMarket called twice |
| `MarketNotSettled` | `MarketNotSettled(address market)` | claimWinnings before settleMarket |
| `MarketHasTrades` | `MarketHasTrades(address market)` | deregisterMarket after trades |
| `CapacityExceeded` | `CapacityExceeded(uint256 projected, uint256 activeCap)` | `newTotalLiab > assets × (100% − reserveBps)` |
| `NotAContract` | `NotAContract(address account)` | market address has no deployed code (EOA) |
| `PayoutExceedsRiskBudget` | `PayoutExceedsRiskBudget(uint256 payout, uint256 budget)` | settleMarket payout > R |
| `PayoutExceedsSettlement` | `PayoutExceedsSettlement(uint256 requested, uint256 remaining)` | winners claim more than authorized |
| `ExceedsMaxMarkets` | `ExceedsMaxMarkets(uint256 active)` | > 10 000 active markets |
| `VaultInsolvent` | `VaultInsolvent(uint256 balance, uint256 liability)` | raw balance < totalLiability (accounting bug) |
| `AccountingInvariantViolated` | `AccountingInvariantViolated()` | `claimedPayout > settledPayout` detected before subtraction in `claimWinnings` (should never occur in correct operation) |

---

## 8. Events

### `MarketRegistered`
```solidity
event MarketRegistered(address indexed market, uint32 outcomeCount, uint256 riskBudget);
```
Emitted by `registerMarket`. Index by `market` to track per-market state off-chain.

### `MarketDeregistered`
```solidity
event MarketDeregistered(address indexed market);
```
Emitted by `deregisterMarket`.

### `MarketSettled`
```solidity
event MarketSettled(address indexed market, uint256 totalPayout, uint256 profit);
```
- `totalPayout` = USDC owed to winners
- `profit` = `riskBudget − totalPayout` (vault spread earned; ≥ 0 always)

### `TradeCostCollected`
```solidity
event TradeCostCollected(address indexed market, address indexed trader, uint256 cost, uint256 newCurrentLiability);
```
Emitted on every buy trade. `newCurrentLiability` is the updated worst-case loss for that market after the buy.

### `RefundDistributed`
```solidity
event RefundDistributed(address indexed market, address indexed trader, uint256 refundAmount, uint256 newCurrentLiability);
```
Emitted on every sell trade inside `distributeRefund`. `refundAmount` is the USDC sent from the vault to the trader. `newCurrentLiability` is the updated worst-case loss after the sell. Index `(market, trader)` to reconstruct a complete per-trader trade history across both buy and sell directions.

### `MarketLiabilityUpdated`
```solidity
event MarketLiabilityUpdated(address indexed market, uint256 oldLiability, uint256 newLiability);
```
Emitted from both `collectTradeCost` and `distributeRefund` when the market's `currentLiability` changes. Off-chain indexers should listen to this to track `totalLiability` decomposed per market across both buy and sell activity.

### `MarketForceSettled`
```solidity
event MarketForceSettled(address indexed market, uint256 lossAbsorbed);
```
- `lossAbsorbed` = the `currentLiability` released from `totalLiability` (absorbed as permanent LP NAV loss).

### `WinningsClaimed`
```solidity
event WinningsClaimed(address indexed market, address indexed winner, uint256 amount);
```
Emitted each time a winner receives USDC.

### `MarketFullyClaimed`
```solidity
event MarketFullyClaimed(address indexed market, uint256 totalPaid);
```
Emitted inside `claimWinnings` when `claimedPayout == settledPayout` — i.e. every winner has been paid and the market is fully closed. At this point MARKET_ROLE is automatically revoked from the market contract. Off-chain indexers should use this event as the definitive "market closed" signal.

### `MarketExpiredUntraded`
```solidity
event MarketExpiredUntraded(address indexed market, uint256 riskBudget);
```
Emitted inside `settleMarket` when `!info.hasTrades` — the market was settled without ever receiving a trade. The full `riskBudget` is freed back into LP NAV. Distinct from `MarketSettled` to prevent misleading "profit" accounting for dead markets in off-chain indexers.

### Parameter update events
```solidity
event AlphaUpdated(uint256 oldAlpha, uint256 newAlpha);
event MaxRiskUpdated(uint256 oldMax, uint256 newMax);
event ReserveBpsUpdated(uint16 oldBps, uint16 newBps);
```

### `LiabilityCapApplied`
```solidity
event LiabilityCapApplied(address indexed market, uint256 reported, uint256 cap);
```
Emitted from both `collectTradeCost` and `distributeRefund` when a market contract reports `newLiability > riskBudget`. This violates the LS-LMSR Proposition 4.9 invariant — the vault silently caps the value to `riskBudget` to maintain solvency, but fires this event so monitors and auditors can immediately identify the misbehaving market. Any occurrence of this event warrants investigation.

---

## 9. Function Reference

### Initializer

#### `initialize`
```solidity
function initialize(
    address usdc,
    address admin,
    uint256 _alpha,
    uint256 _maxRiskPerMarket,
    uint16  _reserveBps
) external initializer
```

**Modifiers:** `initializer` (Initializable — can only be called once, on the proxy)

**Validation:**
- `usdc != address(0)` and `admin != address(0)`
- `_alpha in [1e12, 2e17]`
- `_maxRiskPerMarket > 0`
- `_reserveBps in [MIN_RESERVE_BPS=500, MAX_RESERVE_BPS=5000]`

**Init chain called (must all be called):**
```
__ERC20_init("Believer LP", "bLP")
__ERC4626_init(IERC20(usdc))
__ERC20Permit_init("Believer LP")
__Pausable_init()
__AccessControlDefaultAdminRules_init(0, admin)   ← sets admin + enforces two-step transfer; minDelay=0 at launch
```

**Side effects:** Grants all five privileged roles to `admin` (`DEFAULT_ADMIN_ROLE` via init; remaining four via `_grantRole`). Sets `getRoleAdmin(MARKET_ROLE) = MARKET_MANAGER_ROLE` via `_setRoleAdmin` so on-chain role metadata and the public `grantRole`/`revokeRole` paths are consistent with the operational intent of `registerMarket`/`deregisterMarket`.

> **Note:** `__ReentrancyGuard_init()` and `__UUPSUpgradeable_init()` are **not** called. In OZ 5.x, `ReentrancyGuard` uses transient storage and is stateless; `UUPSUpgradeable` holds no state of its own. Neither requires an init call.

---

### Market Management

#### `registerMarket`
```solidity
function registerMarket(address market, uint32 nOutcomes)
    external onlyRole(MARKET_MANAGER_ROLE) whenNotPaused
```

**Pre-conditions:**
1. `market != address(0)`
2. `market.code.length > 0` (must be a deployed contract, not an EOA)
3. `nOutcomes in [2, 100]`
4. `markets[market].registered == false`
5. `activeMarketCount < MAX_ACTIVE_MARKETS`
6. Capacity: `totalLiability + R ≤ totalAssets() × (BPS_BASE − reserveBps) / BPS_BASE`

**State changes:**
```
markets[market] = MarketInfo{ registered:true, hasTrades:false, ... riskBudget: maxRiskPerMarket, currentLiability: maxRiskPerMarket }
totalLiability += maxRiskPerMarket
activeMarketCount += 1
MARKET_ROLE granted to market
```

**Why `currentLiability = riskBudget` at start:** By LS-LMSR theory, C(q⁰) = R exactly (from the epsilon derivation). The vault is exposed to the full R from block 0.

---

#### `deregisterMarket`
```solidity
function deregisterMarket(address market)
    external onlyRole(MARKET_MANAGER_ROLE) whenNotPaused
```

**Pre-conditions:**
1. `markets[market].registered == true`
2. `markets[market].settled == false`
3. `markets[market].hasTrades == false` ← prevents stranding trader USDC

**State changes:**
```
totalLiability -= info.riskBudget
--activeMarketCount  (reverts on underflow — surfaces accounting bugs)
delete markets[market]
MARKET_ROLE revoked from market
```

---

### Trade Flow

#### `collectTradeCost`
```solidity
function collectTradeCost(address trader, uint256 cost, uint256 newLiability)
    external onlyRole(MARKET_ROLE) nonReentrant whenNotPaused
```

**msg.sender is the market contract** (has MARKET_ROLE).

**Pre-conditions:**
1. `markets[msg.sender].registered == true`
2. `markets[msg.sender].settled == false`
3. `trader != address(0)`

**State changes (before external call — CEI):**
```
capped = min(newLiability, info.riskBudget)

// If cap fires, Prop 4.9 is violated — emit LiabilityCapApplied for monitoring
if newLiability > info.riskBudget: emit LiabilityCapApplied(market, newLiability, riskBudget)

old    = info.currentLiability

// Delta-update totalLiability (live LS-LMSR tracking)
if capped > old:   totalLiability += (capped - old)
elif capped < old: totalLiability -= (old - capped)

info.currentLiability = capped
info.hasTrades = true
```

**`totalLiability` delta logic:** When `newLiability < old` (normal as volume grows), `totalLiability` shrinks → LP NAV rises immediately on each trade. When a market is one-sided early, `newLiability` may momentarily exceed `old` → `totalLiability` rises temporarily.

**External call (after state changes):**
```
IERC20(asset()).safeTransferFrom(trader, address(this), cost)
```

**Gas note:** If `cost == 0`, the USDC transfer is skipped. This handles zero-cost trades (e.g. marginal rebalances that round to zero).

**Re-entrancy analysis:** All state changes (including `totalLiability` update) happen BEFORE `safeTransferFrom`. Even with ERC-777 hooks, vault state is committed. `nonReentrant` adds a second layer.

---

#### `distributeRefund`
```solidity
function distributeRefund(address trader, uint256 refundAmount, uint256 newLiability)
    external onlyRole(MARKET_ROLE) nonReentrant whenNotPaused
```

**msg.sender is the market contract** (has MARKET_ROLE). The sell-side counterpart to `collectTradeCost`.

**Pause-gated** — sell trades must stop when the vault is paused, consistent with `collectTradeCost`.

**Pre-conditions:**
1. `markets[msg.sender].registered == true`
2. `markets[msg.sender].settled == false`
3. `trader != address(0)`

**State changes (before external call — CEI):**
```
capped = min(newLiability, info.riskBudget)

// If cap fires, Prop 4.9 is violated — emit LiabilityCapApplied for monitoring
if newLiability > info.riskBudget: emit LiabilityCapApplied(market, newLiability, riskBudget)

old = info.currentLiability

// Delta-update totalLiability (live LS-LMSR tracking — same logic as collectTradeCost)
if capped > old:   totalLiability += (capped - old)
elif capped < old: totalLiability -= (old - capped)

info.currentLiability = capped
info.hasTrades = true   ← defensive; confirms market has activity even on sell-only paths
```

**External call (after state changes):**
```
IERC20(asset()).safeTransfer(trader, refundAmount)   ← vault PUSHES; no trader approval needed
```

**Post-transfer solvency check:**
```
_assertSolvent()   ← balance must still cover totalLiability after USDC leaves the vault
```

**Gas note:** If `refundAmount == 0`, the USDC transfer is skipped. This handles zero-cost sell ops (e.g. marginal q-vector rebalances that round to zero).

**Why `_assertSolvent()` here but not in `collectTradeCost`:** `collectTradeCost` pulls USDC *into* the vault — the balance can only increase. `distributeRefund` pushes USDC *out*, and a sell may simultaneously raise `currentLiability` (q-vector reverts toward q⁰ under certain skew conditions). Both effects reduce the solvency margin together. `_assertSolvent()` catches any misbehaving market that would leave the vault under-collateralised in a single call.

**Re-entrancy analysis:** All state changes (including `totalLiability` and `currentLiability`) happen BEFORE `safeTransfer`. Even with ERC-777 hooks on a USDC-compatible token, vault state is fully committed. `nonReentrant` adds a second layer.

**Trader approval:** The trader does **not** need a USDC approval for sell trades. The vault already holds the USDC and calls `safeTransfer` (not `safeTransferFrom`). This is the opposite of the buy path.

---

### Settlement Flow

#### `settleMarket`
```solidity
function settleMarket(uint256 totalPayout)
    external onlyRole(MARKET_ROLE) nonReentrant
```

**Not pause-gated** — markets must always be resolvable.

**msg.sender is the market contract.**

**Pre-conditions:**
1. `markets[msg.sender].registered == true`
2. `markets[msg.sender].settled == false`
3. `totalPayout ≤ info.riskBudget` — enforced by `PayoutExceedsRiskBudget`

**State changes:**
```
info.settled = true
info.settledPayout = totalPayout
info.currentLiability = 0

// unchecked — liveLiab ≤ totalLiability by Σ currentLiability invariant
totalLiability -= liveLiab (live value, not riskBudget)

--activeMarketCount  (intentionally checked — underflow surfaces accounting bugs)

// unchecked — riskBudget ≥ totalPayout validated at checks gate
emit MarketSettled(market, totalPayout, riskBudget - totalPayout)

if !hasTrades: emit MarketExpiredUntraded(market, riskBudget)

if totalPayout == 0:
    _revokeRole(MARKET_ROLE, market)   ← zero-payout path; no claim call can ever arrive
    emit MarketFullyClaimed(market, 0)

_assertSolvent() called ← accounting invariant check
```

**No USDC moves here** — only accounting. USDC moves in `claimWinnings`.

**Profit in emitted event:** `riskBudget - totalPayout` is inlined directly into `MarketSettled` inside an `unchecked` block. The check `totalPayout > riskBudget` at the checks gate makes underflow impossible. The difference between worst-case reserved and what was actually owed stays in the vault and increases LP NAV.

**`MarketExpiredUntraded` note:** If `hasTrades == false`, an additional `MarketExpiredUntraded(market, riskBudget)` event is emitted after `MarketSettled`. This signals that the riskBudget was freed with no trade activity — no trader USDC was ever at risk.

**Zero-payout MARKET_ROLE revocation:** When `totalPayout == 0`, `settledPayout = 0` and `claimedPayout` starts at `0`. The equality `claimedPayout == settledPayout` is satisfied before any claim is attempted, but the code enforcing it lives inside `claimWinnings`. No valid call can reach it — `amount == 0` hits `ZeroAmount()` and `amount > 0` hits `PayoutExceedsSettlement` with `remaining = 0`. MARKET_ROLE is therefore revoked inside `settleMarket` for this case, with `MarketFullyClaimed(market, 0)` emitted for indexer consistency.

**`_assertSolvent` note:** Should never revert. If it does, an upstream accounting path has corrupted state. This is an auditing / testing aid, not a runtime guard.

---

#### `forceSettleMarket`
```solidity
function forceSettleMarket(address market)
    external onlyRole(EMERGENCY_ROLE) nonReentrant
```

**Emergency path only.** Use when a market's oracle has permanently failed or the market contract is broken and cannot call `settleMarket` itself. Guarded by `EMERGENCY_ROLE` — a faster-response multisig than `DEFAULT_ADMIN_ROLE` — so urgent action does not depend on full governance quorum.

**Not pause-gated** — emergencies must be handleable at any time.

**Pre-conditions:**
1. `markets[market].registered == true`
2. `markets[market].settled == false`

**State changes:**
```
// unchecked — lossAbsorbed ≤ totalLiability by Σ currentLiability invariant
totalLiability -= lossAbsorbed  ← absorbed as worst-case LP loss

--activeMarketCount  (intentionally checked — underflow surfaces accounting bugs)
info.settled = true
info.settledPayout = 0          ← no winners paid
info.currentLiability = 0
info.riskBudget = 0
MARKET_ROLE revoked from market
emit MarketForceSettled(market, lossAbsorbed)
_assertSolvent() called ← accounting invariant check; consistent with settleMarket
```

**Economic effect:** LP NAV decreases by `lossAbsorbed = info.currentLiability`. This is the cost of unresolvable market risk. Call only when the market is genuinely broken — there is no undo.

**After forceSettleMarket:** The market record remains in `markets` mapping as `settled=true` with zero payouts. `claimWinnings` will revert for any amount because `settledPayout = 0`.

---

#### `claimWinnings`
```solidity
function claimWinnings(address winner, uint256 amount)
    external onlyRole(MARKET_ROLE) nonReentrant
```

**msg.sender is the settled market contract.**

**Not pause-gated** — winners with a verified, settled payout must never be permanently blocked by vault-level operational pauses. Both `settleMarket` and `claimWinnings` are intentionally unpausable so the full resolution flow remains unblocked during emergencies.

**Pre-conditions:**
1. `markets[msg.sender].registered == true`
2. `markets[msg.sender].settled == true`
3. `winner != address(0)`, `amount > 0`
4. `info.claimedPayout <= info.settledPayout` — explicit invariant guard (reverts `AccountingInvariantViolated` if violated)
5. `amount ≤ info.settledPayout - info.claimedPayout`

**State changes (before USDC transfer — CEI):**
```
// Explicit invariant guard — claimedPayout > settledPayout is an accounting bug
if (info.claimedPayout > info.settledPayout): revert AccountingInvariantViolated()

info.claimedPayout += amount
```

**External call:**
```
IERC20(asset()).safeTransfer(winner, amount)
```

**Post-drain role revocation:**
```
if (info.claimedPayout == info.settledPayout):   // only reachable when settledPayout > 0
    _revokeRole(MARKET_ROLE, market)
    emit MarketFullyClaimed(market, info.settledPayout)
```
Once all authorised winnings are paid, MARKET_ROLE is automatically revoked. No admin call needed. For zero-payout markets (`settledPayout == 0`), this branch is unreachable — revocation occurs inside `settleMarket` instead, where `MarketFullyClaimed(market, 0)` is emitted.

**Pull-payment pattern:** Market orchestrates, vault pays. Winners call `market.claim()` → market verifies share balance → market calls `vault.claimWinnings(winner, amount)`.

---

### Parameter Administration

#### `setAlpha(uint256 newAlpha)`
- Range: `[MIN_ALPHA=1e12, MAX_ALPHA=2e17]`
- Affects: markets registered **after** this call only.
- Emits: `AlphaUpdated(old, new)`

#### `setMaxRiskPerMarket(uint256 newMax)`
- Must be > 0.
- Affects: markets registered **after** this call only.
- Emits: `MaxRiskUpdated(old, new)`

#### `setReserveBps(uint16 newBps)`
- Range: `[MIN_RESERVE_BPS=500, MAX_RESERVE_BPS=5000]`
- Controls both the registration capacity ceiling and the LP withdrawal floor simultaneously.
- Lower value → more active capital, more markets permitted, less LP withdrawal headroom.
- Higher value → tighter market capacity, more LP withdrawal headroom.
- Emits: `ReserveBpsUpdated(old, new)`

---

### Pause Control

#### `pause()` / `unpause()`
- `pause()`: Caller must have `PAUSER_ROLE`.
- `unpause()`: Caller must have `DEFAULT_ADMIN_ROLE`.
- **Asymmetric by design:** A compromised `PAUSER_ROLE` key can halt the vault but cannot defeat its own pause by immediately unpausing. Only a DEFAULT_ADMIN_ROLE quorum can resume operations.
- `_update` hook: reverts on ANY token transfer while paused (deposit, withdraw, burn, transfer).
- `collectTradeCost`: gated by `whenNotPaused`.
- `settleMarket`, `forceSettleMarket`, `claimWinnings`: NOT paused — markets resolve and winners claim regardless of vault operational state.

---

### View Functions

#### `availableLiquidity() → uint256`
Returns `_freeLiquidity()`. This is the maximum USDC LPs can collectively withdraw right now: `assets − totalLiability − (assets × reserveBps)`. Both the liability lock and the reserve buffer are anchored to total assets. Because `totalLiability` is live, this value rises organically as markets earn spread.

#### `nav() → uint256`
Returns `totalAssets() − totalLiability`. The net USDC owned by LPs after subtracting all active market loss exposure. Rising NAV signals the vault is accumulating spread from trading activity.

#### `utilizationBps() → uint256`
Returns `totalLiability × 10_000 / activeCap` where `activeCap = totalAssets × (BPS_BASE − reserveBps) / BPS_BASE`.
- Measures against **active capital** (deployable to markets), not total assets. This gives operators an accurate headroom signal: a reading of 7000 means 30% of market-available capital remains, regardless of reserve buffer size.
- 0 = no active markets
- 5000 = 50% of active capital encumbered by live market liabilities
- 10000 = active capital fully encumbered (no new markets possible)
- `type(uint256).max` = reserve buffer consumes all assets (edge case)

Because `totalLiability` tracks live worst-case loss (not fixed riskBudgets), utilisation naturally falls as markets accumulate trading volume.

#### `isSolvent() → bool`
Returns `balanceOf(vault) >= totalLiability`. Off-chain monitors should poll this; it should always be `true`. Equivalent to the `_assertSolvent` internal check without reverting.

#### `getMarketInfo(address market) → MarketInfo`
Returns full struct. All fields zero-valued for unregistered addresses.

#### `isActiveMarket(address market) → bool`
Returns `true` if `registered && !settled`.

---

### ERC-4626 Overrides

#### `totalAssets() → uint256`
```solidity
return IERC20(asset()).balanceOf(address(this));
```
**Why not subtract `totalLiability`?** The USDC backing market liabilities is still LP-owned capital — it just cannot be withdrawn yet. Excluding it from `totalAssets` would incorrectly depress the LP share price.

#### `maxWithdraw(address owner) → uint256`
```solidity
if (paused()) return 0;
return min(
    _convertToAssets(balanceOf(owner), Math.Rounding.Floor),
    _freeLiquidity()
)
```
If the vault is fully utilised, this returns 0 for all LPs — they must wait for markets to settle.

#### `maxRedeem(address owner) → uint256`
```solidity
if (paused()) return 0;
return min(
    balanceOf(owner),
    _convertToShares(_freeLiquidity(), Math.Rounding.Floor)
)
```

#### `maxDeposit / maxMint`
Both return `type(uint256).max` normally, `0` while paused.

---

### ERC-20 Overrides

#### `decimals() → uint8`
Always returns `18`. Explicitly overrides both `ERC20Upgradeable` and `ERC4626Upgradeable` to avoid ambiguity.

#### `_update(from, to, value)`
```solidity
function _update(...) internal override(ERC20Upgradeable) whenNotPaused {
    super._update(from, to, value);
}
```
The `whenNotPaused` modifier blocks all ERC-20 movements (transfers, mints, burns) during pause. This is more efficient than overriding each individual function.

---

### UUPS Override

#### `_authorizeUpgrade(address newImplementation)`
```solidity
function _authorizeUpgrade(address) internal override onlyRole(UPGRADER_ROLE) {}
```
Empty body — the `onlyRole` modifier performs the check. Called automatically by `UUPSUpgradeable.upgradeToAndCall`.

---

### Internal Helpers

#### `_decimalsOffset() → uint8`
Returns `12`. Called by ERC-4626's `decimals()` computation:
```
bLP decimals = USDC.decimals() + _decimalsOffset() = 6 + 12 = 18
```
Also sets the virtual-share count to `10^12` for the inflation-attack protection formula in ERC-4626.

#### `_freeLiquidity() → uint256`
```solidity
uint256 assets  = totalAssets();
uint256 reserve = assets.mulDiv(reserveBps, BPS_BASE, Math.Rounding.Ceil);
uint256 minHeld = totalLiability + reserve;
return assets > minHeld ? assets - minHeld : 0;
```
Applies two constraints anchored to total assets: (1) cannot withdraw below `totalLiability`, (2) `reserveBps` fraction of total assets is always held back as a withdrawal buffer. The reserve is **not** relative to NAV — it is a fixed slice of total assets, making it a real, non-self-referential floor. Used by `maxWithdraw`, `maxRedeem`, `availableLiquidity`.

Because `totalLiability` is live, `_freeLiquidity` rises naturally after every trade that reduces a market's worst-case loss bound.

#### `_assertSolvent()`
```solidity
uint256 rawBalance = IERC20(asset()).balanceOf(address(this));
if (rawBalance < totalLiability) revert VaultInsolvent(rawBalance, totalLiability);
```
Called at the end of `settleMarket`. Should never revert through normal operation — it is an on-chain accounting assertion, not a runtime guard. If it reverts, a code path has incorrectly updated `totalLiability`.

---

## 10. Upgrade Guide

### Upgrade Checklist

- [ ] New implementation compiled with same `pragma solidity 0.8.31`
- [ ] No existing storage variables renamed, reordered, or removed
- [ ] New variables appended BEFORE `__gap` and `__gap` size reduced accordingly
- [ ] `_disableInitializers()` present in new implementation constructor
- [ ] If new initializer needed: use `reinitializer(N)` with incremented version
- [ ] All inherited `__init()` functions still called (add new ones for new OZ modules)
- [ ] `_authorizeUpgrade` still protected by `UPGRADER_ROLE`
- [ ] Tested with `forge test` + `foundry upgrade` fork simulation

### Storage Gap Accounting

Current custom state: 6 variables → `__gap[45]` (conservative; total effectively reserved ≈ 51).

When adding variables in future upgrades:

```solidity
// BEFORE (current):
uint256[45] private __gap;

// AFTER (adding 2 new variables):
uint256 public newVariableA;   // slot 6
uint256 public newVariableB;   // slot 7
uint256[43] private __gap;     // reduced from 45 to 43
```

---

## 11. Integration Patterns

### Market Contract Integration (most important)

```solidity
interface IBlieverV1Pool {
    function collectTradeCost(address trader, uint256 cost, uint256 newLiability) external;
    function settleMarket(uint256 totalPayout) external;
    function claimWinnings(address winner, uint256 amount) external;
    function alpha() external view returns (uint256);
    function maxRiskPerMarket() external view returns (uint256);
    function reserveBps() external view returns (uint16);
}

contract MyMarket {
    IBlieverV1Pool public immutable vault;
    IERC20         public immutable usdc;

    // Market must have MARKET_ROLE granted by vault admin
    // (done automatically by vault.registerMarket)

    function buy(uint256 outcomeIdx, uint256 deltaQ) external {
        // 1. Compute cost via LSMath
        uint256 cost = LSMath.costFunction(q_new, alpha) 
                     - LSMath.costFunction(q_old, alpha);
        uint256 newLiab = LSMath.calculateWorstCaseLoss(q_new, q0, alpha);

        // 2. Update local q-vector (Effects first)
        q[outcomeIdx] += deltaQ;

        // 3. Pull cost AND update live liability atomically
        //    (trader must have approved vault: usdc.approve(address(vault), cost))
        vault.collectTradeCost(msg.sender, cost, newLiab);
        // totalLiability in the vault is now updated to reflect newLiab
    }

    function resolve(uint256 winningOutcome) external onlyOracle {
        // Convert q[winningOutcome] from LSMath 18-dec to 6-dec USDC before calling
        uint256 totalPayout = q[winningOutcome] / 1e12;
        vault.settleMarket(totalPayout);
    }

    function claim() external {
        uint256 payout = shares[msg.sender][winningOutcome];
        shares[msg.sender][winningOutcome] = 0;
        vault.claimWinnings(msg.sender, payout);
    }
}
```

### LP Deposit (Standard ERC-4626)

```solidity
// 1. Approve vault
IERC20(usdc).approve(address(vault), amount);

// 2. Deposit and receive bLP
uint256 shares = vault.deposit(amount, lpAddress);

// 3. Check LP position and available liquidity
uint256 usdcValue    = vault.convertToAssets(vault.balanceOf(lpAddress));
uint256 withdrawable = vault.maxWithdraw(lpAddress);
uint256 currentNAV   = vault.nav();
```

### LP Withdrawal (respects live liability lock + 20% reserve)

```solidity
// Check max withdrawable (accounts for live liability AND 20% reserve floor)
uint256 withdrawable = vault.maxWithdraw(lpAddress);

// Withdraw (reverts if exceeds maxWithdraw)
vault.withdraw(withdrawable, lpAddress, lpAddress);
```

### Off-Chain Monitoring

```solidity
// Poll these to monitor vault health
bool   ok          = vault.isSolvent();        // must always be true
uint256 utilBps    = vault.utilizationBps();   // rising = more markets active
uint256 freeLiq    = vault.availableLiquidity(); // rising = vault earning spread
uint256 netNAV     = vault.nav();              // LP net value
```

---

## 12. Invariants for Testing

The following invariants must hold at **all times**. Use Foundry invariant testing to verify:

```solidity
// Invariant 1: Vault USDC ≥ totalLiability
function invariant_solvency() external {
    uint256 balance = IERC20(vault.asset()).balanceOf(address(vault));
    assertGe(balance, vault.totalLiability());
}

// Invariant 2: totalAssets = raw USDC balance
function invariant_totalAssets() external {
    uint256 balance = IERC20(vault.asset()).balanceOf(address(vault));
    assertEq(vault.totalAssets(), balance);
}

// Invariant 3: totalLiability = Σ currentLiability over all active markets (live tracking)
// Market addresses are tracked off-chain or via the test harness; there is no on-chain list.
function invariant_liabilityAccounting(address[] calldata knownMarkets) external {
    uint256 sum = 0;
    for (uint i = 0; i < knownMarkets.length; i++) {
        BlieverV1Pool.MarketInfo memory info = vault.getMarketInfo(knownMarkets[i]);
        if (info.registered && !info.settled) {
            sum += info.currentLiability;  // live value, not riskBudget
        }
    }
    assertEq(vault.totalLiability(), sum);
}

// Invariant 4: No market payout exceeds its riskBudget; no double-claim
function invariant_noExcessPayout(address[] calldata knownMarkets) external {
    for (uint i = 0; i < knownMarkets.length; i++) {
        BlieverV1Pool.MarketInfo memory info = vault.getMarketInfo(knownMarkets[i]);
        if (info.settled) {
            assertLe(info.settledPayout, info.riskBudget);
            assertLe(info.claimedPayout, info.settledPayout);
        }
    }
}

// Invariant 5: activeMarketCount == count of registered+unsettled markets
function invariant_activeMarketCount(address[] calldata knownMarkets) external {
    uint256 count = 0;
    for (uint i = 0; i < knownMarkets.length; i++) {
        if (vault.isActiveMarket(knownMarkets[i])) count++;
    }
    assertEq(vault.activeMarketCount(), count);
}

// Invariant 6: LP maxWithdraw ≤ availableLiquidity (≤ NAV × 80%)
function invariant_maxWithdrawBound(address lp) external {
    assertLe(vault.maxWithdraw(lp), vault.availableLiquidity());
}

// Invariant 7: bLP totalSupply corresponds to totalAssets (via ERC-4626 exchange rate)
function invariant_exchangeRate() external {
    if (vault.totalSupply() > 0) {
        assertGe(vault.convertToAssets(10**vault.decimals()), 1);
    }
}

// Invariant 8: currentLiability ≤ riskBudget for every active market
function invariant_liabilityBound(address[] calldata knownMarkets) external {
    for (uint i = 0; i < knownMarkets.length; i++) {
        BlieverV1Pool.MarketInfo memory info = vault.getMarketInfo(knownMarkets[i]);
        if (info.registered && !info.settled) {
            assertLe(info.currentLiability, info.riskBudget);
        }
    }
}

// Invariant 9: isSolvent() == true (live view mirror of _assertSolvent)
function invariant_isSolvent() external {
    assertTrue(vault.isSolvent());
}

// Invariant 10: MARKET_ROLE is revoked on every market exit path
// — zero-payout settlement: revoked in settleMarket (settledPayout == 0)
// — normal full-claim:      revoked in claimWinnings (claimedPayout == settledPayout > 0)
// — force-settlement:       revoked in forceSettleMarket
function invariant_roleRevokedOnAllExitPaths(address[] calldata knownMarkets) external {
    for (uint i = 0; i < knownMarkets.length; i++) {
        BlieverV1Pool.MarketInfo memory info = vault.getMarketInfo(knownMarkets[i]);
        // Zero-payout settled: role must be gone immediately after settlement
        if (info.settled && info.settledPayout == 0) {
            assertFalse(vault.hasRole(vault.MARKET_ROLE(), knownMarkets[i]));
        }
        // Non-zero payout fully claimed: role must be gone after last claim
        if (info.settled && info.settledPayout > 0 && info.claimedPayout == info.settledPayout) {
            assertFalse(vault.hasRole(vault.MARKET_ROLE(), knownMarkets[i]));
        }
    }
}

// Invariant 11: vault remains solvent immediately after every distributeRefund call
// (distributeRefund calls _assertSolvent internally; this external check mirrors it)
function invariant_solventAfterRefund() external {
    // isSolvent() is the public mirror of _assertSolvent — always true if accounting is correct
    assertTrue(vault.isSolvent());
}
```

---

## 13. Potential Gotchas

### 1. Trader must approve the VAULT, not the market

```
✗ WRONG:   usdc.approve(marketContract, cost)
✓ CORRECT: usdc.approve(address(vault), cost)
```

The vault calls `safeTransferFrom(trader, vault, cost)` — the trader's approval must be for the vault.

### 2. `collectTradeCost` updates `totalLiability` live — no separate call needed

`collectTradeCost` performs a delta update to `totalLiability` atomically on every trade. Market contracts must not make any separate liability-update call; the liability is committed inside `collectTradeCost` before the USDC transfer.

### 3. `settleMarket` releases `currentLiability`, not `riskBudget`

The accounting on settlement uses the **live** `currentLiability`, not the original `riskBudget`. If a market has accumulated significant trading volume before settlement, `currentLiability < riskBudget`, and `totalLiability` decreases only by the live amount. The difference (`riskBudget - currentLiability`) represents spread the vault has already effectively earned throughout the market's life.

### 4. `totalPayout` in `settleMarket` must be in USDC (6-dec), not LSMath (18-dec)

`q[winningOutcome]` in LSMath uses 18-decimal fixed-point. Market contracts must convert to USDC decimals before calling `settleMarket`:

```solidity
uint256 payoutUSDC = q[winningOutcome] / 1e12;
```

### 5. `registerMarket` requires a deployed contract address AND the capacity check

Two categories of pre-conditions: (0) `market.code.length > 0` (EOA registration is blocked), (1)–(4) administrative checks (zero-address, outcome count, duplicate, market cap). Then one capacity check: `newTotalLiab ≤ totalAssets() × (BPS_BASE − reserveBps) / BPS_BASE`. With default `reserveBps = 2000` this is an 80% ceiling. If the vault has no deposits, the capacity check fails immediately (activeCap = 0).

### 6. `maxWithdraw` returns 0 when vault is at reserve floor — not an error

If `_freeLiquidity()` returns 0 (vault fully utilised or the `reserveBps` floor consuming all available USDC), `maxWithdraw` returns 0 for all LPs. As markets earn spread through trading, `_freeLiquidity` rises automatically and LP withdrawals become available again without any admin action.

### 7. Paused vault: `settleMarket`, `forceSettleMarket`, and `claimWinnings` still work

None of the resolution or payout paths are gated by `whenNotPaused`. Markets can always settle, liabilities can always be cleared, and winners can always claim their verified payouts. This guarantees that an emergency pause cannot permanently strand funds that have already been authorised for distribution.

### 8. `deregisterMarket` after ANY trade is blocked

Once `hasTrades = true`, `deregisterMarket` reverts. For a genuinely broken post-trade market, use `forceSettleMarket` — it is the correct emergency path and clears the liability immediately.

### 9. MARKET_ROLE is automatically revoked when a market is fully closed

Three paths cover all exit states:

| Path | When | Revocation point |
|---|---|---|
| Normal settlement, `totalPayout > 0` | All winners claimed (`claimedPayout == settledPayout`) | Inside `claimWinnings` on the final claim |
| Normal settlement, `totalPayout == 0` | All positions were on losing outcomes; no claim can arrive | Inside `settleMarket` immediately after effects |
| Emergency path | `forceSettleMarket` called by admin | Inside `forceSettleMarket` immediately after effects |

No admin action is needed in any path. The zero-payout case is handled at settlement time because `claimWinnings` would revert on any attempt (`ZeroAmount` if `amount == 0`, `PayoutExceedsSettlement` if `amount > 0` with `remaining = 0`) — the revocation code inside `claimWinnings` is structurally unreachable for this case.

### 10. `forceSettleMarket` sets `settledPayout = 0` — no claims possible

After force-settlement, `claimWinnings` will revert for any amount because `settledPayout = 0`. There is no path for winners to recover funds from a force-settled market. This is by design — only call `forceSettleMarket` for markets whose oracle has genuinely failed.

### 11. `__gap` size must decrease when adding variables

Adding 2 new state variables requires changing `uint256[45] private __gap` to `uint256[43] private __gap`. Forgetting to shrink causes wasted slots (not harmful). Adding variables WITHOUT shrinking causes a storage collision in subsequent upgrades.

### 12. `unpause()` requires DEFAULT_ADMIN_ROLE, not PAUSER_ROLE

`pause()` is intentionally low-privilege (PAUSER_ROLE) for speed. `unpause()` requires DEFAULT_ADMIN_ROLE. Integration scripts that call `unpause` must use the admin multisig, not the ops pauser key.

### 13. `forceSettleMarket` requires EMERGENCY_ROLE

The emergency settlement path is guarded by `EMERGENCY_ROLE`, not `DEFAULT_ADMIN_ROLE`. In production these should be separate multisigs. If EMERGENCY_ROLE is not yet delegated to a separate key, DEFAULT_ADMIN_ROLE holds it by default (granted in `initialize`) and can still execute force-settlement.

### 14. `_grantRole` (internal) vs `grantRole` (public) — different access requirements for MARKET_ROLE

`registerMarket` and `deregisterMarket` use `_grantRole`/`_revokeRole` (the internal `_` variants). Internal variants bypass the `getRoleAdmin` check and always succeed — they are guarded only by the calling function's own modifier (`onlyRole(MARKET_MANAGER_ROLE)`). This is correct.

The public `grantRole(MARKET_ROLE, x)` path uses a different check: `onlyRole(getRoleAdmin(MARKET_ROLE))`. Because the initializer explicitly calls `_setRoleAdmin(MARKET_ROLE, MARKET_MANAGER_ROLE)`, this returns `MARKET_MANAGER_ROLE`, making both paths consistent.

Without `_setRoleAdmin`, `getRoleAdmin(MARKET_ROLE)` would return `DEFAULT_ADMIN_ROLE` (the OZ default for all roles). Any external contract or admin script calling `grantRole(MARKET_ROLE, newMarket)` directly would revert unless the caller held `DEFAULT_ADMIN_ROLE`, creating an invisible discrepancy between on-chain role metadata and actual operational intent.

### 15. `distributeRefund` — trader needs NO USDC approval; vault pushes, not pulls

On a buy trade (`collectTradeCost`) the vault calls `safeTransferFrom(trader → vault, cost)`, so the trader must first approve the vault. On a sell trade (`distributeRefund`) the vault calls `safeTransfer(vault → trader, refundAmount)` — funds flow out of the vault. No ERC-20 approval from the trader is required or possible.

```
Buy:  trader.approve(vault, cost) then market calls vault.collectTradeCost(trader, cost, newLiab)
Sell: no approval needed           then market calls vault.distributeRefund(trader, refund, newLiab)
```

Market contracts must not ask for trader approvals before calling `distributeRefund`.

### 16. `distributeRefund` calls `_assertSolvent()` — `collectTradeCost` does not

`collectTradeCost` always increases the vault balance (USDC flows in), so solvency is structurally preserved. `distributeRefund` decreases the balance (USDC flows out) while the liability update may go in either direction depending on market skew. The `_assertSolvent()` call at the end of `distributeRefund` is the on-chain safety net that prevents a misbehaving market from draining the vault below its total liability in a single call. If it reverts, an upstream accounting path in the market contract is broken and must be investigated immediately.
