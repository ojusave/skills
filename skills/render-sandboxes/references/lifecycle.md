# Lifecycle and product choice

Sources: CLI [sandbox help](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/cmd/sandbox.go), [group help](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/cmd/sandboxgroups.go), [status schema](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/pkg/client/sandboxes/sandboxes_gen.go), Python [client](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/python/render/experimental/sandbox/client.py), and [Sandboxes docs](https://render.com/docs/sandboxes.md).

## Group and status

A group scopes a workspace's sandboxes to a region and optionally an environment. CLI help says early access permits at most one default group; the SDK `list_groups` comment says at most one group, returning zero or one. Do not infer multi-group availability. Inspect the returned group and region instead of hardcoding a region.

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

## When to choose another product

A sandbox fits a bounded run that needs a disposable OS environment. Choose based on how work arrives and how it must continue:

| Need | Route |
|---|---|
| On-demand tasks with orchestration, retries, or composition | `render-workflows` |
| A command on a schedule | `render-cron-jobs`; it can trigger Workflow tasks |
| A continuously running queue consumer | `render-background-workers` |
| An always-on server receiving public or private requests | `render-web-services` or `render-private-services` |

Read `render-workflows` and its [references/service-types.md](https://github.com/render-oss/skills/blob/main/skills/render-workflows/references/service-types.md) for this decision. Fetch the [service-types Markdown](https://render.com/docs/service-types.md) ([HTML](https://render.com/docs/service-types)) before relying on current product capabilities.
