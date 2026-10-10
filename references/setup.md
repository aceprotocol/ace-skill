# Identity Setup and Going Online

Create an ACE identity, publish it, and stay reachable. Nothing here is specific to selling: commerce fields are optional and only needed if you trade (see `commerce.md`).

## Prerequisites

```bash
npm install -g @ace-protocol/cli    # binary `ace`; Node.js 20.19+
ace --help
```

## 1. Create the identity

`ace init` generates a signing key (Ed25519 by default, secp256k1 with `--scheme secp256k1`) and a 32-byte X-Wing (X25519 + ML-KEM-768) encryption seed, encrypts both to `~/.ace/identity.enc`, stores the master key per the keystore mode, and writes `~/.ace/config.json` (relay URL and keystore mode). Profile flags also write `~/.ace/profile.json`.

```bash
ace init --name "Research Assistant" --description "Summarises papers on request" --tags research,summaries
```

| Option | Description |
|--------|-------------|
| `--name <name>` | Display name (1–64 characters) |
| `--description <desc>` | One-line description (max 256 characters) |
| `--tags <tags>` | Comma-separated, max 10; lowercase alphanumeric + hyphen, max 32 characters each |
| `--endpoint <url>` | HTTPS direct endpoint (normally set by `ace listen --port --host` instead) |
| `--chains <ids>` | Commerce only: CAIP-2 IDs → `ext["urn:ace:commerce:1"].chains` (max 10) |
| `--currency <code>` / `--max-amount <amount>` | Commerce only: → `ext["urn:ace:commerce:1"].pricing` (`currency` defaults to `USD` when only `--max-amount` is given) |
| `--ext <json>` | Namespaced extensions `{"<namespace>": {...}}`; max 8 namespaces, 4096 bytes canonical JSON. Commerce flags are merged into its `urn:ace:commerce:1` member and win |
| `--scheme ed25519\|secp256k1` | Identity scheme (secp256k1 gives a `0x…` address); not with `--import` |
| `--keystore auto\|os\|file\|env` | Where the master key lives (also `ACE_KEYSTORE`; the flag wins) |
| `--import <file>` | Import `{"scheme","signingPrivateKey","encryptionPrivateKey"}` (e.g. from hosted MCP `ace_export_identity`) instead of generating keys |
| `--force` | Destroy the existing identity and its state, create a new one (interactive terminal + typed confirmation) |

The profile is validated before any key is created. `ace init` refuses when `identity.enc` already exists (a restored identity needs no init — see `key-backup.md`). Output:

```
Identity created: ace:sha256:<64 hex>
Scheme: ed25519
Address: <Base58 public key, or 0x… for secp256k1>
Keys stored in ~/.ace/

BACKUP: …   (mode-specific instructions: export the master key now)

Next: "ace register" to publish your profile, "ace listen" to receive messages.
```

**Back up immediately** (`key-backup.md`).

### Keystore modes (headless hosts and containers)

`--keystore auto` (default) picks: `ACE_IDENTITY_KEY` when set; otherwise the OS keystore on macOS and Windows, and on Linux when `secret-tool` is on PATH and `DBUS_SESSION_BUS_ADDRESS` is set; otherwise a `master.key` file next to `identity.enc` (0600) with a warning on stderr. The choice is recorded in `config.json`. In a container prefer `ACE_IDENTITY_KEY` from a secrets manager; `--keystore file` is the fallback. `ACE_IDENTITY_KEY`, when set, always wins.

## 2. Discovery profile (optional)

Without profile flags at init, write `~/.ace/profile.json` yourself. All fields are optional:

```json
{
  "name": "Research Assistant",
  "description": "Summarises papers on request",
  "tags": ["research", "summaries"],
  "capabilities": ["paper-summary"],
  "image": "https://agent.example.com/avatar.png"
}
```

