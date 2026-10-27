# Delegation credentials and ERC-8004 bindings

Both are optional statements another party can check. Neither grants execution, payment or delivery rights.

## Delegation: "acts on behalf of" (profile `ace:delegation:1`)

A delegation credential states one edge between two identities: **the issuer declares that the subject acts on its behalf.** Typical use: an organisation or person (issuer) vouches for an agent (subject). Credentials are the official profile `ace:delegation:1`: the issuer signs one, the subject publishes it in its descriptor `ext["ace:delegation:1"]`. A credential lives at most 7 days; to revoke, the issuer simply stops renewing.

```bash
# on the issuer
ace delegation issue <subject-ace-id> --days 7 --out cred.json
# hand cred.json to the subject (it is public data; any channel works); on the subject
ace delegation add cred.json            # looks the issuer up, verifies, publishes it in the descriptor ext
```

- The file is the bare credential `{issuer, subject, issuedAt, expiresAt, signature}`, no KEL. `add` looks the issuer up and verifies the signature against the issuer's live descriptor, so the issuer must be published and reachable; a lookup failure is reported as itself (for example `unknown_peer`), never as "not delegated".
- A descriptor carries at most 4 credentials, one per issuer; `add` replaces one from the same issuer and drops expired ones. The issuer reissues before expiry and the subject adds the new credential.
- Descriptor verification checks only the shape of the extension. A peer that cares whether the subject acts for the issuer verifies the credential itself, against its own lookup of the issuer (SDK: `NetworkClient.verifyDelegation`); the result is bound to that pin of the issuer.
- What a delegation means for trust, pricing or authority is the receiving application's decision. Never treat it as permission to spend or execute.

Hosted MCP: `ace_delegation` (`list`, `issue`, `attach` with `credential`, `detach` with `issuer`; hosted only; [scopes](hosted-mcp.md)).

## ERC-8004: optional on-chain registry binding

ERC-8004 agent registries live on chain. ACE binds to them in both directions; it defines no chain signature format.

1. Card record → chain: `ace network configure --erc8004 eip155:8453:0xREGISTRY:42` (CAIP-10 registry address `:` agentId, decimal for `eip155`; up to 4, comma-separated; `""` clears). The signed card record, next to `pay`, now names the on-chain agent.
2. Chain → ACE: the agent's registration file (its `tokenURI` document) must list a service `{"name": "ACE", "endpoint": "<your ACE ID>"}`. Update it with your own wallet tooling.

Like `pay`, an `erc8004` entry is claimed, not proven: the binding holds only when both directions hold. Verifiers fetch the registration file themselves and call the SDK's pure function `verifyErc8004Registration(registrationJson, aceId)`; the SDK has no chain client. Hosted MCP: `ace_erc8004` (`list`, `verify`, `attach`, `detach`; hosted only; [scopes](hosted-mcp.md)). The chain keeps its own security model: binding a post-quantum identity does not make the chain account post-quantum.
