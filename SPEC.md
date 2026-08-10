# agent-spend-policy — Phase 3 Architecture Specification

Status: **proposed** (pending maintainer approval). Written after the Phase 0 research gate and
the Phase 1 critical review; it supersedes the original "policy oracle" design from the build
prompt for the reasons recorded below.

---

## 0. Context: why this spec is not the original design

Phase 0 found a genuine on-chain competitor that the original prompt assumed did not exist:

- **`deegalabs/stellar-402-spendguard` (SpendGuard)** — verified **live on testnet** via the
  stellar.expert API (`CCABMNFY3VKK7BI3YBWXJEE2EXX2NW5S573NASTCFXA6KBXR5PDWFD6E`, created
  ~Apr 2026, 157 emitted events, 1 storage entry). Its mechanism was confirmed by reading
  `contracts/budget-guard/src/lib.rs` directly:
  - **It is a plain custodial contract, NOT a custom account.** No `SmartAccountTrait`,
    no `__check_auth`. The agent calls `authorize_payment(price, merchant)`; the contract
    does `agent.require_auth()`, runs its checks (paused / amount / max_tx / whitelist /
    daily-limit / balance), then `usdc.transfer(contract_addr → merchant)` out of funds the
    owner moved in via `top_up`.
  - Its README line "Soroban Custom Account" is marketing imprecision; the source shows a
    standard `#[contract]` with `require_auth` at the door.
- **`stellarspend/stellarspend-contracts`** — plain contracts with `require_auth` + token
  `transfer_from` (allowances/budgets), differentiated by Noir/UltraHonk ZK spend-limit proofs.
  Not agent-specific, not a custom account.
- **`valentinahenry37-droid/soroban-vault`** (multisig treasury, plain `require_auth`),
  **`DelegoLabs/Delego`** (TS app, no in-repo contracts), **`vegaforge/tollway`** (TS
  orchestration, off-chain), **`JosueBrenes/stellar-agent-pay`** (CLI plugin, off-chain),
  **`wraith-protocol/contracts`** (stealth privacy, not policy).

**Conclusion:** no known project uses a Soroban custom account (`__check_auth`) for spend
policy. That is this project's mechanism and its differentiator:

| | Fortexa | SpendGuard | StellarSpend | **agent-spend-policy** |
|---|---|---|---|---|
| Enforcement layer | off-chain app firewall | on-chain, custodial vault | on-chain, `require_auth` + ZK | **on-chain, account level** |
| Where funds live | user wallet | inside vault contract | owner/contract | **the agent's own account** |
| Compromised agent key | bypassable | capped within vault | n/a | **cannot exceed policy** |
| Scope | one flow | one vault | budget apps | **any Soroban authorization by the account** |

**Differentiation discipline (required wording):** SpendGuard is **not advisory** — it enforces
on-chain for its scope (payments out of its vault fail the transaction when out of policy). The
differentiation claim must therefore be made at the mechanism level and never as "we enforce,
they don't": (1) *scope* — SpendGuard gates one function (`authorize_payment`) on one payment
rail; a policy account gates every Soroban authorization the account makes, including nested
token transfers; (2) *custody/UX* — SpendGuard requires moving USDC into a vault (`top_up`);
funds for a policy account arrive directly at the agent's own address, and no admin key has any
fund-moving authority (admin changes policy only; only the registered agent key can move funds,
and only within policy); (3) *governance* — one instance per account, policy-admin disjoint from
signer by construction. These claims are the ones the README will state, verbatim in spirit.

---

## 1. Enumerated contracts

