---
name: render-sandboxes
description: Runs code and shell commands in isolated Render Sandboxes (Linux microVMs) instead of on the user's machine. Use when the user wants to run untrusted or generated code, try a package or script on a clean Linux box, reproduce a bug in a fresh environment, run something that needs root or system packages, lock code to specific network domains, or keep a disposable environment they can snapshot and restore. Also use when the user mentions Render Sandboxes, sbx- IDs, or sandbox snapshots.
license: MIT
compatibility: Uses the Render MCP server's sandbox tools (mcp.render.com, signed in with Render). Falls back to Render CLI 2.28.0+ (`render ea sandboxes`). Requires a workspace with Render Sandboxes enabled.
metadata:
  author: Render
  version: "1.0.0"
  category: sandboxes
---

# Render Sandboxes

A Render sandbox is a fresh Linux microVM that starts in a few seconds and is deleted when you're done.
Nothing run there can touch the user's machine or their Render services.

**What you get:** Debian 12 on x86_64, 2 CPUs, 4 GB memory, 10 GB disk, running as root.
`bash`, `curl`, `git`, `jq`, `python3`, and `node` are preinstalled.

**Render Sandboxes are in early access.** Limits and APIs may change.
Docs: [render.com/docs/sandboxes](https://render.com/docs/sandboxes)

## When to use a sandbox

Use one when running the code locally would be risky, messy, or impossible:

- The code is untrusted, generated, or from a stranger.
- It needs `apt-get`, root, or a different OS than the user's.
- The user wants a clean environment to reproduce a bug or test an install.
- The job is long or heavy and shouldn't tie up the user's machine.

Don't use one for quick edits to the user's own project files.

## Tools

These tools come from the Render MCP server. The user signs in with their Render account; no API key is needed.

| Tool | What it does |
| --- | --- |
| `run_in_new_sandbox` | One step: create, write files, run one command, return output, terminate. Best for quick jobs. |
| `create_sandbox` | Create a sandbox and wait until it's ready. Returns an `sbx-...` ID. |
| `run_sandbox_command` | Run a bash command. Returns stdout, stderr, exit code, and whether it timed out. |
| `write_sandbox_file` / `read_sandbox_file` | Move text files in and out. |
| `list_sandbox_files` | List a directory. |
| `snapshot_sandbox` / `list_sandbox_snapshots` | Save a sandbox's filesystem and start new sandboxes from it. |
| `list_sandboxes` | See what's running in the workspace. |
| `terminate_sandbox` | Delete a sandbox and its files. |

If the user has several workspaces, confirm which one with `list_workspaces` before creating sandboxes.

## How to work

1. **Pick the shape of the job.** For one command, call `run_in_new_sandbox`. For several steps, call `create_sandbox` once and reuse the ID.
2. **Run commands.** Every `run_sandbox_command` starts a new shell in `/root`, so `cd` doesn't carry over. Chain steps with `&&` or use absolute paths.
3. **Read results carefully.** A non-zero exit code is a normal result. Show the user the relevant stderr.
4. **Clean up.** Call `terminate_sandbox` when the work is done, and tell the user the sandbox ID.
   Sandboxes also stop when their lifetime runs out. The default is 30 minutes.

## Network access

Pick the narrowest setting that works:

| `network` | Use it when |
| --- | --- |
| `deny-all` | Running untrusted code that needs no internet. |
| `allow-list` | The code needs a few known services. Pass them in `allowedDomains`, for example `["pypi.org", "*.pythonhosted.org"]`. Everything else is blocked. |
| `allow-all` | The default. Use for general work when the user trusts the code. |

Domain matching is exact, and a leading `*.` matches subdomains. Only HTTP and HTTPS can reach allowed domains.

## Good habits

- **Keep secrets out.** Don't put the user's credentials in a sandbox unless they ask.
- **Set timeouts.** Commands stop after 120 seconds by default. Raise `timeoutSeconds` (up to 600) for builds and installs.
- **Install Python packages in a virtualenv.** Debian 12 blocks `pip install` into the system Python.
  Use `python3 -m venv /root/venv && /root/venv/bin/pip install ...`.
- **Reuse setup with snapshots.** After a slow setup, call `snapshot_sandbox` with a name, then pass that name to `create_sandbox`. Snapshots expire after three days.
- **Big outputs get truncated** to about 20,000 characters. Write large results to a file and read the part you need.

## Limits during early access

| Limit | Value |
| --- | --- |
| Resources per sandbox | 2 CPUs, 4 GB memory, 10 GB disk |
| Running sandboxes per workspace | 100 |
| Longest lifetime | 24 hours |
| API requests | 100 per minute |
| Region | Oregon |

## Without the MCP server

The Render CLI does the same things:

```bash
render workspace set tea-...                      # pick the workspace once
render ea sandboxes create --timeout 1800         # prints the sbx-... ID; wait until it's running
render ea sandboxes list --status running
render ea sandboxes exec sbx-... -- bash -c 'python3 --version'
render ea sandboxes copy ./local.txt sbx-...:/root/local.txt
render ea sandboxes stop sbx-... --confirm
```

Pass commands to `exec` through `bash -c '...'`. Otherwise the CLI treats the whole string as a program name.

## Troubleshooting

| What you see | What to do |
| --- | --- |
| "Render Sandboxes are not enabled for this workspace" | The workspace needs early access. Check **+ New > Sandbox Group** in the Dashboard. |
| `429` or rate limit errors | Too many sandboxes or requests. Terminate unused sandboxes and wait a minute. |
| A command returns `timedOut: true` | Raise `timeoutSeconds`, or run it in the background and poll a log file. |
| A download fails under `allow-list` | Add the host to `allowedDomains`. Package managers often use a CDN host as well. |
| Files from an earlier session are gone | Sandboxes are ephemeral. Use a snapshot to keep a filesystem. |
