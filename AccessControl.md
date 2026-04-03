# BlieverV1Pool — Access Control & Role Architecture

> **Scope:** `BlieverV1Pool.sol` (improved V1)
> **Base:** `AccessControlDefaultAdminRulesUpgradeable` + `PausableUpgradeable` + UUPS
> **Chain:** Base L2

---

## Table of Contents

1. [Foundation — How OZ AccessControl Works](#1-foundation--how-oz-accesscontrol-works)
2. [The Five Roles — Declaration to Runtime](#2-the-five-roles--declaration-to-runtime)
3. [Full Architecture Map](#3-full-architecture-map)
4. [Role Lifecycle — MARKET_ROLE in Detail](#4-role-lifecycle--market_role-in-detail)
5. [Role Admin Hierarchy — The Hidden Graph](#5-role-admin-hierarchy--the-hidden-graph)
6. [minDelay = 0 — What It Means and Why It Matters](#6-mindelay--0--what-it-means-and-why-it-matters)
7. [Production Deployment Checklist](#7-production-deployment-checklist)
8. [Attack Scenarios — Before and After Fixes](#8-attack-scenarios--before-and-after-fixes)

---

## 1. Foundation — How OZ AccessControl Works

Every role in OpenZeppelin AccessControl is just a `bytes32` hash. The entire system
rests on one internal mapping:

```solidity
// Simplified internal OZ storage (ERC-7201 namespaced)
mapping(bytes32 role => mapping(address account => bool)) private _roles;
```

`hasRole(MARKET_ROLE, 0xABC...)` is a boolean lookup. Nothing more.

The architecture is entirely about:
- **Who can grant/revoke** each role
- **Which functions check** which role at runtime

`AccessControlDefaultAdminRulesUpgradeable` layers on top of this by adding special
mechanics specifically for `DEFAULT_ADMIN_ROLE` (the root role) — enforcing a two-step
transfer flow with a configurable mandatory delay.

---

## 2. The Five Roles — Declaration to Runtime

### `DEFAULT_ADMIN_ROLE` — Governance Layer
```solidity
// Built into OZ — value is bytes32(0)
```

**Special mechanics (DefaultAdminRules):**
- Transferring this role requires two separate transactions
- A mandatory `minDelay` must pass between `beginDefaultAdminTransfer()` and `acceptDefaultAdminTransfer()`
- The pending transfer can be cancelled at any time before acceptance

**Functions gated in BlieverV1Pool:**
```
setAlpha()             — change global LS-LMSR α parameter
setMaxRiskPerMarket()  — change per-market USDC risk budget
setReserveBps()        — change LP reserve buffer (BPS)
unpause()              — resume vault after emergency pause
```

**Who should hold it in production:** Governance multisig (slowest, highest quorum)

---

### `MARKET_MANAGER_ROLE` — Operational Layer
```solidity
bytes32 public constant MARKET_MANAGER_ROLE = keccak256("MARKET_MANAGER_ROLE");
```

**Functions gated:**
```
registerMarket(market, nOutcomes)   — whitelist market contract, grant MARKET_ROLE
deregisterMarket(market)            — remove trade-free unsettled market, revoke MARKET_ROLE
```

**Blast radius:** Limited to which market contract addresses receive or lose `MARKET_ROLE`.
Cannot touch USDC, cannot pause, cannot upgrade. Day-to-day ops role.

**Who should hold it in production:** Ops multisig

---

### `MARKET_ROLE` — Hot Path Layer (Contracts, Not Humans)
```solidity
bytes32 public constant MARKET_ROLE = keccak256("MARKET_ROLE");
```

This role is **not held by humans**. It is granted to market contract addresses
automatically on `registerMarket()` and revoked automatically on settlement or deregistration.

**Functions gated:**
```
collectTradeCost(trader, cost, newLiability)  — pull USDC from trader, update liability
settleMarket(totalPayout)                      — finalise resolution, record payout
claimWinnings(winner, amount)                  — push USDC to winner
```

Every USDC movement in the protocol flows through these three functions.
No MARKET_ROLE → no trades, no settlements, no payouts.

**Who holds it:** Market contract addresses (auto-granted/revoked by the vault)

---

### `PAUSER_ROLE` — Circuit Breaker (Asymmetric)
```solidity
bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
```

**Functions gated:**
```
pause()    — halt deposits, withdrawals, trades, token transfers
```

`unpause()` requires `DEFAULT_ADMIN_ROLE` — **NOT this role.**

This asymmetry is intentional and critical: a compromised pauser key can stop
the vault but cannot undo its own pause. Resuming operations requires the
higher-authority governance multisig to explicitly act.

**What pause() blocks:**
- `deposit()` / `mint()` — via `maxDeposit`/`maxMint` returning 0
- `withdraw()` / `redeem()` — via `maxWithdraw`/`maxRedeem` returning 0
- `collectTradeCost()` — `whenNotPaused` guard
- All ERC-20 transfers — `_update()` override with `whenNotPaused`

**What pause() does NOT block:**
- `settleMarket()` — markets must always be resolvable
- `claimWinnings()` — winners with verified payouts must never be locked out
- `forceSettleMarket()` — emergency resolution must always be available

**Who should hold it in production:** Fast ops multisig (low-latency, 2-of-3)

---

### `EMERGENCY_ROLE` — Crisis Resolution
```solidity
bytes32 public constant EMERGENCY_ROLE = keccak256("EMERGENCY_ROLE");
```

**Functions gated:**
```
forceSettleMarket(market)  — settle a broken/oracle-stuck market, absorb liability as LP loss
```

**Why it exists separately from DEFAULT_ADMIN_ROLE:**
`forceSettleMarket` is time-critical — a stuck oracle at 2am cannot wait for a full
governance quorum to assemble. `EMERGENCY_ROLE` held by a smaller, faster-response
multisig (e.g. 2-of-5 vs 5-of-9 governance) decouples emergency speed from
governance security.

**What forceSettleMarket does:**
1. Releases `currentLiability` from `totalLiability` (vault absorbs as worst-case loss)
2. Marks market `settled = true`, zeroes `currentLiability` and `riskBudget`
3. Revokes `MARKET_ROLE` from the market contract
4. Emits `MarketForceSettled(market, lossAbsorbed)`
5. Calls `_assertSolvent()` — verifies vault remains solvent after loss absorption

**Who should hold it in production:** Emergency multisig (smaller quorum than governance)

---

### `UPGRADER_ROLE` — Implementation Control
```solidity
bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
```

**Functions gated:**
```
_authorizeUpgrade(newImplementation)  — approve a new UUPS logic contract
```

**Who should hold it in production:** Timelock contract (not a multisig directly).
A timelock forces a minimum delay between "upgrade proposed" and "upgrade executed",
giving LPs time to exit if they disagree with the change.

---

## 3. Full Architecture Map

```
DEFAULT_ADMIN_ROLE  (governance multisig — slowest, highest quorum)
│  Two-step transfer + minDelay enforced by AccessControlDefaultAdminRules
│
├── setAlpha()
├── setMaxRiskPerMarket()
├── setReserveBps()
├── unpause()
│
├──► MARKET_MANAGER_ROLE  (ops multisig)
│       ├── registerMarket()   ──grants──►  MARKET_ROLE to market address
│       └── deregisterMarket() ─revokes─►  MARKET_ROLE from market address
│
├──► MARKET_ROLE  (market contract addresses — not humans)
│       ├── collectTradeCost()   ◄── called on every trade
│       ├── settleMarket()       ◄── called on oracle resolution
│       └── claimWinnings()      ◄── called per winner claim
│           │
│           └── auto-revokes MARKET_ROLE when market fully settled/drained
│
├──► PAUSER_ROLE  (fast ops multisig — 2-of-3)
│       └── pause()              [unpause requires DEFAULT_ADMIN_ROLE]
│
├──► EMERGENCY_ROLE  (emergency multisig — smaller quorum)
│       └── forceSettleMarket()
│
└──► UPGRADER_ROLE  (timelock contract)
        └── _authorizeUpgrade()
```

---

## 4. Role Lifecycle — MARKET_ROLE in Detail

`MARKET_ROLE` is the only role with a dynamic lifecycle — it is granted and revoked
automatically as part of normal protocol operation.

```
registerMarket(market, nOutcomes)
  └── _grantRole(MARKET_ROLE, market)
        │
        ▼
  [market is now active — can trade, settle, claim]
        │
        ├── collectTradeCost() called on every trade
        │     updates currentLiability, pulls USDC from trader
        │
        ├── settleMarket(totalPayout) called on resolution
        │     sets settled = true, releases currentLiability from totalLiability
        │     if totalPayout == 0:
        │       └── _revokeRole(MARKET_ROLE, market)  ← immediate, no claims possible
        │
        └── claimWinnings(winner, amount) called per winner
              if claimedPayout == settledPayout:
                └── _revokeRole(MARKET_ROLE, market)  ← revoked when fully drained
                    emit MarketFullyClaimed(market, totalPaid)

Alternative paths:
  deregisterMarket(market)       ← only if hasTrades == false
    └── _revokeRole(MARKET_ROLE, market)

  forceSettleMarket(market)      ← EMERGENCY_ROLE, oracle failure
    └── _revokeRole(MARKET_ROLE, market)
```

**Why auto-revocation matters:** Every market contract holding `MARKET_ROLE`
is an active write surface on the vault. A settled market that retains `MARKET_ROLE`
indefinitely is an unnecessary, lingering attack vector. Auto-revocation eliminates it
the moment it is no longer needed.

---

## 5. Role Admin Hierarchy — The Hidden Graph

### What "role admin" means

In OZ AccessControl, every role has an **admin role** — a separate role whose
holders are the only ones who can call the public `grantRole(X, ...)` and
`revokeRole(X, ...)` for role X.

```solidity
// OZ internal mapping
mapping(bytes32 role => bytes32 adminRole) private _roleAdmins;

// Public grant path — checks admin role
function grantRole(bytes32 role, address account)
    public
    onlyRole(getRoleAdmin(role))   // ← admin check
{
    _grantRole(role, account);     // internal — no check
}
```

**Default:** Every role's admin is `DEFAULT_ADMIN_ROLE` unless explicitly set otherwise.

### The Fix Applied — `_setRoleAdmin`

```solidity
// In initialize(), after role grants:
_setRoleAdmin(MARKET_ROLE, MARKET_MANAGER_ROLE);
```

**Before this fix:**

| Who can publicly grant/revoke MARKET_ROLE? | DEFAULT_ADMIN_ROLE (the default) |
|--------------------------------------------|----------------------------------|
| What getRoleAdmin(MARKET_ROLE) returns      | DEFAULT_ADMIN_ROLE               |
| What audit tools show                       | "Governance controls markets"    |
| Reality                                     | Ops controls markets             |

**After this fix:**

| Who can publicly grant/revoke MARKET_ROLE? | MARKET_MANAGER_ROLE              |
|--------------------------------------------|----------------------------------|
| What getRoleAdmin(MARKET_ROLE) returns      | MARKET_MANAGER_ROLE              |
| What audit tools show                       | "Ops controls markets"           |
| Reality                                     | Ops controls markets ✓           |

**Important:** The internal `_grantRole`/`_revokeRole` calls inside `registerMarket`
and `deregisterMarket` bypass the admin check entirely — so the vault functioned
correctly before this fix. This change makes the **on-chain role graph reflect reality**,
not just the function guards.

### Full role admin table (post-fix)

| Role                 | Admin Role               | Who holds admin     |
|----------------------|--------------------------|---------------------|
| `DEFAULT_ADMIN_ROLE` | itself (special OZ rules)| governance multisig |
| `MARKET_MANAGER_ROLE`| `DEFAULT_ADMIN_ROLE`     | governance multisig |
| `MARKET_ROLE`        | `MARKET_MANAGER_ROLE` ✓  | ops multisig        |
| `PAUSER_ROLE`        | `DEFAULT_ADMIN_ROLE`     | governance multisig |
| `EMERGENCY_ROLE`     | `DEFAULT_ADMIN_ROLE`     | governance multisig |
| `UPGRADER_ROLE`      | `DEFAULT_ADMIN_ROLE`     | governance multisig |

---

## 6. `minDelay = 0` — What It Means and Why It Matters

### The Problem `AccessControlDefaultAdminRules` Solves

Plain `AccessControlUpgradeable` allows this single-transaction attack:

```
Block 1:
  tx1: admin.grantRole(DEFAULT_ADMIN_ROLE, attacker)
       → attacker immediately has full admin
  tx2: attacker.setReserveBps(0)
  tx3: attacker.setMaxRiskPerMarket(1e30)
  tx4: attacker.registerMarket(attacker_contract)
       → all LP capital reserved, vault drained in principle
```

One compromised key. One block. Protocol dead.

`AccessControlDefaultAdminRulesUpgradeable` forces a mandatory two-step process:

```
Step 1: admin calls beginDefaultAdminTransfer(newAdmin)
          → pendingAdmin = newAdmin
          → transferSchedule = block.timestamp + minDelay
          → nothing granted yet
          → event DefaultAdminTransferScheduled(newAdmin, schedule) fires

        [minDelay must pass]

Step 2: newAdmin calls acceptDefaultAdminTransfer()
          → only succeeds if: msg.sender == pendingAdmin
          → only succeeds if: block.timestamp >= transferSchedule
          → NOW the role transfers
```

The delay is your **incident response window.**

### What `minDelay = 0` Does

```solidity
__AccessControlDefaultAdminRules_init(0, admin);  // minDelay = 0
```

With `minDelay = 0`:
```
transferSchedule = block.timestamp + 0 = block.timestamp
block.timestamp >= block.timestamp  →  immediately true
```

Both steps can happen **in the same block**, even in a single flashbot bundle:

```
Bundle:
  tx1: attacker.beginDefaultAdminTransfer(attacker)
  tx2: attacker.acceptDefaultAdminTransfer()     ← succeeds, same block
```

The two-step requirement is technically enforced — two calls are needed.
But the time window between them is **zero**.

### What Still Protects You at `minDelay = 0`

The event fired in Step 1 is observable before Step 2 completes:

```
tx1 fires: DefaultAdminTransferScheduled(attacker, schedule)
```

An on-chain monitor watching for this event could alert — but cannot *prevent* the
transfer if both transactions land in the same block. Protection is monitoring-dependent,
not cryptographically enforced.

### What a Real `minDelay` Gives You

With `minDelay = 48 hours`:

```
Day 0:   Attacker compromises admin key
         Calls beginDefaultAdminTransfer(attacker)
         → Event fires immediately, monitors alert

Day 0-2: Security team sees the alert
         Current admin calls cancelDefaultAdminTransfer()
         → Transfer cancelled, attacker gets nothing

Day 2:   If uncancelled, transfer would complete here
         The 48-hour window was the incident response time
```

The delay converts a zero-response-time exploit into a **cancellable, observable,
time-bounded process.**

### Recommended Delays by Role

| Role               | Recommended Delay | Reason                                    |
|--------------------|-------------------|-------------------------------------------|
| `DEFAULT_ADMIN_ROLE` | 48–72 hours      | Full incident response cycle              |
| `UPGRADER_ROLE` (timelock) | 24–48 hours | LP exit window before upgrade executes  |

Note: `minDelay` in `AccessControlDefaultAdminRules` only applies to `DEFAULT_ADMIN_ROLE`
transfers. Other role grants/revokes are immediate (no delay). The timelock for
`UPGRADER_ROLE` should be a separate timelock contract, not this delay.

---

## 7. Production Deployment Checklist

The `initialize()` function grants all roles to a single `admin` address.
This is acceptable **only for testnet.** Before mainnet, every item below
must be completed and signed off.

```
BLIEVER V1 — MAINNET DEPLOYMENT CHECKLIST
══════════════════════════════════════════

PRE-DEPLOY
□ Governance multisig deployed and tested (DEFAULT_ADMIN_ROLE)
□ Ops multisig deployed and tested (MARKET_MANAGER_ROLE, PAUSER_ROLE)
□ Emergency multisig deployed and tested (EMERGENCY_ROLE)
□ Timelock contract deployed with target delay (UPGRADER_ROLE)

DEPLOY
□ Deploy implementation contract — verify _disableInitializers() present
□ Deploy proxy — call initialize() with deployer EOA as initial admin
□ Verify all initialize() params on-chain (alpha, maxRisk, reserveBps)
□ Run: forge inspect BlieverV1Pool storage-layout --pretty → save as storage-layout-v1.json

ROLE DISTRIBUTION (must complete before any LP deposits)
□ grantRole(MARKET_MANAGER_ROLE, ops_multisig)
□ grantRole(PAUSER_ROLE,         ops_multisig)
□ grantRole(EMERGENCY_ROLE,      emergency_multisig)
□ grantRole(UPGRADER_ROLE,       timelock_contract)

□ revokeRole(MARKET_MANAGER_ROLE, deployer_EOA)
□ revokeRole(PAUSER_ROLE,         deployer_EOA)
□ revokeRole(EMERGENCY_ROLE,      deployer_EOA)
□ revokeRole(UPGRADER_ROLE,       deployer_EOA)

MINDELAY — LAUNCH BLOCKER ⚠️
□ Call changeDefaultAdminDelay(48 hours)   ← REQUIRED before LP deposits
  [This call itself has a delay before taking effect — plan accordingly]
□ Wait for delay change to take effect and verify on-chain

ADMIN TRANSFER
□ Call beginDefaultAdminTransfer(governance_multisig)
□ Wait for minDelay to pass
□ governance_multisig calls acceptDefaultAdminTransfer()
□ Verify deployer_EOA no longer holds DEFAULT_ADMIN_ROLE
□ Verify governance_multisig holds DEFAULT_ADMIN_ROLE

VERIFICATION
□ Verify getRoleAdmin(MARKET_ROLE)      == MARKET_MANAGER_ROLE
□ Verify getRoleAdmin(PAUSER_ROLE)      == DEFAULT_ADMIN_ROLE
□ Verify getRoleAdmin(EMERGENCY_ROLE)   == DEFAULT_ADMIN_ROLE
□ Verify getRoleAdmin(UPGRADER_ROLE)    == DEFAULT_ADMIN_ROLE
□ Verify deployer_EOA holds zero roles
□ Verify isSolvent() == true
□ Verify utilizationBps() == 0

MONITORING (must be live before LP deposits)
□ Alert on: DefaultAdminTransferScheduled event
□ Alert on: RoleGranted(DEFAULT_ADMIN_ROLE, ...)
□ Alert on: LiabilityCapApplied event (Prop 4.9 violation signal)
□ Alert on: MarketForceSettled event
□ Alert on: isSolvent() == false (poll every block or use event-based)
```

---

## 8. Attack Scenarios — Before and After Fixes

### Scenario A — Compromised Pauser Key

**Before fix (PAUSER_ROLE controlled unpause):**
```
Attacker compromises pauser key
  → calls pause()    // halts vault
  → calls unpause()  // immediately reverses the pause
  → circuit breaker is useless
```

**After fix (unpause requires DEFAULT_ADMIN_ROLE):**
```
Attacker compromises pauser key
  → calls pause()    // halts vault ✓
  → cannot call unpause() — wrong role, reverts
  → governance must convene to resume
  → pause holds until reviewed
```

---

### Scenario B — Compromised Admin Key (No DefaultAdminRules)

**Before fix (plain AccessControlUpgradeable):**
```
Block 1: attacker.grantRole(DEFAULT_ADMIN_ROLE, attacker)
         → immediately effective, one transaction
         → all admin functions open to attacker
```

**After fix (AccessControlDefaultAdminRules, minDelay = 48h):**
```
Block 1: attacker.beginDefaultAdminTransfer(attacker)
         → event fires, monitors alert
         → transfer not yet effective

Block 1 to +48h: security team calls cancelDefaultAdminTransfer()
         → attacker gets nothing
```

---

### Scenario C — Oracle Failure (No EMERGENCY_ROLE)

**Before fix (forceSettleMarket required DEFAULT_ADMIN_ROLE):**
```
Oracle fails at 2am
  → market is stuck, totalLiability not released
  → LP withdrawals constrained by dead liability
  → governance quorum required (5-of-9 multisig)
  → takes hours to assemble, market stays stuck
```

**After fix (EMERGENCY_ROLE):**
```
Oracle fails at 2am
  → emergency multisig (2-of-3) signs forceSettleMarket()
  → liability released, market resolved in minutes
  → governance not required for emergency action
```

---

## Quick Reference

```
Role                  Held By              Gates
─────────────────────────────────────────────────────────────────
DEFAULT_ADMIN_ROLE    governance multisig  setAlpha, setMaxRisk,
                                           setReserveBps, unpause
MARKET_MANAGER_ROLE   ops multisig         registerMarket,
                                           deregisterMarket
MARKET_ROLE           market contracts     collectTradeCost,
                      (auto-managed)       settleMarket,
                                           claimWinnings
PAUSER_ROLE           fast ops multisig    pause()
EMERGENCY_ROLE        emergency multisig   forceSettleMarket()
UPGRADER_ROLE         timelock contract    _authorizeUpgrade()
```
