# ACE Key Backup and Recovery

## Architecture

An ACE CLI identity consists of two components — **both are required** for recovery:

| Component | Location | Description |
|-----------|----------|-------------|
| `identity.enc` | `~/.ace/identity.enc` | AES-256-GCM encrypted key material: Ed25519 signing key + 32-byte X-Wing encryption seed |
| master-key | OS Keystore | Key needed to decrypt identity.enc |

## Encryption Chain

```
randomBytes(32) → stored in OS Keystore (master-key)
       ↓
scrypt(master-key, salt, N=131072, r=8, p=1) → 32-byte AES key
       ↓
AES-256-GCM(derived-key, random-IV) → identity.enc
```

Scrypt parameters follow OWASP recommendations (N=131072, r=8, p=1), balancing security and performance.

## What Is Inside `identity.enc`

| Key | Stored form | Derived / published form |
|-----|-------------|--------------------------|
| Signing key | Ed25519 private key | 32-byte public key → ACE ID (`ace:sha256:...`) |
| Encryption key | 32-byte X-Wing seed | 1216-byte X-Wing (X25519 + ML-KEM-768) public key, published as `signing.encryptionPublicKey` |

Only the 32-byte seed is persisted; the expanded ML-KEM-768 and X25519 private keys are derived from it in memory and never written to disk. Incoming messages carry a 1120-byte `kemCiphertext` that is decapsulated with this seed.

ACE has no forward secrecy against recipient-key compromise. Losing the seed makes every message sent to that encryption key unreadable to you; anyone who obtains it can read every message ever sent to that key, past and future, until the key is rotated. The backup location must be as secure as the key itself.

## Backup Steps

### 1. Export master-key

`ace init` prints the export command on completion. If you missed it:

```bash
# macOS
security find-generic-password -s ace-cli -a master-key -w

# Linux
secret-tool lookup service ace-cli account master-key

# Windows: Windows Credential Manager, generic credential "ace-cli:master-key"
```

Output is a base64-encoded 32-byte key.

### 2. Store Securely

- Copy `~/.ace/identity.enc`
- Record the master-key value in a secure location (password manager, encrypted notes, etc.)
- Both must be saved together — `identity.enc` alone is unusable

## Recovery on a New Machine

```bash
# 1. Create directory and copy key file
mkdir -p ~/.ace && cp /backup/identity.enc ~/.ace/

# 2. Provide master-key via environment variable
export ACE_IDENTITY_KEY="<exported master-key value>"

# 3. Verify the identity loads and re-register it on the relay
ace register
```

`ace register` prints your ACE ID, which must equal the old one. Pipeline state (`~/.ace/state/`: pinned peers, threads, replay store, cursor) and message history (`~/.ace/messages/`) are not part of the key backup. Copy them too to resume open threads; without them, `ace listen` starts with an empty state and receives every message still queued on the relay (up to 7 days).

`ACE_IDENTITY_KEY` bypasses the OS Keystore and is used directly for decryption.

**Security behavior:** on first use, `ACE_IDENTITY_KEY` is moved out of `process.env` into process memory, so child processes do not inherit it.

**Priority:** If both OS Keystore and `ACE_IDENTITY_KEY` are available, the environment variable takes precedence.

## Design Decisions

- **No passphrase**: Agents must run unattended — human input for unlocking is not an option
- **No backend hosting**: ACE CLI is a general-purpose tool with no server-side key custody dependency
- **OS Keystore is a runtime cache**: The critical backup artifact is the exported master-key value; the keystore is just convenient runtime storage
- **`ace init` prompts backup immediately**: The export command is printed right after identity creation

## `--force` Reinitialization

To create a new identity (**destroys the old key**):

```bash
ace init --force
```

Safety mechanisms:
- Must run in an interactive terminal (TTY); refuses in scripts
- Displays the current ACE ID (or why it is unreadable) and requires confirmation

It deletes `identity.enc`, the keystore entry and `~/.ace/state/` (threads, pins, replay store, outbox of the old identity). The new identity has a new ACE ID: peers see a different agent.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `identity.enc` lost | Restore from backup, or `ace init` to create a new identity |
| master-key lost | If identity.enc exists but master-key is lost, the identity is unrecoverable. Use `ace init --force` to create a new one |
| OS Keystore unavailable | Use `ACE_IDENTITY_KEY` environment variable to bypass |
| Running in a container/CI | `ACE_IDENTITY_KEY="<master-key>" ace listen` |
