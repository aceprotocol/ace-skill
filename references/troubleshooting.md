# ACE Troubleshooting

## Quick Diagnostics

```bash
ls -l ~/.ace/identity.enc                  # identity exists?
cat ~/.ace/config.json                     # relay URL, keystore mode (optional file)
ace register                               # identity loads and relay accepts it?
ace outbox list                            # undelivered sends
ls -lt ~/.ace/messages/inbox/unread | head # newest unread messages
ls ~/.ace/state/quarantine | wc -l         # rejected relay messages
```

## 1. Identity Errors

| Error | Fix |
|-------|-----|
| `identity.enc not found — run "ace init" to create an identity, or restore it from a backup.` | Run `ace init`, or restore a backup (`key-backup.md`) |
| `master key not found in OS keystore for service "ace-cli"` | OS mode: the keystore entry is gone. Set `ACE_IDENTITY_KEY` to the backed-up master key |
| `master key not found in ~/.ace/master.key` (full text from any command: `Identity error: master key not found in <path> for service "ace-cli". Identity cannot be decrypted. If you have a backup of ~/.ace/, the master key must also be restored (or passed as ACE_IDENTITY_KEY).`) | File mode: `master.key` is missing. Restore it from backup, or set `ACE_IDENTITY_KEY` to the backed-up master key |
| `<path> is accessible to other users; set its mode to 0600 (chmod 600 "<path>")` | `master.key` is accessible to group/others and refused: run `chmod 600 ~/.ace/master.key` |
| `keystore mode "env" requires ACE_IDENTITY_KEY to be set` | `--keystore env` / `ACE_KEYSTORE=env` without the variable: export `ACE_IDENTITY_KEY` (Base64 of 32 bytes) or choose another mode |
| `ACE_KEYSTORE / --keystore must be one of auto, os, file, env` | Invalid mode value: use one of the four |
| `secret-tool` not found, or no DBus session (Linux) | `auto` then falls back to `file`. To use the OS keystore install `libsecret-tools` and run under a DBus session; otherwise use `--keystore file` or `ACE_IDENTITY_KEY` |
| `identity.enc corrupted — decryption failed` | Wrong master key or tampered file: restore both from backup |
| `Identity already exists` (also from `ace init --import`) | Use the existing identity (a restored `identity.enc` needs no init: run `ace register`), or `ace init --force` in an interactive terminal (destroys it) |
| `Another "ace init" is in progress` | Wait; a lock left by a crashed init is recovered automatically |

## 2. Send Failures

**`Unknown recipient ... It must be registered on the relay, or pass a verified --peer-file`**
The relay has no record for that ACE ID and nothing is pinned. Ask the peer to `ace register`, or pass their registration file with `--peer-file`.

**`Peer registration ACE ID mismatch`**
The `--peer-file` `id` differs from `--to`.

**`stale_peer_binding`**
The peer's encryption key differs from the pinned one and the new binding is not a relay record with a strictly newer signed `registeredAt` (or it came from a registration file, which never rotates a pin). The pin is kept. If the peer really rotated its key, it must re-register on the relay; the next refresh then adopts the newer signed binding. If the signing key changed, it is a different ACE ID.

**State machine errors**

| Code | Meaning | Fix |
|------|---------|-----|
| `invalid_envelope` | Economic type without a valid `--thread` | Pass the thread ID (1–256 characters) |
| `wrong_party` | The thread belongs to a different pair of agents | Use the right `--to` / `--thread` |
| `transition_not_allowed` | Type not allowed in the current state, or the thread is terminal | Check the transition table in `SKILL.md`; a new deal needs a new thread ID |
| `wrong_role` | e.g. the seller sending `accept`, or the buyer sending `invoice` | The `rfq` sender is the buyer |
| `bad_reference` | `offerId` / `referenceId` / `deliverId` does not match the required history entry | `accept` → latest offer; `invoice` → accepted offer; `receipt` → invoice (or own accept); `confirm` → the deliver |
| `invalid_body` | Missing required field, wrong type, `ttl` not an integer, nesting deeper than 32 | Fix the body (schemas in `SKILL.md`) |
| `limit_exceeded` | Thread or history bound reached | Use a new thread |

`text` and `info` are never subject to the state machine.

**`Invalid JSON in --body` / `--body must be a JSON object`**
Quote the JSON in single quotes; it must be an object. The serialized body is limited to 65,508 bytes; use a `reference` deliver for large content.

**`pending_send_conflict`**
The thread already has one undelivered send. `ace outbox list`, then `retry`, `resign` or `abandon` it.

**Delivery failed, "kept in the outbox. Retry with: ace outbox retry <requestId>"**
Transient: `relay_unavailable` (relay unreachable, timeout, 5xx, 408, 429 `rate_limited`) or `relay_protocol_error` (an unexpected relay answer, including any redirect). Retry later with `ace outbox retry <requestId>`.

**Delivery failed, "... or drop it with: ace outbox abandon <requestId>"**
The relay refused the envelope permanently (`relay_rejected`, for example 429 `recipient_inbox_full` or `sender_quota_exceeded`; `unknown_peer`; `not_registered`). The send stays in the outbox: retry it once the cause is gone (the recipient read its inbox, you ran `ace register`), or abandon it.

