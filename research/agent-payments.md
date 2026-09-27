# Research notes: how agents pay for API calls on NEAR today

For engagement #1. Every claim below links to source code, documentation, or an
on-chain transaction that was checked on 2026-09-27.

## 1. Protocol: x402 `exact` on NEAR

- **x402 v2.** A paid HTTP endpoint answers `402 Payment Required` with a
  base64 `PAYMENT-REQUIRED` header listing what it `accepts`: scheme, network,
  asset, amount, `payTo`, and `maxTimeoutSeconds`. The client retries with a
  `PAYMENT-SIGNATURE` header, and the server returns the resource plus a
  `PAYMENT-RESPONSE` header that names the settlement transaction.
  - Observed live: `accepts: [{scheme: "exact", network: "near:testnet", amount: "1000", asset: "3e2210e1…b8af", payTo: "x402-merchant.agency.testnet", maxTimeoutSeconds: 300}]`.
- **NEAR settlement shape.** The payer signs a NEP-366 `SignedDelegateAction`
  wrapping exactly one `ft_transfer` (asset contract, `receiver_id = payTo`,
  exact amount, 1 yoctoNEAR deposit). A relayer submits it inside its own
  transaction (`Action::Delegate`) and pays the gas.
  - Sources: `fastnear/x402-facilitator` README "Deliberate scope"; FastNear
    builder docs, `docs/transaction-flow/advanced-features.mdx` (NEP-366).
- **Replay protection is on chain.** The delegate's signed hash is a
  single-use anchor, and it expires at `max_block_height`, which is set from
  `maxTimeoutSeconds`.

## 2. What a paying agent needs

| Requirement | Why | Evidence |
| --- | --- | --- |
| A NEAR account with a **full-access key** | `ft_transfer` needs 1 yocto attached, and function-call keys cannot attach a deposit. The facilitator rejects function-call payer keys. | facilitator `crates/x402-chain-near/src/mechanism.rs` (FunctionCall rejection, 1-yocto check) |
| **USDC** in that account | The `exact` scheme moves exactly the quoted amount. | payer `ft_balance_of` preflight in the facilitator |
| **No NEAR for gas** | The relayer sponsors gas; the payer only signs. | settle tx `EbmtPJhq…` is signed by `x402-relayer.agency.testnet` |
| A client library | `@fastnear/x402` (`createNearX402Client`, `createLocalNearSigner`) wraps the official `@x402/near`. | `fastnear/js-monorepo` `packages/x402` |

Caveats found:

- **The full-access check happens only at the facilitator.** Neither
  `@fastnear/x402` nor `@x402/near` checks the key type before signing, so
  clients should check it themselves.
- **Spend controls.** `@x402/core` 2.25 clients refuse any payment above $1 by
  default. `createNearPaymentFetch` in `@fastnear/x402@2.5.0` does not expose
  that setting, so larger payments need `createNearX402Client(...).setSpendControls(...)`.
- **Version drift.** The published `@fastnear/x402@2.5.0` pins `@x402/*`
  `~2.18.0`; the facilitator's examples pin 2.25.0. Overriding to 2.25.0 worked
  in testing.
- **Browser wallets are not ready.** The fastnear static demo's wallet
  manifest gives no wallet both `signDelegateActions` and
  `signDelegateActionsWithTtl`, which the browser x402 path requests.

## 3. What a merchant must run

1. **A resource server** that emits `PAYMENT-REQUIRED` and forwards payments to
   a facilitator, for example `@fastnear/x402/server` `createNearResourceServer` or
   `@x402/express`.
2. **A facilitator** that verifies and settles. `fastnear/x402-facilitator` is
   a production Rust service. It uses PostgreSQL journaling and exactly-once
   claims, and it succeeds only when the inner token receipt is
   `SuccessValue`. It needs a funded relayer account and an API client per
   resource server, with exact network, asset, and `payTo` policy rows.
3. **A USDC storage registration** for the `payTo` account (`storage_deposit`,
   0.00125 NEAR), or the transfer fails.

Public reference facilitators exist (`x402.mikedotexe.com`,
`test.x402.mikedotexe.com`), but they issue API keys by manual approval only.

## 4. Evidence from this engagement's own infrastructure

| Event | Transaction |
| --- | --- |
| Agent buys an API response (0.001 USDC) | [EbmtPJhq…](https://testnet.nearblocks.io/txns/EbmtPJhqMXiUdTuW8N1d87Z1Wd3gSSckmUYL4CisDyPh) |
| Acme pays the engagement deposit (3 USDC) to `multiagency.sputnikv2.testnet` | [3ppdvTz5…](https://testnet.nearblocks.io/txns/3ppdvTz5dmsw4ot7GePdT39oKPDjGyiUByK2RMZ6pD18) |

Both were checked independently of the facilitator. The transaction reached
`FINAL` with no failed receipt, and the payee's balance moved by exactly the
amount.

## 5. Current limits

- `exact` only; one network and one USDC contract per facilitator process.
- Testnet has a single canonical USDC (`3e2210e1…b8af`, 6 decimals), funded
  from Circle's faucet (20 USDC per address every 2 hours).
- The NEAR Intents route (payers with no NEAR account) is specified upstream
  (x402-foundation/x402#2948) but is not implemented in the facilitator, and
  it is mainnet-only.
- Indexed data (FastNear API balances, Transactions API) lags the chain by
  seconds and is advisory. Settlement authority is RPC `tx_status` at `FINAL`.
