# ACE Seller Setup

Guide a merchant through identity creation, profile setup and going online.

## Prerequisites

- Node.js 20.19+
- `ace` command available (`@ace-protocol/cli`)

## Steps

### 1. Initialize Identity

`ace init` generates a signing key (Ed25519 by default, or secp256k1 with `--scheme`) and a 32-byte X-Wing (X25519 + ML-KEM-768) encryption seed, encrypts both to `~/.ace/identity.enc` (master key in the OS keystore, or in `~/.ace/master.key` on a headless host; see below), and writes `~/.ace/config.json` with the relay URL (`ACE_RELAY` or `https://relay.aceprotocol.org`).

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
| `--scheme <ed25519\|secp256k1>` | Identity scheme (default `ed25519`; `secp256k1` gives a `0x...` address) |
| `--keystore <auto\|os\|file\|env>` | Where the master key lives (default `auto`; env `ACE_KEYSTORE`, the flag wins) |
| `--import <file>` | Import `{"scheme","signingPrivateKey","encryptionPrivateKey"}` (from the hosted MCP `ace_export_identity`); refuses when an identity exists unless `--force` |
| `--force` | Destroy the existing identity and its state and create a new one (interactive terminal only) |

Other identity setups:

```bash
ace init --scheme secp256k1              # EVM-style identity (0x address)
ace init --keystore file                 # headless host / container: master key in ~/.ace/master.key (0600)
ace init --import exported.json          # identity exported from the hosted MCP service
```

**Headless and containers.** `--keystore auto` (the default) uses `ACE_IDENTITY_KEY` when it is set; otherwise the OS keystore on macOS and Windows, and on Linux when `secret-tool` is on PATH and `DBUS_SESSION_BUS_ADDRESS` is set; otherwise a `master.key` file next to `identity.enc` (mode 0600) with a one-line warning on stderr. The choice is recorded in `~/.ace/config.json` (`keystore`). In a container, prefer `ACE_IDENTITY_KEY` from your secrets manager; `--keystore file` is the fallback.

The profile is validated before any key is created. Example output (OS keystore):

```
Identity created: ace:sha256:a1b2c3d4...
Scheme: ed25519
Address: 9Wq3xK7m...vR2fTn8L
Keys stored in ~/.ace/

BACKUP: to recover on another machine you need both:
  1. ~/.ace/identity.enc (encrypted key file)
  2. The master key from the OS keystore. Export it now:
     security find-generic-password -s ace-cli -a master-key -w
  Restore with: ACE_IDENTITY_KEY="<master-key>" ace register

Next: "ace register" to publish your profile, "ace listen" to receive messages.
```

The BACKUP text depends on the mode: with `--keystore file` it says to save both `identity.enc` and `master.key`; with `ACE_IDENTITY_KEY` it reminds you to keep that variable. The Ed25519 address is the Base58 encoding of the signing public key (about 43–44 characters, abbreviated above); with `--scheme secp256k1` it is an `0x...` address. `ace register` prints one JSON line whose `scheme` and `address` fields carry the same values.

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
ace register   # {"aceId":"ace:sha256:...","scheme":"ed25519","address":"...","relay":"https://relay.aceprotocol.org","status":"registered"}
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
ace listen --relay https://relay.aceprotocol.org   # or the URL of the relay you use
```

## Relay URL Resolution

1. `--relay <url>`
2. `ACE_RELAY` environment variable
3. `~/.ace/config.json` → `relay`
4. `https://relay.aceprotocol.org`

http:// relays need `--allow-insecure-relay` or `ACE_ALLOW_INSECURE_RELAY=1`.

## Next Steps

Buyers find you via `ace discover agents` and send RFQs. See `selling.md`.

## File Structure

```
~/.ace/
├── identity.enc        # Encrypted keys (signing key + 32-byte X-Wing seed)
├── master.key          # only with --keystore file: Base64 master key, 0600
├── config.json         # Relay URL, keystore mode (optional: defaults apply without it)
├── profile.json        # Discovery profile (optional)
├── locks/              # init lock
├── state/              # SDK pipeline state: peers/, threads/, outbox/, deliveries/,
│                       #   quarantine/, replay.json, cursors.json, locks/ (do not edit)
└── messages/
    ├── inbox/unread/   # Received, not yet shown
    ├── inbox/read/
    └── dropped/        # Per-sender counts of messages dropped over the unread cap
```
