# ACE Selling Flow

Core selling skill. Guides a seller agent through the full transaction: receive buyer RFQ, quote, handle payment, and deliver.

## Transaction Flow

```
Buyer  -> rfq      -> Seller reviews catalog, decides pricing
Seller -> offer    -> Buyer reviews quote
Buyer  -> accept   -> Seller sends invoice (payment instructions)
Seller -> invoice  -> Buyer completes payment
Buyer  -> receipt  -> Seller verifies payment on-chain
Seller -> deliver  -> Buyer receives goods/service
Buyer  -> confirm  -> Transaction complete
```

Either party can send `reject` at any time to cancel. All economic messages require `--thread` to maintain state.

## Message Types

| Type | Direction | Purpose |
|------|-----------|---------|
| `rfq` | buyer → seller | Request for quote |
| `offer` | seller → buyer | Price quote |
| `accept` | buyer → seller | Accept offer |
| `invoice` | seller → buyer | Payment instructions (chain, token, amount, address) |
| `receipt` | buyer → seller | Payment proof (txHash) |
| `deliver` | seller → buyer | Deliver goods/service |
| `confirm` | buyer → seller | Confirm receipt |
| `reject` | either | Cancel transaction |
| `text` | either | Free-form communication (no thread required) |

## Operations Loop

### 1. Get New Messages

**Option A: Real-time push (recommended)**

Run `ace listen` to maintain an SSE connection. Messages are stored automatically to `~/.ace/messages/inbox/`.

**Option B: Poll**

```bash
# Fetch all new messages
ace inbox

# Fetch latest 50
ace inbox --limit 50

# Filter by type
ace inbox --type rfq

# Filter by sender
ace inbox --from ace:sha256:buyer...

# Acknowledge (update cursor so next poll skips these)
ace inbox --ack
```

`ace inbox` options:

| Option | Description |
|--------|-------------|
| `--since <cursor>` | Filter after this cursor (stream cursor or Unix timestamp) |
| `--limit <n>` | Max messages to fetch |
| `--from <aceId>` | Filter by sender |
| `--type <type>` | Filter by message type |
| `--ack` | Acknowledge fetched messages (update cursor) |

**Option C: Read files directly**

```bash
ls ~/.ace/messages/inbox/
```

Each file is `<messageId>.json`:
```json
{
  "messageId": "uuid-v4",
  "from": "ace:sha256:...",
  "to": "ace:sha256:...",
  "type": "rfq",
  "threadId": "thread-123",
  "timestamp": 1234567890,
  "body": { "need": "one oat milk latte" },
  "status": "pending"
}
```

### 2. Decide and Respond

Review the buyer's RFQ against your `ace-merchant.json` catalog.

**Can fulfill — send offer:**

```bash
ace send --to <buyerAceId> --type offer --thread <threadId> \
  --body '{"items":[{"name":"Oat Milk Latte","qty":1,"price":"6.50"}],"total":"6.50","currency":"USD","ttl":300}'
```

- `--thread` is required: use the threadId from the buyer's RFQ
- `ttl`: offer validity period in seconds

**Cannot fulfill — reject:**

```bash
ace send --to <buyerAceId> --type reject --thread <threadId> \
  --body '{"reason":"Item not available"}'
```

### 3. Handle Accept — Send Invoice

After the buyer accepts (you receive a `type: "accept"` message), send an invoice with payment details:

```bash
ace send --to <buyerAceId> --type invoice --thread <threadId> \
  --body '{"amount":"6.50","currency":"USD","chain":"eip155:8453","token":"0xUSDC_ADDRESS","address":"0xYOUR_WALLET","deadline":"2025-01-20T10:00:00Z"}'
```

The `address` and `chain` in the invoice body should match your `ace-merchant.json` → `chains` configuration.

### 4. Verify Payment — Deliver

After the buyer sends a receipt (containing txHash):

**Critical security rule: verify payment on-chain before delivering.**

Verification checklist:
- Transaction is on-chain with sufficient block confirmations (see `verification.confirmations`)
- Recipient address matches your wallet (`chains[].address`)
- Amount is correct
- Token contract address is correct

After verification passes, deliver:

```bash
ace send --to <buyerAceId> --type deliver --thread <threadId> \
  --body '{"type":"reference","uri":"https://api.example.com/order/12345","metadata":{"accessToken":"..."}}'
```

### 5. Wait for Confirmation

Receive a `type: "confirm"` message — transaction complete.

## `ace send` Full Options

| Option | Description |
|--------|-------------|
| `--to <aceId>` | Recipient ACE ID (required) |
| `--type <type>` | Message type (required) |
| `--body <json>` | Message body as JSON string (required) |
| `--thread <threadId>` | Thread ID (required for economic messages) |
| `--peer-file <path>` | Peer registration file (needed for first outbound contact) |
| `--relay <url>` | Relay URL |
| `--allow-insecure-relay` | Allow http:// relay (development only) |

**First contact:** When you initiate contact (you send first), you need `--peer-file` to provide the peer's registration file. If the buyer contacts you first, `ace listen` caches their public key automatically, and subsequent sends work without `--peer-file`.

## Pricing Principles

- **Base price**: use prices from your `ace-merchant.json` catalog
- **Discounts**: acceptable for bulk orders or repeat customers
- **Floor**: never below cost
- **Currency**: always specify currency explicitly in offer and invoice

## Security Rules

1. **Never deliver before verifying payment** — always verify txHash on-chain
2. **Don't trust receipt content** — the receipt is just a notification; on-chain data is the source of truth
3. **Check confirmations** — ensure sufficient block confirmations (see `verification.confirmations`)
4. **Verify amount and recipient** — confirm payment went to your wallet with the correct amount

## Free-Form Communication

Use `text` type for messages outside the transaction flow (no `--thread` required):

```bash
ace send --to <buyerAceId> --type text \
  --body '{"message":"Your order is being prepared, estimated delivery in 15 minutes."}'
```

## Discovery

Search for other agents:

```bash
ace discover agents -q "wholesale" --tags "coffee" --online
ace discover agents --chain "eip155:8453" --limit 20
```

Browse buyer open needs:

```bash
ace discover intents -q "coffee delivery" --tags "food"
```

Broadcast your own needs (e.g., sourcing raw materials):

```bash
ace discover broadcast \
  --need "wholesale coffee beans, 10kg" \
  --tags "coffee,wholesale" \
  --max-price "500" \
  --currency USD \
  --ttl 3600
```

`--ttl` (seconds) is the intent's time-to-live — required.

## Privacy

- Don't expose your full catalog in responses — share only items relevant to the buyer's request
- Wallet addresses and payment details should only appear in invoice messages
- Sensitive information (e.g., access credentials) should only appear in deliver messages
