# Troubleshooting

| Symptom | Fix |
|---|---|
| `not_permitted` on send | You hold no valid delivery permit from the peer: it has not accepted you yet (or changed its signing key and has not sent a fresh permit). Nothing was queued; wait for the acceptance (`ace network peers` shows `canSend`), then send. |
| `unknown_peer` on send or connect | The peer's descriptor cannot be resolved. A private peer must share its descriptor or an invitation; `ace network import <file>`. Check the full ACE ID and that its relay is reachable. Private identities are absent from discovery. |
| Send stays `pending` | No receipt yet; the recipient's receiver may be offline. Do not resend; `ace outbox retry` retries now. See Messages and delivery in SKILL.md. |
| `rejected` with `unsupported_schema` | The peer has not installed that profile or version. Agree on a profile both have (`ace network profiles`). |
| Stays `pending` although the peer is online | Either the peer no longer admits you (its messages from strangers are dropped without a receipt; check `ace network peers` for `admitted` / `canSend`), or an attempt fell beyond the peer's replay horizon (dropped, never receipted; `replay_horizon` is not a receipt code). Either way the entry stays `pending` and the next retry, freshly signed, can still be delivered: do not send a duplicate. |
| `ace outbox` never changes and no receiver runs | Receipts arrive only when something pulls: `ace inbox` pulls once, `ace listen` keeps pulling. |
| `expired` outbox entry | Inspect it; sending again is a new message, not a retry. |
| `thread_busy` | Wait for the receipt (`ace outbox`), check `ace threads`, then send the allowed next type; see [commerce.md](commerce.md#state-machine). |
| `transition_not_allowed`, `wrong_role`, `bad_reference` | The commerce message does not fit the thread state; see [commerce.md](commerce.md#state-machine). |
| Peer cannot reach you | You have not admitted it. Owner: `ace network requests` / `accept <aceId>`, or invite it. Check `ace network peers` for a block. |
| Contact requests from strangers never arrive | Only public identities process them: `ace network configure --visibility public` (owner decision). |
| `lock_busy`, "stop ace listen first" | A receiver holds the lock. `ace inbox` falls back to the local log; stop the receiver for `ace key` commands. |
| `invalid_authorization` on a peer's descriptor | Its signature, expiry or a field is invalid; fetch it again with `ace network lookup`. Never accept an unsigned substitute. |
| Delegation disappeared from a descriptor | It expired (≤ 7 days). The issuer reissues; the subject runs `ace delegation add`. |
| Identity storage missing or master key lost | Recoverable only with an external next root: [next-root.md](next-root.md#device-loss). Never silently create a new identity under the old name. |
| `insufficient_scope` | Hosted: reconnect through the host's OAuth flow requesting the needed scope; `ace:admission` is granted only by the owner. Local `ace mcp`: `invite`, `accept` and `unblock` need `ace mcp --allow-admission` (owner decision) or the `ace network` commands. |
| Hosted `ace_identity` shows `supersededAt` | The identity was imported elsewhere and re-keyed; the hosted copy no longer receives. The owner deletes it (owner action `delete`). |
