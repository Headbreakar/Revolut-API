# Revolut Merchant API — Payment Implementation Guide

**Author:** Prem Kumar
**Scope:** Order creation → payment processing → refunds, with DB schema, webhooks, and security
**API version referenced:** `2026-04-20` (Revolut Merchant API, header-versioned)

---

## 1. How the whole thing fits together

Revolut's Merchant API is built around one core object: the **Order**. You don't call a "charge card" endpoint directly like some older gateways — instead you create an Order, then a payment method (card field, Revolut Pay, Apple/Google Pay, hosted checkout) pays *into* that order. A refund is technically a second Order of `type: refund`, linked back to the original via `related_order_id`. Once you get used to this "everything is an order" model, the rest of the API is pretty consistent.

Three layers in a production integration:

1. **Your backend** — creates orders, stores payment state, never touches raw card data.
2. **Revolut's checkout surface** — either the hosted checkout page (redirect) or the embedded Merchant Web SDK (card field / widgets rendered in an iframe on your site). Either way, card data goes straight from the customer's browser to Revolut, never through your server.
3. **Webhook listener** — the source of truth for order state changes, since payment confirmation is asynchronous (3DS challenges, bank processing delays, etc.).

Authentication is a static bearer token, no OAuth dance:

```
Authorization: Bearer sk_1234567890ABCdefGHIjklMNOpqrSTUvwxYZ...
Revolut-Api-Version: 2026-04-20
```

The secret key lives only on your server. There's a separate public key that goes to the frontend for initializing the Merchant Web SDK. Every request that touches money must also carry the `Revolut-Api-Version` header — if you omit it, the request is rejected outright. I'm pinning `2026-04-20` for this implementation since it's the latest stable version as of writing.

---

## 2. Order creation

**Endpoint:** `POST /api/orders`

This is always a server-to-server call. It must never be initiated from the frontend because it requires the secret key.

### Request (minimum viable + the fields I'd actually use)

```json
{
  "amount": 4999,
  "currency": "GBP",
  "description": "Order #ORD-10234",
  "customer": {
    "email": "customer@example.com",
    "full_name": "Jane Doe"
  },
  "line_items": [
    {
      "name": "Wireless Mouse",
      "type": "physical",
      "quantity": { "value": 1 },
      "unit_price_amount": 4999,
      "total_amount": 4999
    }
  ],
  "capture_mode": "automatic",
  "merchant_order_data": {
    "reference": "ORD-10234"
  },
  "redirect_url": "https://mystore.com/checkout/success",
  "expire_pending_after": "PT30M"
}
```

Notes that actually matter in practice:

- **`amount` is always in minor units.** 4999 in GBP is £49.99. This trips people up constantly, especially with currencies like JPY or ISK where there's no minor unit at all (100 = 100 units, not 1.00). Always look up the currency's exponent before hardcoding cent math.
- **`line_items` total must exactly equal `amount`.** If you're doing tax/discount math, the sum of `total_amount` across line items has to match the order total or the request 400s.
- **`capture_mode: automatic` vs `manual`** is the single most important architectural decision here. Automatic captures funds the moment authorization succeeds — good for digital goods, bad for anything with a fulfillment delay (you'd be holding customer money before you've shipped). Manual capture authorizes and holds the funds, and you capture explicitly later (see §4). For e-commerce with physical shipping, I'd default to manual and capture on dispatch.
- **`merchant_order_data.reference`** is your own internal order ID. Always set this — it's what you match against in webhooks and reconciliation, since Revolut's order `id` is meaningless to your existing systems.
- **`expire_pending_after`** auto-fails abandoned orders (someone opens checkout, never pays, tab sits open forever). Without this, pending orders can sit indefinitely.

