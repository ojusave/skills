# Sandbox networking

Choose the policy before executing generated or untrusted code. Use `deny-all` when the workload needs no network. If it needs downloads or an API, use an explicit `allow-list` containing only the required destinations. If no available client or public API path can express the requested restrictions, stop before running the workload. Never widen access to resolve a compatibility error.

| Policy | Meaning |
|---|---|
| `allow-all` | Allows outbound traffic. Do not use as an implicit fallback for untrusted code. |
| `deny-all` | Denies outbound traffic. Supported by the reviewed CLI and both SDKs. |
| `allow-list` | Admits only the listed destinations and protocols described by the matching schema. Requires destination rules. |

Do not label a policy or sandbox as universally secure.

## Check client support

| Client | What the reviewed interface can express |
|---|---|
| CLI 2.28.0 | `--network-policy deny-all` or `allow-all`. No flag supplies the destinations required for `allow-list`. |
| Python 1.2.0 high-level client | `network_policy="deny-all"` or `"allow-all"`. `create` takes a string. No high-level `allowed_domains` or `rules` keyword exists. |
| TypeScript 1.2.0 | Uses `default` and `allowedDomains`. Its allow-list request was rejected by the live API on October 9, 2026; do not treat this shape as a working fallback. |
| TypeScript main at `4699a103` | Uses `type` and `rules`. This source revision does not establish that an installed or published package supports it. |
| Public REST create | Accepts `type` and HTTPS `rules`; verified live in an enabled workspace on October 9, 2026. The returned ID works with the reviewed CLI/SDK lifecycle operations. |

Do not pass a policy dictionary into Python's string parameter, invent CLI flags, or suppress TypeScript errors to pass an unsupported shape. Check the installed schema and current [TypeScript SDK Markdown](https://render.com/docs/sandboxes-sdk-typescript.md) ([HTML](https://render.com/docs/sandboxes-sdk-typescript)). Discover any connected MCP tool's real parameters before choosing it.

The [released TypeScript schema](https://github.com/render-oss/sdk/blob/typescript/v1.2.0/typescript/src/generated/schema.ts) describes `default: "allow-list"` with `allowedDomains`, including HTTP and HTTPS. The live API rejected that request with HTTP 400 because `networkPolicy.rules` was missing. A package satisfying the documented minimum version does not prove every operation is compatible.

## Public API path for destination rules

Source: the public [`POST /sandboxes`, `sandboxNetworkPolicyPOST`, and `sandboxEgressRule` definitions](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/typescript/src/generated/schema.ts).

From the host, send `POST https://api.render.com/v1/sandboxes` with `Content-Type: application/json` and the host's Render API key in the `Authorization: Bearer ...` header. Set a request deadline, supply the intended workspace ID, and send this body with the required hostname substituted:

```json
{
  "ownerId": "tea-YOUR_WORKSPACE_ID",
  "timeoutSeconds": 300,
  "networkPolicy": {
    "type": "allow-list",
    "rules": [{ "domain": "example.com", "protocol": "https" }]
  }
}
```

Save the successful response's sandbox ID immediately, then use the normal CLI/SDK readiness, transfer, execution, and termination steps. Before uploading or running the workload, use `GET /v1/sandboxes/{id}` to verify its effective `networkPolicy` has exactly the requested type and rules. A deprecated `default` alias may also appear in the response. If the policy differs, terminate the task-owned sandbox and report the mismatch. A create timeout has an unknown outcome; follow [failure recovery](lifecycle.md#recover-from-failures) without blindly repeating the POST.

The current schema requires `rules` for `allow-list` and rejects them otherwise. Each domain can be listed once. `example.com` does not include `api.example.com`; a leftmost wildcard such as `*.example.com` must be explicitly needed and does not replace the apex. Do not add wildcards speculatively. Only `https` is accepted for a rule's protocol; the schema says other outbound protocols are dropped. `default` is a deprecated alias for `type`; if both are sent, they must agree.

The live check allowed `https://example.com`, blocked `https://example.org`, and blocked `http://example.com`, with all three destinations reachable from the host as positive controls. This verifies those cases in the tested workspace, not universal isolation against every possible bypass. For a different workload, check required destinations and a disallowed destination before executing its code.

## Omitted defaults are not a policy choice

Python's high-level create comment says an omitted policy uses the workspace default. Current SDK main's `sandboxNetworkPolicyPOST` says omission uses `allow-all`. Set a restrictive policy explicitly; do not use omission to resolve that disagreement.

For package downloads, enumerate the actual package index and artifact/CDN hosts needed by the workload. Keep the Render API key in the host process; never pass it into the sandbox.
