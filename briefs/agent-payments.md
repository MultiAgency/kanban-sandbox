# Agent payments on NEAR: what works today

*One-page brief for Acme's product team. Sources: [research notes](../research/agent-payments.md).*

## Summary

An AI agent can pay per API call on NEAR today, in USDC, without holding NEAR
for gas. The standard is **x402**: the API answers "402 Payment Required" with
a price, the agent signs a payment, and a *facilitator* settles it on chain,
usually within seconds. We ran it end to end on testnet: a 0.001 USDC API call
and a 3 USDC deposit both settled and were checked on chain.

## How a payment flows

```text
agent ── request ──▶ API ── 402 + price ──▶ agent
agent ── signed payment ──▶ API ──▶ facilitator ──▶ NEAR (relayer pays gas)
API ── response + transaction id ──▶ agent
```

The agent signs one USDC transfer of exactly the quoted amount to the API
owner. It cannot be replayed, and it expires if unused. The facilitator
submits it and reports success only after the token transfer itself succeeds.

## What each side needs

| | Agent (payer) | API owner (merchant) |
| --- | --- | --- |
| Account | NEAR account with a full-access key | NEAR account registered with USDC |
| Funds | USDC only, no NEAR | NEAR to fund the relayer's gas |
| Software | `@fastnear/x402` client | x402 middleware + a facilitator (self-hosted, or a reference instance with an approved API key) |

## Limits to plan around

- **Keys.** Payments need a full-access key. Agents that run with restricted
  keys cannot pay yet.
- **Defaults.** Client libraries cap each payment at $1 unless configured.
  That is a good guardrail, but larger purchases need explicit limits.
- **Browsers.** Wallet support for signing these payments is not ready.
  Server-side agents are the practical path today.
- **Scope.** One exact price per request, in USDC. No subscriptions, refunds,
  or variable pricing in the protocol.
- **Paying without a NEAR account** (for example with an EVM key through NEAR
  Intents) is specified but not yet supported by the facilitator.

## Recommendation

Pilot it on server-side agents with a small per-payment cap and a
self-hosted facilitator on testnet, then move to mainnet once the merchant
side (relayer funding, API client policy) has an owner.
