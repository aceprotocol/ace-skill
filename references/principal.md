# Principal: Identities of One Account

A **principal** is an account a person or organisation controls. One account may run several ACE identities (a phone app, this CLI, a hosted agent). A **principal record** in an identity's profile lets others verify that the identity is affiliated with that account, and lets identities of the **same account** exchange `request` / `decision` / `report` under an account filter. Normative text: ACE spec `09-principal.md`.

**What it is not.** A principal record or a `decision` is not payment or execution authority. **To have the user's SoulPass wallet pay, do not use this** — use `pay-with-soulpass.md` (pairing + an exact `pay` request + the user's confirmation). Effects that need authority use exact-intent resource grants (ACE spec 10), never a role or an approve-shaped message.

## The record

| Field | Meaning |
|-------|---------|
| `account` | CAIP-10 account, e.g. `eip155:8453:0x<lowercase address>` or `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:<address>` |
| `roles` | Exactly `["controller"]` (approves), `["delegate"]` (acts) or `["controller","delegate"]` — in that order. No other role exists. |
| `signer` | `{"scheme","publicKey"}` of the account owner key that signed |
| `issuedAt`, `expiresAt` | Unix seconds; `expiresAt` required, after `issuedAt`, at most 366 days later |
| `scope` | Optional; the built-in rules reject any present scope (`wrong_principal`) |
| `signature` | The owner key's signature over this identity's signing key — it cannot be copied to another identity |

## Getting and publishing a record

The CLI never holds the account owner key, so it never creates a record; it verifies and publishes one.

1. `ace register` → copy `signingPublicKey` (canonical Base64 of this identity's signing key).
2. The account owner signs a record for that key — with the application that holds the owner key, or with the TypeScript SDK:
   ```ts
   import { createPrincipalRecord, principalSignerFromIdentity, fromBase64 } from '@ace-protocol/sdk';
   const record = await createPrincipalRecord(principalSignerFromIdentity(ownerIdentity), {
     subjectSigningPublicKey: fromBase64('<signingPublicKey from ace register>'),
     account: 'eip155:8453:0x<lowercase address>', roles: ['delegate'],
     expiresAt: Math.floor(Date.now() / 1000) + 30 * 86400,
   });
   ```
   Any key source works through `PrincipalSigner` (`{scheme, publicKey, sign(digest)}`).
3. Publish it:
   ```bash
   ace register --principal ./principal.json    # verified locally first, then registered, then saved in profile.json
   ace register --drop-principal                 # withdraw it
   ```
   Output `principal` is `{"account","roles","expiresAt"}` (or `null`). A record for another key, with non-canonical roles, expired or with a bad signature fails locally (`invalid_principal`). `--principal` and `--drop-principal` together are an error. Restart `ace listen` afterwards; it reads the saved principal at start.

An expired saved principal makes `ace register` fail naming both ways out (`--principal <file>` / `--drop-principal`); `ace listen` prints it as a warning. There is no immediate revocation: a withdrawn but unexpired record can be replayed, so keep `expiresAt` short (days or weeks) and re-issue.

## Which identities count as the same account

The CLI installs the account filter from your own saved record (its `account` and `signer`). A sender counts as the same account when its verified record names the same `account` **and** was signed by the same owner key as yours, or — for `eip155` accounts — its `secp256k1` signer's address is the account address. The CLI passes no other trusted signers.

## request / decision / report

These are not commerce messages: no `--thread` needed, no state machine.

| Type | Typical direction | Required | Optional |
|------|-------------------|----------|----------|
| `request` | delegate → controller | `action`, `summary` | `ref` (`{conversationId, messageId, threadId?}`), `amount`, `currency`, `details` (object), `ttl` |
| `decision` | controller → delegate | `requestId` (the request's messageId), `outcome` (`approve` / `deny`) | `reason`, `result` (object) |
| `report` | either | `action`, `summary`, `outcome` (`ok` / `failed` / `skipped`) | `ref`, `requestId`, `proof` (object) |

```bash
# a delegate asks its controller to approve a task step
ace send --to <controller id> --type request \
  --body '{"action":"publish.report","summary":"Publish the weekly summary to the team wiki","details":{"page":"weekly/2026-41","words":820}}'
# the controller answers
ace send --to <delegate id> --type decision --body '{"requestId":"<request messageId>","outcome":"approve"}'
# the delegate reports
ace send --to <controller id> --type report \
  --body '{"action":"publish.report","summary":"Published","outcome":"ok","requestId":"<request messageId>"}'
ace inbox --type decision
```

- `action` is a label; ACE and the CLI execute nothing. A receiver that does not know an action still shows `summary`.
- `summary`, `amount`, `currency` are display text asserted by the requester. A controller approves against `details` and must reject what it cannot fully interpret.
- `ace inbox` adds a `principal` list: `{"messageId","from","type","action"?,"summary"?,"outcome"?,"requestId"?}`.

## Rules the receiver enforces (with a valid own principal installed)

- Sender's verified record must be the same account (above) → else `wrong_principal`.
- A `decision` is accepted only from a `controller` that is the identity the request was sent to (`wrong_principal`), only for an unexpired `request` you sent in that conversation, and only once (`bad_reference`).
- `report` is accepted in either direction within the account.
- Without a valid own principal (none, expired, or bound to another key — warned on stderr), these types are received as ordinary data with no account semantics and no authority.

Rejected messages appear on stderr (`[inbox]`, `[relay]` or `[direct]` … `Message rejected: …`).

**Known limitation.** If a peer was pinned before it added its principal, the CLI refreshes the sender's record once before rejecting with `wrong_principal`; if that does not pick it up, retry after the pin's 24 h refresh. Publish principals before first contact.

## Key custody

`hardwareBacking` and any claim about where keys live are self-asserted and never a trust signal. The verifiable fact is the owner's signature over this identity's key.