### Response — the fields that matter for what comes next

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "token": "ord_tok_9fc01989...",
  "state": "pending",
  "amount": 4999,
  "currency": "GBP",
  "outstanding_amount": 4999,
  "checkout_url": "https://checkout.revolut.com/pay/550e8400...",
  "created_at": "2026-08-03T14:22:00Z"
}
```

- **`id`** — the permanent order ID. Store this. Every subsequent operation (capture, cancel, refund, retrieve) uses this ID.
- **`token`** — a *temporary* ID, valid only until the payment is authorized. This is what you hand to the frontend Merchant Web SDK when embedding checkout directly on your site instead of redirecting to `checkout_url`. Because it expires after authorization, never persist it as a long-term reference — use `id` for that.
- **`checkout_url`** — if you're using the hosted checkout page approach (redirect flow, least frontend work), this is the URL you redirect the customer to.
- **`state: pending`** — order created, no payment attempt yet. This is the state you write to your `orders` table immediately after this response returns.

At this point your backend has one job left in the synchronous flow: save the order row and redirect the customer (or hand `token` to the embedded widget). Everything after this is async and driven by webhooks.

---

## 3. Payment processing (what happens between order creation and completion)

The customer pays via whichever method you've enabled — card, Apple Pay, Google Pay, Revolut Pay, or Pay by Bank. Regardless of method, the order moves through this lifecycle:

```
pending → processing → authorised → completed
                    ↘ failed
authorised → (manual capture) → completed
authorised → (cancel) → cancelled
```

| State | Meaning |
|---|---|
| `pending` | Order created, no successful payment attempt yet |
| `processing` | Payment submitted, being verified/authorized |
| `authorised` | Funds reserved on the card, not yet captured (manual capture mode only) |
| `completed` | Funds captured and settling to your account |
| `cancelled` | Order or authorization cancelled before capture |
| `failed` | Payment declined or order expired |

Each order also carries a `payments` array — a single order can have multiple payment *attempts* (e.g., customer's first card declines, they retry with a different card). Each payment object has its own `state` (much more granular — `authentication_challenge`, `authorisation_passed`, `declined`, etc.) plus a `decline_reason` when applicable (`insufficient_funds`, `invalid_cvv`, `do_not_honour`, and so on — there's a fixed enum of about 25 reasons).

### 3DS handling

If a payment's state is `authentication_challenge`, the response includes an `authentication_challenge` object with an `acs_url` — this is the bank's 3D Secure challenge page. If you're using the embedded Merchant Web SDK, it handles the redirect/iframe for this automatically. If you built your own custom flow against the raw API, you're responsible for redirecting the customer to `acs_url` and handling the return.

### What you actually need to store from a payment object

From the `payments[]` array once a payment settles, the fields worth persisting:

- `id` — payment ID (distinct from order ID)
- `state`, `decline_reason`, `bank_message` — for support/dispute purposes
- `payment_method.type` — card / apple_pay / google_pay / revolut_pay_card / revolut_pay_account / sepa_direct_debit
- `payment_method.card_last_four`, `card_brand`, `card_expiry` — display-safe card info, never the PAN
- `payment_method.fingerprint` — a stable 44-char hash identifying the payment instrument across transactions, useful for fraud/dedup logic (e.g., blocking a card associated with chargebacks) without storing the actual card number
- `authorisation_code` — issuer's approval code, needed if you ever have to manually reference a transaction with the bank
- `network_transaction_id` — needed for linking recurring/retry attempts to the original authorization

---

## 4. Capturing (manual capture mode)

**Endpoint:** `POST /api/orders/{order_id}/capture`

If you set `capture_mode: manual` at creation, the order sits in `authorised` state holding funds until you explicitly capture. This is idempotent — calling capture twice with the same amount just returns the current state instead of double-charging, but calling it twice with *different* amounts errors out. Good practice: fire capture the moment your fulfillment pipeline confirms the order is going out (e.g., from your shipping/inventory service), not before.

There's also a hard deadline: `capture_deadline` on the payment object tells you when the issuer will auto-release the hold if you never capture. Don't sit on authorized-but-uncaptured orders indefinitely.

---

## 5. Refunds

**Endpoint:** `POST /api/orders/{order_id}/refund`

```json
{
  "amount": 4999,
  "currency": "GBP",
  "description": "Refund for returned item",
  "merchant_order_data": {
    "reference": "REFUND-ORD-10234-1"
  }
}
```

Key mechanics:

- Refunds only work on orders in `completed` state. You cannot refund a pending or authorized-but-uncaptured order — for that you'd cancel instead.
- `currency` must match the original order's currency exactly.
- Partial refunds are supported, and you can issue multiple partial refunds against one order as long as the cumulative total doesn't exceed the original `amount`. This means your DB schema needs to support many-refunds-per-order, not a single refund flag.
- The response is itself a new Order object, `type: refund`, with `related_order_id` pointing back to the original. Same lifecycle states apply (`pending` → `completed`) — refunds aren't instant either, they go through processing.
- **Always send an `Idempotency-Key` header on refund requests.** If your request times out and you retry, without an idempotency key you risk double-refunding. Use something derived from your internal refund record ID.

```
Idempotency-Key: refund-req-a3f9c2e1
```

I'd generate this key deterministically from your own refund record's primary key (or a UUID stored *before* the request fires) so retries — even across server restarts — reuse the same key.

---

## 6. Webhooks

Polling order status is a bad idea at any scale — webhooks are the intended source of truth for state transitions.

**Setup:** `POST /api/webhooks` with a URL and list of subscribed events (at minimum `ORDER_COMPLETED` and `ORDER_AUTHORISED`; also worth adding refund and dispute events if you support those flows). Revolut returns a `signing_secret` on creation — store this securely, it's what you use to verify incoming payloads are actually from Revolut and not spoofed.

**Important gotcha:** delivery order isn't guaranteed. For a normal completed payment, you'd expect `ORDER_AUTHORISED` then `ORDER_COMPLETED`, but if the first delivery attempt fails and gets requeued, you might receive `ORDER_COMPLETED` before `ORDER_AUTHORISED`. Your handler needs to be idempotent and state-driven (i.e., "set order to X" not "assume this is the Nth event"), never assume ordering.

### Signature verification (this is the part people skip and regret)

Every webhook POST includes a `Revolut-Signature` header and a `Revolut-Request-Timestamp` header. You reconstruct the signed payload and compare HMACs:

```
payload_to_sign = "v1." + timestamp + "." + raw_request_body
expected_signature = "v1=" + HMAC_SHA256(signing_secret, payload_to_sign)
```

Node/Express implementation:

```javascript
const crypto = require('crypto');

