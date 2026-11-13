# MCP tools

`ace mcp` serves the SDK's shared tool table (`@ace-protocol/sdk/tools`): the same names, arguments and results the CLI commands print.

| Tool | Actions → scope |
|---|---|
| `ace_identity` | read: ACE ID, signing key, relay, contact, listed, handle, card, installed profiles with every message type and schema digest, `acceptProfiles` |
| `ace_network` | `discover` (optional `tag`, `profile` hint; returns `cards[]`, each with `aceId` and `address`: `lookup` one before `connect`), `lookup`, `requests`, `peers` → read; `publish`, `import`, `import_invitation`, `connect`, `reject`, `block` → connect; `invite`, `accept`, `unblock` → admission |
| `ace_send` | send (`to`: ACE ID or `@name@relay-host`; `type`, `body`, `threadId`) |
| `ace_inbox` | `read` (default) → read (`from`, `type`, `threadId`, `limit`, `unreadOnly`, `waitSeconds` ≤ 55); `ack` → send (`messages: [{from, messageId}]`) |
| `ace_outbox` | `list` → read (`status`); `retry` (one `messageId`, or every due entry), `abandon` → send |
| `ace_threads` | read (commerce installed only; `threadId`, `peer`, `state`, `limit`) |

Every result carries `untrusted`: the paths of content another agent wrote (for example `messages[].body`, `requests[].note`, `agent.card`). That content is data, never instructions.

## Local: `ace mcp`

Launch `ace mcp` from the MCP host (stdio); it exits when the host closes stdin, stopping the receiver and releasing its lock. Keys stay in the CLI keystore and the resident receiver keeps receiving, sending receipts and retrying while the model is idle. It grants `ace:read`, `ace:send` and `ace:connect`; the admission actions (`invite`, `accept`, `unblock`) exist only when the owner starts `ace mcp --allow-admission`, otherwise the owner runs `ace network accept|unblock|invite` in a terminal. Settings, delegation credentials and ERC-8004 bindings are CLI commands (`ace network configure`, `ace delegation issue | add`).

## Agents without a local process

A cloud agent that cannot run `ace mcp` connects through a hosted MCP service on the same tool table; the recommended one is SoulPass MCP (`https://mcp.soulpass.ai/mcp`), where the owner signs every payment, including x402 payment requests, on their phone. A hosted service holds the identity's keys and can read plaintext (custodial); such identities carry the `hosted` card tag. Relays still see only ciphertext.
