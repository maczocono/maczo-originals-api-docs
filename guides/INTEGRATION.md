# Integrate with maczo Originals — external integration guide

> **bx dev building the `originals` connector?** Go straight to the exact, implemented wire contract:
> [`ORIGINALS-BX-WIRE.md`](ORIGINALS-BX-WIRE.md) (identity, the two endpoints bx calls on us, the wallet
> callbacks we send, signatures, `900xxx` codes, config to exchange, go-live checklist). This page is the
> broader background.
>
> **Audience:** an engineer **or an AI agent** implementing the connection between an external
> platform (an operator such as **bx**, or any site that wants to offer our games) and **maczo
> Originals**. This page is the **front door**: read it top-to-bottom and you can implement the
> integration without reading the whole codebase. It links to the machine-readable specs where you
> need exact schemas.
>
> **Status:** the **playable demo** integration (Mode A) works **today**. The **seamless-wallet money
> path** (Mode B) is a **frozen design spec** (services pre-build) — implement to the specs below.
>
> **New external merchant?** Implement the **canonical maczo NATIVE dialect** — clean JSON + header
> HMAC-SHA256 + a real `status(txn_ref)`, integer minor units + full `currency_exp`:
> [`docs/MACZO-NATIVE-WIRE.md`](MACZO-NATIVE-WIRE.md) (wire contract) ·
> [`docs/wallet-api/maczo-native-seamless.v1.yaml`](wallet-api/maczo-native-seamless.v1.yaml) (OpenAPI) ·
> [`docs/ONBOARDING.md`](ONBOARDING.md) (self-serve dev→prod). The bx/SG/Regem dialects below are
> **legacy** alternatives kept for existing operators.
>
> **Authoritative sources this guide ties together (drill in for detail):**
> - Central wallet contract (code, dialect-independent): [`libs/wallet_spec/`](../libs/wallet_spec/) — `types.py`, `codes.py`, `errors.py`, `sequence.py`
> - OpenAPI (machine-readable): [`docs/wallet-api/maczo-native-seamless.v1.yaml`](wallet-api/maczo-native-seamless.v1.yaml) (**canonical**, money: maczo→merchant) · [`docs/wallet-api/seamless-wallet.v1.yaml`](wallet-api/seamless-wallet.v1.yaml) (legacy SG dialect) · [`docs/wallet-api/maczo-provider.v1.yaml`](wallet-api/maczo-provider.v1.yaml) (launch + catalog: operator→maczo)
> - Wire protocol reference (Regem dialect, legacy): [`docs/wallet-api/maczo-wallet-protocol.md`](wallet-api/maczo-wallet-protocol.md)
> - Architecture + money-safety: [`docs/DESIGN.md`](DESIGN.md) · Build rules: [`.claude/skills/originals-platform/SKILL.md`](../.claude/skills/originals-platform/SKILL.md)
> - bx-specific facts: [`docs/BX-INTEGRATION.md`](BX-INTEGRATION.md)

---

## 0. TL;DR for an implementer

There are **two ways** to connect, pick by what you need:

