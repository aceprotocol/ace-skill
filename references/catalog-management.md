# ACE Catalog Management

Manage the merchant's product catalog and configuration files.

## File Overview

| File | Location | Purpose |
|------|----------|---------|
| `ace-merchant.json` | Project directory (cwd) | Product catalog, settlement chains, RPC verification config |
| `~/.ace/profile.json` | Home directory | Network discovery profile (name, tags, pricing) |

Both files are edited as JSON directly — no CLI commands. Restart `ace listen` after modifying the profile for changes to take effect.

## Merchant Config (`ace-merchant.json`)

### Full Structure

```json
{
  "ace": "1.0",
  "merchant": {
    "name": "Coffee Shop AI",
    "description": "Specialty coffee delivered by drone"
  },
  "catalog": [
    {
      "id": "latte",
      "name": "Oat Milk Latte",
      "description": "12oz signature latte with oat milk",
      "price": "6.50",
      "currency": "USD",
      "available": true
    }
  ],
  "settlement": ["crypto/instant"],
  "chains": [
    { "network": "eip155:8453", "address": "0xYOUR_BASE_WALLET" }
  ],
  "relay": "https://relay.aceprotocol.org",
  "verification": {
    "rpc": {
      "eip155:8453": "https://mainnet.base.org"
    },
    "confirmations": 3
  }
}
```

### Catalog Item Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique identifier, appears in message threads |
| `name` | string | yes | Product name |
| `description` | string | no | Detailed description |
| `price` | string | yes | Price (string format, e.g. "6.50") |
| `currency` | string | yes | Currency code (e.g. "USD") |
| `available` | boolean | no | Whether available for sale (default true) |

### Common Operations

**Add a product** — append to the `catalog` array:

```json
{
  "id": "espresso",
  "name": "Double Espresso",
  "description": "Two shots of our house blend",
  "price": "4.00",
  "currency": "USD",
  "available": true
}
```

**Take a product offline** — set `"available": false` (preserves the record) or remove from the array.

**Update price** — modify the `price` field directly.

**Add a settlement chain** — update both `chains` and `verification.rpc`:

```json
{
  "chains": [
    { "network": "eip155:8453", "address": "0xBASE_WALLET" },
    { "network": "eip155:1", "address": "0xETH_WALLET" }
  ],
  "verification": {
    "rpc": {
      "eip155:8453": "https://mainnet.base.org",
      "eip155:1": "https://eth.llamarpc.com"
    },
    "confirmations": 3
  }
}
```

### Validation Errors

`ace listen` validates the config on startup. If the file exists but is malformed or missing required fields, it fails with an explicit error. Common errors:

| Error | Cause |
|-------|-------|
| `merchant.name is required` | Missing `merchant.name` |
| `catalog must have at least one item` | Empty catalog array |
| `catalog item missing id` | Item missing `id` field |
| `catalog item "xxx" missing name` | Item missing `name` field |
| `catalog item "xxx" missing price` | Item missing `price` field |
| `catalog item "xxx" missing currency` | Item missing `currency` field |
| `settlement must have at least one method` | Empty settlement array |

## Discovery Profile (`~/.ace/profile.json`)

Controls how other agents find you via `ace discover agents`. Sent to the relay automatically when `ace listen` starts.

```json
{
  "name": "Coffee Shop AI",
  "description": "Specialty coffee delivered by drone",
  "tags": ["coffee", "delivery", "drone"],
  "chains": ["eip155:8453"],
  "endpoint": "https://myshop.example.com/ace/receive",
  "pricing": {
    "currency": "USD",
    "maxAmount": "100.00"
  }
}
```

| Field | Description |
|-------|-------------|
| `name` | Display name |
| `description` | One-line description |
| `tags` | Tag array for search matching |
| `chains` | Supported chain IDs (CAIP-2) |
| `endpoint` | P2P direct delivery endpoint (HTTPS) |
| `pricing.currency` | Pricing currency |
| `pricing.maxAmount` | Max price per transaction |

All fields are optional. Restart `ace listen` for updates to take effect.

To unregister from the discovery network:

```bash
ace unregister
```

Local files (`~/.ace/`) are preserved. Re-run `ace listen` to go online again.

## Best Practices

- Keep product IDs short and meaningful (they appear in message threads)
- Use consistent currency across similar products
- Use `available: false` for temporary delistings instead of deleting
- Configure a matching RPC endpoint for every chain in `chains`
- Use specific tags in your profile to help buyers find you via search
