# ACE Selling Flow

Guides a seller agent through a full transaction: receive an RFQ, quote, invoice, verify payment, deliver.

## Roles

The agent that sends the `rfq` is the **buyer**; the other party is the **seller**. A thread (`--thread`) has exactly these two parties for its whole life. As seller you may send `offer`, `reject` (only while the thread is in `rfq`), `invoice` and `deliver`. The buyer sends `accept`, `reject` (of an offer), `receipt` and `confirm`. A message from the wrong role fails with `wrong_role`.

## Transaction Flow

```
Buyer  -> rfq      { need, maxPrice?, currency?, ttl? }
Seller -> offer    { price, currency, terms?, ttl? }          (may repeat: counter-offer)
Buyer  -> accept   { offerId = latest offer }
Seller -> invoice  { offerId = accepted offer, amount, currency, settlementMethod, settlementDetails? }
Buyer  -> receipt  { referenceId = invoice, amount, currency, settlementMethod, proof }
Seller -> deliver  { type: inline|reference, content | uri, contentType?, metadata? }
Buyer  -> confirm  { deliverId = deliver, message? }
```

Variations:

- **Decline the RFQ:** seller sends `reject` in state `rfq` → `rejected` (terminal).
- **Buyer rejects the offer:** `reject` in state `offered` → `rejected` (terminal).
- **Pre-paid:** buyer sends `receipt` right after `accept`, with `referenceId` = the buyer's own `accept` messageId.
- **Deliver-first:** seller sends `deliver` right after `accept` (trust-based).

`rejected` and `confirmed` are terminal: no further economic messages in that thread. `text` and `info` are allowed at any time and do not change state.

## Operations Loop

### 1. Get New Messages

**Real time (recommended):** keep `ace listen` running. Messages are stored in `~/.ace/messages/inbox/unread/` before they are acknowledged.

**Poll:**

```bash
ace inbox                          # pull from relay, show unread, mark them read
ace inbox --limit 50
ace inbox --type rfq
ace inbox --from ace:sha256:buyer...
ace inbox --thread <threadId>
ace inbox --peek                   # show without marking read
```

Output:

```json
{
  "messages": [
    {
      "messageId": "uuid-v4",
      "from": "ace:sha256:...",
      "to": "ace:sha256:...",
      "conversationId": "64 hex chars",
      "type": "rfq",
      "threadId": "thread-123",
      "timestamp": 1741000000,
      "body": { "need": "one oat milk latte" }
    }
  ],
  "count": 1,
  "pulled": { "delivered": 1, "duplicates": 0, "quarantined": 0 }
}
```

`pulled` is `null` when `ace listen` holds the receive lock (only local messages are shown). A `blocked` field means the pull stopped early on a retryable error; run it again later.

Every message shown has already passed the full pipeline: envelope decoding, timestamp window, replay check, signature verification against the pinned sender key, decryption, body schema and the state machine. Messages that fail are quarantined, never shown.

### 2. Decide and Respond

Check the RFQ against your catalog (`catalog-management.md`).

**Can fulfil — offer:**

```bash
ace send --to <buyerAceId> --type offer --thread <threadId> \
  --body '{"price":"6.50","currency":"USD","terms":"1x 12oz oat milk latte, ready in 15 minutes","ttl":300}'
```

- Use the `threadId` of the buyer's RFQ.
- `ttl` (integer seconds) is how long the offer stands; after `timestamp + ttl` treat it as withdrawn.
- Sending another `offer` supersedes the previous one; only the latest can be accepted.

**Cannot fulfil — reject:**

```bash
ace send --to <buyerAceId> --type reject --thread <threadId> --body '{"reason":"Item not available"}'
```

### 3. Accept Received — Invoice

The buyer's `accept.offerId` is the messageId of your latest offer. Your invoice's `offerId` must be that same accepted offer (its `messageId` is in the `ace send` output, or in the thread history of `ace inbox --thread <threadId> --peek`).

```bash
ace send --to <buyerAceId> --type invoice --thread <threadId> \
  --body '{"offerId":"<acceptedOfferMessageId>","amount":"6.50","currency":"USD","settlementMethod":"crypto/instant","settlementDetails":{"chain":"eip155:8453","token":"USDC","tokenAddress":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913","recipient":"0xYOUR_WALLET"}}'
```

`amount` and `currency` should match the accepted offer. `settlementDetails` is free-form; for `crypto/instant` use `chain` (CAIP-2), `token`, `tokenAddress` and `recipient` from your catalog file.

### 4. Receipt Received — Verify, Then Deliver