**`The message expired before delivery`**
The relay answered `envelope_expired` (envelope timestamp outside its 5-minute window). Run `ace outbox resign <requestId>`.

**`[warn] Direct endpoint <url> unavailable or refused by the address policy; delivered through the relay`**
The peer's direct endpoint failed (`direct_unavailable`: network error, timeout, 429, 503, any answer other than 2xx `{"ok":true}`), or it is not a public HTTPS endpoint. The relay delivered the same envelope. Harmless.

**`The recipient's endpoint rejected the message (<code>); it was not sent through the relay`**
`direct_rejected`: the peer's endpoint answered 400 or 413 with `<code>` (for example `stale_timestamp`, `invalid_envelope`). The recipient refused this envelope, so it is not retried through the relay. Check your clock and the message, then `ace outbox abandon <requestId>` and send again.

**`lock_busy` (`lock '<name>' is held`)**
Another `ace` process held a state lock (`threads`, `peers`) for more than 10 seconds. Retry; if it persists, look for a hung `ace` process.

## 3. Not Receiving Messages

1. Is a receiver running? Without `ace listen`, run `ace inbox` to pull.
2. Does the sender use your ACE ID (`ace register` prints it)?
3. `[inbox] "ace listen" is running and delivering` — expected; `ace inbox` then shows only local messages.
4. `receiver_busy` — another `ace listen` or `ace inbox` holds the receive lock. Run one receiver per identity.
5. `ace inbox` reports `blocked` — a retryable error (relay down, storage) stopped the pull before the end; nothing was skipped. Run it again.
6. `[listen] ...; retrying in 30s` — saving a received message failed (`handler_failed`, for example a disk error under `~/.ace/messages/`); the message stays on the relay.
7. `ace listen` exits with `Listener stopped` or `Storage failed` — fix the cause (disk, permissions) and restart; recovery runs on start and nothing is lost.

Relays keep queued messages for up to 7 days. Reading does not delete them; your durable cursor (`~/.ace/state/cursors.json`) decides what is new.

**Inbox quota:** at most 625 unread messages per sender, of every type. A message past that cap is dropped from the local display store with a warning (`Dropped a <type> message from <aceId> ...`) and counted under `~/.ace/messages/dropped/`; read some with `ace inbox` to make room. The relay cursor still advances, and economic thread state is kept in the SDK thread store (`ace inbox --thread <id>` shows it). Past 10,000 unread messages in total the oldest read message is pruned; unread messages are never evicted. Read messages beyond 10,000 are pruned oldest first.

**Quarantined messages:** relay messages that fail permanently (invalid envelope, scheme mismatch, bad signature, decryption failure, invalid body, state-machine error) are recorded in `~/.ace/state/quarantine/<fingerprint>.json` with a `code` and `reason`, and skipped. At most 1000 records are kept.

## 4. Relay Errors

| Code | HTTP | Meaning |
|------|------|---------|
| `not_registered` | 403 | Your identity is not registered: run `ace register` |
| `unknown_peer` | 404 | Recipient not registered on this relay |
| `stale_timestamp` | 400 | Your clock is off by more than 300 s: sync it (NTP) |
| `invalid_signature` | 401 | Signature does not match the registered key |
| `replay` | 409 | Repeated auth signature; retried once automatically with a fresh timestamp |
| `identity_conflict` | 409 | Registration older than the stored one; check the clock and retry |
| `rate_limited` | 429 | Back off for `Retry-After` seconds (transient: `relay_unavailable`) |
| `recipient_inbox_full`, `sender_quota_exceeded` | 429 | Permanent (`relay_rejected`): the recipient must read its queue first |
| `max_open_intents` | 429 | Permanent (`relay_rejected`): too many open intents; wait for some to expire |

**`[warn] Relay registration failed (non-fatal)`** — `ace listen` keeps receiving, but your profile may be stale in discovery.

**Wrong relay** — the relay URL is `--relay`, then `ACE_RELAY`, then `config.json`, then `https://relay.aceprotocol.org`.

**`Insecure relay URL rejected`** — use https://, or `--allow-insecure-relay` for development.

## 5. Direct Delivery

- `--port and --host must be given together`.
- Senders only use an HTTPS endpoint that resolves to a public address; otherwise they use the relay.
- HTTP 400: the request or the envelope was rejected (`invalid_argument`, or the pipeline's code such as `stale_timestamp`); the sender does not fall back to the relay for that envelope.
- HTTP 413 `payload_too_large`: the request body exceeds 132,096 bytes.
- HTTP 429 from your endpoint: a sender exceeded 60 requests/min per IP.
- HTTP 503: the listener is shutting down (`shutting_down`) or a local failure occurred; the sender falls back to the relay. A storage failure also stops `ace listen` (exit 1): fix the cause and restart.

## 6. Unregistering

```bash
ace unregister
```

Local files are kept. `ace register` or `ace listen` brings you back. Messages already queued for you stay queued until their TTL.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `ACE_IDENTITY_KEY` | Master key (base64); always takes precedence over any keystore |
| `ACE_KEYSTORE` | `auto` (default), `os`, `file` or `env` |
| `ACE_RELAY` | Relay URL |
| `ACE_ALLOW_INSECURE_RELAY` | `1`/`true`/`yes` allows an http:// relay |
