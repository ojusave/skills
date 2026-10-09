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

Python returns `Sandbox`, `SandboxList.sandboxes`, and `SandboxGroupList.groups`; pages have `next_cursor`. TypeScript list results contain cursor-bearing wrappers (`sandbox` or `sandboxGroup`). Inspect the wrapper before accessing its ID.

Python `copy_to` accepts a local file or directory and returns no value. `copy_from` writes locally and returns the path written. TypeScript `upload` accepts `Buffer | Uint8Array | string | Readable`, not a local filename; read the local file first. TypeScript `download` returns `{data: Buffer, size, contentType?}`; write `data` locally. Directory transfers use archives. Do not assume the TypeScript client extracts a downloaded archive for you.

## Create, wait, copy, execute, retrieve, terminate

Both examples use an existing local `input.txt` and write `output.txt`. The command copies the input inside the sandbox; the host verifies the downloaded bytes. The five-minute lifetime and one-minute readiness deadline are example choices, not platform defaults.

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
} finally {
  await sandboxes.terminate(sandbox.id);
}
```

The SDK command string runs through `bash -c`. Nonzero exits arrive as `SandboxExecExit` in Python or `{type: "exit", exit_code}` in TypeScript; they are not automatically thrown exceptions. Stream/transport errors can throw. Consume the terminal event and check it explicitly.

After termination, query the sandbox state using the saved ID. If cleanup fails, retain that ID and report the failure. Aborting a TypeScript stream only cancels the client request in the reviewed source; it does not call `terminate`. Do not describe stream cancellation as verified remote process termination.

For persistence, insert the required [snapshot](snapshots.md) before termination and wait for availability. The upstream Python example demonstrates file transfer and cleanup but does not wait for readiness; the examples here add that check from the current docs.
