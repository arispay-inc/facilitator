# ArisPay x402 Facilitator

The x402 facilitator for USD and EUR agent payments — non-custodial, live on
Base mainnet at `https://facilitator.arispay.app`.

Accept x402 agent payments in USD or EUR without running chain
infrastructure, holding funds, or giving up settlement control.

**Non-custodial by construction.** The facilitator verifies signed payment
authorizations (the buyer's EIP-3009 signature, the EIP-712 domain, and that
the payment matches the seller's declared requirements) and submits the
settlement transaction on-chain. The transfer moves funds **directly from
buyer to seller** in a single transaction. ArisPay does not hold customer
funds, maintain merchant balances, operate escrow, or release payouts —
settlement *is* the payout. It verifies payments, not fulfillment: the
resource server remains responsible for delivering what it sold.

No auth, no signup — point your server's facilitator URL at it and you're
done. **No facilitator fee**: verification is free, and settlement gas is the
chain's cost, paid by the seller. Buyers never pay gas (that's the EIP-3009
design). See **Fees** below.

## Integration

Any x402 v2 server built on upstream `@x402/core`:

```ts
import { HTTPFacilitatorClient } from "@x402/core/server";

const facilitator = new HTTPFacilitatorClient({ url: "https://facilitator.arispay.app" });
```

