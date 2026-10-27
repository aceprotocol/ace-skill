# Next root, rotation, recovery

## How an identity is anchored

An ACE identity is a key event log (KEL, spec `01-identity.md`). The inception event (`icp`) names a root key (`ml-dsa-65`) and commits the SHA-256 digest of the **next root**. The ACE ID is `ace:sha256:<SHA-256 of icp>`, so it never changes. The root signs only log events:

```
icp  (root R0, next = digest(R1))  → ACE ID
dlg  signing key S1 (ml-dsa-65)    → signs packets, receipts, descriptors, permits, credentials, relay auth
rot  (root R1, next = digest(R2))  → signed by R1 alone; no signing key until the next dlg
dlg  signing key S2
```

There is one delegated key, the signing key. The X-Wing prekey used to receive messages is not in the KEL: it is published in the signed descriptor and rotated every 24 hours.

`ace key show` prints `root {scheme, digest}`, `next {digest, held: local|external}`, `signing {seq, scheme, notAfter}`, the events and the keystore mode. No key material.

## Why the next root matters

- **Root compromise.** A `rot` is signed by the revealed next root alone; whoever holds only the current root cannot rotate. The holder of the next root appends a `rot` that forks the log at the start of the current root epoch, revoking every delegation a thief added. Peers adopt it as `extended` or `recovered`.
- **Device loss.** A `rot` needs nothing from the lost device if the next root is kept elsewhere.
- **Algorithm agility.** The commitment is a hash, so a `rot` can move to a new key or, later, a future scheme (today every root is `ml-dsa-65`).

Anyone holding the committed next root controls the identity: keep it offline.

## Where it lives

| Option | How | Protects against |
|---|---|---|
| In this keystore (default) | `ace init` generates and seals it with the root | Routine rotation. Not against theft of the whole machine; device loss means identity loss |
| External | `ace key next-root --out <file>` elsewhere; commit with `ace init --next-root <digest>` or `ace key rotate --next-root <digest>` | Root compromise on this machine and device loss (recover with `ace key recover`) |

```bash
# on another machine or offline medium (no identity needed)
ace key next-root --out /media/offline/ace-next-root.json        # prints {digest, scheme, file}
# on the agent host
ace init --next-root <64-hex digest>                               # at creation, or later:
ace key rotate --next-root <64-hex digest>
```

The file is created 0600, never overwritten, and holds a root seed; the seed is never printed. To use it, copy it to the host (mode 0600, owned by you) and run `ace key rotate --root-file <file> [--next-root <digest of a fresh external file>]`; afterwards delete the file.

## Rotation and delegation

| Command | Events | Effect |
|---|---|---|
| `ace key rotate [--next-root <hex>] [--root-file <file>]` | `rot`, `dlg` | Next root becomes the root, a new next root is committed, a fresh signing key is delegated |
| `ace key delegate [--not-after <unix>]` | `dlg` | Fresh signing key (suspected signing-key leak); `notAfter` at most 366 days ahead |

Both refuse while a receiver (`ace listen`, `ace mcp`, `ace inbox`) holds the lock: stop it first. They save the identity, publish, and print `{aceId, seq, signingSeq, published}`; if publishing failed, run `ace network publish`. Then restart the receiver. Pending Outbox entries are re-signed automatically on their next attempt. Peers adopt the extension of the log they pinned; the ACE ID and admissions stay.

## Device loss

```bash
ace key recover ace:sha256:<id> --root-file /media/offline/ace-next-root.json [--descriptor-file saved.json]
```

The KEL is taken from the relay's descriptor or, once that lapsed, the card record; otherwise pass a saved descriptor (`ace network publish --out`). The card record's set-once handle is kept and nothing else of it: re-declare the rest with `ace network configure`, as the replaced keys may have signed it. The log is extended with a `rot` signed by the file's key and a fresh signing delegation; every earlier delegation, including any a thief added, is revoked. The result is installed and published (`{aceId, seq, published}`). Afterwards place a new external next root (`ace key next-root` elsewhere, then `ace key rotate --next-root`). Admissions and messages from the lost device do not carry over: exchange invitations or reconnect.

With the next root held locally and no copy of `~/.ace` plus master key, the identity is lost: `ace init --force` creates a new one (new ACE ID; peers must pair again).

## Import re-keys at once

`ace init --import <export.json>` takes an export holding the root, next root and signing seeds with the KEL (e.g. the hosted MCP owner export). It refuses an export without the next root, then appends `rot`, `rot`, `dlg`, so no key the previous custodian knows is current, and publishes. A hosted copy stops receiving once the relay has the new keys; delete it with the owner `delete` action. This protects against a custodian compromised later, not one already hostile (it could rotate first). Delete the export afterwards.

## Troubleshooting

| Problem | Fix |
|---|---|
| `the next root key is not held by this identity` | It is external: `ace key rotate --root-file <file>` |
| `--root-file: … must be mode 0600 and owned by the current user` | `chmod 600 <file>` |
| `keys saved but not published` | `ace network publish` |
| `recover: …; nothing changed` | The file is not the committed next root of that ACE ID, or the KEL could not be read from the relay (pass `--descriptor-file`) |
| A peer reports `kel_conflict` after recovery | Its pin diverges before the current epoch; confirm over a trusted channel, then it re-pins |
