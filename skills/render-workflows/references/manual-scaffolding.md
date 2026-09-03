# Direct SDK Project Setup

Use this reference to add the Render Workflows SDK directly to an existing codebase or create a minimal workflow service without `render workflows init`. Preserve an existing project's dependency, module, and build conventions. For a new standalone service, prefer adapting the current official [Python examples](https://github.com/render-examples/render-workflows-examples-python) or [TypeScript examples](https://github.com/render-examples/render-workflows-examples-ts) over copying this minimal structure blindly.

## Choose the Language and Boundary

Use the language requested by the user. Otherwise infer it from the project:

| Indicators | Language |
|---|---|
| `pyproject.toml`, `requirements.txt`, `Pipfile`, or Python source | Python |
| `package.json`, `tsconfig.json`, or TypeScript source | TypeScript |

If both ecosystems are present, prefer the language already used by the component that will trigger or own the workflow. Ask only when the choice materially changes the integration.

Create a self-contained workflow service directory when the workflow has an independent build or deployment boundary. Do not overwrite unrelated root dependency files.

## Python

Minimum files:

```text
workflows/
|-- main.py
`-- requirements.txt
```

`requirements.txt`:

```text
render>=1.0.1
```

`main.py`:

```python
from render import TaskContext, Workflows

app = Workflows()

@app.task
def ping(_ctx: TaskContext) -> str:
    return "pong"

if __name__ == "__main__":
    app.start()
```

Install dependencies in an isolated environment:

```bash
cd workflows
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Use an equivalent activation command on Windows or non-POSIX shells.

## TypeScript

Minimum files:

```text
workflows/
|-- src/
|   `-- main.ts
|-- package.json
`-- tsconfig.json
```

`package.json`:

```json
{
  "name": "my-workflow",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "tsx src/main.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@renderinc/sdk": "^1.0.0"
  },
  "devDependencies": {
    "tsx": "^4.20.2",
    "typescript": "^5.0.0"
  }
}
```

`src/main.ts`:

```typescript
import { task, type TaskContext } from "@renderinc/sdk/workflows";

task(
  { name: "ping" },
  function ping(_ctx: TaskContext): string {
    return "pong";
  },
);
```

Install dependencies:

```bash
cd workflows
npm install
```

Use the project's existing TypeScript configuration when integrating into an established service. For a standalone service, choose module and compiler settings compatible with the current Node runtime and verify with `npm run typecheck`.

## Verify

From the workflow service directory, start the local task server:

```bash
# Python
render workflows dev -- .venv/bin/python main.py

# TypeScript
render workflows dev -- npm start
```

In another terminal:

```bash
render workflows tasks list --local -o text
render workflows start ping --local --input='[]' -o json
render workflows tasks runs show <task-run-id> --local -o json
```

Copy the task run ID returned by `workflows start`. If the run is still queued or running, poll `tasks runs show` for a bounded period. Verify that the completed result contains `"pong"`; run creation alone is not sufficient. If an agent starts the server, it should capture the result and stop the server before handing the workspace back.

For deployment, use the workflow directory as the service's root directory and preserve the build/start commands validated locally. The current CLI can generate a `render workflows create` command from `workflows init`; when scaffolding manually, construct that command from the project's actual configuration.