That's the whole integration. The client calls the endpoints below;
request/response shapes are the upstream x402 wire format. With the
[`paygate`](https://www.npmjs.com/package/paygate) SDK (Express/Fastify)
this facilitator is already the default.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/verify` | Dry-run verification of a signed payment payload (response carries `requestId`) |
| POST | `/settle` | Verify + settle on-chain (idempotent on the EIP-3009 nonce; response carries `requestId`) |
| GET | `/supported` | Supported kinds (scheme/network/version), signer addresses, per-kind asset + fee metadata, and the additive `live` production block |
| GET | `/facilitator` | Discovery: live vs planned networks, validated assets, currencies, signers, settlement mode, fee policy, endpoint map |
| GET | `/receipts/{txHash}` | Settlement receipt by transaction hash (asset, amount, payer, payTo, gas, explorer link) |
| GET | `/settlements/{nonce}` | Settlement status by payment id — the EIP-3009 nonce (`settled` \| `pending`, tx hash, timestamps; `?network=` optional, defaults to the deployment network) |
| GET | `/status` | Service status: network, settlement mode, uptime, deployment commit |
| GET | `/metrics` | Uptime, per-endpoint latency, trailing 24h/1h settlement counts + success rate |
| GET | `/trust` | Trust page: custody model, relayer address + explorer link, verified EIP-712 domains, fee policy, security model, version |
| GET | `/events/schema` | Webhook event model (schema published now, delivery upcoming — see **Webhooks**) |
| GET | `/discovery/resources` | Bazaar catalog: paginated list of x402 resources settling through this facilitator (x402 spec discovery layer) |
| GET | `/health` | Liveness |

Everything except `/health` is also mirrored under `/facilitator/*`
(`/facilitator/verify`, `/facilitator/settle`, `/facilitator/supported`, …)
for clients configured with a path prefix.

The Bazaar catalog merges two sources: the PayGate v2 product registry (the
hosted proxy at `paygate.arispay.app/{slug}/*` settles through this
facilitator), and organic sightings — third-party servers emitting the
`bazaar` discovery extension in their 402s, cataloged on successful verify.
Each item's `accepts` mirrors what the resource's real 402 advertises.
Indexers such as x402scan and Coinbase's Bazaar crawl this endpoint to list
our merchants; sellers make themselves directory-visible just by settling
through us with the extension enabled.

## Networks and assets

One deployment settles on one network.

**Live production: Base mainnet (`eip155:8453`) only** — EIP-3009 exact
scheme, USDC and EURC (yes, you can price your API in euros):

| Asset | Currency | Contract | Decimals | Settlement |
|---|---|---|---|---|
| USDC | USD | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 | EIP-3009 `transferWithAuthorization` |
| EURC | EUR | `0x60a3E35Cc302bFA44Cb288Bc5a4F316Fdb1adb42` | 6 | EIP-3009 `transferWithAuthorization` |

The same list is served live by `GET /facilitator` (with the full EIP-712
domain info), `GET /trust`, and `GET /supported` — both in each v2 kind's
`extra.assets` and in the additive top-level `live` block, which separates
the production network from compatibility echoes (v1 kinds and legacy
network identifiers are kept in `kinds` for older clients but are **not**
live settlement surfaces).

**Planned (not live):** Ethereum mainnet (`eip155:1`) for EUR-stablecoin
work is registered but has an empty asset registry until entries are
verified on-chain at deploy time. `GET /facilitator` reports it under
`plannedNetworks` — never point production traffic at a planned network.

### EUR pricing example

```ts
// paygate — price a route in euro cents, settled as EURC on Base
const pw = paygate({ merchantId: "m_123", currency: "EUR" });
app.get("/api/eu-thing", pw({ priceCents: 50 }), handler); // €0.50
```

## Fees

ArisPay charges **no facilitator fee**. Settlement gas is the chain's cost,
paid by the seller.

| Cost | Who pays | How |
|---|---|---|
| Verification | Nobody | Free — no chain writes; serve it at any scale |
| Settlement gas (~$0.001 on Base today) | The seller | Self-settle with your own relayer key (SDK support rolling out), **or** netted payout: our relayer fronts the gas within the epoch and recovers the exact cost — zero margin — at distribution |
| Facilitator margin | — | None. Ever. Value-added services are priced separately if ever offered |

**Launch-period gas fronting:** relayer-submitted direct-payout settles
currently draw on limited relay gas that ArisPay fronts (a bounded daily
budget) while self-settle and the netted pass-through roll out. Any change
to that allowance — including ending it — comes with at least 30 days'
notice, announced here and in the `/supported` response (each v2 kind's
`extra.fees`, `launchGasFronting` + `transitionNoticeDays`).

A note on mechanics: on the EVM exact scheme the payer's EIP-3009 signature
binds the transfer's `to` and `value`, and verification enforces that they
match the seller's requirements — so a facilitator cannot deduct a fee from
the payment itself. Gas pass-through happens at netted payout distribution
(or not at all, when the seller self-settles), never by mutating the
exact-scheme transfer.

## Merchant observability

No account needed for any of this:

- **Request ids.** Every `/verify` and `/settle` response (success or error)
  carries a `requestId` — quote it when investigating a payment. The field is
  additive; every pre-existing wire field is unchanged.
- **Receipts.** `GET /receipts/{txHash}` returns the settlement record: asset,
  amount, payer, payTo, success, gas used/cost (filled async from the
  receipt), and a block-explorer link.
- **Settlement status.** `GET /settlements/{nonce}` looks up a payment by its
  EIP-3009 nonce — the payment identifier on the exact scheme — and reports
  `settled` (with tx hash + timestamp) or `pending`.
- **Idempotency / duplicate protection.** The nonce is claimed atomically
  before settlement; a duplicate settle for the same authorization is
  rejected with `nonce_already_used`, never double-settled.
- **Structured error codes.** `missing_fields`, `internal_error`,
  `nonce_already_used`, `daily_gas_budget_exceeded`, `not_found` — plus
  scheme-level reasons passed through from the upstream x402 kernel.
- **Status + metrics.** `GET /status` (network, mode, uptime, deployment
  commit) and `GET /metrics` (uptime, per-endpoint latency, trailing 24h/1h
  settlement counts and success rate).
- **Trust page.** `GET /trust` — the custody model, relayer address,
  on-chain-verified EIP-712 domains, fee policy, and security model in one
  machine-readable document.

## Webhooks (upcoming)

The webhook **event schema is published now** at `GET /events/schema`;
**delivery is not yet implemented**. Event types:

`payment.verified` · `payment.settled` · `payment.failed` ·
`settlement.confirmed` · `settlement.failed`

Each event will carry `eventId`, `type`, `paymentId` (the EIP-3009 nonce),
`requestId`, `network`, `asset`, `amount`, `payTo`, `payer`, `txHash`,
`status`, `timestamp`, and `error` (`{ code, message }`, on `*.failed` only).
Planned delivery: HTTPS POST, HMAC-SHA256 signature header
(`X-Facilitator-Signature: t=<unix>,v1=<hex>` over `<t>.<rawBody>`),
at-least-once with exponential backoff — consumers dedupe on `eventId`. The
shapes are frozen ahead of delivery so handlers can be built early.

## Why ArisPay instead of another facilitator — or self-hosting?

- **Non-custodial:** funds move buyer → seller directly; there is no balance
  to trust us with and no payout to wait for.
- **No account, no API key:** point `facilitatorUrl` here and you're live.
- **USD and EUR:** USDC + EURC on Base mainnet, EIP-712 domains verified
  on-chain.
- **Transparent fee and gas policy:** no facilitator fee; the machine-readable
  policy in `/supported` is the contract, with ≥30 days' notice on any change
  to the launch gas allowance.
- **Observability built in:** request ids, receipts, settlement status,
  metrics — without signing up.
- **Simple integration:** the default facilitator in
  [`paygate`](https://www.npmjs.com/package/paygate) (accept), paired with
  [`payagent`](https://www.npmjs.com/package/payagent) (pay).
- **Vs self-hosting:** no relayer key management, RPC infrastructure, replay
  protection, or monitoring to run — and you keep settlement control, since
  nothing custodial sits between you and the chain.

Every claim above is verifiable in-band: `/supported`, `/facilitator`,
`/trust`, `/metrics`, and the relayer wallet on Basescan.

## Protocol notes

- **x402Version 2.** The wire format is x402 v2 (`amount`, CAIP-2 network
  ids). v1 sellers keep working: requirements carrying the v1
  `maxAmountRequired` field are normalized to v2 `amount` before
  verification, and `/supported` still echoes v1 kinds for legacy clients.
- **Settlement is EIP-3009.** The payer signs an EIP-712
  `transferWithAuthorization`; the facilitator submits it. Settlement is
  atomic — it either moves the funds or reverts.
- **Replay protection.** The authorization nonce is claimed (unique on
  `(nonce, network)`) before settlement is attempted; a duplicate nonce
  aborts the settle. The service fails closed if the nonce database is
  unreachable.
- **Metering.** Every settle attempt — success or failure — is recorded
  (payer, payTo, asset, amount, tx hash, gas cost) for abuse monitoring.
  Gas fields are filled async from the on-chain receipt, best-effort;
  metering never blocks or masks a settlement outcome.
- **Relayer gas budget.** The launch-period gas fronting is bounded: settles
  abort with `daily_gas_budget_exceeded` once the current UTC day's confirmed
  gas spend crosses the daily budget.
- **Rate limit:** 120 requests/minute per client IP, shared across all
  endpoints.
- **CORS:** explicit origin allow-list, no wildcard. Server-to-server calls
  are unaffected.

## Source and status

This repo carries the facilitator's public documentation. The service is
hosted and operated at `https://facilitator.arispay.app` — there is nothing
to run to use it. Source release is planned as the codebase graduates out
of ArisPay's private monorepo; watch this repo.

Company: [ArisPay](https://arispay.app) · Service status: [status.arispay.app](https://status.arispay.app)
