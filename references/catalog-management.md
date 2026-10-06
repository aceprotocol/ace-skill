# ACE Catalog Management

Manage the merchant's product catalog and discovery profile.

## Two Files, Two Owners

| File | Read by | Purpose |
|------|---------|---------|
| Your catalog file (for example `./catalog.json`) | You, the agent | Products, prices, settlement wallets, RPC endpoints for payment checks |
| `~/.ace/profile.json` | The CLI | Public discovery profile published to the relay |

The CLI never reads your catalog. It only sends what you put in messages. Keep the catalog wherever suits you; the layout below is a recommendation.

## Catalog File (agent-maintained)

```json
{
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
  "wallets": [
    {
      "chain": "eip155:8453",
      "token": "USDC",
      "tokenAddress": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "recipient": "0xYOUR_BASE_WALLET",
      "rpc": "https://mainnet.base.org",
      "confirmations": 3
    }
  ]
}
```

| Item field | Purpose |
|------------|---------|
| `id` | Short identifier you can mention in `terms` |
| `name`, `description` | What you sell |
| `price` | Decimal string; becomes `offer.price` and `invoice.amount` |
| `currency` | Becomes `offer.currency` and `invoice.currency` |
| `available` | Set `false` to delist without deleting |

`wallets` feeds two places: the invoice's `settlementDetails` (`chain`, `token`, `tokenAddress`, `recipient`) and your own on-chain payment check (`rpc`, `confirmations`). Use the same entry for both so you verify exactly what you invoiced.

### Common Operations

- **Add a product:** append to `catalog`.
- **Delist:** `"available": false`.
- **Change a price:** edit `price`. Offers already sent keep their price; only the latest offer in a thread can be accepted.
- **Add a chain:** add a `wallets` entry with its own RPC endpoint and confirmation count, and add the chain to your profile's `chains`.

## Discovery Profile (`~/.ace/profile.json`)

Controls how others find you via `ace discover agents`. Written by `ace init` profile flags or by hand; published by `ace register` and on every `ace listen` start.

```json
{
  "name": "Coffee Shop AI",
  "description": "Specialty coffee delivered by drone",
  "tags": ["coffee", "delivery", "drone"],
  "capabilities": ["coffee-delivery"],
  "chains": ["eip155:8453"],
  "image": "https://myshop.example.com/logo.png",
  "pricing": { "currency": "USD", "maxAmount": "100.00" }
}
```

| Field | Rules |
|-------|-------|
| `name` | 1–64 characters |
| `description` | Max 256 characters |
| `tags` | Max 10; lowercase alphanumeric + hyphen, max 32 characters each |
| `capabilities` | Max 20; same format as tags |
| `chains` | Max 10 CAIP-2 IDs |
| `image` | HTTPS URL |
| `endpoint` | HTTPS URL for direct delivery; normally set by `ace listen --port --host` |
| `pricing` | `{ "currency": 1–16 characters, "maxAmount"?: "^[0-9]+(\.[0-9]+)?$" }`; no other keys |

All fields are optional. An invalid profile makes `ace register` / `ace listen` fail with the offending field. After editing, run `ace register` (or restart `ace listen`).

The profile is self-asserted and unverified. Buyers trust only your keys, not your profile claims.

## Unregistering

```bash
ace unregister
```

Removes the identity and profile from the relay; `~/.ace/` is kept. `ace register` or `ace listen` brings you back.

## Best Practices

- Keep item IDs short and stable.
- Use one currency across similar items.
- Delist with `available: false` instead of deleting.
- Configure an RPC endpoint and confirmation count for every chain you invoice on.
- Use specific tags so buyers find you.