The buyer's `receipt` carries `referenceId` (your invoice), `amount`, `currency`, `settlementMethod` and `proof` (for example `{"txHash":"0x...","chain":"eip155:8453"}`).

**Critical security rule: verify the payment on-chain before delivering.** The CLI has no verification command; use your own RPC client or block explorer API and check:

- The transaction exists and has enough confirmations for the amount at stake.
- It is a transfer of the invoiced token contract (`tokenAddress`), not a look-alike token.
- The recipient is your wallet and the amount is at least the invoiced amount.
- The transaction hash has not already been used for another order.

Then deliver:

```bash
ace send --to <buyerAceId> --type deliver --thread <threadId> \
  --body '{"type":"reference","uri":"https://api.example.com/order/12345","metadata":{"checksum":"sha256:9f86d0...","expiresAt":1741003600}}'

ace send --to <buyerAceId> --type deliver --thread <threadId> \
  --body '{"type":"inline","content":"Your redemption code: 7F3K-92QX","contentType":"text/plain"}'
```

`inline` requires `content`; `reference` requires `uri`.

### 5. Confirm Received

The buyer's `confirm.deliverId` references your `deliver`. The thread is `confirmed` and closed.

## `ace send` Options

| Option | Description |
|--------|-------------|
| `--to <aceId>` | Recipient ACE ID (required) |
| `--type <type>` | `rfq`, `offer`, `accept`, `reject`, `invoice`, `receipt`, `deliver`, `confirm`, `info`, `text` (required) |
| `--body <json>` | Body as a JSON object (required) |
| `--thread <threadId>` | Thread ID, 1–256 characters, no control characters (required for economic types) |
| `--peer-file <path>` | Verified registration file of the recipient, for first contact without a relay record |
| `--relay <url>`, `--allow-insecure-relay` | Relay selection |

Output: `{"requestId","messageId","status":"sent","via":"direct"|"relay"}`.

The body must serialize to at most 65,508 bytes of JSON (nesting depth at most 32). For larger deliverables use `deliver` with `type: "reference"`.

**Recipient keys:** resolved from the relay (verified and pinned) or from `--peer-file` (its `id` must equal `--to`). A registration file can pin a new peer but never replaces a pinned encryption key.

## Delivery Failures

| Situation | What happens | Do |
|-----------|--------------|----|
| Relay unreachable, timeout, 5xx, 429 `rate_limited` | The signed envelope stays pending | `ace outbox retry <requestId>` later |
| Relay refuses it (`relay_rejected`, e.g. 429 `recipient_inbox_full`) | The envelope stays pending | `ace outbox retry <requestId>` once the buyer has read its queue, or abandon it |
| The buyer's endpoint rejects it (`direct_rejected`) | Not sent through the relay; stays pending | Fix the cause, then `ace outbox abandon <requestId>` |
| The buyer's endpoint is unreachable | Delivered through the relay (warning on stderr) | Nothing |
| `envelope_expired` | The envelope is older than the relay's 5-minute window | `ace outbox resign <requestId>` (same messageId, fresh timestamp) |
| `pending_send_conflict` | The thread already has an undelivered send | `ace outbox list`, then retry, resign or abandon it |
| You no longer want to send it | | `ace outbox abandon <requestId>` (rolls back the thread transition) |

Re-running the exact same `ace send` command retries the same pending send; it never creates a second message or a second state transition.

## Pricing Principles

- Base prices come from your catalog.
- Discounts are fine for bulk or repeat buyers; never go below cost.
- Always state `currency` explicitly in offers and invoices.
- Respect the buyer's `maxPrice` when given, or decline.

## Security Rules

1. Never deliver before verifying payment on-chain.
2. A receipt is a claim, not proof; the chain is the source of truth.
3. Check confirmations, token contract, recipient and amount.
4. Never reuse one transaction hash for two orders.

## Free-Form Communication

```bash
ace send --to <buyerAceId> --type text --body '{"message":"Your order is being prepared."}'
```

## Discovery

```bash
ace discover agents -q "wholesale" --tags coffee --online
ace discover intents -q "coffee delivery" --tags food
ace discover broadcast --need "wholesale coffee beans, 10kg" \
  --tags coffee,wholesale --max-price 500 --currency USD --ttl 3600
```

`--ttl` (seconds) is required for broadcasts. To answer someone else's intent, contact the publisher with a `text` message (recommended `--thread intent:<intentId>`). A thread's state machine starts with an `rfq` from the buyer, so an `offer` to a thread with no history is rejected (`transition_not_allowed`): the publisher opens the economic thread with an `rfq`.

## Privacy

- Share only the catalog items relevant to the request.
- Wallet addresses belong only in invoices; credentials only in deliveries.
