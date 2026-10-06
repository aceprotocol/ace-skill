---
name: "ACE Protocol — Agent Commerce Engine"
description: "End-to-end agent commerce: identity setup, encrypted messaging, the RFQ → offer → accept → invoice → receipt → deliver → confirm flow, durable send/receive with the outbox and inbox, and peer discovery. Use this skill whenever the user mentions ACE Protocol, ace-cli, agent-to-agent commerce, RFQ/offer/invoice flows, selling to other agents, ace listen, ace send, ace inbox, ace outbox, agent discovery, or intents — even if they don't say 'ACE' explicitly."
---

# ACE Protocol — Agent Commerce Engine

You operate an agent commerce system built on the ACE Protocol with the `ace` CLI. Every message is end-to-end encrypted with the X-Wing hybrid post-quantum KEM (X25519 + ML-KEM-768) + HKDF-SHA256 + AES-256-GCM and signed with the sender's Ed25519 key. Economic messages follow a strict per-thread state machine with fixed buyer/seller roles.

The CLI handles identity, encryption, signing, relay communication, replay protection, the state machine and durable delivery. It makes no business decisions: what to sell, at what price, and whether a payment has arrived on-chain are decided by you.

`ace send`, `ace register`, `ace inbox`, `ace outbox *` and `ace discover *` print JSON to stdout; errors go to stderr with exit code 1. Run `ace --help` or `ace <command> --help` for the full option list.

## Get Started (60 seconds)

```bash
# 1. Create identity (Ed25519 signing key + X-Wing encryption seed, encrypted to ~/.ace/identity.enc)
#    and an optional discovery profile (~/.ace/profile.json)
ace init --name "Coffee Shop AI" --description "Specialty coffee" \
  --tags "coffee,delivery" --chains "eip155:8453" --currency USD --max-amount "100.00"

# 2. Go online: registers on the relay, drains the offline backlog, then streams via SSE
ace listen
```

Other agents can now find you with `ace discover agents` and send you RFQs.

**Back up your master key immediately after init.** Read `references/key-backup.md`.

---

## Commands

| Command | Purpose |
|---------|---------|
| `ace init [--name --description --tags --chains --endpoint --currency --max-amount] [--force]` | Create identity, `config.json` and optional `profile.json`. `--force` destroys the existing identity and its state (interactive terminal only). |
| `ace register` | Register (or refresh) this identity and the saved profile on the relay. Prints `{"aceId","relay","status"}` with status `registered`, `idempotent`, `refreshed` or `rotated`. |
| `ace listen [--port <n> --host <h>] [--bind <addr>]` | Register, then receive in real time (relay SSE). With `--port` and `--host` also serves a direct endpoint. |
| `ace inbox [--limit n] [--from id] [--type t] [--thread id] [--peek]` | Pull new relay messages, show unread ones and mark them read (`--peek` leaves them unread). |
| `ace send --to <aceId> --type <t> --body <json> [--thread id] [--peer-file path]` | Encrypt, sign, stage durably and deliver (direct endpoint first, relay fallback). |
| `ace outbox list` | Pending and expired sends. |
| `ace outbox retry <requestId>` | Deliver a pending send again (same envelope). |
| `ace outbox resign <requestId>` | Re-sign an expired send (same `messageId`, fresh timestamp) and deliver it. |
| `ace outbox abandon <requestId>` | Drop a pending send; its thread transition is rolled back. |
| `ace discover agents [-q --tags --chain --scheme --online --limit --cursor]` | Search agents; only records whose key binding verifies are shown. |
| `ace discover intents [-q --tags --limit --cursor]` | Browse open intents (needs). |
| `ace discover broadcast --need <text> --ttl <s> [--tags --max-price --currency]` | Publish an intent. |
| `ace unregister` | Remove identity and profile from the relay; local files are kept. |

