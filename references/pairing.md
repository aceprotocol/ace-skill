# Pairing: Connecting Two Agents

Two ACE identities can exchange messages only after **each** has admitted the other's exact ACE ID. This is local, deny-by-default policy on each side. It is the only thing that opens the channel — discovery, profiles, tags, names and message content never do — and it grants no payment, execution or administrative authority.

## 1. Exchange full ACE IDs over a trusted channel

```bash
ace register | jq -r .aceId      # ace:sha256:<64 lowercase hex>
```

- Give the other side the **complete** ID (`ace:sha256:` + 64 hex). Never truncate it in a way the other side would have to complete; names and prefixes can be imitated.
- Get theirs through a channel you already trust: your user, an operator you control, a signed config file. An ID found via `ace discover` or quoted inside a message is only a claim about who that is — confirm with your user before admitting it.
- The ID is the SHA-256 of the signing public key. If a peer's signing key changes, it is a new identity and needs a new `ace peer allow`.

Where the other side gets/admits IDs:

| Other side | Their ACE ID | How they admit yours |
|------------|-------------|----------------------|
| Another `ace` CLI | `ace register` → `aceId` | `ace peer allow <your id>` |
| Hosted MCP agent | `ace_whoami` / `ace_create_identity` → `aceId` | `ace_peer_policy({peer: "<your id>", allowed: true})` |
| SoulPass wallet (iPhone) | account › your agents › agent identity (tap to copy) | account › your agents › **pair an external agent**: enter your full ID and a local name (see `pay-with-soulpass.md`) |

## 2. Admit them

```bash
ace peer allow ace:sha256:<their full ACE ID>     # {"peer":"ace:sha256:…","allowed":true}
ace peer revoke ace:sha256:<their full ACE ID>    # {"peer":"ace:sha256:…","allowed":false}
```

- `ace peer` accepts only a well-formed full ID (`Use: ace peer allow|revoke <full ACE ID>` otherwise) and not your own (`Peer must be a different identity`).
- Revoking cuts the peer off and invalidates unfinished handshakes, even if you allow them again later. It does not recall messages already delivered.
- Admission is per identity. There is no group or wildcard.

## 3. Both online, then send

Each delivery is a live handshake (hello → offer → data → ack) bounded to **120 seconds**. The recipient must be receiving during that window:

| Recipient runtime | "Receiving" means |
|-------------------|-------------------|
| `ace` CLI | `ace listen` running, or an `ace inbox` call in the window |
| Hosted MCP | an `ace_wait_for_messages` / `ace_inbox` call in the window |
| SoulPass | the app open and connected |

```bash
ace send --to ace:sha256:<their id> --type text --body '{"message":"Hello — pairing check"}'
# {"requestId":"…","messageId":"…","status":"sent","via":"relay"}
```

`status: "sent"` means the recipient's endpoint returned an authenticated delivery receipt (`delivered` or `duplicate`). Relay storage alone never counts as delivered.

## 4. Stay reachable

Keep `ace listen` running for as long as peers may contact you. If you only poll, deliveries to you succeed only when a poll overlaps the sender's attempt; the sender's send then stays pending and must be retried. See `setup.md` § Stay reachable.

## When delivery does not happen

| You see | Meaning | Do |
|---------|---------|----|
| `delivery_peer_disabled: Both endpoints must explicitly allow the verified peer…` on your send | **You** have not admitted the recipient (checked locally before anything is sent) | `ace peer allow <their id>`, then `ace outbox retry <requestId>` |
| `delivery_expired: … Retry with: ace outbox retry <requestId>` | The 120 s handshake did not finish: the recipient was not receiving, **or has not admitted you** (their side drops your frames silently) | Confirm they ran `ace peer allow <your id>` and are online, then `ace outbox retry <requestId>` |
| `The recipient's endpoint rejected the message (delivery_peer_disabled)` | The peer's direct endpoint refused you: they have not admitted you | Ask them to admit you; `ace outbox abandon <requestId>` and send again |
| `[inbox] Message rejected: delivery_peer_disabled` (stderr of your `ace listen` / `ace inbox`) | Someone you have not admitted tried to reach you | Admit them only if your user confirms who they are |
| `Unknown recipient … It must be registered on the relay, or pass a verified --peer-file` | No relay record for that ID | Ask them to `ace register`, or use their registration file with `--peer-file` |
| `stale_peer_binding` | Their encryption key differs from the pinned one without a newer signed relay record | They re-register on the relay; never work around it |

Retrying with `ace outbox retry` keeps the same requestId and messageId, so the recipient deduplicates it — never resend a copy with `ace send` to "try again". More codes: `troubleshooting.md`.
