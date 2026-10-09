# Sandbox networking

Choose the policy before executing generated or untrusted code. Use `deny-all` when the workload needs no network. If it needs downloads or an API, use an explicit `allow-list` through a client that can encode the required destinations. If the installed client cannot express the requested restrictions, stop before running the workload and identify the missing client capability.

| Policy | Meaning |
|---|---|
| `allow-all` | Allows outbound traffic. Do not use as an implicit fallback for untrusted code. |
| `deny-all` | Denies outbound traffic. Supported by the reviewed CLI and both SDKs. |
| `allow-list` | Admits only the listed destinations and protocols described by the matching schema. Requires destination rules. |

These values remain in the reviewed schemas. Do not label a policy or sandbox as universally secure.

## Check client support

| Client | What the reviewed interface can express |
|---|---|
| CLI 2.28.0 and main | `--network-policy deny-all` or `allow-all`. The validator's schema contains `allow-list`, but the command has no flag for its required destinations. |
| Python 1.2.0 and main high-level client | `network_policy="deny-all"` or `"allow-all"`. `create` takes a string; `api.py` constructs the policy without destinations. No high-level `allowed_domains` or `rules` keyword exists. |
| TypeScript 1.2.0 | Structured `networkPolicy` with `default` and, for allow-list, `allowedDomains`. |
| TypeScript current main | Structured `networkPolicy` with `type` and, for allow-list, `rules`. `default` is a deprecated alias for `type`. |

Do not pass a policy dictionary into Python's string parameter or invent CLI flags. Use a TypeScript package whose schema matches the required operation, then verify the effective policy returned by the API before running code.

## Released TypeScript 1.2.0 shape

Source: [`typescript/src/generated/schema.ts` at typescript/v1.2.0](https://github.com/render-oss/sdk/blob/typescript/v1.2.0/typescript/src/generated/schema.ts), `sandboxNetworkPolicy`. The CLI [generated schema](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/pkg/client/sandboxes/sandboxes_gen.go) also carries this shape.

```typescript
const networkPolicy = {
  default: "allow-list" as const,
  allowedDomains: ["example.com", "*.example.com"],
};
// Supply networkPolicy to sandboxes.create after checking the installed schema.
```

`allowedDomains` is required for `allow-list` and rejected for the other policies. Matching is exact: `example.com` does not cover `api.example.com`. Only a leftmost wildcard such as `*.example.com` is allowed. List the apex separately when it is needed. This schema says only HTTP and HTTPS traffic is matched; other outbound TCP is dropped.

## Current SDK main shape differs

Source: [`typescript/src/generated/schema.ts` at 4699a103](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/typescript/src/generated/schema.ts), `sandboxNetworkPolicyPOST` and `sandboxEgressRule`; Python [`sandbox_egress_rule.py`](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/python/render/public_api/models/sandbox_egress_rule.py).

```typescript
const networkPolicy = {
  type: "allow-list" as const,
  rules: [
    { domain: "example.com", protocol: "https" as const },
    { domain: "*.example.com", protocol: "https" as const },
  ],
};
// This is the main-source schema, not a claim about an installed 1.2.0 release.
```

`rules` is required for `allow-list` and rejected otherwise; each domain can be listed once. Exact and leftmost-wildcard matching remain. Only HTTPS is accepted, and the schema says other outbound protocols are dropped. `default` is a deprecated alias for `type`; if both are sent, they must agree.

Do not merge these examples. Released schema's HTTP/HTTPS claim and main's HTTPS-only claim are different. Read the installed generated schema and fetch the current [TypeScript SDK Markdown](https://render.com/docs/sandboxes-sdk-typescript.md) ([HTML](https://render.com/docs/sandboxes-sdk-typescript)) before choosing a shape. If the service rejects it, report the incompatibility rather than widening access.

## Omitted defaults are not a policy choice

Python's high-level create comment says an omitted policy uses the workspace default. Current SDK main's `sandboxNetworkPolicyPOST` says omission uses `allow-all`. Set a restrictive policy explicitly; do not use omission to resolve that disagreement.

For package downloads, enumerate the actual package index and artifact/CDN hosts needed by the workload. A hostname allow-list does not justify sending the host's Render API key into the sandbox.
