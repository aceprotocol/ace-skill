# ACE Key Backup and Recovery

## Architecture

An ACE CLI identity consists of two components — **both are required** for recovery:

| Component | Location | Description |
|-----------|----------|-------------|
| `identity.enc` | `~/.ace/identity.enc` | AES-256-GCM encrypted Ed25519 keypair |
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

## Backup Steps

### 1. Export master-key

`ace init` prints the export command on completion. If you missed it:

```bash
# macOS
security find-generic-password -s ace-cli -a master-key -w

# Linux
secret-tool lookup service ace-cli account master-key

# Windows (PowerShell)
cmdkey /list:ace-cli:master-key
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

# 3. Verify identity works
ace listen
```

`ACE_IDENTITY_KEY` bypasses the OS Keystore and is used directly for decryption.

**Security behavior:** `ACE_IDENTITY_KEY` is consumed (deleted from `process.env`) after first read to minimize the exposure window. This means the same process cannot rely on reading this variable multiple times.

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
- Must run in an interactive terminal (TTY)
- Displays current ACE ID and requires confirmation
- Refuses to execute in scripts or non-TTY environments

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `identity.enc` lost | Restore from backup, or `ace init` to create a new identity |
| master-key lost | If identity.enc exists but master-key is lost, the identity is unrecoverable. Use `ace init --force` to create a new one |
| OS Keystore unavailable | Use `ACE_IDENTITY_KEY` environment variable to bypass |
| Migrating to container/CI | `ACE_IDENTITY_KEY="<master-key>" ace listen` |
