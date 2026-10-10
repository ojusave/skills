# Sandbox SDK

Verified release sources: Python [`client.py`](https://github.com/render-oss/sdk/blob/python/v1.2.0/python/render/experimental/sandbox/client.py), [`types.py`](https://github.com/render-oss/sdk/blob/python/v1.2.0/python/render/experimental/sandbox/types.py), TypeScript [`sandboxes/index.ts`](https://github.com/render-oss/sdk/blob/typescript/v1.2.0/typescript/src/experimental/sandboxes/index.ts), and the [Python lifecycle example](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/python/example/sandbox/main.py). Main was also checked at `4699a1035c5fa4ab0df95b44dc9344368f620dec`; do not assume main's schema is in an installed release.

Before generating code, check the project's actual interpreter/package manager:

```bash
python -m pip show render
python -c 'import render; print(render.__file__)'
npm ls @renderinc/sdk
```

Use Python `render>=1.2.0` (Python 3.10+) or `@renderinc/sdk` 1.2.0+ (Node 18+). Read installed `render/experimental/sandbox/client.py`, `types.py`, and `api.py`, or the TypeScript package's sandbox declarations and generated schema. Fetch the current [Python reference Markdown](https://render.com/docs/sandboxes-sdk-python.md) ([HTML](https://render.com/docs/sandboxes-sdk-python)) or [TypeScript reference Markdown](https://render.com/docs/sandboxes-sdk-typescript.md) ([HTML](https://render.com/docs/sandboxes-sdk-typescript)) when signatures differ.

## Credentials and methods

`RENDER_API_KEY` is a Render API key from [Account Settings](https://dashboard.render.com/u/settings?add-api-key). `RENDER_WORKSPACE_ID` is the `tea-...` ID from workspace Settings. Both belong in the host environment. A plugin OAuth session supplies neither. The clients throw if sandbox operations lack a workspace ID or explicit owner; `create` is not anonymous just because its arguments are optional.

Python async client: `from render import RenderAsync`, then `RenderAsync().experimental.sandboxes`. Python sync client: `from render import Render`, then the same methods without `await`, with ordinary iteration for exec. TypeScript: `import { Render } from "@renderinc/sdk"`, then `new Render().experimental.sandboxes`.

| Operation | Python async | TypeScript |
|---|---|---|
| Create | `await create(...)` | `await create({...})` |
| Read by ID | `await from_id(id)` | `await get(id)` |
| List | `await list(status=[...], cursor=..., limit=...)` | `await list({status: [...], cursor, limit})` |
| Groups | `await list_groups()` | `await listGroups()` |
| Upload | `await copy_to(id, local_path, remote_path)` | `await upload(id, remotePath, data)` |
| Execute | `async for event in exec(id, command)` | `for await (const event of await exec(id, command))` |
| Download | `await copy_from(id, remote_path, local_path)` | `await download(id, remotePath)` |
| Terminate | `await terminate(id)` | `await terminate(id)` |

`owner_id=` / `ownerId` can override the default workspace. TypeScript ID methods take the optional owner ID positionally; create/list/group methods take it in an options object. Use the exact installed signature.

Create fields in the reviewed clients:

| Python keyword | TypeScript property | Type / constraint |
|---|---|---|
| `owner_id` | `ownerId` | Workspace ID, TypeScript template type `tea-${string}` |
| `plan` | `plan` | `starter`, `standard`, `pro` in the reviewed schema |
| `timeout_seconds` | `timeoutSeconds` | Lifetime in seconds; set explicitly because defaults conflict |
| `region` | `region` | String; client default then workspace default when omitted |
| `network_policy` | `networkPolicy` | Python string versus TypeScript policy object; see [network.md](network.md) |
| `env` | `env` | String-to-string map injected into the sandbox |
| `snapshot_id` | `snapshotId` | Available snapshot in the same group |
| `snapshot_name` | `snapshotName` | Mutually exclusive with snapshot ID; verify installed support |

Python returns `Sandbox`, `SandboxList.sandboxes`, and `SandboxGroupList.groups`; pages have `next_cursor`. TypeScript list results contain cursor-bearing wrappers (`sandbox` or `sandboxGroup`). Inspect the wrapper before accessing its ID. For paginated sandbox and snapshot lists, pass Python's `next_cursor` or the last TypeScript wrapper's `cursor` to the next request. Stop on an empty page or absent cursor; detect a repeated cursor and report incomplete traversal instead of looping. A short page can still have a cursor. Terminated sandboxes are excluded by default, so include `terminated` explicitly when reconciling cleanup.

Python `copy_to` accepts a local file or directory and returns no value. `copy_from` writes locally and returns the path written. TypeScript `upload` accepts `Buffer | Uint8Array | string | Readable`, not a local filename; read the local file first. TypeScript `download` returns `{data: Buffer, size, contentType?}`; write `data` locally. Directory transfers use archives. Do not assume the TypeScript client extracts a downloaded archive for you. Python spools the archive before extraction, but directory extraction itself is not atomic and can leave partial contents after failure. Download into a fresh task-owned staging directory, then verify the complete expected file set and contents before promoting it as the result. On failure, preserve existing outputs and retry into another fresh staging directory within a bounded budget; do not merge partial attempts or claim automatic resume. Remove only staging directories created by this task after preserving needed diagnostics.

## Operations outside the high-level SDK

The [public API schema](https://github.com/render-oss/sdk/blob/4699a1035c5fa4ab0df95b44dc9344368f620dec/typescript/src/generated/schema.ts) defines additional operations that are not methods on `experimental.sandboxes` in the reviewed clients:

| Need | Public API definition, relative to `https://api.render.com/v1` |
|---|---|
| Execution history | `GET /sandboxes/{sandboxId}/execs`; supports `ownerId`, `cursor`, and `limit`. |
| One execution record | `GET /sandboxes/{sandboxId}/execs/{execId}`; optional `ownerId`. |
| Sandbox-wide logs and lifecycle events | `GET /sandboxes/{sandboxId}/logs`; schema describes SSE with `since`, `follow`, and `execId`. Availability must be verified. |
| Directory metadata | `GET /sandboxes/{sandboxId}/files/list`; schema requires `path` and accepts `depth`. Availability must be verified. |
| Low-level execution and transfer integration | Run/file connect-token endpoints, plus execution-status reporting. Prefer the SDK's `exec`, upload, and download abstractions for ordinary tasks. |

These are schema definitions, not additional high-level SDK methods or a claim that every deployed endpoint works. Check current documentation and the actual response in the target environment. An exposed MCP tool can provide an operation without a local SDK method; inspect its parameters first. Direct public API requests use the Render API key in the host's authorization header. Never use a private backend endpoint as a fallback.

For live SSE, use a streaming HTTP client and a deadline. The generated Python `stream_sandbox_logs` helper reads `response.text`, so it is not an incremental event iterator. Keep the working `exec` output stream distinct from historical sandbox-wide logs.

## Create, wait, copy, execute, retrieve, terminate

Both examples use an existing local `input.txt` and write `output.txt`. They demonstrate the normal lifecycle, not automatic retry or durable recovery. Use the [failure recovery procedure](lifecycle.md#recover-from-failures) for timeouts, interruptions, and cleanup errors; a create response lost before an ID is returned cannot be handled by these `finally` blocks. The command copies the input inside the sandbox; the host verifies the downloaded bytes. The five-minute lifetime and one-minute readiness deadline are example choices, not platform defaults.

Python:

```python
import asyncio
from pathlib import Path

from render import RenderAsync
from render.experimental.sandbox import SandboxExecExit, SandboxExecOutput


async def main():
    expected = Path("input.txt").read_bytes()
    sandboxes = RenderAsync().experimental.sandboxes
    sandbox = await sandboxes.create(timeout_seconds=300, network_policy="deny-all")
    print(f"sandbox: {sandbox.id}")
    try:
        for _ in range(40):
            status = (await sandboxes.from_id(sandbox.id)).status
            if status == "running":
                break
            if status in ("errored", "terminated"):
                raise RuntimeError(f"Sandbox {sandbox.id} is {status}")
            await asyncio.sleep(1.5)
        else:
            raise TimeoutError(f"Sandbox {sandbox.id} did not become running")

        await sandboxes.copy_to(sandbox.id, "input.txt", "/tmp/input.txt")
        exit_code = None
        async for event in sandboxes.exec(
            sandbox.id, "cat /tmp/input.txt > /tmp/output.txt"
        ):
            if isinstance(event, SandboxExecOutput):
                print(f"[{event.stream}] {event.data}", end="")
            elif isinstance(event, SandboxExecExit):
                exit_code = event.exit_code
        if exit_code != 0:
            raise RuntimeError(f"Command did not succeed: exit={exit_code}")

        written = await sandboxes.copy_from(sandbox.id, "/tmp/output.txt", "output.txt")
        if Path(written).read_bytes() != expected:
            raise RuntimeError("Downloaded output differs from input")
    finally:
        await sandboxes.terminate(sandbox.id)


asyncio.run(main())
```

TypeScript (the `default` policy key is from released SDK 1.2.0; main also accepts it as a deprecated alias):

```typescript
import { readFile, writeFile } from "node:fs/promises";
import { Render } from "@renderinc/sdk";

const expected = await readFile("input.txt");
const sandboxes = new Render().experimental.sandboxes;
const sandbox = await sandboxes.create({
  timeoutSeconds: 300,
  networkPolicy: { default: "deny-all" },
});
console.log(`sandbox: ${sandbox.id}`);
let operationFailed = false;
let operationError: unknown;
try {
  let ready = false;
  for (let attempt = 0; attempt < 40; attempt++) {
    const status = (await sandboxes.get(sandbox.id))?.status;
    if (status === "running") {
      ready = true;
      break;
    }
    if (status === "errored" || status === "terminated") {
      throw new Error(`Sandbox ${sandbox.id} is ${status}`);
    }
    await new Promise((resolve) => setTimeout(resolve, 1500));
  }
  if (!ready) throw new Error(`Sandbox ${sandbox.id} did not become running`);

  await sandboxes.upload(sandbox.id, "/tmp/input.txt", expected);
  let exitCode: number | undefined;
  for await (const event of await sandboxes.exec(
    sandbox.id, "cat /tmp/input.txt > /tmp/output.txt",
  )) {
    if (event.type === "output") {
      process.stdout.write(`[${event.stream}] ${event.data}`);
    } else {
      exitCode = event.exit_code;
    }
  }
  if (exitCode !== 0) throw new Error(`Command did not succeed: exit=${exitCode}`);

  const downloaded = await sandboxes.download(sandbox.id, "/tmp/output.txt");
  await writeFile("output.txt", downloaded.data);
  if (!downloaded.data.equals(expected)) throw new Error("Downloaded output differs from input");
} catch (error) {
  operationFailed = true;
  operationError = error;
  throw error;
} finally {
  try {
    await sandboxes.terminate(sandbox.id);
  } catch (cleanupError) {
    if (operationFailed) {
      throw new AggregateError(
        [operationError, cleanupError],
        `Sandbox ${sandbox.id}: operation and cleanup both failed`,
      );
    }
    throw new Error(`Cleanup failed for sandbox ${sandbox.id}`, { cause: cleanupError });
  }
}
```

The SDK command string runs through `bash -c`. Nonzero exits arrive as `SandboxExecExit` in Python or `{type: "exit", exit_code}` in TypeScript; they are not automatically thrown exceptions. Stream/transport errors can throw. Consume the terminal event and check it explicitly.

After termination, query the sandbox state using the saved ID. If cleanup fails, retain that ID and the original operation error, then follow the bounded recovery procedure before reporting final status. Aborting a TypeScript stream only cancels the client request in the reviewed source; it does not call `terminate`. Do not describe stream cancellation as verified remote process termination.

For persistence, insert the required [snapshot](snapshots.md) before termination and wait for availability. The upstream Python example demonstrates file transfer and cleanup but does not wait for readiness; the examples here add that check from the current docs.
