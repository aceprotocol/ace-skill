# ACE Troubleshooting

Common issues and solutions.

## Quick Diagnostics

```bash
# Does the identity exist?
ls ~/.ace/identity.enc

# Is ace listen running?
cat ~/.ace/listen.pid

# Current relay connection?
cat ~/.ace/relay.url

# Recent messages?
ls -lt ~/.ace/messages/inbox/ | head -10

# Sync cursor position?
cat ~/.ace/sync-cursor.json

# Is merchant config valid?
cat ace-merchant.json | python3 -m json.tool
```

## Common Issues

### 1. `ace listen` Fails to Start

Check in order:

| Check | Fix |
|-------|-----|
| `~/.ace/identity.enc` missing | Run `ace init` |
| `OS keystore master key not found` | Set `ACE_IDENTITY_KEY` environment variable (see `key-backup.md`) |
| `identity.enc corrupted` or decryption failure | Restore from backup, or `ace init --force` to rebuild |
| `Another listen instance may be running` | Check if process exists; if it has exited, delete `~/.ace/listen.pid` |
| Relay connection failure | Check network connectivity and relay URL |
| `ace-merchant.json` validation error | Check required fields (see "Config Validation Errors" below) |
| `Insecure relay URL rejected` | Use an HTTPS relay, or add `--allow-insecure-relay` for development |

### 2. Message Send Failures

**"Unknown recipient"**

Cause: No cached encryption public key for the buyer, and relay lookup returned nothing.

Fix:
- Wait for the buyer to message you first (`ace listen` caches their public key automatically)
- Or use `--peer-file` to provide their registration file

**"Invalid transition" state machine error**

Cause: The economic message type doesn't match the current thread state. The transaction flow is fixed: `rfq → offer → accept → invoice → receipt → deliver → confirm`.

Fix:
- Check thread state in `~/.ace/threads/`
- Confirm you're using the correct `--thread` ID
- `text` type messages are not subject to state machine constraints — they can be sent anytime

**"Message body as JSON string" parse error**

Cause: The `--body` JSON is malformed.

Fix: Ensure the body is a valid JSON string and under 256KB.

### 3. Not Receiving Messages

Troubleshooting steps:

1. Confirm `ace listen` is running (check `~/.ace/listen.pid`)
2. Confirm the buyer is using your correct ACE ID
3. SSE auto-reconnects on network interruptions, up to a maximum of 50 attempts. After that it stops with `SSE exceeded 50 reconnect attempts, giving up`. Restart `ace listen` to resume.
4. Restarting `ace listen` automatically syncs offline messages from the relay
5. You can also pull manually with `ace inbox`

**Inbox quota:** Each identity stores up to 10,000 messages, with a per-sender cap of 250. Oldest messages are deleted automatically when the limit is exceeded.

### 4. Config Validation Errors

`ace listen` validates `ace-merchant.json` on startup. If the file exists but has invalid JSON or missing fields, it fails with an explicit error (it does not silently ignore malformed configs).

| Error | Fix |
|-------|-----|
| `merchant.name is required` | Add the `merchant.name` field |
| `catalog must have at least one item` | Add at least one item to the catalog array |
| `catalog item missing id/name/price/currency` | Fill in all required fields for each item |
| `settlement must have at least one method` | Add `"settlement": ["crypto/instant"]` |

### 5. Relay Connection Issues

**Registration failed (non-fatal)**

`[warn] Relay registration failed (non-fatal)` — listen continues running, but you may not appear in discovery results. Check the relay URL and network connectivity.

**No relay URL configured**

If you see `No relay URL configured`, configure one via any of these (in resolution order):
1. `--relay <url>` flag
2. `ACE_RELAY` environment variable
3. `~/.ace/config.json` → `relay` field
4. `./ace-merchant.json` → `relay` field

### 6. TOFU Key Change Warning

If you see `[security] TOFU violation: peer ... encryption key changed!`:

**What it means:** A previously cached peer encryption public key doesn't match the newly received one. This could indicate a relay man-in-the-middle attack.

**What happens:** The CLI automatically keeps the previously cached key and rejects the new one.

**What to do:**
- If you can confirm the peer genuinely re-initialized their identity (e.g., ran `ace init` again), delete the corresponding cache file in `~/.ace/peers/` and retry
- If you cannot confirm, **do not** delete the cache — contact the peer to verify

### 7. Direct Delivery Rate Limit (HTTP 429)

If the other party reports receiving HTTP 429 when delivering messages, they've hit the rate limit (60 requests/min per IP). Normal commerce transactions won't trigger this — it's typically caused by bulk testing or abnormal retry loops.

### 8. First Contact Issues

**You initiate contact (you send first):**
- You need `--peer-file` to provide the peer's registration file
- Or wait for their registration info to become available on the relay

**They contact you:**
- `ace listen` automatically queries and caches their public key from the relay
- Subsequent sends work directly without `--peer-file`

**Peer cache** is stored in `~/.ace/peers/` with a 24-hour TTL.

### 9. Unregistering from the Network

```bash
ace unregister
```

Local files are preserved. Run `ace listen` to go back online at any time.

## Environment Variable Reference

| Variable | Description |
|----------|-------------|
| `ACE_IDENTITY_KEY` | Master key (base64), bypasses OS Keystore |
| `ACE_RELAY` | Relay URL |
| `ACE_ALLOW_INSECURE_RELAY` | Set to `1`/`true`/`yes` to allow http:// relay |
| `ACE_ENV` | Set to `test` for test mode (e.g., localhost HTTPS→HTTP downgrade) |
