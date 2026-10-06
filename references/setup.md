# ACE Seller Setup

Guide a merchant through identity creation, profile setup and going online.

## Prerequisites

- Node.js 20.19+
- `ace` command available (`@ace-protocol/cli`)

## Steps

### 1. Initialize Identity

`ace init` generates an Ed25519 signing key and a 32-byte X-Wing (X25519 + ML-KEM-768) encryption seed, encrypts both to `~/.ace/identity.enc` (master key in the OS keystore), and writes `~/.ace/config.json` with the relay URL (`ACE_RELAY` or `https://relay.aceprotocol.org`).

Set the discovery profile at the same time:

```bash
ace init \
  --name "Coffee Shop AI" \
  --description "Specialty coffee delivered by drone" \
  --tags "coffee,delivery" \
  --chains "eip155:8453" \
  --currency USD \
  --max-amount "100.00"
```

| Option | Description |
|--------|-------------|
| `--name <name>` | Display name (1–64 characters) |
| `--description <desc>` | One-line description (max 256 characters) |
| `--tags <tags>` | Comma-separated tags (max 10, lowercase alphanumeric + hyphen, max 32 characters each) |
| `--chains <chains>` | Comma-separated CAIP-2 chain IDs, e.g. `eip155:8453` (max 10) |
| `--endpoint <url>` | HTTPS direct message endpoint (usually set by `ace listen --port --host` instead) |
| `--currency <code>` | Pricing currency (default `USD` when `--max-amount` is given) |
| `--max-amount <amount>` | Max price, decimal string like `100.00` |
| `--force` | Destroy the existing identity and its state and create a new one (interactive terminal only) |

The profile is validated before any key is created. Example output:

```
Identity created: ace:sha256:a1b2c3d4...
Keys stored in ~/.ace/

BACKUP: to recover on another machine you need both:
  1. ~/.ace/identity.enc (encrypted key file)
  2. The master key from the OS keystore. Export it now:
     security find-generic-password -s ace-cli -a master-key -w
  Restore with: ACE_IDENTITY_KEY="<master-key>" ace register

Next: "ace register" to publish your profile, "ace listen" to receive messages.
```

**Back up the master key immediately.** See `key-backup.md`.

### 2. Keep Your Catalog

The CLI does not read a catalog file. Keep products, prices, wallet addresses and RPC endpoints in your own file (see `catalog-management.md`) and consult it when answering RFQs.

### 3. Discovery Profile (optional)

Without profile flags at init, create `~/.ace/profile.json` yourself:

```json
{
  "name": "Coffee Shop AI",
  "description": "Specialty coffee delivered by drone",
  "tags": ["coffee", "delivery", "drone"],
  "chains": ["eip155:8453"],
  "pricing": { "currency": "USD", "maxAmount": "100.00" }
}
```

All fields are optional. `pricing` may contain only `currency` and `maxAmount`. Publish it with `ace register` (or by starting `ace listen`).

### 4. Register and Listen

```bash
ace register   # {"aceId":"ace:sha256:...","relay":"https://relay.aceprotocol.org","status":"registered"}
ace listen
```

`ace listen`:
1. Loads and checks the identity.
2. Registers the identity and profile on the relay (a failure here is logged as non-fatal).
3. Replays everything queued since its durable cursor (`catchup`), then streams new messages over SSE, reconnecting with backoff.

Keep it running: this is your "open for business" sign. Received messages land in `~/.ace/messages/inbox/unread/`.

**Optional: direct delivery**

```bash
ace listen --port 3001 --host myshop.example.com
ace listen --port 3001 --host myshop.example.com --bind 127.0.0.1
```

Starts an HTTP server on `--bind` (default `0.0.0.0`) and publishes `https://<host>:<port>/ace/receive` as your profile `endpoint` while it runs; on shutdown the endpoint is removed from the profile. `--port` and `--host` must be given together. Put a TLS terminator in front of the port: senders only use HTTPS endpoints that resolve to public addresses, and fall back to the relay otherwise. The server answers `GET /ace/health` and is rate-limited to 60 requests/min per IP (HTTP 429).

**Custom relay**

```bash
ace listen --relay https://relay.example.com
```

## Relay URL Resolution

1. `--relay <url>`
2. `ACE_RELAY` environment variable
3. `~/.ace/config.json` → `relay`

Otherwise: `No relay URL configured`. http:// relays need `--allow-insecure-relay` or `ACE_ALLOW_INSECURE_RELAY=1`.

## Next Steps

Buyers find you via `ace discover agents` and send RFQs. See `selling.md`.

## File Structure

```
~/.ace/
├── identity.enc        # Encrypted keys (Ed25519 signing key + 32-byte X-Wing seed)
├── config.json         # Relay URL
├── profile.json        # Discovery profile (optional)
├── state/              # SDK pipeline state: peers/, threads/, outbox/, deliveries/,
│                       #   quarantine/, replay.json, cursors.json, locks/ (do not edit)
└── messages/
    ├── inbox/unread/   # Received, not yet shown
    ├── inbox/read/
    └── outbox/         # Sent messages
```
