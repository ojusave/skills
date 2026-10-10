# Sandbox snapshots

Sources: CLI [`cmd/sandboxsnapshots.go`](https://github.com/render-oss/cli/blob/c4d106cfdaf8eec6d23475155e43f4baeeed2013/cmd/sandboxsnapshots.go) and sibling create/get/list/delete commands, SDK [Python snapshot client](https://github.com/render-oss/sdk/blob/python/v1.2.0/python/render/experimental/sandbox/client.py), [TypeScript snapshot client](https://github.com/render-oss/sdk/blob/typescript/v1.2.0/typescript/src/experimental/sandboxes/index.ts), and current [Sandboxes Markdown](https://render.com/docs/sandboxes.md) ([HTML](https://render.com/docs/sandboxes)).

## Capture and restore constraints

| Kind | Captures | Restore plan |
|---|---|---|
| `filesystem` (default) | Writable filesystem | Any supported plan |
| `runtime` | Writable filesystem, memory, and CPU state | Use the snapshot's `plan`, as required by the documentation |

Validate the runtime plan match before sending the request; do not depend on server rejection. A live test on October 9, 2026 accepted a mismatched plan and reached `running`, contrary to the documented constraint. That does not prove a supported resize or increased compute capacity. The current [Python reference](https://render.com/docs/sandboxes-sdk-python.md) ([HTML](https://render.com/docs/sandboxes-sdk-python)) says the plan parameter has no effect on available compute. Follow the documented matching-plan path and report a request for larger compute as requiring a supported alternative.

Both restore into a new sandbox in the source sandbox's group. Snapshot creation requires a running source and does not stop it. Use a snapshot only when state must outlive the sandbox; download individual outputs when that is sufficient.

A new snapshot is `creating`, then `available` or `failed`. Poll with a deadline. Do not restore until `available`; on `failed`, inspect `error`. A polling deadline does not prove the operation failed, so inspect the existing ID before repeating it.

## CLI

```bash
render ea sandboxes snapshots create sbx-... --kind filesystem --output json
# Save the snapshot ID. Repeat until available, or stop on failed/deadline.
render ea sandboxes snapshots get snp-... --output json
render ea sandboxes create --snapshot-id snp-... --timeout 300 --network-policy deny-all
# Save the new sandbox ID and wait until it is running before use.
render ea sandboxes list --all --output json
```

For runtime capture, use `--kind runtime`. On restore add `--plan` with the exact value returned in the snapshot. The CLI does not infer that requirement from the example.

Snapshot get/list/delete use the active workspace's default group unless given `--group sbg-...`. Use `render ea sandbox-groups list` to inspect groups. Main source includes snapshot `--name` and create `--from`; installed 2.28.0 does not, so baseline examples use IDs.

## SDK methods

On `sandboxes.snapshots`:

| Operation | Python async | TypeScript |
|---|---|---|
| Capture | `await create(sandbox_id, kind="filesystem")` | `await create({sandboxId, kind: "filesystem"})` |
| Get | `await from_id(sandbox_group_id=group_id, snapshot_id=id)` | `await get({sandboxGroupId, snapshotId})` |
| List | `await list(sandbox_group_id=group_id, status="available")` | `await list({sandboxGroupId, status: ["available"]})` |
| Delete | `await delete(sandbox_group_id=group_id, snapshot_id=id)` | `await delete({sandboxGroupId, snapshotId})` |

Python sync uses the same methods without `await`. Create also accepts `name` and `expires_at` (Python datetime) / `expiresAt` (TypeScript ISO 8601 string). Omit expiry for Render's default, then inspect the returned expiry field. Get/list/delete require the group ID. List accepts cursor/limit and returns `SnapshotList.snapshots` plus `next_cursor` in Python, or cursor-bearing `snapshot` wrappers in TypeScript. Follow the [pagination procedure](sdk.md#credentials-and-methods); a single or short page is not necessarily the complete list.

Restore uses ordinary sandbox creation: Python `await sandboxes.create(snapshot_id=snapshot.id, timeout_seconds=300, network_policy="deny-all")`; TypeScript `await sandboxes.create({snapshotId: snapshot.id, timeoutSeconds: 300, networkPolicy: {default: "deny-all"}})` for released 1.2.0. Add `plan=snapshot.plan` / `plan: snapshot.plan` for runtime snapshots, then wait for the new sandbox to be `running`.

## Expiry and deletion

Fetch current docs for the default lifetime; use `expires_at` / `expiresAt` as the specific snapshot's deadline. After expiry it cannot be retrieved or restored. Lists omit expired and deleted snapshots. Error classes alone do not establish which state occurred: a live restore attempt after deletion returned `SnapshotNotReadyError` with code `snapshot_not_available`, rather than `SnapshotNotFoundError`. Inspect the saved snapshot ID and operation history before deciding to poll or retry; do not repeatedly create from a known deleted snapshot.

Deleting a snapshot prevents future restores and does not affect sandboxes already restored from it. A snapshot still `creating` cannot be deleted. SDK deletion is documented as idempotent for already-deleted/expired snapshots, but a never-valid ID errors. The CLI retrieves first and can report an expired/deleted ID as not found.

```bash
render ea sandboxes snapshots list --status available --output json
# Preview deletion without changing anything:
render ea sandboxes snapshots delete snp-...
# When deletion is part of the requested cleanup:
render ea sandboxes snapshots delete snp-... --confirm
```

Terminate source and restored sandboxes when their work is done. Deleting a snapshot does not perform that cleanup.