- **`contracts/policy-engine`** — the core: a **Soroban custom account** contract. It owns
  all policy state and implements `SmartAccountTrait::__check_auth`. One WASM, one instance
  per governed agent (Stellar's account model: each account has exactly one account contract).
- **`examples/demo-agent-payment`** — a thin consumer example. It holds **no policy state of
  its own**; it demonstrates that a consumer contract/agent needs zero policy code and that
  enforcement happens inside the policy account. Its function `pay(...)` merely drives a
  SAC `transfer` whose `from` is the policy account, signed by the agent key.

There is deliberately no second "app" repo (see build-prompt Phase 2 reasoning).

---

## 2. Storage (`policy-engine`)

All keys are `#[contracttype]` enums. Persistent keys get **explicit TTL extension**; instance
keys get TTL bumps via `env.storage().instance().extend_ttl` where required.

| Key | Type | Kind | Notes |
|---|---|---|---|
| `Initialized` | `bool` | instance | one-time flag |
| `Admin` | `Address` | instance | policy manager; set at `initialize` |
| `AgentKey` | `BytesN<32>` | instance | agent's Ed25519 public key; only signer accepted by `__check_auth` |
| `Policy` | `PolicyConfig` | persistent | the policy; TTL bumped on every write |
| `Window` | `WindowState` | persistent | `{ window_start: u64, spent: i128 }`; TTL bumped on every write |

```rust
#[contracttype]
pub struct PolicyConfig {
    pub per_tx_cap: i128,            // 0 = rule disabled
    pub window_cap: i128,            // 0 = rule disabled; fixed 86_400s bucket (rolling window = backlog)
    pub allowlist: Vec<Address>,     // empty Vec = no recipient restriction (documented, explicit)
    pub active_start: u64,           // unix seconds (ledger time); 0 = no restriction
    pub active_end: u64,             // unix seconds; 0 = no end
    pub paused: bool,                // admin kill switch
    pub allow_classic_payments: bool,// default false — classic ops cannot be arg-inspected, so deny unless explicitly allowed
}

#[contracttype]
pub struct WindowState {
    pub window_start: u64,
    pub spent: i128,
}
```

**TTL rule:** every policy mutation (`set_policy`, `revoke_policy`, `initialize`) and every
successful authorization in `__check_auth` extends the TTL of `Policy`/`Window` to
`DEFAULT_TTL` (target ~31 days of ledgers, computed from protocol constants at build time).
In `__check_auth` the extension is conditional if `storage().persistent().get_ttl` is
available in soroban-sdk 27 (extend only when remaining TTL < 7 days); otherwise extend
unconditionally and note the per-auth fee in the README.

---

## 3. Public surface (exact signatures)

```rust
// ── Lifecycle ────────────────────────────────────────────────────────────
pub fn initialize(env: Env, admin: Address, agent_key: BytesN<32>) -> Result<(), Error>
    // require_auth(admin). Exactly once (AlreadyInitialized otherwise).

// ── Policy management (admin only) ───────────────────────────────────────
pub fn set_policy(env: Env, config: PolicyConfig) -> Result<(), Error>
    // require_auth(Admin). Validates: per_tx_cap >= 0, window_cap >= 0,
    // active_end == 0 || active_end > active_start. Stores config, resets Window.
pub fn revoke_policy(env: Env) -> Result<(), Error>
    // require_auth(Admin). Removes Policy + Window → account becomes default-deny.

// ── Read / advisory ──────────────────────────────────────────────────────
pub fn query_policy(env: Env) -> Option<PolicyConfig>          // no auth
pub fn check(env: Env, recipient: Address, amount: i128) -> PolicyResult
    // no auth. Pure read-and-decide; emits the same allowed/blocked event as __check_auth.
    // This is the direct replacement for the original prompt's check_policy(): it is the
    // pre-flight/advisory surface agents and off-chain tooling call (via simulation).

// ── Enforcement (host-invoked) ───────────────────────────────────────────
impl SmartAccountTrait for PolicyEngineContract {
    fn __check_auth(env: Env, signature_payload: SignaturePayload,
                    signatures: Vec<Signature>) -> Result<(), Error>
}
```

`PolicyResult`:

```rust
#[contracttype]
pub enum PolicyResult { Allowed, Blocked(BlockReason) }

#[contracttype]
pub enum BlockReason { NoPolicy, CapExceeded, RecipientNotAllowed, OutsideTimeWindow,
                       Paused, InvalidAmount, ClassicOpNotAllowed }
```

### Caller map — every function maps to a real caller (no speculative functions)

| Function | Real caller | Why it is called | Auth |
|---|---|---|---|
| `__check_auth` | **The Soroban host** — invoked automatically whenever the governed account must authorize an operation in any transaction. No user chooses to call it; it fires on every agent payment. | Enforcement gate: verify agent key signature, inspect operation, apply policy, panic on block. | host-invoked; no `require_auth` (reasoned below) |
| `initialize` | **Deployer/admin** (human setting up the agent's account), via CLI/wallet. | One-time setup: set admin + register agent signing key. | `require_auth(admin)` |
| `set_policy` | **Admin** (policy manager), via CLI/wallet, whenever limits / allowlist / window change. | The only way policy changes; resets window state. | `require_auth(admin)` |
| `revoke_policy` | **Admin**, on compromise or decommission. | Removes policy → account becomes default-deny (stops all movement). | `require_auth(admin)` |
| `query_policy` | **Admin, agent runtime, or off-chain auditor** (dashboard / monitoring). | Read current policy; no state change. | none (pure read) |
| `check(recipient, amount)` | **Agent runtime pre-flight** and **off-chain tooling** (simulated call before submitting a payment). Replaces the original prompt's `check_policy`; is the richer "why was this blocked" surface (returns `BlockReason`). | Advisory decision + advisory allowed/blocked event; no state change. | none (pure read-and-decide) |

Explicitly deferred to the Phase 8 backlog (not in MVP): agent-key rotation (`set_agent_key`),
rolling (not fixed) windows, multi-signature policy updates, second example integration, policy
templates.

### Auth placement — the reasoning (this is the security-critical decision)

1. **`__check_auth` is the enforcement point and does NOT call `require_auth`.** The Soroban
   host invokes it only for operations this account must authorize; inside, the contract:
   (a) verifies the agent's Ed25519 signature from `signatures` against `AgentKey` (rejects a
   wrong signer → the account is unusable by anyone but the registered agent key);
   (b) extracts the operation from `signature_payload` and evaluates policy;
   (c) **panics with a specific `Error` on any block** — a panic inside `__check_auth` fails
   the entire transaction at the ledger, which is the enforcement. There is no path where the
   agent's key can move the account's funds without policy approval, because the host will not
   authorize any Soroban operation for this account without a passing `__check_auth`.
2. **Admin functions use `require_auth(Admin)`** (the address stored at `initialize`), not the
   agent key and not a parameter-supplied address. Taking `owner` as an argument invites
   argument-spoofing confusion; one instance has one admin, stored once. The admin can change
   policy but **cannot move funds** (only the agent key can authorize operations), and the
   agent key can move funds only within policy — the two roles are disjoint by construction.
3. **`check` / `query_policy` require no auth** — they are pure reads that mutate nothing.
   They are safe to expose to anyone and are the off-chain pre-flight surface.

### What `__check_auth` inspects (enforcement semantics)

- **Soroban authorization payloads:** `signature_payload` carries the `HashIDPreimage::
  SorobanAuthorization` preimage, whose `invocation` includes the target contract, function
  symbol, and args. The contract matches SAC token calls:
  - `transfer(from, to, amount)` → `to = args[1]`, `amount = args[2]`
  - `transfer_from(spender, from, to, amount)` → `to = args[2]`, `amount = args[3]`
  and applies: paused → amount > 0 → `per_tx_cap` → allowlist (if non-empty) → window cap
  (with 86 400 s lazy reset) → active time window.
- **Other Soroban calls:** window / paused / no-policy checks only (recipient and amount are
  not extractable); a non-payment call cannot move the account's tokens *directly*, and any
  nested token transfer the account must authorize is itself inspected. Documented limitation.
- **Classic payment operations:** the payload does not expose destination/amount to the
  contract. Default is **deny** (`ClassicOpNotAllowed`) unless `allow_classic_payments` is set,
  in which case they pass (all-or-nothing — cannot be finer-grained). Documented limitation.

### Blocked-attempt recordability (honest statement)

A blocked `__check_auth` panics, so the transaction fails; the emitted event is observable in
the failed transaction's **diagnostic events** and the block is provable by the failed tx hash
plus the specific `Error` code. The advisory `check()` path emits the same event as a normal
contract event on success. Both claims are re-verified live in Phase 6.

---

## 4. Events (a stated primitive, not optional logging)

```rust
// every decision emits exactly one of these (from __check_auth or check())
(symbol_short!("spend"), symbol_short!("allowed"))  // (recipient, amount, spent_in_window)
(symbol_short!("spend"), symbol_short!("blocked"))  // (recipient, amount, reason_symbol)

// policy lifecycle
(symbol_short!("policy"), symbol_short!("set"))      // (admin)
(symbol_short!("policy"), symbol_short!("revoked"))  // (admin)
(symbol_short!("policy"), symbol_short!("init"))     // (admin)
```

`reason_symbol ∈ {no_policy, cap, allowlist, window, paused, amount, classic}` — short symbols
so explorers render them readably.

---

## 5. Errors, arithmetic, code rules

- `#[contracterror]` enum `Error` with one discriminant per `BlockReason` plus
  `AlreadyInitialized`, `NotInitialized`, `Unauthorized`, `BadSignature`, `InvalidConfig`,
  `ArithmeticOverflow`.
- **No `unwrap()` outside tests** (use `ok_or`/`unwrap_or`/`match`). **No floats.** All
  amounts `i128`; use `checked_add`/`checked_sub`/`checked_mul`. No percentages in MVP, so no
  basis points; if any appear later they are integer bps math.
- Signature verification uses `soroban_sdk::crypto::ed25519` — never hand-rolled.

---

## 6. Tech stack (verified this session, re-verify at build time)

- `soroban-sdk = "27"` → resolves **27.0.5** (crates.io, 2026-08-03); stellar CLI **27.0.0**
  (installed); testnet at **protocol 27** (verified via Soroban RPC `getLatestLedger`).
- Rust 1.96.0; `wasm32-unknown-unknown` target; workspace edition from the `stellar contract
  init` template defaults.
- The exact `SmartAccountTrait` / `SignaturePayload` / `Signature` module paths and the
  account-contract signer setup commands are pinned against soroban-sdk 27.0.5 docs and the
  reference `stellar/soroban-account-contract` at implementation time (Phase 5/6) — this spec
  fixes the *semantics*, not unverified type paths.

---

## 7. Repo layout (single repo)

```
agent-spend-policy/
  Cargo.toml                    # Rust workspace
  SPEC.md                       # this document
  contracts/
    policy-engine/
      src/lib.rs                # contractimpl: lifecycle, policy mgmt, check
      src/auth.rs               # __check_auth + payload parsing
      src/storage.rs            # DataKey, PolicyConfig, WindowState
      src/policy.rs             # pure check_policy(config, state, ctx) -> PolicyResult
      src/events.rs
      src/error.rs
      src/test.rs               # unit + integration tests (testutils)
    examples/
      demo-agent-payment/
        src/lib.rs              # consumer example: pay(token, to, amount) via policy account
  README.md  CONTRIBUTING.md  SECURITY.md
  .github/workflows/ci.yml
  tests/fixtures/README.md      # Phase 6: real IDs, hashes, event output
```

---

## 8. Git workflow (non-negotiable, restated from the build prompt)

- Never `git add .`; stage explicitly.
- One commit per logical unit; push immediately after each commit.
- Conventional format: `type(scope): description`.
- Branch protection Ruleset + maintainer bypass configured and tested (Phase 4) **before** the
  first real commit.

---

## 9. Numbered build sequence (Phase 5 execution order)

1. `stellar contract init` scaffold the workspace; pin sdk `27`; verify `cargo build` +
   `wasm32-unknown-unknown` toolchain present.
2. `error.rs` + `storage.rs` (DataKey, PolicyConfig, WindowState) + unit tests for validation.
3. `initialize` / `set_policy` / `revoke_policy` / `query_policy` with admin auth + TTL bumps.
4. `policy.rs` — pure `check_policy(config, state, now, recipient, amount) -> PolicyResult`
   (window lazy-reset, cap, allowlist, time window, pause, amount) — fully unit-tested.
5. `auth.rs` — `__check_auth`: signature verification, payload parsing, SAC arg extraction,
   panics with reason; events from `events.rs` wired into `check()` and `__check_auth`.
6. `examples/demo-agent-payment` — consumer contract + integration tests via testutils.
7. Full workspace: `cargo fmt --check`, clippy `all` + `pedantic` deny, `cargo test`.
8. `.github/workflows/ci.yml` (job named `build`, matching Phase 4's required status check),
   README/CONTRIBUTING/SECURITY.
9. Phase 6 testnet deployment + scenario proof; fixtures recorded.
10. Phase 8 backlog issues.

---

## 10. Deployment prerequisites (Phase 6 checklist)

- Funded testnet keypairs: `test-key` exists and is funded (~20k XLM; verified this session).
- Custom-account signer setup: create the governed account and set `SIGNER_TYPE_CONTRACT` to
  the deployed `policy-engine` instance (exact `stellar` CLI sequence confirmed against the
  reference account-contract docs before executing).
- Scenario to prove: set policy (cap 100, allowlist [merchant], window 1 day) → agent-key
  signed SAC transfer to merchant within cap (succeeds, `allowed` event) → same transfer over
  cap (tx fails, `blocked` diagnostic event, hash recorded) → non-allowlisted recipient
  (fails) → record all IDs/hashes/event output in `tests/fixtures/README.md`.
