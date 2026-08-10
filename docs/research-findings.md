# Phase 0 Research Findings — agent-spend-policy

Recorded per the build prompt's evidence rule: every claim below was produced this session by
fetching the cited source directly. Nothing is inherited from the build prompt's partial
findings. Date of session: 2026-08-10.

The build prompt noted two Phase-0 searches were previously blocked by network restrictions;
both were completed here with direct access, plus the Fortexa re-check.

---

## 1. Drips Wave Stellar approved-repo list (fetched live)

- URL: `https://www.drips.network/wave/stellar/repos` — shows **667 approved repos** for the
  Stellar Wave Program (fetched 2026-08-10).
- `karagozemin/Fortexa` **is approved** — tagged *Stellar Agentic Hackathon*, 2× Points.
- Other approved agent/payment repos seen on the list: `enliven17/talos-stellar`,
  `Flamki/stellarmind`, `winsznx/routedock`, `wraith-protocol/{demo,sdk,contracts}`,
  `Emmy123222/Stellar-GreenPay`, `Emmy123222/Stellar-MarketPay-`, `karagozemin/OverSync`,
  `Stellopay/stellopay-core`, `Trustless-Work/trustlesswork-smart-contract-stellar`.
- **None of these is an on-chain spend-policy engine contract.** (Caveat: the list UI ignored
  search-filter URLs, so the full 667 could not be exhaustively scanned; each competitor
  candidate found via GitHub search was instead verified individually — see below.)

## 2. On-chain competitor search (GitHub API, direct)

Queries run via `https://api.github.com/search/repositories`:
`q=soroban+agent+policy` (11 results), `q=soroban+spend+limit` (12 results), sorted by stars.
Candidates inspected individually:

### 2a. `deegalabs/stellar-402-spendguard` (SpendGuard) — DIRECT ON-CHAIN COMPETITOR

**Deployment proof** — `GET https://api.stellar.expert/explorer/testnet/contract/CCABMNFY3VKK7BI3YBWXJEE2EXX2NW5S573NASTCFXA6KBXR5PDWFD6E`:
```json
{"contract":"CCABMNFY3VKK7BI3YBWXJEE2EXX2NW5S573NASTCFXA6KBXR5PDWFD6E",
 "created":1775581789,"creator":"GBF5LCVZQ5VQ5DOE57DXY4PDDWS2BGACEJBGJUJYAJSGJKOWHZ5TTLOY",
 "wasm":"e625b29e867dc6593375db0a49ed7bb626e2753c0f0c52662b9f6c1a7a475680",
 "events":157,"storage_entries":1,"validation":{"status":"unverified"}}
```
`created=1775581789` ≈ April 2026; 157 events ⇒ invoked and used on testnet.

**Mechanism proof** — `contracts/budget-guard/src/lib.rs` (11,566 bytes, read in full):
```rust
use soroban_sdk::{contract, contractimpl, contracttype, token, Address, Env};
```
Entire `#[contractimpl]` surface: `initialize, authorize_payment, top_up, set_daily_limit,
set_max_tx, whitelist_merchant, remove_merchant, emergency_pause, emergency_unpause,
set_agent, get_status`. `src/` = `error.rs, events.rs, lib.rs, storage.rs, test/`.
**No `__check_auth`. No `SmartAccountTrait`. No account trait import.**
Auth + execution inside `authorize_payment`:
```rust
    // 2. Check caller is agent
    let agent: Address = env.storage().instance().get(&DataKey::Agent).unwrap();
    agent.require_auth();
    ...
    // 9. Execute transfer
    usdc.transfer(&contract_addr, &merchant, &price);
```
`top_up`: `usdc.transfer(&from, &contract_addr, &amount);` — funds moved INTO the vault.

**Conclusion: `require_auth`-gated custodial enforcement. Not advisory (out-of-policy fails
the tx). Not a custom account (no `__check_auth`; agent key signs a vault call; funds in the
vault).**

### 2b. `stellarspend/stellarspend-contracts` — on-chain, require_auth, ZK flavor

README: "Soroban smart contracts for automated budgets, savings goals, and private spending
limit verification on Stellar… Stellar Hacks: Real-World ZK Submission". Contract list:
`zk-verifier, spending-limits, batch-payment, batch-rewards, escrow, savings-goals, fee,
audit, access-control`.
`contracts/allowances/src/lib.rs` (read):
```rust
use soroban_sdk::{contract, contractimpl, panic_with_error, symbol_short, token, Address, Env, Vec};
...
owner.require_auth();
...
token_client.transfer_from(&env.current_contract_address(), &allowance.owner, &allowance.recipient, &allowance.amount);
```
Plain `require_auth` + `transfer_from`. Not a custom account. Personal-finance/ZK focus.
Metadata: Rust, 2★, 199 forks, created 2026-01-17.

### 2c. `valentinahenry37-droid/soroban-vault` — on-chain multisig treasury

`src/lib.rs` (read in full, real working code):
```rust
use soroban_sdk::{contract, contractimpl, token, Address, Bytes, Env, Vec};
```
Surface: `initialize, propose_transfer, propose_add_signer, propose_remove_signer,
propose_change_threshold, approve, reject, execute, deposit, get_proposal, get_config,
get_balance`. Spending control in `execute` → `check_and_update_spending` (per-period limit,
lazy reset) then `token_client.transfer(&env.current_contract_address(), &recipient, &amount);`.
Custodial (`deposit` moves funds in), `require_auth` on proposer/signer/executor.
**Not a custom account.** Metadata: 0★, created 2026-08-06, Rust.

### 2d. Others (off-chain or not policy)

- `DelegoLabs/Delego` — API desc: "AI-powered commerce platform built on Stellar and Soroban
  … approvals and spending limits." TypeScript, 6★, 113 forks. Root listing: full-stack TS app,
  **no `contracts/` dir, no `Cargo.toml`** at root.
- `vegaforge/tollway` — `package.json` (read): "Orchestration, policy, and observability layer
  for agent payments on Stellar (x402 + MPP)", pnpm/turbo TS monorepo, deps only
  biome/changesets/turbo/typescript. Off-chain orchestration.
- `JosueBrenes/stellar-agent-pay` — API desc: "Stellar CLI plugin that lets AI agents unlock
  x402 paid HTTP resources with USDC under an auditable spending policy." TypeScript CLI
  plugin, npm `stellar-agent-pay`, off-chain.
- `wraith-protocol/contracts` — README (read): stealth-address privacy contracts (EVM/Soroban/
  Solana/CKB); Stellar: `stealth-announcer, stealth-registry, stealth-sender, wraith-names`.
  Privacy infra, not spend policy.

## 3. Fortexa re-verification (fresh, not inherited)

`https://github.com/karagozemin/Fortexa` README (read in full): "Policy-Controlled Payment
Firewall for Autonomous Agent Actions on Stellar". Next.js app: `/api/decision`,
`/api/stellar/build-payment`, `/api/stellar/submit-signed`; wallet-bound login (SEP-53
challenge); unsigned-XDR → wallet sign → submit; SHA-256 hash-chained audit log;
`docs/SCF_TRANCHE_PLAN.md` (SCF funding). **Zero Soroban mentions.** Off-chain.

## 4. Net conclusion

Every on-chain project found is (a) `require_auth` + custody (SpendGuard vault, StellarSpend
allowances, soroban-vault multisig) or (b) off-chain (Fortexa, tollway, stellar-agent-pay,
Delego). **No known project implements agent spend policy as a Soroban custom account
(`__check_auth`).** That is the mechanism and differentiator for agent-spend-policy; see
`SPEC.md` for the design and the required differentiation wording.
