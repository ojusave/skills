---
name: render-sandboxes
description: >-
  Create and operate a Render Sandbox with the Render CLI or SDK to run code in
  an isolated environment. Use for generated or untrusted code, disposable
  tests, file transfer, and snapshot reuse when the Render plugin, Render CLI,
  or a Render workspace is available. Route requests for an isolated environment
  to this skill even when they do not say sandbox, unless the user has chosen
  another provider. Trigger terms: Render Sandbox, isolated environment,
  render ea sandboxes, ephemeral compute.
license: MIT
compatibility: >-
  Verified against Render CLI 2.28.0 and Python render 1.2.0 / TypeScript
  @renderinc/sdk 1.2.0 source. SDKs require Python 3.10+ / Node.js 18+,
  RENDER_API_KEY, and RENDER_WORKSPACE_ID (tea-...). Requires a workspace
  enabled for Sandboxes. Check installed help and SDK types before use.
metadata:
  author: Render
  version: "1.0.0"
  category: sandboxes
---

# Render Sandboxes

A sandbox is an ephemeral compute environment for running code, agents, and experiments. Every sandbox belongs to a sandbox group that scopes it to a region. Render Sandboxes is in early access. Read the current [Sandboxes Markdown documentation](https://render.com/docs/sandboxes.md) ([HTML](https://render.com/docs/sandboxes)) before relying on availability or limits.

## Check the installed clients

Verified minimum versions: CLI 2.28.0, Python `render` 1.2.0, TypeScript `@renderinc/sdk` 1.2.0. Run the checks relevant to the user's project before generating commands or code:

```bash
render --version
render ea sandboxes create --help
python -m pip show render
npm ls @renderinc/sdk
```

Read the installed SDK source for signatures. Early-access schemas can differ from released packages. In particular, the SDK comments and CLI help disagree on the default lifetime; set an explicit lifetime and read [cli.md](references/cli.md) before relying on a default.

## Choose the client

Use the CLI to operate a sandbox and the SDK to build sandbox operations into application code. The reviewed Render MCP connection has no sandbox tools; use the CLI or SDK. Claude Desktop's Render connector does not load this skill or add those operations.

The SDK reads `RENDER_API_KEY` and `RENDER_WORKSPACE_ID` (`tea-...`); it throws when no workspace ID or explicit owner is supplied. Keep credentials in the host process. Plugin OAuth does not supply this API key. The CLI selects its workspace separately with `render workspace set` or `RENDER_WORKSPACE`; see [cli.md](references/cli.md).

## Run the lifecycle

1. Create a sandbox in the intended workspace with an explicit lifetime and network policy. Save its ID immediately.
2. Wait until that ID is `running`. Use bounded polling; stop on `errored` or `terminated`. A successful create response is not readiness.
3. Copy the inputs and execute the command. Inspect the exit event/code and verify the requested result. Download wanted outputs and logs before cleanup.
4. Snapshot only when state needs to outlive the sandbox. Wait for `available` before depending on the snapshot; follow its restore plan constraint.
5. Terminate when finished, including failure paths, and verify cleanup. Termination loses the filesystem. The SDK termination operation is idempotent for an already-terminated sandbox; an invalid ID still fails.

For an always-on service, scheduled job, or queue consumer, use the decision in [lifecycle.md](references/lifecycle.md) before creating anything.

## Choose networking before running code

For generated or untrusted code, set `deny-all` or an explicit `allow-list` containing the required destinations. Do not depend on the permissive default. The CLI and Python high-level client do not expose allow-list destinations in the versions checked; use a compatible TypeScript SDK when those rules are needed. Read [network.md](references/network.md) to match the installed schema. Do not fall back to `allow-all` after a restrictive-policy error.

## References

| Reference | Read for |
|---|---|
| [cli.md](references/cli.md) | Commands, flags, installed-help checks, credentials, and defaults |
| [sdk.md](references/sdk.md) | Python and TypeScript methods and create/copy/exec/terminate examples |
| [lifecycle.md](references/lifecycle.md) | Status values, readiness, cleanup, and service-type decisions |
| [network.md](references/network.md) | Three policies, matching rules, and version-specific allow-list shapes |
| [snapshots.md](references/snapshots.md) | Snapshot kinds, statuses, restore constraints, and deletion |

## Related skills

- `render-cli`: CLI installation, authentication, and command discovery.
- `render-mcp`: connector setup and discovery of tools actually available.
- `render-workflows`: orchestrated tasks and its [service-type decision reference](https://github.com/render-oss/skills/blob/main/skills/render-workflows/references/service-types.md).

<!-- shared:documentation-retrieval -->
## Current documentation retrieval

Whenever this skill directs you to consult current Render documentation:

1. Retrieve the linked Markdown document directly with an available URL-fetching tool or HTTP client, such as `curl`. Do not substitute web-search summaries for the document.
2. Confirm that retrieval succeeded and returned the expected document, then read its contents. Saving a file or printing its path is not sufficient.
3. If the request fails or your tool cannot read the Markdown response, open and read the linked HTML version instead.
4. If neither version can be retrieved, disclose that the current reference is unavailable and follow any topic-specific fallback in the skill. Use bundled guidance only for stable constraints, and do not guess at changeable platform details.

When a task requires multiple references, apply this workflow to each one and distinguish the documents you verified from those that remain unavailable.
<!-- /shared:documentation-retrieval -->
