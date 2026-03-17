# BlieverV1Pool — Architecture & Theory

> **Audience:** Analysts, auditors, researchers, and protocol designers who want to understand *what* the vault does and *why* it is designed this way.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The LS-LMSR Foundation](#2-the-ls-lmsr-foundation)
3. [Why a Single Global Vault?](#3-why-a-single-global-vault)
4. [Capital Flow Architecture](#4-capital-flow-architecture)
5. [Risk Model and Loss Bound](#5-risk-model-and-loss-bound)
6. [LP Share Token (bLP)](#6-lp-share-token-blp)
7. [Market Lifecycle](#7-market-lifecycle)
8. [Security Architecture](#8-security-architecture)
9. [Upgradeability Strategy](#9-upgradeability-strategy)
10. [Design Trade-offs and Constraints](#10-design-trade-offs-and-constraints)

---

## 1. Executive Summary

**BlieverV1Pool** is a single, upgradeable USDC vault that simultaneously acts as the Automated Market Maker (AMM) for every Believer prediction market. It is the only contract in the system that ever holds real USDC.

| Property | Value |
|---|---|
| Contract name | `BlieverV1Pool` |
| LP token | **bLP** (Believer Liquidity Provider) |
| Underlying asset | USDC (Base-chain canonical, 6 decimals) |
| bLP decimals | 18 |
| AMM algorithm | LS-LMSR (Liquidity-Sensitive LMSR) |
| Standard | ERC-4626 tokenised vault |
| Upgradeability | UUPS proxy (ERC-1822) |
| Chain | Base (any EVM-compatible) |

The vault is **not** a constant-product AMM (Uniswap-style). It is an obligation-tracking market maker where:

- Every trade **pays into** the vault.
- Liquidity grows **automatically** with volume.
- Worst-case vault loss per market is **mathematically bounded** by a pre-set risk budget R.

---

## 2. The LS-LMSR Foundation

### 2.1 Cost Function

The LS-LMSR cost function (Othman et al., 2013) is:

$$C(\mathbf{q}) = b(\mathbf{q}) \cdot \ln\!\left(\sum_{i=1}^{n} e^{q_i / b(\mathbf{q})}\right)$$

where:

- **q** = obligation vector (shares outstanding per outcome)
- **b(q) = α · Σqᵢ** — the *variable* liquidity parameter
- **α** = commission constant (e.g. 0.03 for 3%)

### 2.2 Why Variable Liquidity is Important

In a fixed-b LMSR, the market maker charges the same spread regardless of trading volume. In LS-LMSR, as total volume Σqᵢ grows, b(q) grows, prices become less elastic, and the effective spread narrows. **Volume creates its own liquidity** — no manual re-capitalisation is needed.

### 2.3 Key Properties Used by the Vault

| Property | Formal Reference | Vault Implication |
|---|---|---|
| C(q) ≥ max(qᵢ) | Lemma 4.5 | Payout always ≤ money collected |
| Loss ≤ C(q⁰) = R | Proposition 4.9 | Per-market loss is bounded by R |
| C(γq) = γ·C(q) | Proposition 4.10 | Scale-invariant; currency-independent |
| Path independence | §3.5 | Order of trades doesn't matter |

### 2.4 Synthetic Initialisation Vector

Every market starts with **q⁰ = (ε, ε, …, ε)** where:

$$\varepsilon = \frac{R}{1 + \alpha \cdot n \cdot \ln(n)}$$

This choice ensures **C(q⁰) = R exactly** — so from block 0 the market's worst-case loss is exactly the risk budget R. These are *accounting entries only*; no USDC moves. The vault pre-reserves R at market registration to cover this initial liability.

---

## 3. Why a Single Global Vault?

### 3.1 Problem with Per-Market Pools

If every market had its own USDC pool:

- Liquidity is fragmented → thin markets early.
- Capital efficiency is poor — each market holds its own USDC idle.
- LP management overhead is enormous (100s of pool tokens).
- Flash loan and composability surface grows unmanageably.

### 3.2 The Global Pool Advantage

A single vault means:

- **Shared liquidity** — all markets draw on the same USDC base.
- **Capital efficiency** — uncorrelated markets can use the same capital (only one wins at a time).
- **Single LP token** — LPs hold one bLP representing exposure to all markets.
- **Simple composability** — one ERC-4626 vault, one standard interface.
- **Guaranteed solvency** — vault-level capacity check ensures total liability ≤ assets × (100% − reserveBps).

### 3.3 Architecture Diagram

```
BlieverV1Pool (single USDC vault — the only contract holding real USDC)
│
├── Market Contract #1  (NYC Mayor 2026)
│   ├─ stores q-vector (synthetic accounting only)
│   ├─ stores α, R (from vault at registration)
│   └─ calls vault: collectTradeCost / distributeRefund / settleMarket / claimWinnings
│
├── Market Contract #2  (US Presidential Election 2028)
│   └─ (same structure)
│
└── Market Contract #N  (up to 10 000 simultaneously active)
```

**Markets are pure math wrappers**. They track who owns what shares. The vault is the settlement layer.

---

## 4. Capital Flow Architecture

### 4.1 LP Deposit / Withdrawal

```
LP deposits USDC
    │
    ▼
BlieverV1Pool.deposit(assets, receiver)
    │  Vault mints bLP shares proportional to TVL
    │  ERC-4626: shares = assets × totalSupply / totalAssets
    ▼
LP holds bLP (appreciates as vault earns spread from markets)

LP redeems bLP
    │
    ▼
BlieverV1Pool.redeem(shares, receiver, owner)
    │  Limited to availableLiquidity = totalAssets − totalLiability
    ▼
LP receives USDC
```

### 4.2 Trade Flow

#### Buy trade

```
Trader wants to buy outcome-i shares in Market #1

1. Market #1 computes:
      cost       = C(q_new) − C(q_old)   [in USDC, 6-dec]
      newLiab    = LSMath.calculateWorstCaseLoss(q_new, q0, α)

2. Trader approves vault: USDC.approve(vault, cost)

3. Market #1 calls:
      vault.collectTradeCost(trader, cost, newLiab)

4. Vault (atomically in one call):
      a. Computes delta = newLiab − old currentLiability
      b. Applies delta to totalLiability  ← live liability tracking
      c. updates markets[market].currentLiability = min(newLiab, riskBudget)
      d. sets hasTrades = true
      e. safeTransferFrom(trader → vault, cost USDC)
      f. emits TradeCostCollected + MarketLiabilityUpdated (if delta ≠ 0)
```

#### Sell trade

```
Trader wants to sell outcome-i shares back to Market #1

1. Market #1 computes:
      refundAmount = C(q_old) − C(q_new)   [in USDC, 6-dec]
      newLiab      = LSMath.calculateWorstCaseLoss(q_new, q0, α)

2. Market #1 calls:
      vault.distributeRefund(trader, refundAmount, newLiab)
      (trader does NOT need a USDC approval — the vault pushes, not pulls)

3. Vault (atomically in one call):
      a. Computes delta = newLiab − old currentLiability
      b. Applies delta to totalLiability  ← live liability tracking
      c. updates markets[market].currentLiability = min(newLiab, riskBudget)
      d. sets hasTrades = true (defensive — sell confirms market activity)
      e. safeTransfer(vault → trader, refundAmount USDC)
      f. emits RefundDistributed + MarketLiabilityUpdated (if delta ≠ 0)
      g. calls _assertSolvent() — vault balance must still cover totalLiability
```

**Key insight — buy side:** As volume grows, `newLiab < old` on most trades → `totalLiability` shrinks → LP NAV rises. The vault becomes healthier with every trade, not just at settlement.

**Key insight — sell side:** When a trader sells, the vault pushes USDC out. The q-vector partially reverts toward q⁰, which may raise or lower the worst-case loss bound depending on market skew. The `_assertSolvent()` check at the end of every `distributeRefund` call guarantees the balance never falls below total liabilities regardless of the liability direction.

### 4.3 Settlement Flow

```
Oracle resolves Market #1 → Outcome i wins

1. Market #1 calls:
      vault.settleMarket(totalPayout)
      where totalPayout = q[i]  (USDC owed to all outcome-i holders)

2. Vault:
      a. marks market as settled
      b. sets settledPayout = totalPayout
      c. releases currentLiability (live value) from totalLiability   [unchecked — invariant proven]
      d. emits MarketSettled(market, totalPayout, profit=riskBudget−totalPayout)  [unchecked — check-gated]
      e. emits MarketExpiredUntraded(market, riskBudget) if no trades ever occurred
      f. if totalPayout == 0: revokes MARKET_ROLE, emits MarketFullyClaimed(market, 0)
      g. calls _assertSolvent() — accounting sanity check

3. Each winner (if totalPayout > 0) calls market.claim() → market calls:
      vault.claimWinnings(winner, amount)

4. Vault:
      a. checks claimedPayout + amount ≤ settledPayout
      b. increments claimedPayout
      c. safeTransfer(vault → winner, amount USDC)
      d. emits WinningsClaimed
      e. when claimedPayout == settledPayout:
            revokes MARKET_ROLE from market contract
            emits MarketFullyClaimed(market, totalPaid)
```

### 4.4 Emergency Settlement Flow

```
Oracle has permanently failed for Market #1

Admin calls:
      vault.forceSettleMarket(market)

Vault:
      a. releases currentLiability from totalLiability (worst-case loss absorbed into LP NAV)
      b. marks market as settled with settledPayout = 0 (winners receive nothing)
      c. revokes MARKET_ROLE from market contract
      d. emits MarketForceSettled(market, lossAbsorbed)
      e. calls _assertSolvent() — accounting sanity check
```

### 4.5 Net Value Flow

| Event | Vault USDC Balance | totalLiability | LP NAV Effect |
|---|---|---|---|
| LP deposits $1000 | +1000 | 0 | +1000 |
| Market registered (R=$1) | 0 | +1 | −1 reserved |
| Trade A (buy): cost $0.10, liability $0.90 | +0.10 | −0.10 (live delta) | +0.20 |
| Trade B (buy): cost $0.05, liability $0.80 | +0.05 | −0.10 (live delta) | +0.15 |
| Trade C (sell): refund $0.03, liability $0.85 | −0.03 | +0.05 (live delta) | −0.08 |
| Market settles, payout $0.70 | −0.70 | −0.85 released | profit absorbed |

`totalLiability` tracks the live loss bound, not the fixed riskBudget. Every trade that narrows the worst-case loss immediately improves LP NAV.

---

## 5. Risk Model and Loss Bound

### 5.1 Per-Market Bound

From LS-LMSR Proposition 4.9:

$$\text{Loss}_{\text{market}} = C(\mathbf{q}^0) - C(\mathbf{q}_{\text{final}}) + \max_i q_i \leq C(\mathbf{q}^0) = R$$

The vault sets C(q⁰) = R = `maxRiskPerMarket` by construction. This is a **hard mathematical guarantee** — no combination of trades can make the vault lose more than R per market, regardless of which outcome wins.

### 5.2 Reserve Buffer Model

The vault splits its total assets into two permanent zones on every `registerMarket` call:

```
Total vault assets (100%)
│
├── Active Capital = assets × (100 % − reserveBps)
│     Markets draw from here — totalLiability must stay within this ceiling.
│
└── Reserve Buffer = assets × reserveBps
      Always free — guarantees LP withdrawals are never fully blocked.
```

**The single registration check:**
$$\text{totalLiability} + R \leq \text{assets} \times \frac{BPS\_BASE - reserveBps}{BPS\_BASE}$$

With `reserveBps = 2000` (20%), the active capital ceiling is 80% of assets. One computation, one comparison — semantically complete and non-redundant.

**Admin control over `reserveBps` ([MIN_RESERVE_BPS=5%, MAX_RESERVE_BPS=50%]):**

| Scenario | Direction | Rationale |
|---|---|---|
| Vault consistently profitable, volume high | Lower (e.g. 20%→10%) | Idle capital is waste; liabilities shrink fast |
| Election season / volatile events | Raise (e.g. 20%→40%) | LP protection under uncertainty |
| LP redemption pressure elevated | Raise | Ensures withdrawals remain available |
| New unproven market types | Raise | Conservative mode during uncertainty |

For example, with `reserveBps = 2000` (20% reserve):
- Active capital ceiling = 80% of assets.
- With $1M TVL and $1 risk per market: up to 800,000 markets can register before the vault is full.
- LPs can always withdraw at least the 20% reserve buffer regardless of market count.

### 5.3 Solvency Invariant

The vault enforces one hard invariant at all times:

$$\text{rawBalance} \geq \text{totalLiability}$$

`_assertSolvent()` is called at the end of `settleMarket` and `distributeRefund` to verify this on-chain. `isSolvent()` is a public view function for off-chain monitoring. The invariant cannot be violated through normal operation paths — its purpose is to surface accounting bugs immediately rather than silently corrupting state.

### 5.4 Outcome-Independent Profit Region

When sufficient volume has traded on a market, the vault enters an **outcome-independent profit** zone:

$$\mathfrak{R}(\mathbf{q}) = C(\mathbf{q}) - \max_i q_i - C(\mathbf{q}^0) > 0$$

In this zone, the vault profits regardless of which outcome resolves. This is the expected long-run state for all active markets (Othman et al., §4.5).

---

## 6. LP Share Token (bLP)

### 6.1 Token Properties

| Feature | Implementation |
|---|---|
| Standard | ERC-4626 + ERC-20 |
| Name | Believer LP |
| Symbol | bLP |
| Decimals | 18 |
| Underlying | USDC (6 dec) |
| Mintable | Yes — via ERC-4626 `deposit` / `mint` |
| Burnable | Yes — via ERC-4626 `redeem` / `withdraw` (internally calls `_burn`) |
| Permit | Yes — EIP-2612 gasless approvals |
| Pausable | Yes — all transfers halt during emergencies |

Standalone `burn()` (ERC20Burnable) and `flashLoan()` (ERC-3156) are intentionally excluded. ERC-4626 `redeem`/`withdraw` cover all legitimate burning scenarios, and flash-minting bLP introduces unnecessary complexity with no concrete V1 use case.

### 6.2 Share Price Mechanics

Share price is determined by the ERC-4626 exchange rate:

$$\text{price per bLP} = \frac{\text{totalAssets (USDC)}}{\text{totalSupply (bLP)}}$$

As markets collect more USDC than they pay out (spread profit), `totalAssets` increases while `totalSupply` stays constant → **share price appreciates**. LPs profit passively by holding bLP.

### 6.3 Decimal Offset (Inflation Attack Protection)

bLP uses an 18-decimal representation with a 12-decimal offset over USDC (6 dec):

$$\text{bLP decimals} = 6 + 12 = 18$$

The non-zero decimals offset in ERC-4626 provides **virtual share protection**: even with zero total supply, the virtual shares make a first-depositor donation attack economically infeasible without requiring the protocol to burn dead shares to a zero address.

---

## 7. Market Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│                    Market Lifecycle                      │
│                                                          │
│  UNREGISTERED                                            │
│       │                                                  │
│       │  registerMarket(market, nOutcomes)               │
│       │  ── reserves riskBudget in totalLiability        │
│       │  ── grants MARKET_ROLE                           │
│       ▼                                                  │
│   REGISTERED (active)                                    │
│       │         │             │                          │
│       │         │             │ forceSettleMarket(market)│
│       │         │             │ ── admin emergency path  │
│       │         │             │ ── absorbs currentLiab   │
│       │         │             │ ── revokes MARKET_ROLE   │
│       │         │             │ ── _assertSolvent()      │
│       │         │             ▼                          │
│       │         │      FORCE SETTLED                     │
│       │         │                                        │
│       │         │  deregisterMarket(market)              │
│       │         │  ── only if !hasTrades                 │
│       │         │  ── releases riskBudget                │
│       │         ▼                                        │
│       │    DEREGISTERED                                  │
│       │                                                  │
│  collectTradeCost(...)  [buy trades — many times]                │
│  distributeRefund(...)  [sell trades — many times]               │
│  ── totalLiability updated live (delta per trade, both sides)    │
│       │                                                  │
│       │  settleMarket(totalPayout)                       │
│       │  ── releases currentLiability from totalLiability│
│       │  ── emits MarketExpiredUntraded if !hasTrades    │
│       │  ── _assertSolvent() called                      │
│       ▼                                                  │
│   SETTLED                                                │
│       │                                                  │
│  claimWinnings(winner, amount)  [many times]             │
│       │                                                  │
│       │  (all settledPayout claimed)                     │
│       │  ── revokes MARKET_ROLE                          │
│       │  ── emits MarketFullyClaimed                     │
│       ▼                                                  │
│   FULLY CLAIMED                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 8. Security Architecture

### 8.1 Role-Based Access Control

```
DEFAULT_ADMIN_ROLE  (two-step transfer via AccessControlDefaultAdminRules)
    ├── MARKET_MANAGER_ROLE  ──  register / deregister markets
    │       └── administers MARKET_ROLE  (via _setRoleAdmin — on-chain getRoleAdmin is consistent)
    ├── PAUSER_ROLE          ──  pause vault (low-latency ops multisig in production)
    ├── UPGRADER_ROLE        ──  authorize UUPS upgrades
    ├── EMERGENCY_ROLE       ──  force-settle broken/stuck markets (faster-response multisig)
    ├── unpause()            ──  reserved to DEFAULT_ADMIN_ROLE only (asymmetric with pause)
    └── (can grant any role to any address)

MARKET_ROLE (auto-granted per market by MARKET_MANAGER via registerMarket;
             auto-revoked on forceSettleMarket or when all winnings are claimed)
    ├── collectTradeCost()       ← buy trades: pulls USDC in, updates live totalLiability
    ├── distributeRefund()       ← sell trades: pushes USDC out, updates live totalLiability, asserts solvency
    ├── settleMarket()
    └── claimWinnings()          ← role auto-revoked when claimedPayout == settledPayout
```

The vault uses OpenZeppelin `AccessControlDefaultAdminRulesUpgradeable`, which enforces a two-step, minimum-delay process for transferring `DEFAULT_ADMIN_ROLE`. A compromised admin key cannot silently hand control to an attacker without a pending acceptance window.

The initializer explicitly calls `_setRoleAdmin(MARKET_ROLE, MARKET_MANAGER_ROLE)`. Without this, OZ defaults `getRoleAdmin(MARKET_ROLE)` to `DEFAULT_ADMIN_ROLE` for all roles, causing a split between the on-chain metadata and the actual intent: audit tools and external `grantRole` callers would see governance as the owner of MARKET_ROLE, when market managers are. The explicit `_setRoleAdmin` call makes the public role management paths, dashboards, and internal functions all consistent.

All five privileged roles are initially granted to the deployer-specified `admin` address. In production, each role should be held by a distinct multisig or timelock tuned to its required response speed.

### 8.2 Defence-in-Depth

| Threat | Mitigation |
|---|---|
| Re-entrancy in USDC transfer | `nonReentrant` on all state-changing external functions |
| Trader sells shares but vault cannot send refund | `distributeRefund` (MARKET_ROLE) handles sell-side USDC push with full CEI and `_assertSolvent()` |
| Sell drains vault below its liabilities | `_assertSolvent()` at end of every `distributeRefund` reverts if balance < totalLiability |
| Trader can't withdraw locked USDC | `maxWithdraw`/`maxRedeem` enforce `_freeLiquidity` |
| LP drains vault before market loss realises | Reserve buffer (`reserveBps`) anchored to total assets in both `_freeLiquidity` and `registerMarket` |
| Vault over-commits to too many markets | Single capacity check: `newTotalLiab ≤ assets × (100% − reserveBps)` |
| EOA registered as market (MARKET_ROLE abuse) | `registerMarket` reverts with `NotAContract` if `market.code.length == 0` |
| Market over-reports payout | `PayoutExceedsRiskBudget` revert if payout > riskBudget |
| Winners double-claim | `claimedPayout` counter prevents exceeding `settledPayout`; `AccountingInvariantViolated` revert if counter ever exceeds budget |
| Oracle permanently fails / market stuck | `forceSettleMarket` (EMERGENCY_ROLE) emergency path clears liability |
| Accounting drift undetected | `_assertSolvent()` called after every settlement (normal and emergency) |
| Bad market contract registered | `MARKET_MANAGER_ROLE` guard + protocol governance |
| Smart contract upgrade risk | UUPS + `UPGRADER_ROLE` + `_disableInitializers()` |
| Emergency circuit breaker | `pause()` (PAUSER_ROLE) halts deposits, withdrawals, trades, transfers |
| Compromised pauser defeats its own pause | `unpause()` requires DEFAULT_ADMIN_ROLE — asymmetric with `pause()` |
| Admin key compromised silently | `AccessControlDefaultAdminRules` enforces two-step transfer + minimum delay for DEFAULT_ADMIN_ROLE |
| Audit tools / dashboards show wrong MARKET_ROLE owner | `_setRoleAdmin(MARKET_ROLE, MARKET_MANAGER_ROLE)` in `initialize` makes `getRoleAdmin` consistent with operational intent; prevents hidden reverts in external `grantRole` callers |
| Winners locked out during emergency pause | `claimWinnings` is NOT pause-gated — settled payouts are always claimable |
| Settled market retains MARKET_ROLE indefinitely | Role automatically revoked when `claimedPayout == settledPayout` |
| Market Prop 4.9 violation goes undetected | `LiabilityCapApplied` event fired when market reports `newLiability > riskBudget` |
| Share inflation attack | 12-decimal ERC-4626 offset provides virtual protection |

### 8.3 CEI Pattern

Every function that transfers assets follows Checks → Effects → Interactions:

1. **Checks**: validate inputs, roles, state
2. **Effects**: update all storage variables
3. **Interactions**: perform external USDC transfer last

---

## 9. Upgradeability Strategy

### 9.1 UUPS (ERC-1822)

The vault uses the **Universal Upgradeable Proxy Standard**:

- Upgrade logic lives in the **implementation** contract.
- The proxy is a minimal forwarding contract (lower deployment cost vs. Transparent Proxy).
- `_authorizeUpgrade()` is gated by `UPGRADER_ROLE` — no upgrade without explicit authorisation.
- `_disableInitializers()` in the implementation constructor prevents direct initialisation attacks.

### 9.2 Storage Safety Rules

The storage layout must **never** be reordered or removed in upgrades. The `__gap` array reserves 45 future slots so new state variables can be appended without collision:

```
Custom storage layout (slots post-inheritance):
  slot 0:  alpha
  slot 1:  maxRiskPerMarket
  slot 2:  reserveBps (uint16 — 2 of 32 bytes used)
  slot 3:  totalLiability
  slot 4:  activeMarketCount
  slot 5:  markets (mapping — pointer, 1 slot)
  slots 6–50:  __gap[45]
```

> **Note on inherited storage:** OZ 5.x uses ERC-7201 namespaced storage for all inherited modules (`AccessControlDefaultAdminRulesUpgradeable`, `ERC4626Upgradeable`, `ERC20PermitUpgradeable`, etc.). Inherited state does **not** occupy sequential slots in the implementation's storage — each module hashes its own deterministic slot. `ReentrancyGuard` in OZ 5.5+ uses transient storage (EIP-1153) and contributes no persistent storage at all.

V2 upgrades may **append** variables before `__gap` and shrink `__gap` accordingly.

---

## 10. Design Trade-offs and Constraints

### 10.1 Decisions Made

| Decision | Rationale |
|---|---|
| Single vault, not per-market | Capital efficiency, simplicity, single LP experience |
| Live `totalLiability` tracks `currentLiability` sum | LP NAV reflects true position; improves with volume |
| Single `reserveBps` capacity check | One parameter controls both the registration ceiling and the LP withdrawal floor simultaneously; no three-tier complexity |
| `forceSettleMarket` emergency path (EMERGENCY_ROLE) | Oracle failures must be recoverable without permanent liability lockup; separate role enables faster-response multisig without needing full admin quorum |
| `_assertSolvent()` at settlement (normal and emergency) | On-chain invariant assertion catches accounting bugs at the earliest possible moment; consistent across both paths |
| `AccessControlDefaultAdminRules` for DEFAULT_ADMIN_ROLE | Two-step transfer with configurable minimum delay prevents silent admin key compromise; required for a vault holding user USDC |
| Asymmetric pause/unpause privilege | `pause()` held by a fast ops multisig (PAUSER_ROLE); `unpause()` requires DEFAULT_ADMIN_ROLE — a compromised pauser key cannot defeat its own pause |
| `EMERGENCY_ROLE` for `forceSettleMarket` | Emergency actions should not require full governance quorum; a purpose-built emergency multisig can act faster with lower coordination cost |
| `utilizationBps()` measured against activeCap | Measuring against total assets (including reserve) misleads operators about available headroom; activeCap gives the real deployable picture |
| `LiabilityCapApplied` event on Prop 4.9 violation | Silent cap hides misbehaving market contracts; emitting an observable event lets monitors detect and investigate anomalies without reverting trades |
| `AccountingInvariantViolated` explicit guard in `claimWinnings` | The `claimedPayout - settledPayout` subtraction is structurally sound but a misrouted upgrade could violate it silently; an explicit check surfaces it as a named error with full context |
| `_setRoleAdmin(MARKET_ROLE, MARKET_MANAGER_ROLE)` in `initialize` | `registerMarket`/`deregisterMarket` use internal `_grantRole`/`_revokeRole` which bypass the admin check and work correctly regardless. Without `_setRoleAdmin`, `getRoleAdmin(MARKET_ROLE)` returns `DEFAULT_ADMIN_ROLE` — audit tools display the wrong owner and any external `grantRole(MARKET_ROLE, x)` call reverts unless made by DEFAULT_ADMIN_ROLE. The single line aligns on-chain metadata with actual operational intent |
| No ERC20Burnable standalone `burn()` | ERC-4626 `redeem`/`withdraw` handles all legitimate burn scenarios; standalone burn adds confusion |
| No ERC-3156 Flash Mint on bLP | No concrete V1 use case; bLP flash-mint can temporarily inflate `totalSupply` creating accounting edge cases |
| UUPS upgradeability kept | Pre-launch code; critical bugs must be fixable. Immutability is earned after external audit, not assumed before it |
| No idle yield in V1 (Compound/Moonwell) | Reduces attack surface; add in V2 with thorough audit |
| Settlement and winner claims NOT pause-gated | Markets must always settle and winners must always be able to claim their verified payouts; blocking claims during a pause would strand funds indefinitely |
| `deregisterMarket` only if no trades | Prevents stranding trader USDC without recourse |
| Separate `collectTradeCost` (buy) and `distributeRefund` (sell) vault endpoints | Keeps buy/sell accounting explicit and independently auditable; sell path carries `_assertSolvent()` while buy path does not — the asymmetry is intentional because only sells reduce the vault balance |
| `_assertSolvent()` called in `distributeRefund` | Sell trades push USDC out and may simultaneously raise `currentLiability`; both effects reduce the solvency margin together — post-interaction check catches any misbehaving market contract in a single call |
| MARKET_ROLE auto-revoked on all exit paths | Removes attack surface after a market is fully closed regardless of payout amount: zero-payout markets are revoked inside `settleMarket`; non-zero payout markets are revoked inside `claimWinnings` on the final claim; force-settled markets are revoked inside `forceSettleMarket` |
| `MarketFullyClaimed` event on full drain | Off-chain indexers and dashboards have a definitive signal that a market is closed |
| `MarketExpiredUntraded` event on zero-trade settlement | Distinguishes genuine market profit from riskBudget release on dead markets; prevents misleading indexer accounting |

### 10.2 Known Limitations

- **No per-market alpha override** — all markets use global alpha. V2 can add per-market overrides.
- **No trade fee revenue** — no fee mechanism is active in this version; fee accrual logic is deferred to a future upgrade.
- **No idle yield** — vault capital earns nothing when markets are sparse.
- **No on-chain market enumeration** — there is no stored list of ever-registered market addresses. Historical enumeration is done off-chain by querying `MarketRegistered` and `MarketSettled` events.
- **LP withdrawal is pro-rated** — if vault is fully utilised, all LPs are equally constrained (no priority queue).
- **`forceSettleMarket` zeroes riskBudget** — winners in a force-settled market receive nothing; this is intentional (admin should only force-settle truly broken markets).

### 10.3 V2 Roadmap Hooks

The `__gap` storage reserve and UUPS upgradeability are designed for:

- Idle-yield strategy (e.g. Moonwell/Compound integration).
- Per-market alpha parameterisation.
- Protocol fee accrual and collection (new state appended via `__gap`).
- Timelocked governance.
- Additional oracle integrations.
- ERC-3156 flash loan on USDC (distinct from bLP flash-mint, with full accounting guards).
