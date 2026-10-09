# Lifecycle and product choice

Sources: CLI [sandbox help](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/cmd/sandbox.go), [group help](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/cmd/sandboxgroups.go), [status schema](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/pkg/client/sandboxes/sandboxes_gen.go), Python [client](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/python/render/experimental/sandbox/client.py), and [Sandboxes docs](https://render.com/docs/sandboxes.md).

## Operation coverage

The CLI and high-level SDK paths cover the core sandbox lifecycle and snapshots. They are not the entire sandbox API. Use the connected MCP tool schema for its exact coverage; a connector does not imply every operation is exposed.

| Requested operation | Available path and boundary |
|---|---|
| Create, list, inspect, execute, upload/download, terminate | CLI and SDK paths in [cli.md](cli.md) and [sdk.md](sdk.md). CLI inspection uses the list response; SDKs have a read-by-ID method. |
| Snapshot create, list, inspect, delete, and restore | CLI and both SDKs; restore creates a new sandbox. See [snapshots.md](snapshots.md). |
| List sandbox groups | CLI and both SDKs. The reviewed public interfaces do not expose group create/update/delete methods. |
| Read execution history or a specific execution record | Public API schema defines list/retrieve endpoints; the reviewed high-level SDKs and sandbox CLI subcommands have no dedicated method. See [SDK/API gaps](sdk.md#operations-outside-the-high-level-sdk). |
| Replay or follow sandbox-wide logs; list directory metadata | Public schema defines endpoints, but high-level clients do not wrap them. Confirm the endpoint is callable before relying on it. Live stdout/stderr from `exec` is already supported. |
| Restrict outbound destinations | Requires a tool/client that accepts destination rules; see [network.md](network.md). |
| Explicit suspend/resume, resize, or change a running sandbox's policy | Not exposed by the reviewed CLI/high-level SDK or public REST schema. Do not invent a command from a status enum. |

For a request that needs an uncovered operation, inspect current public documentation and the available tool/client schema. Use a confirmed public interface or report the specific gap. Do not substitute an internal endpoint or claim full coverage from generated types. Source review is not a live operation test.

## Group and status

A group scopes a workspace's sandboxes to a region and optionally an environment. Reviewed CLI help says at most one default group; the SDK `list_groups` comment says at most one group, returning zero or one. Do not infer multi-group availability. Inspect the returned group and region instead of hardcoding a region.

| Status | What the procedure does |
|---|---|
| `creating` | Wait with a deadline. |
| `running` | Copy files and execute. |
| `suspended` | Not ready; inspect state and wait only within the deadline. |
| `resuming` | Wait for `running` within the deadline. |
| `errored` | Stop the workload and inspect the failure. |
| `terminated` | Stop using this sandbox; its filesystem is gone. |

All six values exist in the reviewed schemas. CLI list help mentions only four. The clients reviewed here do not expose suspend/resume commands; a status value does not imply a callable operation.

## Complete a run

Creation returns an initial state. Poll `from_id` (Python), `get` (TypeScript), or the CLI list for the exact ID until `running`. Bound the wait and report the ID if readiness fails. Do not blindly repeat create after a lost response.

Copy inputs, then execute. SDK exec sends a command string to `bash -c`. A nonzero process exit is an exit event, not an SDK exception; transport and terminal stream errors can throw. Treat a missing exit event as incomplete, and verify the expected output independently of the command's success message.

Download wanted artifacts and logs before termination. If the filesystem must be reused, capture a [snapshot](snapshots.md) while running and wait for `available`. Termination permanently loses the sandbox filesystem; a snapshot is a separate resource with an expiry.

Put SDK termination in `finally` and preserve the ID if cleanup fails. Python documents termination as idempotent for an already-terminated ID (204); a never-valid ID errors. CLI stop without `--confirm` only previews. Verify actual state after cleanup rather than treating a local message as proof. Terminate only resources owned by this task or explicitly included in the request.

## Recover from failures

Record the workspace, creation time, requested lifetime, and returned IDs as the run proceeds. Preserve the original operation error separately from any cleanup error. The SDK examples demonstrate the normal lifecycle; they are not durable job supervisors and cannot guarantee cleanup if the host process is killed.

- **Create times out before returning an ID:** Report an unknown outcome. Do not repeat create automatically. Check request logs and paginated workspace listings, including terminated resources, to reconcile the original request. A timestamp or similar configuration alone is not proof of ownership. Continue or terminate only an ID attributable to this task. If attribution remains ambiguous, preserve the request time and inputs, report unresolved cleanup, and ask for the missing evidence or operator help. The explicit lifetime bounds resource lifetime but is not proof of termination.
- **Readiness or another read fails:** Retry transient transport errors, HTTP 429, or 5xx within a deadline, with bounded backoff and `Retry-After` where provided. Stop on authentication, permission, or validation errors and report the cause. A failed read must not trigger a replacement create. On deadline expiry, clean up the saved task-owned ID.
- **Execution is interrupted:** A missing terminal exit event is an incomplete result. Do not replay a potentially side-effecting command without checking its outcome. Aborting a stream does not terminate the sandbox. Retrieve needed outputs if possible, then explicitly terminate when the run is over.
- **Termination fails or its response is lost:** Retain the exact owned ID, inspect its state, and retry termination only for that ID within a small bounded budget, such as three attempts with backoff. Termination is idempotent for an already-terminated sandbox. Preserve both errors if the workload and cleanup fail; do not claim success while cleanup is unresolved.

Verify `terminated` through an exact-ID read, or an exact match in a paginated terminated listing if the client's get helper cannot return terminated resources. An unexplained 404, an authentication failure, a CLI preview, or elapsed lifetime is not sufficient evidence. If verification still fails, report the ID, last confirmed state, and next cleanup action. Never delete an unrelated sandbox to make a workspace look clean.

## When to choose another product

A sandbox fits a bounded run that needs a disposable OS environment. Choose based on how work arrives and how it must continue:

| Need | Route |
|---|---|
| On-demand tasks with orchestration, retries, or composition | `render-workflows` |
| A command on a schedule | `render-cron-jobs`; it can trigger Workflow tasks |
| A continuously running queue consumer | `render-background-workers` |
| An always-on server receiving public or private requests | `render-web-services` or `render-private-services` |

Read `render-workflows` and its [references/service-types.md](https://github.com/render-oss/skills/blob/main/skills/render-workflows/references/service-types.md) for this decision. Fetch the [service-types Markdown](https://render.com/docs/service-types.md) ([HTML](https://render.com/docs/service-types)) before relying on current product capabilities.
