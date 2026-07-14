# maczo NATIVE wire contract (canonical dialect for external merchants)

> The exact, implemented wire an external **merchant** integrates against. This is the **canonical**
> dialect (clean JSON, header HMAC-SHA256, integer minor units + full `currency_exp`, a real
> `status(txn_ref)`). The bx/SG/Regem dialects ([`ORIGINALS-BX-WIRE.md`](ORIGINALS-BX-WIRE.md),
> [`wallet-api/maczo-wallet-protocol.md`](wallet-api/maczo-wallet-protocol.md)) are **legacy** and kept
> only for operators already integrated on them.
>
> Machine-readable: [`wallet-api/maczo-native-seamless.v1.yaml`](wallet-api/maczo-native-seamless.v1.yaml)
> (OpenAPI 3.0.3, validated). Central types (code): [`libs/wallet_spec/`](../libs/wallet_spec/).

## 1. Roles & directions

- **maczo = game provider.** Runs the provably-fair games + UI.
- **merchant (operator) = money source of truth.** Holds the player's real balance. maczo's ledger is a
  display-only mirror; maczo never authorizes a bet on a balance it computed.

Two independent directions:

```
(1) LAUNCH + CATALOG + KICK   merchant ──▶ maczo   (POST /GameLogin, /GetGameList, /KickPlayer, /ReplayLogin)
(2) SEAMLESS WALLET (money)   maczo ──▶ merchant   (POST {base}{prefix}/debit|credit|rollback|status|balance)
```

You (the merchant) implement **both**: the small launch/catalog surface you *call on us*, and the
**seamless wallet** we *call on you*.

## 2. Identity & signing (header HMAC-SHA256)

Each merchant is registered in maczo's operator registry with an `operator_id`, a wallet callback base
URL + path prefix, an inbound IP allowlist, allowed currencies/games, and one or more **signing keys**
(versioned by `key_id`, so a key can be rotated with an overlap window). Your signing secret never
crosses the wire.

Every request (both directions) is signed:

- `X-Signature`: `HMAC_SHA256(secret, canonical)` lowercase hex
- `X-Timestamp`: unix seconds (integer string), within **±60 s** of now
- `X-Nonce`: single-use opaque nonce (reject a seen nonce)
- `X-Key-Id`: selects the per-merchant key

`canonical` is the newline-joined string (no trailing newline):

```
METHOD
PATH
X-Timestamp
X-Nonce
sha256hex(rawBody)      # sha256 of the exact request-body bytes; empty body -> sha256("")
```

Reference implementation: [`libs/native_sig/sig.py`](../libs/native_sig/sig.py). Reject outside the
window or on a replayed nonce (fail closed → `SIGNATURE_OR_CLOCK`).

## 3. Money model (non-negotiable)

- **HTTP status is ALWAYS `200`.** Classify the outcome on the body `code`, never on the HTTP status.
- Amounts are **integer minor units** (`amount_minor`) + `currency` (ISO-4217) + `currency_exp`
  (minor-unit exponent). Never a float; the full `currency_exp` is supported (no 2-dp cap).
- **Idempotency key = `txn_ref`.** A duplicate returns `code: DUPLICATE`, `duplicate: true`, and the
  **live** balance — it never re-moves money.
- A `credit` (win) or `rollback` MUST reference a prior applied debit via `ref_txn_ref`; a settle with no
  prior debit returns `code: TXN_NOT_FOUND` (never silent).
- `status(txn_ref)` is a **read-only** query of whether a move applied — it MUST NOT move money.
- Unknown / unrecognised `code` ⇒ fail closed (treated as `BAD_REQUEST`).

**Canonical codes** (`libs/wallet_spec/codes.py`): `OK`, `DUPLICATE`, `INSUFFICIENT_FUNDS`,
`SESSION_INVALID`, `SESSION_REPLACED`, `PLAYER_BLOCKED`, `ALREADY_SETTLED`, `IDEMPOTENCY_CONFLICT`,
`TXN_NOT_FOUND`, `OPERATOR_TIMEOUT`, `OPERATOR_UNAVAILABLE`, `RATE_LIMITED`, `BAD_REQUEST`,
`OPERATOR_CONFIG_ERROR`, `SIGNATURE_OR_CLOCK`.

## 4. Endpoints you implement (maczo → you)

`POST {wallet_callback_base}{wallet_callback_prefix}/{func}` — always HTTP 200; body is JSON.

| func | body (money fields) | response |
|---|---|---|
| `debit` | `operator_id, player_ref, txn_ref, round_ref, game_code, amount_minor, currency, currency_exp` | `{status, code, duplicate, old_balance_minor, new_balance_minor, currency, currency_exp, operator_ref}` |
| `credit` | + `payout_minor, valid_bet_minor, ref_txn_ref` | same |
| `rollback` | `amount_minor` (== original debit), `ref_txn_ref` | same |
| `status` | `operator_id, txn_ref` | `{status, applied, old_balance_minor?, new_balance_minor?, currency, currency_exp}` |
| `balance` | `operator_id, player_ref, currency` | `{status, balance_minor, currency, currency_exp}` |

The **ordering invariant** maczo follows: validate session → freeze check → persist PENDING intent →
**debit you & confirm APPLIED** → allocate nonce → resolve → commit round → credit (or internal
zero-credit on a loss — **no `credit` call for a 0 payout**) → reveal. A loss never calls you.

## 5. Endpoints you call (you → maczo)

The launch/catalog/kick surface (`POST /GameLogin`, `/GetGameList`, `/KickPlayer`, `/ReplayLogin`) is
verified per-operator against your registered dialect + signing key + IP allowlist. `/GameLogin` returns
a launch `Url`; open it in an iframe/redirect. See [`INTEGRATION.md`](INTEGRATION.md) §3 for the launch
body and [`ONBOARDING.md`](ONBOARDING.md) for the dev→prod steps.

## 6. Provably fair

Every result is re-derivable from `(server_seed, client_seed, nonce)` via
`HMAC-SHA256(server_seed, "client_seed:nonce:cursor")`. The server-seed hash is committed before use and
revealed on rotation. See [`INTEGRATION.md`](INTEGRATION.md) §4 and the public verifier
(`verify.maczo.co`).

## 7. Go-live checklist

1. Register the merchant in the maczo backoffice → get `operator_id` + a `key_id`.
2. Implement the five wallet endpoints (§4) + verify our signature (§2).
3. Set your `wallet_callback_base` + prefix; hand maczo your signing secret out-of-band.
4. Test on the **dev sandbox** (`originals-api-dev.maczo.co`) against the `native-mock` reference merchant.
5. maczo staff approve go-live (adds your callback FQDN to the egress allowlist) → flip `status:active`.

See [`ONBOARDING.md`](ONBOARDING.md) for the full self-serve flow + endpoint tables.