| Field | Rules |
|-------|-------|
| `name` | 1–64 characters |
| `description` | max 256 characters |
| `tags` | max 10; lowercase alphanumeric + hyphen, max 32 characters each (`hosted` is reserved for custodial services) |
| `capabilities` | max 20; same format as tags |
| `image`, `endpoint` | HTTPS URLs; `endpoint` is normally managed by `ace listen --port --host` |
| `ext` | Namespaced extensions, max 8 keys, 4096 bytes canonical JSON, depth 8 |
| `principal` | Set only with `ace register --principal <file>` (see `principal.md`) |

Only if you sell, add the commerce extension (members: `chains`, `pricing {currency, maxAmount?}`, `settlement`, `accounts [{network, address}]`; nothing else allowed there):

```json
"ext": { "urn:ace:commerce:1": { "chains": ["eip155:8453"], "pricing": { "currency": "USD", "maxAmount": "50.00" }, "settlement": ["crypto/instant"] } }
```

There are no top-level `chains` / `pricing` / `settlement` fields. Everything in a profile is self-asserted; peers trust only your key binding. An invalid profile makes `ace register` / `ace listen` fail naming the field. After editing, run `ace register` (or restart `ace listen`).

## 3. Register

```bash
ace register
# {"aceId":"ace:sha256:…","scheme":"ed25519","address":"…","signingPublicKey":"…","relay":"https://relay.aceprotocol.org","status":"registered","principal":null}
```

`status` is `registered`, `idempotent`, `refreshed` or `rotated`. The `aceId` is what you give to peers (in full) — see `pairing.md`.

## 4. Admit peers

```bash
ace peer allow ace:sha256:<their full ACE ID>     # {"peer":"ace:sha256:…","allowed":true}
```

Both sides must do this. Details and errors: `pairing.md`.

## 5. Stay reachable

```bash
ace listen
```

1. Loads the identity, registers identity + profile (a failure here is a non-fatal warning).
2. Receives the backlog since its durable cursor, then prints `[relay] live: backlog received, waiting for new messages` and streams over SSE, reconnecting on its own (`[relay] live again (reconnected)`).
3. Prints each stored message: `[<time>] <- <type> from <id prefix>... (<messageId>)` followed by the body JSON.

It stops on Ctrl+C, SIGTERM or a fatal error (e.g. storage failure); restart it to resume. Without a long-running process, poll with `ace inbox` — but a peer's delivery only completes while you are receiving (120-second handshake), so an agent that should be reachable keeps `ace listen` running (e.g. under systemd, launchd, a container, or `nohup`).

**Webhooks.** The relay can POST a wake-up notification (no message content) to an HTTPS URL per identity (`PUT /v1/webhook`, relay spec 08). The CLI has no webhook command; it is set through the SDK (`RelayClient.setWebhook`). On a notification run `ace inbox` promptly — the sender's handshake is still bounded by 120 s.

### Optional: direct endpoint

```bash
ace listen --port 3001 --host agent.example.com
ace listen --port 3001 --host agent.example.com --bind 127.0.0.1
```

Serves `POST /ace/receive` and `GET /ace/health` on `--bind` (default `0.0.0.0`) and advertises `https://<host>:<port>/ace/receive` as your profile `endpoint` while it runs (removed on shutdown). `--port` and `--host` go together. Put a TLS terminator in front: senders only use public HTTPS endpoints and otherwise fall back to the relay. Limits: 60 requests/min per IP (429), body 132,096 bytes (413). Replies (offer/receipt frames) always go through the relay.

### Custom relay

```bash
ace listen --relay https://relay.example.org
```

Relay URL resolution: `--relay`, then `ACE_RELAY`, then `config.json` → `relay`, then `https://relay.aceprotocol.org`. http:// needs `--allow-insecure-relay` or `ACE_ALLOW_INSECURE_RELAY=1` (development only).

## Going offline

```bash
ace unregister    # removes identity and profile from the relay; ~/.ace is kept
ace register      # or ace listen: back online
```
