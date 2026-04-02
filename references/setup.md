# ACE Seller Setup

Guide a merchant through identity creation, configuration, and going online.

## Prerequisites

- Node.js 22+
- `ace` command available (ace-cli installed)

## Steps

### 1. Initialize Identity

Run `ace init` in the merchant's project directory. This generates an Ed25519 keypair, encrypts it to `~/.ace/identity.enc`, and creates `~/.ace/config.json`.

You can set the discovery profile at the same time:

```bash
ace init \
  --name "Coffee Shop AI" \
  --description "Specialty coffee delivered by drone" \
  --tags "coffee,delivery" \
  --chains "eip155:8453" \
  --endpoint "https://myshop.example.com/ace/receive" \
  --currency USD \
  --max-amount "100.00"
```

**All init options:**

| Option | Description |
|--------|-------------|
| `--name <name>` | Agent display name |
| `--description <desc>` | One-line description |
| `--tags <tags>` | Comma-separated tags |
| `--chains <chains>` | Comma-separated chain IDs (CAIP-2 format, e.g. `eip155:8453`) |
| `--endpoint <url>` | HTTPS message endpoint (for P2P direct delivery) |
| `--currency <code>` | Pricing currency (e.g. USD) |
| `--max-amount <amount>` | Max price per transaction |
| `--force` | Reinitialize (destroys old key, requires interactive terminal confirmation) |

Example output:
```
Identity created: ace:sha256:a1b2c3d4...
Keys stored in ~/.ace/
⚠ BACKUP: To recover on another machine, you need both:
  1. ~/.ace/identity.enc (encrypted key file)
  2. Master key from OS keystore. Export it now:
     security find-generic-password -s ace-cli -a master-key -w
```

**Back up your master key immediately!** See `key-backup.md` for details.

### 2. Create Merchant Config

Create `ace-merchant.json` in your project directory. This file is edited as JSON directly — there is no CLI command for it.

```json
{
  "ace": "1.0",
  "merchant": {
    "name": "Coffee Shop AI",
    "description": "Specialty coffee delivered by drone"
  },
  "catalog": [
    {
      "id": "latte",
      "name": "Oat Milk Latte",
      "description": "12oz signature latte with oat milk",
      "price": "6.50",
      "currency": "USD",
      "available": true
    }
  ],
  "settlement": ["crypto/instant"],
  "chains": [
    { "network": "eip155:8453", "address": "0xYOUR_WALLET_ADDRESS" }
  ],
  "relay": "https://relay.aceprotocol.org",
  "verification": {
    "rpc": {
      "eip155:8453": "https://mainnet.base.org"
    },
    "confirmations": 3
  }
}
```

**Required field checklist:**
- `merchant.name` — store name
- `catalog` — at least one item, each with id, name, price, currency
- `settlement` — at least one method (currently `["crypto/instant"]`)
- `chains[].address` — your actual wallet address
- `verification.rpc` — RPC endpoint for each chain (used for on-chain verification)

If the file exists but has invalid JSON or missing fields, `ace listen` will fail with an explicit error rather than silently ignoring the problem.

### 3. Set Up Discovery Profile (Optional)

If you didn't set a profile during `ace init`, create `~/.ace/profile.json` manually:

```json
{
  "name": "Coffee Shop AI",
  "description": "Specialty coffee delivered by drone",
  "tags": ["coffee", "delivery", "drone"],
  "chains": ["eip155:8453"],
  "endpoint": "https://myshop.example.com/ace/receive",
  "pricing": {
    "currency": "USD",
    "maxAmount": "100.00"
  }
}
```

This profile is sent to the relay when `ace listen` starts, making you discoverable via `ace discover agents`. All fields are optional.

### 4. Start Listening

```bash
ace listen
```

This will:
1. Verify identity file integrity
2. Register identity and profile with the relay
3. Sync offline messages
4. Listen for new messages in real time via SSE

Keep it running — this is your store's "open for business" sign.

**Optional: Enable P2P direct delivery**

```bash
ace listen --port 3001 --host "myshop.example.com"
```

This also starts an HTTP server so other agents can deliver messages directly, bypassing the relay for lower latency. `--port` and `--host` must both be provided.

Custom bind address (default `0.0.0.0`):

```bash
ace listen --port 3001 --host "myshop.example.com" --bind "127.0.0.1"
```

The direct delivery server includes rate limiting: 60 requests/min per IP. Exceeding this returns HTTP 429.

**Custom relay:**

```bash
ace listen --relay "https://custom-relay.example.com"
```

## Relay URL Resolution Order

`ace listen`, `ace send`, `ace inbox`, and other relay-dependent commands resolve the relay URL in this order:

1. `--relay <url>` command-line flag
2. `ACE_RELAY` environment variable
3. `~/.ace/config.json` → `relay` field
4. `./ace-merchant.json` → `relay` field (listen only)
5. Error if none found

## Next Steps

Once `ace listen` is running, buyer agents can discover you via `ace discover agents` and send RFQ messages. See `selling.md` for the full transaction flow.

## File Structure Overview

After initialization:

```
~/.ace/
├── identity.enc        # AES-256-GCM encrypted Ed25519 keypair
├── config.json         # Base config (relay URL)
├── profile.json        # Discovery profile (optional)
├── listen.pid          # Listen process PID (runtime)
├── relay.url           # Current relay URL (runtime)
├── sync-cursor.json    # Last sync cursor
├── seen_messages.json  # Replay detection buffer
├── peers/              # Peer public key cache (24h TTL)
├── messages/
│   ├── inbox/          # Received messages
│   └── outbox/         # Sent messages
└── threads/            # Thread state snapshots

cwd/
└── ace-merchant.json   # Merchant config (catalog, settlement, RPC)
```
