# Sandbox CLI

Baseline: installed Render CLI 2.28.0 and its [release source](https://github.com/render-oss/cli/tree/v2.28.0). Also reviewed [main at c4d106c](https://github.com/render-oss/cli/tree/c4d106cfdaf8eec6d23475155e43f4baeeed2013). Check help for the binary being used. Fetch the current [CLI reference Markdown](https://render.com/docs/sandboxes-cli-reference.md) ([HTML](https://render.com/docs/sandboxes-cli-reference)) for changes.

## Access and command discovery

Use `render-cli` for installation and authentication. `RENDER_API_KEY` authenticates automation; interactive CLI login is a separate option. The plugin's OAuth connection does not authenticate your SDK process.

```bash
render --version
render ea sandboxes --help
render ea sandboxes create --help
render workspace current --output json
```

Select the intended workspace with `render workspace set tea-...`. The CLI also accepts `RENDER_WORKSPACE=tea-...` in its process environment. The SDK's `RENDER_WORKSPACE_ID` does not select the CLI workspace. Do not print credentials or pass the Render key into sandbox environment variables.

## Command surface

All sandbox commands use the `ea` prefix. Use non-interactive `--output json`, `yaml`, or `text` when an agent parses results.

| Command | Behavior and flags |
|---|---|
| `render ea sandbox-groups list` | Lists groups in the active workspace. No create-group command was found. |
| `render ea sandboxes create` | Creates asynchronously; flags below. |
| `render ea sandboxes list` | Excludes terminated by default. `--all` defaults to false; repeat `--status` to filter. |
| `render ea sandboxes exec <sandboxId> -- <command>` | Streams stdout/stderr and exits with the remote command's exit code. |
| `render ea sandboxes copy <src> <dst>` | Copies a file/directory; `cp` is an alias. Exactly one endpoint must be local. |
| `render ea sandboxes stop <sandboxId>` | Preview only unless `--confirm` is passed. |
| `render ea sandboxes snapshots create <sandboxId>` | `--kind filesystem` or `runtime`; omitted kind defaults to filesystem at the API. |
| `render ea sandboxes snapshots get <snapshotId>` | Uses default group unless `--group` is supplied. |
| `render ea sandboxes snapshots list` | `--group` and repeatable `--status`; newest first, omits expired/deleted. |
| `render ea sandboxes snapshots delete <snapshotId>` | `--group`; preview only unless `--confirm` is passed. |

These commands are defined in `cmd/sandbox.go`, `cmd/sandboxgroups*.go`, `cmd/sandboxcreate.go`, `cmd/sandboxlist.go`, `cmd/sandboxexec.go`, `cmd/sandboxcopy.go`, `cmd/sandboxstop.go`, and `cmd/sandboxsnapshots*.go` in the linked source. `--confirm` is an execution flag, not authorization for unrelated changes.

## Create flags and defaults

| Flag | CLI 2.28.0 default and behavior |
|---|---|
| `--plan` | Empty, omitted from request. Reviewed schema values: `starter`, `standard`, `pro`. |
| `--region` | Empty, omitted from request; API chooses workspace default. The sandbox group fixes the region. |
| `--timeout` | `0`; help says this uses the default and maximum of 86400 seconds. Positive values set lifetime; negative values fail validation. |
| `--network-policy` | Empty, omitted from request. Help lists `allow-all`, `deny-all`; there is no destination-list flag. |
| `--env-var` | No entries. Repeat `KEY=VALUE`; inline entries override env files. |
| `--env-file` | No files. Later files override earlier ones; every named file must exist. |
| `--snapshot-id` | Empty, starts from the base image. Otherwise snapshot must be available in the same group; runtime restores require matching `--plan`. |

Do not infer resource sizes or prices from plan names. Fetch the [current Sandboxes docs](https://render.com/docs/sandboxes.md) and run `render ea sandboxes create --help` for the installed CLI.

**Lifetime conflict:** `cmd/sandboxcreate.go` says default/max 86400; SDK Python `python/render/experimental/sandbox/client.py` and TypeScript `SandboxCreateInput` comments say 7200. Released TypeScript 1.2.0 schema says 7200; SDK main schema says 86400. These are distinct source claims. Do not choose a universal default. Read installed help and use an explicit lifetime, such as the five-minute example below.

**Newer source is not the release:** main adds create `--from` (snapshot ID or name) and snapshot create `--name`. Neither appears in installed CLI 2.28.0 help. Use them only if installed help confirms support; `--from` and `--snapshot-id` are mutually exclusive.

## One operating path

Replace `tea-...` and `sbx-...` with actual IDs. The example copies an existing local `main.py` and expects that script to write `/tmp/output.json`.

```bash
render workspace set tea-...
render ea sandboxes create --timeout 300 --network-policy deny-all --output json
# Save the returned ID. Repeat with a bounded wait until this ID is running.
render ea sandboxes list --all --output json
# Only after the target ID is running:
render ea sandboxes copy ./main.py sbx-...:/tmp/main.py
render ea sandboxes exec sbx-... -- python3 /tmp/main.py
render ea sandboxes copy sbx-...:/tmp/output.json ./output.json
# After preserving wanted files, including on a failed run:
render ea sandboxes stop sbx-... --confirm
render ea sandboxes list --status terminated --output json
```

Inspect the downloaded result, the command exit code, and the target ID's cleanup status. Stop polling on terminal failure; after a timeout or lost response, inspect the existing ID before creating another sandbox.

`exec` reconstructs a shell-quoted command from argv. Use `-- echo hello` for a simple command. For shell operators, pass an explicit shell, for example `-- bash -c 'printf hello > /tmp/output.txt'`. Passing one quoted string containing the whole simple command can instead quote it as one executable token.

Remote copy paths use `sbx-ID:path`: relative paths resolve under the sandbox's home directory; absolute paths address its filesystem. A directory is transferred as an archive to the requested path, without an extra nested basename. Sandbox-to-sandbox copy is not supported.
