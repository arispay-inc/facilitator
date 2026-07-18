# ArisPay x402 Facilitator


Open x402 facilitator, live on Base mainnet at `https://facilitator.arispay.app`.

Verifies and settles x402 payments for any x402-speaking seller. No auth, no
signup — point your server's facilitator URL at it and you're done. Gas is
sponsored by the facilitator's relayer wallet during the launch phase;
payers and sellers never pay gas. See **Fees** below.

## Integration

Any x402 v2 server built on upstream `@x402/core`:

```ts
import { HTTPFacilitatorClient } from "@x402/core/server";

const facilitator = new HTTPFacilitatorClient({ url: "https://facilitator.arispay.app" });
```

That's the whole integration. The client calls the endpoints below;
request/response shapes are the upstream x402 wire format.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/verify` | Dry-run verification of a signed payment payload |
| POST | `/settle` | Verify + settle on-chain |
| GET | `/supported` | Supported kinds (scheme/network/version), signer addresses, per-kind asset + fee metadata |
| GET | `/facilitator` | Discovery: networks, validated assets, currencies, signers, settlement mode |
| GET | `/discovery/resources` | Bazaar catalog: paginated list of x402 resources settling through this facilitator (x402 spec discovery layer) |
| GET | `/health` | Liveness |

The three spec endpoints are also mirrored under `/facilitator/*`
(`/facilitator/verify`, `/facilitator/settle`, `/facilitator/supported`) for
clients configured with a path prefix. `/discovery/resources` is likewise
mirrored at `/facilitator/discovery/resources`.

The Bazaar catalog merges two sources: the PayGate v2 product registry (the
hosted proxy at `paygate.arispay.app/{slug}/*` settles through this
facilitator), and organic sightings — third-party servers emitting the
`bazaar` discovery extension in their 402s, cataloged on successful verify
(see `src/discovery.ts`). Each item's `accepts` mirrors what the resource's
real 402 advertises. Indexers such as x402scan and Coinbase's Bazaar crawl
this endpoint to list our merchants; sellers make themselves
directory-visible just by settling through us with the extension enabled.

## Networks and assets

One deployment settles on one network. Production runs **Base mainnet
(`eip155:8453`)** only.

| Asset | Contract | Decimals | Settlement |
|---|---|---|---|
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 | EIP-3009 `transferWithAuthorization` |
| EURC | `0x60a3E35Cc302bFA44Cb288Bc5a4F316Fdb1adb42` | 6 | EIP-3009 `transferWithAuthorization` |

The same list is served live by `GET /facilitator` (with the full EIP-712
domain info) and by `GET /supported` in each v2 kind's `extra.assets`.

## Fees

| Phase | Fee |
|---|---|
| **Launch (now)** | $0 — gas is sponsored, bounded by a daily relayer budget and rate limits |
| **Scheduled** | Cost recovery: estimated gas + buffer, floored at $0.001 per settle |

Fees activate with at least 30 days' notice, announced here and in the
`/supported` response (each v2 kind's `extra.fees`). Margin is never taken
on raw settlement.

A note on mechanics: on the EVM exact scheme the payer's EIP-3009 signature
binds the transfer's `to` and `value`, and verification enforces that they
match the seller's requirements — so a facilitator cannot deduct a fee from
the payment itself. When the fee activates it will be collected through a
mechanism that respects that, never by mutating the exact-scheme transfer.

## Protocol notes

- **x402Version 2.** The wire format is x402 v2 (`amount`, CAIP-2 network
  ids). v1 sellers keep working: requirements carrying the v1
  `maxAmountRequired` field are normalized to v2 `amount` before verification
  (`normalizeRequirements` in `src/routes.ts`), and `/supported` still echoes
  v1 kinds for legacy clients.
- **Settlement is EIP-3009.** The payer signs an EIP-712
  `transferWithAuthorization`; the facilitator submits it and pays the gas.
  Settlement is atomic — it either moves the funds or reverts.
- **Replay protection.** The authorization nonce is claimed in a
  `FacilitatorNonce` table (unique on `(nonce, network)`) before settlement is
  attempted; a duplicate nonce aborts the settle. The service fails closed if
  the nonce database is unreachable.
- **Metering.** Every settle attempt — success or failure — is recorded in a
  `FacilitatorSettle` table (payer, payTo, asset, amount, tx hash, gas cost)
  for abuse monitoring and future pricing. Gas fields are filled async from
  the on-chain receipt, best-effort; metering never blocks or masks a
  settlement outcome.
- **Relayer gas budget.** Sponsored gas is bounded: settles abort with
  `daily_gas_budget_exceeded` once the current UTC day's confirmed gas spend
  crosses `FACILITATOR_DAILY_GAS_BUDGET_ETH` (default 0.01 ETH).
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

