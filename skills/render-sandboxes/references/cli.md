# Sandbox tasks with the Render CLI

Use this path when the connected MCP server has no sandbox tools. Reuse an existing CLI login; if needed, ask the user to run `render login`. Never ask them to paste credentials into chat.

## Check the installed commands

The commands below were checked with Render CLI 2.28.0. Inspect help before using them on a different version:

```bash
render ea sandboxes --help
render ea sandboxes create --help
render workspace current -o json
```

Use the user's confirmed workspace. If the current workspace differs, explain that before changing it with `render workspace set <workspace-id>`.

## Create, execute, inspect, stop

For an offline task:

```bash
render ea sandboxes create --timeout 600 --network-policy deny-all --confirm -o json
```

Read the returned sandbox ID. Poll `render ea sandboxes list -o json` until that ID is `running`; stop on a terminal error or after a bounded wait. Do not execute while it is still creating.

Use the actual ID in place of `<sandbox-id>`:

```bash
render ea sandboxes copy ./main.py <sandbox-id>:/root/main.py
render ea sandboxes exec <sandbox-id> -- bash -c 'cd /root && timeout --kill-after=5 60 python3 main.py'
render ea sandboxes copy <sandbox-id>:/root/result.txt ./result.txt
render ea sandboxes stop <sandbox-id> --confirm -o json
render ea sandboxes list -o json
```

Copy only the inputs and outputs required for the task. Skip the output-copy step if the script does not create that file. Inspect the command output and downloaded artifact; confirm termination for the same ID. If a step fails, still attempt cleanup and report any cleanup error. When automating this sequence, put cleanup in a `finally` block or shell trap after the ID is known.

The `--` separates Render flags from the executable and its arguments. Use `bash -c` for shell expressions. Do not pass an entire shell command as a single executable name.

## Network restrictions

CLI 2.28.0 advertises `allow-all` and `deny-all` for `--network-policy`, but no domain allow-list flag. If the user requires specific allowed domains, use an MCP server that exposes `network: "allow-list"`, or report the capability gap. Do not substitute `allow-all`.

## Snapshots

Inspect `render ea sandboxes snapshots create --help` and `render ea sandboxes snapshots list --help`. Save the returned snapshot ID, wait until it is available, and restore with `render ea sandboxes create --snapshot-id <snapshot-id>`. Verify the expected file in the restored sandbox and clean up task-created sandboxes. Keep the snapshot when the user asked for reusable setup.