function verifyRevolutWebhook(req, signingSecret) {
  const timestamp = req.headers['revolut-request-timestamp'];
  const receivedSignature = req.headers['revolut-signature'];
  const rawBody = req.rawBody; // must be the raw, unparsed body string

  const payloadToSign = `v1.${timestamp}.${rawBody}`;
  const expectedSignature = 'v1=' + crypto
    .createHmac('sha256', signingSecret)
    .update(payloadToSign)
    .digest('hex');

  // timing-safe comparison to avoid timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(expectedSignature),
    Buffer.from(receivedSignature)
  );
}
```

Two things that will bite you if you're not careful:

1. **You need the raw request body, not the parsed JSON.** If you're using `express.json()` globally, by the time your handler runs the body's already been parsed and re-serializing it won't byte-for-byte match what was signed (key ordering, whitespace). Use a raw body middleware scoped just to the webhook route.
2. **Reject anything older than a few minutes** based on the timestamp header, to guard against replay attacks even with a valid signature.

Also reject on signature mismatch with a 401 — don't process the event, don't 200 it, don't log-and-continue.

---

## 7. PCI-compliant database schema

The single most important design decision: **your database should never contain a full card number, CVV, or any raw payment credential.** Revolut's checkout surface (hosted page or embedded widget) handles all of that — card data never transits your server, which is what keeps you at PCI DSS SAQ A (the lightest self-assessment tier) instead of SAQ D. The schema below only stores *references and display-safe metadata*.

```sql
-- Orders: mirrors Revolut's order lifecycle, keyed by our own PK + their ID
CREATE TABLE orders (
    id                  BIGSERIAL PRIMARY KEY,
    reference           VARCHAR(64) UNIQUE NOT NULL,      -- our internal order ref, e.g. ORD-10234
    revolut_order_id    UUID UNIQUE NOT NULL,              -- Revolut's `id`
    customer_id         BIGINT NOT NULL REFERENCES customers(id),
    amount_minor        BIGINT NOT NULL,                   -- always minor units, never float
    currency            CHAR(3) NOT NULL,
    state               VARCHAR(20) NOT NULL,               -- pending/processing/authorised/completed/cancelled/failed
    capture_mode        VARCHAR(10) NOT NULL DEFAULT 'automatic',
    outstanding_amount_minor BIGINT,
    refunded_amount_minor    BIGINT DEFAULT 0,
    checkout_url         TEXT,
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_revolut_id ON orders(revolut_order_id);
CREATE INDEX idx_orders_state ON orders(state);

-- Payments: one order can have multiple payment attempts
CREATE TABLE payments (
    id                   BIGSERIAL PRIMARY KEY,
    order_id             BIGINT NOT NULL REFERENCES orders(id),
    revolut_payment_id   UUID UNIQUE NOT NULL,
    state                VARCHAR(30) NOT NULL,
    decline_reason        VARCHAR(50),
    bank_message          TEXT,
    amount_minor          BIGINT NOT NULL,
    currency               CHAR(3) NOT NULL,
    payment_method_type    VARCHAR(30),        -- card / apple_pay / google_pay / revolut_pay_card / etc.
    card_brand              VARCHAR(20),        -- visa/mastercard/amex — display only
    card_last_four            CHAR(4),
    card_expiry                CHAR(5),          -- MM/YY, display only
    payment_fingerprint          CHAR(44),       -- for fraud/dedup logic, not reversible to PAN
    authorisation_code            VARCHAR(10),
    network_transaction_id         VARCHAR(64),
    created_at                      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_order_id ON payments(order_id);
CREATE INDEX idx_payments_fingerprint ON payments(payment_fingerprint);

-- Refunds: separate table, not a flag on orders, since multiple partials are allowed
CREATE TABLE refunds (
    id                   BIGSERIAL PRIMARY KEY,
    original_order_id    BIGINT NOT NULL REFERENCES orders(id),
    revolut_refund_order_id UUID UNIQUE NOT NULL,  -- Revolut's refund order `id`
    amount_minor           BIGINT NOT NULL,
    currency                 CHAR(3) NOT NULL,
    state                     VARCHAR(20) NOT NULL,
    reason                     TEXT,
    idempotency_key             VARCHAR(64) UNIQUE NOT NULL,
    created_at                   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refunds_original_order ON refunds(original_order_id);

-- Webhook event log: for replay protection, debugging, and audit trail
CREATE TABLE webhook_events (
    id                BIGSERIAL PRIMARY KEY,
    revolut_event_id  VARCHAR(64),           -- dedupe key if Revolut provides one, else hash of payload+timestamp
    event_type         VARCHAR(50) NOT NULL,   -- ORDER_COMPLETED, ORDER_AUTHORISED, etc.
    order_id             BIGINT REFERENCES orders(id),
    raw_payload            JSONB NOT NULL,
    signature_valid          BOOLEAN NOT NULL,
    processed                  BOOLEAN NOT NULL DEFAULT false,
    received_at                 TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at                  TIMESTAMPTZ
);

CREATE INDEX idx_webhook_events_order_id ON webhook_events(order_id);
CREATE UNIQUE INDEX idx_webhook_events_dedupe ON webhook_events(revolut_event_id) WHERE revolut_event_id IS NOT NULL;
```

Design decisions worth explaining if asked in interview:

- **No card number, no CVV, anywhere.** Only `card_last_four`, `card_brand`, `card_expiry` — all display-safe fields that come straight back from Revolut's API response, never captured by our own forms.
- **`payment_fingerprint`** lets us do fraud detection (block a card that charged back before) without ever holding the reversible card number.
- **Amounts are always `BIGINT` in minor units**, never `FLOAT`/`DECIMAL` derived from float math. Floating point + money is a classic bug source.
- **`webhook_events` is an append-only audit log**, separate from the order state itself. This means if a webhook handler crashes mid-processing, we can replay from this table, and it also gives us a forensic trail for support disputes ("customer says they paid, we say we didn't get confirmation — check the log").
- **`idempotency_key` on refunds is UNIQUE**, enforced at the DB layer, not just as an API header — belt and suspenders against double-refunding from a retried request.

---

## 8. Error handling

A few categories to handle distinctly:

**Validation errors (400)** — malformed request, e.g. line items not summing to order amount, missing required address fields for airline/marketplace industry data. Caught before the request even reaches the payment network; fix and retry.

**Authentication errors (401)** — bad or expired secret key. Should never happen in production unless a key was rotated and the deploy didn't pick it up; alert loudly if this occurs.

**Payment declines** — not an API error at all; the request succeeds (201), but the *payment* inside the order has `state: declined` or `failed` with a `decline_reason`. Map these to customer-facing messages instead of showing raw enum values — `insufficient_funds` and `invalid_cvv` need very different UI copy than `suspected_fraud` (where you probably don't want to tell the customer exactly why, for security reasons).

**Network/timeout errors on your side** — always retry idempotent GETs freely. For POSTs that create money movement (create order, refund), only retry with an `Idempotency-Key` — otherwise a timeout followed by a blind retry can double-charge or double-refund.

**Webhook delivery failures** — Revolut retries failed webhook deliveries automatically on their side, but your endpoint should also be defensive: return 200 only after you've durably persisted the event (write to `webhook_events` first, ack second, process asynchronously if the business logic is heavy). If your endpoint 500s, treat that as expected — the retry will come.

---

## 9. Security best practices (beyond PCI scope avoidance)

- **Secret key**: environment variable / secrets manager, never in source control, never in frontend bundles. Rotate on any suspected leak.
- **Webhook signature verification is mandatory**, not optional — an unauthenticated webhook endpoint is an open door to fake "payment completed" events that could trigger free fulfillment.
- **TLS everywhere**, obviously, including the webhook receiver endpoint (Revolut requires HTTPS for webhook URLs).
- **Rate-limit and validate webhook payload size** before parsing, standard hardening against malformed/oversized payloads.
- **Idempotency keys on all money-moving mutations** (orders, refunds), generated once per logical operation and reused across retries.
- **Least-privilege on the DB**: the service account writing `payments`/`refunds` shouldn't have blanket table access if you're running this in a larger system with other services touching the same database.
- **Log payment metadata, never payment credentials** — even in error logs / stack traces, make sure card data (which you never have) and full webhook payloads with sensitive customer PII are handled per your data retention policy, not dumped into plaintext logs indefinitely.

---

## 10. Assumptions made in this implementation

- Assumed **hosted checkout page (redirect) flow** as the primary integration path rather than fully embedding the Merchant Web SDK card field, since it requires the least custom frontend PCI-sensitive code — worth calling out explicitly if the assessment expected the embedded widget approach instead.
- Assumed **manual capture mode** for the reference implementation, on the basis that most e-commerce with physical fulfillment shouldn't capture funds before shipment; automatic capture is a one-line config change if the business model is digital goods instead.
- Assumed **GBP** as the reference currency in examples; currency handling (minor unit conversion) generalizes to any ISO 4217 currency Revolut supports.
- Assumed a **single Postgres database** for the schema; in a real production system, `webhook_events` might reasonably live in a separate append-only store depending on volume.
- Did not implement **subscriptions, payouts, or disputes** endpoints, since the brief scoped this to order creation, payment processing, and refunds specifically.

---

## References

- [Revolut Merchant API — Orders](https://developer.revolut.com/docs/merchant/orders)
- [Create an order](https://developer.revolut.com/docs/merchant/create-order)
- [Refund an order](https://developer.revolut.com/docs/merchant/refund-order)
- [Webhooks](https://developer.revolut.com/docs/merchant/webhooks)
- [Verify the payload signature](https://developer.revolut.com/docs/guides/merchant/monitor-and-observe/webhooks/verify-the-payload-signature)
