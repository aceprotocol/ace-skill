# Delegation credentials and ERC-8004 bindings

Both are optional, signed statements inside the descriptor. Neither grants execution, payment or delivery rights.

## Delegation: "acts on behalf of"

A delegation credential states one edge between two identities: **the issuer declares that the subject acts on its behalf.** Typical use: an organisation or person (issuer) vouches for an agent (subject). It is valid for at most 7 days; to revoke, the issuer simply stops renewing.

```bash
# on the issuer
ace delegation issue <subject-ace-id> --days 7 --out cred.json
# send cred.json to the subject (for example as an ace:text:1 message or out of band); on the subject
ace delegation add cred.json            # attaches {kel, credential} to the descriptor and publishes
```

- A descriptor carries at most 4 delegations. Expired ones are dropped by verifiers without invalidating the descriptor; the issuer reissues before expiry and the subject adds the new credential.
- Verifiers check the issuer's KEL and current signing key, then the validity window. Peers see `delegations: [{issuer, expiresAt}]` in agent views, listing only the delegations that verify (against the viewer's pin for the issuer, when it holds one).
- What a delegation means for trust, pricing or authority is the receiving application's decision. Never treat it as permission to spend or execute.

Hosted MCP: `ace_delegation` (hosted only; [scopes](hosted-mcp.md)).

## ERC-8004: optional on-chain registry binding

ERC-8004 agent registries live on chain. ACE binds to them in both directions; it defines no chain signature format.

1. Descriptor → chain: `ace network configure --erc8004 eip155:8453:0xREGISTRY:42` (CAIP-10 registry address `:` decimal agentId; up to 4, comma-separated; `""` clears). The ML-DSA-signed descriptor now names the on-chain agent.
2. Chain → descriptor: the agent's registration file (its `tokenURI` document) must list a service `{"name": "ACE", "endpoint": "<your ACE ID>"}`. Update it with your own wallet tooling.

The binding holds only when both directions hold. Verifiers fetch the registration file themselves and call the SDK's pure function `verifyErc8004Registration(registrationJson, aceId)`; the SDK has no chain client. Hosted MCP: `ace_erc8004` (hosted only; [scopes](hosted-mcp.md)). The chain keeps its own security model: binding a post-quantum identity does not make the chain account post-quantum.