| | **Mode A — Embed & catalog** | **Mode B — Seamless wallet (operator)** |
|---|---|---|
| **What** | Show our games in your site; we run the wallet (demo) | You hold the player's real money; we call you to move it |
| **Money** | Our in-memory demo wallet | **Your** wallet is the source of truth |
| **You build** | Read `/api/config`, iframe `/embed/{game_id}` | A **wallet callback service** + a **launch client** conforming to our spec |
| **Status** | **Works today** against the running app | Design/pre-build — implement to the specs |
| **Start at** | [§2](#2-mode-a--embed--catalog-works-today) | [§3](#3-mode-b--seamless-wallet-operator-integration) |

**The one thing to internalize (Mode B):** the operator's wallet is the **source of truth for money**;
maczo never authorizes a bet on a balance it computed. Every money move is **idempotent on a
server-derived `txn_ref`**, follows a strict **ordering invariant** ([§3.4](#34-the-ordering-invariant-non-negotiable)),
and every game outcome is **provably fair** and re-derivable ([§4](#4-provably-fair-verify-any-result)).

---

## 1. Roles & call directions

- **maczo = game provider.** We run the games (pure, provably-fair engine) + the UI.
- **operator (e.g. bx) = the money source of truth.** Holds the player's real balance.

Two independent directions (Mode B):

```
(1) LAUNCH + CATALOG   operator ──▶ maczo    "start a session for player X in game Y" / "list your games"
(2) WALLET (money)     maczo ──▶ operator    "debit the stake" / "credit the win" / "rollback"
```

- Only maczo's `wallet-adapter` egresses to the operator; only `bff-gateway` is public. (Network
  isolation — [SKILL §8](../.claude/skills/originals-platform/SKILL.md).)
- The operator **hosts** the wallet endpoints; **we call them**. We **host** the launch + catalog
  endpoints; the **operator calls them**.

---

## 2. Mode A — Embed & catalog (works today)

Point at a running instance (local: `http://127.0.0.1:8000` via `bash scripts/run-web.sh`; hosted:
`https://originals.maczo.co`). All responses are JSON, no-store.

### 2.1 Read the catalog — `GET /api/config`

Returns global defaults + **every game** with its per-game config and a normalized **bet-form input
spec** (build your UI / bound-checks from this — no scraping):

```jsonc
{
  "currency": "USD", "currency_exp": 2, "e8": 100000000,
  "rtp_e8": 99000000, "min_bet_minor": 101, "max_win_minor": 1000000,
  "autobet_delay_ms": 1500,
  "currencies_meta": { "USD": 2, "THB": 2, "USDT": 8, "BTC": 8, "...": 0 },  // code → minor-unit exponent
  "games": [
    {
      "id": "dice", "name": "Dice", "blurb": "...",
      "rtp_e8": 99000000, "enabled": true, "config_version": 1,
      "min_bet_minor": 101, "max_bet_minor": 1000000, "max_win_minor": 1000000,
      "currencies": ["USD"],
      "inputs": [                       // uniform pre-bet input descriptors
        { "key": "direction", "type": "enum", "label": "Direction", "default": "under", "values": ["under","over"] },
        { "key": "target", "type": "int", "label": "Target roll", "default": 5000, "min": 1, "max": 9999, "scale": 100 }
      ]
    }
    // ... 33 games (dice, limbo, mines, plinko, coinflip, keno, wheel, dragon, hilo, ...)
  ]
}
```

- **Money is integer minor units** + `currency` + `currency_exp` (never float). Display value =
  `minor / 10^currency_exp`. Multipliers/RTP are integers scaled by `e8` (`1e8` = 1.00×).
- `inputs` is the machine-readable bet form: `enum` → render the `values`; `int`/`mult_e8` → a
  stepper/slider within `min..max` (`scale` = display divisor). Games with no pre-bet inputs → `[]`.

### 2.2 Embed a game — `GET /embed/{game_id}`

Each game is its **own standalone page** (no lobby chrome in the DOM). Drop it in an iframe:

```html
<iframe src="https://originals.maczo.co/embed/dice"
        style="width:1366px;height:768px;border:0" allow="autoplay"></iframe>
```

- The page loads exactly that one game (`window.MACZO_EMBED_GAME` is injected server-side).
- The embed posts `maczo:miniplayer` messages to the parent for the pop-out control (optional to handle).
- Unknown `game_id` → `404 {code:"UNKNOWN_GAME"}`.

### 2.3 Drive it via the JSON API (if you build your own UI)

The demo wallet is per-session and in-memory (it's exactly the seam the seamless wallet replaces):

| Endpoint | Purpose |
|---|---|
| `POST /api/session` → `{sid, balance_minor, client_seed, server_seed_hash, nonce, ...}` | create a play session (commits a server-seed **hash** up front) |
| `GET /api/session?sid=` | session state + recent history |
| `POST /api/session/client-seed` `{sid, client_seed}` | set your client seed (`^[A-Za-z0-9_-]{1,64}$`) |
| `POST /api/quote` `{sid, game_id, params, bet_minor}` | odds preview (win chance, multiplier, worst-case max multiplier) — no money |
| `POST /api/bet` `{sid, game_id, params, bet_minor}` | one-shot settle → `{win, multiplier_e8, payout_minor, outcome, server_seed_hash, client_seed, nonce, rtp_e8, config_version, balance_minor}` |
| `POST /api/{game}/start` \| `/reveal` \| `/pick` \| `/cashout` … | interactive rounds (mines, dragon, hilo, crash, blackjack, …) |
| `POST /api/verify` `{game_id, server_seed, client_seed, nonce, params, [rtp_e8]}` | recompute any round from a revealed seed ([§4](#4-provably-fair-verify-any-result)) |

Example one-shot bet:

```bash
SID=$(curl -s -X POST http://127.0.0.1:8000/api/session | jq -r .sid)
curl -s -X POST http://127.0.0.1:8000/api/bet -H 'content-type: application/json' \
  -d "{\"sid\":\"$SID\",\"game_id\":\"dice\",\"params\":{\"direction\":\"under\",\"target\":5000},\"bet_minor\":1000}"
# → {"win":true,"multiplier_e8":198000000,"payout_minor":1980,"outcome":{"roll":...},
#    "rtp_e8":99000000,"config_version":1,"balance_minor":...}
```

`params` for each game = the `key`s from that game's `inputs` (§2.1). Bets are gated per-game
(`enabled`, min/max bet, max-win) and stamped with the `config_version` they settled under.

---

## 3. Mode B — Seamless wallet (operator integration)

This is the production money path. It has **two layers** — implement to both:

1. **The central contract** ([§3.1](#31-the-central-contract-implement-to-this)) — stable, **dialect-independent** semantics
   (operations, idempotency, ordering, money, errors). This is `libs/wallet_spec` and never changes
   when a new operator is added.
2. **The wire dialect** ([§3.2](#32-the-wire-dialect-sggsc-current)) — the concrete HTTP/JSON encoding + signature.
   Two are specified; **SG/GSC+ is the current dialect** (the validated OpenAPI specs). The concrete
   host, `OperatorId`/`operator_code`, and secret are **assigned per operator** — [confirm with us](#7-open-items-confirm-with-the-maczo-side).

> **Adapter pattern:** the operator dialect lives **only** in one `OperatorAdapter` implementation
> ([`libs/wallet_spec/types.py`](../libs/wallet_spec/types.py)). Adding an operator = implement that
> interface + register an `operator_id`. The engine/game-round import only the central types.

### 3.1 The central contract (implement to this)

Four money operations + a status probe. Source of truth: [`libs/wallet_spec/types.py`](../libs/wallet_spec/types.py).

| Op | Meaning | Idempotency key (`txn_ref`) | Notes |
|---|---|---|---|
| `balance` | read spendable balance (display only) | — (read) | money-critical decisions never use this — authorize on the debit response |
| `debit` | take the stake; opens the round | `txn_ref` | operator authorizes (real+bonus); `INSUFFICIENT_FUNDS` = terminal reject |
| `credit` | pay the win; settles | `{txn_ref}:WIN` | **only when `payout_minor > 0`**; must reference the debit's `txn_ref` + same `round_ref` |
| `rollback` | reverse an unsettled debit (recovery) | `{txn_ref}:RB` | `amount_minor` = the **stored original** stake, asserted equal before sending |
| `status` | did this exact `txn_ref` apply? | (the move's key) | for operators with no status endpoint, implemented as **replay-of-same-id** |

Request/response shapes (integer minor units throughout) — `DebitRequest`, `CreditRequest`,
`RollbackRequest`, `MoneyResult`, `BalanceResult`, `StatusResult` in `types.py`. Key fields:

- `operator_id`, `player_ref` (the operator's durable player id, **echoed verbatim**), `txn_ref`,
  `round_ref` (the credit MUST echo the debit's, byte-identical — operators route win→real/bonus by
  it), `game_code`, `amount_minor`/`payout_minor`, `currency`, `currency_exp`, `request_hash`.
- `MoneyResult.duplicate=true` ⇒ the operator reported an **idempotent replay**: treat as
  **authoritative success**, and use **your persisted original-apply snapshot** for the player-facing
  answer — **never** the live balance from the duplicate response.

**Canonical error codes** ([`libs/wallet_spec/codes.py`](../libs/wallet_spec/codes.py)) — your adapter maps
the operator's dialect codes onto these; game-round only ever sees these:

`OK · INSUFFICIENT_FUNDS · SESSION_INVALID · SESSION_REPLACED · PLAYER_BLOCKED · DUPLICATE ·
ALREADY_SETTLED · IDEMPOTENCY_CONFLICT · TXN_NOT_FOUND · OPERATOR_TIMEOUT · OPERATOR_UNAVAILABLE ·
RATE_LIMITED · BAD_REQUEST · OPERATOR_CONFIG_ERROR · SIGNATURE_OR_CLOCK`.
Classify with `is_success()` (OK **or** DUPLICATE), `is_terminal_reject()`, `is_retryable()`,
`needs_reconciliation()` — **branch on these, not on raw operator strings**.

### 3.2 The wire dialect (SG/GSC+, current)

The validated OpenAPI specs are the machine-readable contract:
- **Money (maczo → operator):** [`seamless-wallet.v1.yaml`](wallet-api/seamless-wallet.v1.yaml) —
  `balance`, `withdraw` (=debit/bet), `deposit` (=settle/win/refund), `pushbetdata`; `100x` status
  codes; idempotent on `transaction.id`.
- **Launch + catalog (operator → maczo):** [`maczo-provider.v1.yaml`](wallet-api/maczo-provider.v1.yaml) —
  `launch-game`, `available-products`, `provider-games` ("GameList").

**Signatures** (lowercase-hex MD5, ±300 s window, case-insensitive constant-time compare):
- **Launch/catalog (operator → maczo):** `md5(request_time + secret_key + action + operator_code)`,
  `action ∈ launchgame | productlist | gamelist`. On POST `sign`+`request_time` are body fields; on
  GET they are query params.
- **Wallet callbacks (Regem dialect, [`maczo-wallet-protocol.md`](wallet-api/maczo-wallet-protocol.md)):**
  player-bound `MD5(Func + RequestDateTime + OperatorId + SecretKey + PlayerId)`; transaction-bound
  `MD5(Func + TranId + RequestDateTime + OperatorId + SecretKey + PlayerId)` (`TranId` = `BetId` for
  Bet/Rollback, `ResultId` for GameResult). Verified test vector:
  `MD5("Bet"+"9988"+"2026-06-12T10:00:00Z"+"opX"+"secretY"+"playerZ") = 821041b7763e350fcf08470274e61714`.

> ⚠ **Dialect note (must confirm):** the OpenAPI specs + [`wallet-api/README.md`](wallet-api/README.md)
> record **SG/GSC+** as the decided wire (2026-06-20); [`SKILL §5.1`](../.claude/skills/originals-platform/SKILL.md)
> and [`maczo-wallet-protocol.md`](wallet-api/maczo-wallet-protocol.md) describe the **Regem** dialect.
> Both map onto the same central contract (§3.1). **Confirm the concrete dialect with the maczo side
> before coding the wire layer** — see [§7](#7-open-items-confirm-with-the-maczo-side). The invariants
> (§3.3–§3.5) are identical either way.

### 3.3 Money representation & precision

- **Internal & on our side of the wire:** integer **minor units** + `currency` + `currency_exp`.
  Per-currency exponent is in `libs/game_config` `CURRENCIES` (and `/api/config.currencies_meta`):
  fiat = ISO-4217 (USD/THB/EUR **2**, JPY/VND **0**), stablecoins + crypto (**USDT/USDC/BTC/ETH…**) = **8**,
  aligned with bx's Multi-Currency Wallet (`member_wallets DECIMAL(28,8)`).
- **On the wire to the operator:** send **decimal**, **floored (truncated toward zero)** to the
  operator's precision (bx = **2 dp** for fiat/USD; confirm crypto). Never emit more precision than the
  operator persists; never rely on the operator's rounding. `payout = bet * multiplier_e8 // 1e8`
  (floor) everywhere.
- **int64 id mapping (Regem):** operator ids (`BetId`/`ResultId`/`RoundId`) are **int64**. Our
  `txn_ref` is a string ULID → the adapter maps each to a **persisted, collision-free, stable
  per-operator int64** (`txn_ref→BetId`, `txn_ref:WIN→ResultId`, original `BetId` reused for rollback).
  Never send a string id on a Regem wire (it collapses to `0`).

### 3.4 The ordering invariant (NON-NEGOTIABLE)

Every money path follows this exact order ([SKILL §2](../.claude/skills/originals-platform/SKILL.md),
[DESIGN §3/§7](DESIGN.md), [`libs/wallet_spec/sequence.py`](../libs/wallet_spec/sequence.py)):

```
validate session → freeze/block check → persist PENDING intent (primary, w:majority)
→ DEBIT operator & confirm APPLIED → allocate PF nonce → resolve outcome (pure)
→ commit round (w:majority) → CREDIT (or internal zero-credit on a loss) → reveal to player
```

- **Never** burn a nonce or reveal an outcome before the debit is confirmed **and** the round is
  durably committed. A rejected/unfunded bet leaves the provably-fair nonce chain **gap-free**.
- A **loss** is closed by an internal **zero-credit ledger record** (no operator call) — never leave a
  round debited-but-unsettled. Exactly one debit + one credit (or zero-credit) per round.

### 3.5 Idempotency & recovery (the safety core)

- **`txn_ref` is server-derived and deterministic across retries** (`HMAC(server_secret, operator|player|session|Idempotency-Key)`), never fresh-random per attempt. It is **our unique-index key AND the operator's idempotency key**.
- **One doc per money MOVE:** `BET=txn_ref`, `WIN=txn_ref:WIN`, `ROLLBACK=txn_ref:RB`, each under a unique index — the Mongo unique index is the idempotency authority (no broker, no Redis-as-authority).
- **On duplicate:** the operator returns its dup code (Regem `900409` / SG dup) + live balance → treat as **authoritative success**, **replay your persisted original result**, never re-call the operator.
- **On timeout / unknown:** leave the intent **PENDING**, return a retry/pending signal, let the reconciler resolve via **`status(txn_ref)`** — implemented as **replay-of-same-id** when the operator has no read-only status endpoint (bx does not). **Never auto-credit on an unconfirmed debit.**
- **Same `txn_ref`, different `request_hash` → `IDEMPOTENCY_CONFLICT`** (never overwrite a tx).
- **Rollback** only for a provably-`applied` debit, amount sourced from the stored original; a win owed money → **manual review**, never auto-rolled-back.

### 3.6 Launch & identity

1. Operator calls our **launch** (`launch-game` / Regem `GameLogin`) with the **scoped player id**
   (`{envChar}{tenantHex8}_{username}`, ≤ 32–50 chars) — this **IS the durable identity key**, there is
   no separate launch token; launch = session creation (idempotent on relaunch).
2. We bind that player id ↔ a fresh game session, return a player-facing `url` (open in iframe/redirect).
3. We **echo that exact player id on every wallet callback**. If the operator's launch→identity mapping
   is missing/expired, every wallet call fails closed (bx: `900404`).

### 3.7 Game catalog registration

Operator's game-sync pulls our catalog (`available-products` + `provider-games` / GetGames). Convention:
`game_code = "maczo_dice" | "maczo_limbo" | …` (or the bare id per the provider spec examples), stable
`product_code`. **New games arrive DISABLED** on the operator side — an operator must enable each before
bets succeed (until then a bet is blocked, e.g. bx `900416`).

---

## 4. Provably fair (verify any result)

Every outcome is fixed **before** the bet and re-derivable by anyone. Recipe is **frozen, never
versioned** ([DESIGN §3.1](DESIGN.md)):

- `HMAC-SHA256(key = server_seed, msg = f"{client_seed}:{nonce}:{cursor}")`; consume the digest **4
  bytes at a time, big-endian**, as uint32 (`u / 2^32`). Settlement uses **integer comparisons on the
  uints**, never floats.
- `sha256(server_seed)` is **committed** to you before any bet on that seed; the raw `server_seed` is
  **revealed on rotation** (when it can no longer produce new bets).

**Client-side verification recipe (publish to players):**
```
1. Check sha256(server_seed) == the server_seed_hash you were shown before betting.
2. Recompute uints = uint_stream(server_seed, client_seed, nonce, count_for(game_id, params)).
3. Run the documented game function on those uints + params (integer comparisons).
4. Assert outcome == what you were paid on; assert payout == bet * multiplier_e8 // 1e8.
```
The running app exposes `POST /api/verify` (recompute any round from a revealed seed) and ships a
zero-dependency JS + Python reference verifier.

---

## 5. RTP & per-game config (config-admin)

- **RTP is per-game, versioned, and fail-closed.** Default **0.99**, hard effective floor **0.98**
  (`validate_rtp_e8` rejects ≤ `98_000_000`; a config below the floor never activates). Edge is baked
  into the multipliers; a boot-time sweep validates every game.
- Per-game config = `{rtp_e8, enabled, limits{min_bet_minor, max_bet_minor, max_win_minor}, currencies}`,
  stored versioned. **Every round records the `config_version` it settled under** — pass/pin it so a
  dispute re-derives exactly ([`libs/game_config/`](../libs/game_config/)).
- Staff configure games in the backoffice (`/admin/`, "Games & RTP" → edit; "Config Audit"), backed by
  `POST /api/admin/config/{game_id}` (validated, versioned, audited).
- **A bet's max-win is capped per game;** reject `floor(bet * max_multiplier) > max_win` before
  accepting (the client also caps the stake so the player isn't surprised).

---

## 6. Implementation checklist (for an AI/engineer)

**Mode A (embed) — hours:**
- [ ] `GET /api/config` → cache the catalog; build the game grid + per-game bet form from `inputs`.
- [ ] Embed `/embed/{game_id}` in an iframe (1366×768 works; it's responsive).
- [ ] (optional) Handle the `maczo:miniplayer` postMessage for pop-out; use `/api/verify` to expose fairness.

**Mode B (seamless wallet) — build in this order:**
1. [ ] **Confirm with us** ([§7](#7-open-items-confirm-with-the-maczo-side)): wire **dialect** (SG/GSC+ vs Regem), host/base URLs, `OperatorId`/`operator_code`, `SecretKey`, currency set, win→real routing.
2. [ ] Implement the **signature** for your dialect (§3.2) + verify against the test vector; enforce the ±300 s clock window (NTP).
3. [ ] Implement the **launch** endpoint you host: verify sign → bind scoped `player_ref` ↔ session → return `url`. Idempotent on relaunch.
4. [ ] Expose the **catalog** (`available-products` / `provider-games` / GetGames); games default **disabled** until enabled.
5. [ ] Implement the **wallet callbacks** (`balance`, `debit`/withdraw, `credit`/deposit, `rollback`) → map onto the **central contract** (§3.1): integer minor units, `duplicate`=authoritative-success, one-doc-per-move idempotency, `request_hash` conflict → `IDEMPOTENCY_CONFLICT`.
6. [ ] Map your dialect status codes → **canonical `WalletCode`** (§3.1). Classify on the **body**, not HTTP status (operators return HTTP 200 on business errors).
7. [ ] Honor the **ordering invariant** (§3.4) and **money precision** (§3.3, floor to your wire dp; int64 id mapping if Regem).
8. [ ] Implement **`status(txn_ref)` via replay-of-same-id** for unknown/timeout recovery; **never auto-credit an unconfirmed debit**; wins-owed → manual review.
9. [ ] Contract tests: idempotent replay returns the persisted original; duplicate→success; settle/rollback with no prior debit → reconciliation (never silent); business-error-on-200.

---

## 7. Open items (confirm with the maczo side)

These are genuinely unresolved and **must be pinned before the wire layer is final** (also in
[`docs/BX-INTEGRATION.md`](BX-INTEGRATION.md) §remaining-unknowns and the protocol doc §11):

1. **Wire dialect** — SG/GSC+ (OpenAPI specs, current) vs Regem (protocol doc). Which is live for this operator?
2. **Host + base routes** the operator assigns maczo (launch/catalog) and where we POST wallet callbacks; `OperatorId`/`operator_code` + rotatable `SecretKey`.
3. **Win routing** — real vs bonus; confirm `round_ref`-based routing won't mis-route our real-money wins.
4. **Currency + precision** — currency set; crypto precision on the wire (bx MCW = 8 dp; older ledger = 2 dp); does any currency need > 2 dp on the wire?
5. **Status recovery** — will the operator add a read-only `status(txn_ref)` endpoint, or formally bless **replay-of-same-id**?
6. **int64 id space** (Regem) — range fine for our per-operator monotonic sequence; operator never reuses/wraps.

---

## 8. Sandbox / testing

- Run the app: `bash scripts/run-web.sh` (or `pwsh scripts/run-web.ps1`) → `http://127.0.0.1:8000`.
  Mode-A endpoints are live immediately; `/admin/` (config-admin) prints its login in the boot log.
- Money-path services (Mode B) are pre-build; dev topology (docker-compose + single-node Mongo `rs0` +
  a `bx-mock` honoring idempotency keys) and contract tests live under `tests/contract/` — see
  [SKILL §8–§9](../.claude/skills/originals-platform/SKILL.md).
- **Never** authorize a bet on a cached/computed balance; **never** re-call the operator on a duplicate;
  **never** reveal an outcome before debit-confirmed + round-committed. When in doubt, re-read
  [SKILL §2](../.claude/skills/originals-platform/SKILL.md) before touching money-path code.