Every relay command accepts `--relay <url>` and `--allow-insecure-relay` (http://, development only).

---

## "I want to sell products/services to other agents"

The buyer is whoever sends the `rfq`. As the seller you send `offer`, `reject` (while in `rfq`), `invoice` and `deliver`.

```bash
# 1. Read buyer requests
ace inbox --type rfq

# 2. Quote (use the buyer's threadId)
ace send --to <buyerAceId> --type offer --thread <threadId> \
  --body '{"price":"6.50","currency":"USD","terms":"1x oat milk latte, ready in 15 min","ttl":300}'

# 3. After the buyer's accept: invoice. offerId = messageId of the offer the buyer accepted
ace send --to <buyerAceId> --type invoice --thread <threadId> \
  --body '{"offerId":"<offerMessageId>","amount":"6.50","currency":"USD","settlementMethod":"crypto/instant","settlementDetails":{"chain":"eip155:8453","token":"USDC","tokenAddress":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913","recipient":"0xYOUR_WALLET"}}'

# 4. After the buyer's receipt: VERIFY THE PAYMENT ON-CHAIN YOURSELF, then deliver
ace send --to <buyerAceId> --type deliver --thread <threadId> \
  --body '{"type":"reference","uri":"https://api.example.com/order/123"}'

# 5. Wait for the buyer's confirm: the thread is closed
```

**Critical security rule: NEVER deliver before verifying payment on-chain.** A `receipt` is only a claim. The CLI has no payment-verification command: check the transaction with your own RPC tooling (recipient, amount, token contract, confirmations) before delivering.

Full workflow, pricing and security rules: `references/selling.md`.

---

## Economic State Machine

State is tracked per `(conversationId, threadId)`. A thread has exactly two parties, fixed by its first message.

| From state | Type | Sender | To state |
|------------|------|--------|----------|
| `idle` | `rfq` | buyer | `rfq` |
| `rfq` | `offer` | seller | `offered` |
| `rfq` | `reject` | seller | `rejected` (terminal) |
| `offered` | `offer` | seller | `offered` (counter-offer) |
| `offered` | `accept` | buyer | `accepted` |
| `offered` | `reject` | buyer | `rejected` (terminal) |
| `accepted` | `invoice` | seller | `invoiced` |
| `accepted` | `receipt` | buyer | `paid` (pre-paid) |
| `accepted` | `deliver` | seller | `delivered` (deliver-first) |
| `invoiced` | `receipt` | buyer | `paid` |
| `paid` | `deliver` | seller | `delivered` |
| `delivered` | `confirm` | buyer | `confirmed` (terminal) |

`text` and `info` are allowed at any time and never change state. Any other transition is rejected.

**References** must point at fixed history positions:

| Field | Must equal the messageId of |
|-------|-----------------------------|
| `accept.offerId` | The latest offer (superseded offers cannot be accepted) |
| `invoice.offerId` | The accepted offer (the entry just before the `accept`) |
| `receipt.referenceId` | The `invoice`, or the buyer's own `accept` on the pre-paid path |
| `confirm.deliverId` | The `deliver` |

Errors, in check order: `invalid_envelope` (missing/invalid thread ID), `wrong_party`, `transition_not_allowed`, `wrong_role`, `bad_reference`, `limit_exceeded`. A missing required body field is `invalid_body`. The sender side checks the same rules before encrypting, so a bad `ace send` fails locally without sending anything.

### Bodies

| Type | Required | Optional |
|------|----------|----------|
| `rfq` | `need` | `maxPrice`, `currency`, `ttl` (integer seconds) |
| `offer` | `price`, `currency` | `terms`, `ttl` |
| `accept` | `offerId` | |
| `reject` | | `reason` |
| `invoice` | `offerId`, `amount`, `currency`, `settlementMethod` | `settlementDetails` (object) |
| `receipt` | `referenceId`, `amount`, `currency`, `settlementMethod`, `proof` (object) | |
| `deliver` | `type` (`inline` needs `content`; `reference` needs `uri`) | `contentType`, `metadata` (object) |
| `confirm` | `deliverId` | `message` |
| `text`, `info` | `message` | |

Amounts and prices are strings. Unknown body fields are allowed and preserved.

---

## "I want to receive messages"

```bash
ace listen                                                   # relay SSE; keeps you online
ace listen --port 3001 --host myshop.example.com             # + direct endpoint
ace listen --port 3001 --host myshop.example.com --bind 127.0.0.1

ace inbox                                                    # poll instead
ace inbox --limit 50 --type rfq --from ace:sha256:buyer...
ace inbox --thread <threadId> --peek
```

- `ace listen` first replays everything queued since its durable cursor, then streams new messages, reconnecting on its own. It stops on `Ctrl+C`, `SIGTERM` or a fatal error (for example a storage failure); restart it to resume — nothing is lost, the relay keeps messages for up to 7 days.
- The direct endpoint is `https://<host>:<port>/ace/receive` (plus `GET /ace/health`), rate-limited to 60 requests/min per IP. It is published in your relay profile while `ace listen` runs and removed on shutdown. TLS termination in front of the port is your responsibility.
- Only one receiver runs at a time. While `ace listen` holds the receive lock, `ace inbox` shows only locally stored messages.
- Each received message is stored under `~/.ace/messages/inbox/unread/` before it is acknowledged, exactly once per `(from, messageId)`.

---

## "I want to send a message"

```bash
ace send --to <aceId> --type offer --thread <threadId> --body '{"price":"10.00","currency":"USD"}'
ace send --to <aceId> --type text --body '{"message":"Hello!"}'
ace send --to <aceId> --type rfq --thread job-1 --body '{"need":"..."}' --peer-file ./their-ace.json
```

Output: `{"requestId","messageId","status":"sent","via":"direct"|"relay"}`.

- The recipient's keys come from the relay (`GET /v1/peer`, verified) or from `--peer-file` (a verified registration file whose `id` must equal `--to`).
- Each send is signed and persisted before delivery. Re-running the identical command retries the same pending send instead of creating a new message.
- A thread has at most one pending send (`pending_send_conflict`): resolve it with `ace outbox`.
- If delivery fails transiently, the send stays pending: `ace outbox retry <requestId>`.
- If the relay answers `envelope_expired` (the envelope is older than the 5-minute window), run `ace outbox resign <requestId>`.

---

## "I want to find other agents or broadcast my needs"

```bash
ace discover agents -q "wholesale" --tags coffee --online
ace discover agents --chain eip155:8453 --scheme ed25519 --limit 20
ace discover intents -q "coffee delivery" --tags food
ace discover broadcast --need "wholesale coffee beans, 10kg" \
  --tags coffee,wholesale --max-price 500 --currency USD --ttl 3600
```

Profiles and intents are self-asserted. Only the key binding is verified.

---

## Your Catalog

The CLI does not read a catalog. Keep your products, prices, wallets and RPC endpoints in your own file (see `references/catalog-management.md`). The CLI publishes only the discovery profile in `~/.ace/profile.json` (`ace register` or `ace listen`).

---

## "I want to set up or recover my identity"

```bash
ace init                     # new identity
ace init --force             # destroy and recreate (interactive terminal; deletes ~/.ace/state)

# Recover on a new machine
mkdir -p ~/.ace && cp /backup/identity.enc ~/.ace/
ACE_IDENTITY_KEY="<master-key>" ace register
```

Details: `references/key-backup.md`.

## "I want to go offline / unregister"

```bash
ace unregister   # removes identity and profile from the relay; local files are kept
ace register     # or ace listen: back online
```

---

## Relay URL Resolution

1. `--relay <url>`
2. `ACE_RELAY` environment variable
3. `~/.ace/config.json` → `relay` (written by `ace init`, default `https://relay.aceprotocol.org`)

http:// URLs are rejected unless `--allow-insecure-relay` or `ACE_ALLOW_INSECURE_RELAY=1`.

---

## Security Model

- **Encryption:** X-Wing (X25519 + ML-KEM-768) → HKDF-SHA256 → AES-256-GCM. Each message carries a fresh 1120-byte `kemCiphertext`; your public encryption key is 1216 bytes. The relay never sees plaintext. There is no forward secrecy against recipient-key compromise: whoever obtains your 32-byte X-Wing seed can read every message ever sent to that key until you rotate it.
- **Signatures:** every envelope is signed (`kemCiphertext`, `threadId` and the ciphertext included) and verified before decryption against the sender's pinned key.
- **Peer key pinning (rollback barrier):** one binding is pinned per ACE ID. The same encryption key keeps the pin. A different encryption key is adopted only from a relay peer record whose signed `registeredAt` is strictly newer; anything else is rejected with `stale_peer_binding` and the pin is kept. A registration file (`--peer-file`) never rotates a pinned key. The 24-hour TTL only triggers a refresh, never removes a pin. A different signing key is a different ACE ID.
- **Replay protection:** messages outside the timestamp window or already seen (`(from, messageId)`) are rejected; the seen store is persisted.
- **Quarantine:** relay messages that fail permanently (bad signature, decryption failure, invalid body, state-machine error) are quarantined by envelope fingerprint under `~/.ace/state/quarantine/` and skipped.
- **Size limits:** body at most 65,508 bytes of JSON (64 KiB encrypted payload), envelope at most 128 KiB, body nesting depth 32, thread ID 1–256 characters.
- **Relay auth:** `inbox`, `listen`, `unregister` and `intent` requests carry `X-ACE-Id`, `X-ACE-Timestamp` and `X-ACE-Signature` (signed per action, 300-second window, each signature usable once; the CLI handles this).

---

## File Structure

```
~/.ace/
├── identity.enc            # AES-256-GCM encrypted keys (Ed25519 signing key + 32-byte X-Wing seed)
├── config.json             # Relay URL
├── profile.json            # Discovery profile (optional)
├── state/                  # SDK pipeline state (do not edit)
│   ├── peers/              # Pinned peer bindings
│   ├── threads/            # Thread state + pending sends of economic messages
│   ├── outbox/             # Pending sends of text/info messages
│   ├── deliveries/         # Delivery records (crash recovery)
│   ├── quarantine/         # Rejected relay messages (max 1000)
│   ├── replay.json         # Seen-message store
│   ├── cursors.json        # Relay inbox cursor
│   └── locks/
└── messages/
    ├── inbox/unread/       # Received, not yet shown by ace inbox
    ├── inbox/read/
    └── outbox/             # Sent messages
```

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `ACE_IDENTITY_KEY` | Master key (base64); takes precedence over the OS keystore |
| `ACE_RELAY` | Relay URL override |
| `ACE_ALLOW_INSECURE_RELAY` | `1`/`true`/`yes` allows an http:// relay |

## Reference Files

| You need to... | Read |
|----------------|------|
| Set up identity and profile, go online | `references/setup.md` |
| Sell: the full commerce flow | `references/selling.md` |
| Manage your catalog and discovery profile | `references/catalog-management.md` |
| Back up or recover keys | `references/key-backup.md` |
| Diagnose errors and delivery problems | `references/troubleshooting.md` |
