# Principal: several identities acting for one account

A **principal** is the account a person or organisation controls (for SoulPass users: the wallet account). One account often runs several ACE identities: an iPhone, this CLI on a Mac, a hosted MCP agent. A **principal record** in an identity's profile lets anyone verify that the identity acts for that account. It is signed by the account's owner key over this identity's signing key, so it cannot be copied to another identity. Normative text: ACE spec `09-principal.md`.

| Field | Meaning |
|-------|---------|
| `account` | CAIP-10 account, e.g. `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:<address>` |
| `roles` | `["controller"]` (approves), `["agent"]` (acts), or `["controller","agent"]` — exactly these, in this order |
| `signer` | `{"scheme","publicKey"}` of the owner key that signed |
| `issuedAt` / `expiresAt` | Unix seconds; `expiresAt` is required, later than `issuedAt` and at most 366 days after it; an expired record is invalid |
| `scope` | Optional free text your tools agree on (e.g. `copy:solana,hl`); ACE does not interpret it |
| `signature` | The owner key's signature |

## Getting a record

The CLI never holds your account's owner key, so it never creates a record; it only checks and publishes one.

1. Run `ace register` and copy `signingPublicKey` from its JSON output (canonical Base64 of this identity's signing key). That is what the owner signs over.
2. The owner signs a record for that key:
   - **SoulPass** signs principals for its own identities: the iPhone app does it with your passkey, and `soulpass agent attest` does it on a Mac whose device key is the wallet root. Signing a record for a separate ace-cli or hosted identity from the SoulPass app is not available yet.
   - Anyone holding the owner key can sign with the TypeScript SDK:
     ```ts
     import { createPrincipalRecord, principalSignerFromIdentity, fromBase64 } from '@ace-protocol/sdk';
     const record = await createPrincipalRecord(principalSignerFromIdentity(ownerIdentity), {
       subjectSigningPublicKey: fromBase64('<signingPublicKey from ace register>'),
       account: 'solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:<address>', roles: ['agent'],
       expiresAt: Math.floor(Date.now() / 1000) + 30 * 86400,
     });
     ```
     Any key source works through `PrincipalSigner` (`{scheme, publicKey, sign(digest)}`).
3. Save the record as JSON and publish it:
   ```bash
   ace register --principal ./principal.json
   ```
   The CLI verifies the record against this identity's own signing key before any network call (`--principal: ... is not a valid principal record for this identity` otherwise: wrong key, non-canonical roles, expired, bad signature), then registers, then saves it in `~/.ace/profile.json`. The output's `principal` is `{"account","roles","expiresAt"}` (`null` without one).

To withdraw it: `ace register --drop-principal` (this deletes `profile.json` when nothing else is in it). `--principal` and `--drop-principal` together are an error.

An expired saved principal makes `ace register` fail with an error naming both ways out (`--principal <file>` or `--drop-principal`); `ace listen` prints the same as a warning and keeps running. There is no immediate revocation: a withdrawn record that has not expired can still be replayed, so keep `expiresAt` short (days or weeks) and re-issue.

## Which identities count as the same account

The CLI opens the inbox with your own record's `account` and its `signer`. A peer's record counts as the same account when it names the same `account` **and** either it was signed by the same owner key as yours, or (for `eip155` accounts) its `secp256k1` signer's address is the account address. Records signed by different owner keys for one account are not matched by the CLI.

## The three principal messages

Between identities of **the same account** only. They never touch the commerce state machine and need no `--thread`.

```bash
# an agent asks its controller
ace send --to <controller aceId> --type request \
  --body '{"action":"pay","summary":"Pay 1 USDC to seller X","amount":"1","currency":"USDC","details":{"chain":"solana","asset":"USDC","to":"<address>","amount":"1"}}'
# the controller answers (requestId = the request's messageId)
ace send --to <agent aceId> --type decision --body '{"requestId":"<messageId>","outcome":"approve"}'
# the agent reports what it did
ace send --to <controller aceId> --type report \
  --body '{"action":"pay","summary":"Paid 1 USDC","outcome":"ok","requestId":"<messageId>","proof":{"txHash":"0x..."}}'
```

| Type | Direction | Required | Optional |
|------|-----------|----------|----------|
| `request` | agent → controller | `action`, `summary` | `ref` (`{conversationId, messageId, threadId?}`), `amount`, `currency`, `details` (object), `ttl` |
| `decision` | controller → agent | `requestId`, `outcome` (`approve` / `deny`) | `reason`, `result` (object) |
| `report` | either direction | `action`, `summary`, `outcome` (`ok` / `failed` / `skipped`) | `ref`, `requestId`, `proof` (object) |

Example action names: `pay`, `x402.pay`, `copy.run`, `sign`. They are labels only; ACE and the CLI execute nothing (no x402 or payment support is implied). A receiver that does not know an action still shows its `summary`.

**Approve against `details`.** `summary`, `amount` and `currency` are display text asserted by the requesting agent. A controller decides on what it will actually execute (`details`) and never treats `summary` / `amount` as authoritative when they disagree with `details`.

`ace inbox` adds a `principal` list: `{"messageId","from","type","action"?,"summary"?,"outcome"?,"requestId"?}` for each shown principal message. `ace inbox --type decision` shows only decisions.

## Rules the receiver enforces

- You need your **own** saved principal (`profile.json`): without one, every incoming `request` / `decision` / `report` is quarantined `wrong_principal`.
- The sender's verified record must name the same account (see above). A sender outside the account → `wrong_principal`.
- A `decision` is accepted only from a `controller` that is the identity the `request` was sent to (`wrong_principal` otherwise), only for a `request` you sent in that conversation that has not expired, and only once (`bad_reference` for an unknown, expired or already-decided request).
- `report` is accepted in either direction within the account.

Rejected messages are reported on stderr (`[inbox] Message rejected: ...`) and skipped.

**Known limitation.** If a controller pinned a delegate before the delegate added its principal, the pin has no principal. The SDK does one authenticated refresh of the sender's record before raising `wrong_principal`; if that refresh does not pick it up, messages stay `wrong_principal` until the pin refreshes (24 h TTL). Publish the principal before first contact where possible.

## Key custody

`hardwareBacking` in a registration file is self-asserted and cannot be verified; never treat it, or any claim about where keys live, as a trust signal. The principal binding is the verifiable fact: the owner's signature over this identity's key.
