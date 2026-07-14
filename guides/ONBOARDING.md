# Merchant onboarding — dev sandbox → production

Self-serve guide to connect an external **merchant** to maczo Originals as a seamless aggregator. Uses
the **canonical native dialect** ([`MACZO-NATIVE-WIRE.md`](MACZO-NATIVE-WIRE.md)); existing operators on
the legacy bx/SG/Regem dialect follow [`ORIGINALS-BX-WIRE.md`](ORIGINALS-BX-WIRE.md) instead.

## Endpoints

| Surface | Dev / sandbox | Production |
|---|---|---|
| Merchant → us (launch / catalog / kick) | `https://originals-api-dev.maczo.co` | `https://originals-api.maczo.co` |
| Merchant portal (your reports + integration) | `https://portal-dev.maczo.co` | `https://portal.maczo.co` |
| Staff backoffice (maczo) | `https://admin-dev.maczo.co` | `https://admin.maczo.co` |
| Public docs | `https://docs.maczo.co` | `https://docs.maczo.co` |

(Dev + prod run the SAME images with the SAME env-var contract — code never branches on environment;
only config/registry data differs.)

## What you provide / what maczo provides

| maczo provides | you provide |
|---|---|
| `operator_id`, a `key_id`, the launch/catalog endpoints, the games catalog (`/GetGameList`) | Your **wallet callback base URL** + path prefix (`{base}{prefix}/debit|credit|rollback|status|balance`) |
| The native wire contract + OpenAPI + the `native-mock` reference merchant | Your **signing secret** (handed to maczo out-of-band, never on the wire) |
| Dev sandbox + a staff go-live approval | Your inbound **IP allowlist** (hosts that call our launch surface) |

## Steps

1. **Register** — maczo staff onboard your merchant in the backoffice (`admin[-dev].maczo.co`):
   `operator_id`, `dialect: native`, base currency + allowed currencies, allowed games, commission
   terms. You receive portal credentials (`merchant-admin`).
2. **Get a key** — in the portal (or via staff), add a signing `key_id`; hand maczo the secret
   out-of-band (it is stored only by reference, never in plaintext).
3. **Implement the wallet** — the five endpoints in [`MACZO-NATIVE-WIRE.md`](MACZO-NATIVE-WIRE.md) §4,
   verifying our header-HMAC signature (§2). Validate the OpenAPI:
   `python -m openapi_spec_validator docs/wallet-api/maczo-native-seamless.v1.yaml`.
4. **Set your callback** — in the portal, set `wallet_callback_base` + prefix and your inbound IP
   allowlist.
5. **Test on the sandbox** — point at `originals-api-dev.maczo.co`. maczo runs a `sandbox` test-merchant
   backed by `native-mock` (tools/native-mock) so you can drive launch → bet → settle → reveal end-to-end
   and confirm your wallet honors idempotency (`txn_ref`), insufficient-funds, and `status`.
   The local money stack (`deploy/compose/docker-compose.dev.yml`) brings up the full path + a
   `native-mock` merchant if you want to test entirely offline.
6. **Go-live approval (staff)** — maczo adds your `callback_fqdn` to the production egress allowlist
   (only `wallet-adapter` egresses to merchants) and flips your `status: active`. Self-service handles
   credentials + config; production egress + enablement stay staff-approved.
7. **Reconcile** — use the portal reports (GGR / turnover / RTP-actual / settlement) and your own ledger
   to reconcile daily; commission statements are per your configured basis — **GGR (the merchant's win
   = bets − payouts) by default**, or turnover / player-payouts.

## Money-safety you can rely on

- The **operator (you) is the source of truth for money**; maczo never authorizes a bet on a computed
  balance. Every money move is idempotent on a server-derived `txn_ref` (double idempotency: our unique
  index AND your idempotency key).
- Strict **ordering invariant** (debit confirmed before any nonce is burned or outcome revealed); a loss
  writes an internal zero-credit and does **not** call your wallet.
- Every outcome is **provably fair** and re-derivable; commission is a billing figure computed on
  aggregated totals, **never** deducted from a player payout.
