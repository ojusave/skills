---
name: render-workflows
description: Build, validate locally, deploy, and troubleshoot Render Workflows with the current Python or TypeScript SDK. Use for defining or chaining tasks, integrating the SDK into an existing project, running tasks with the local development server, optionally scaffolding a starter, triggering runs from application code, and releasing workflows.
license: MIT
compatibility: Requires Render SDK 1.x. Local validation requires Render CLI 2.12.0+; optional scaffolding and CLI deployment require Render CLI 2.16.0+.
metadata:
  author: Render
  version: "1.1.2"
  category: workflows
---

# Render Workflows

Render Workflows is an SDK-based product for orchestrating long-running distributed tasks. A workflow service registers Python or TypeScript functions as tasks; each task run executes independently and can chain additional runs.

Use Render SDK 1.x APIs (`render>=1.0.1` for Python or `@renderinc/sdk` version 1.x for TypeScript). Local validation requires Render CLI 2.12.0 or later. Use CLI 2.16.0 or later when using the complete optional scaffolding and CLI deployment flow. Prefer the latest compatible releases.

Render Workflows and its SDKs can introduce breaking changes. Treat the installed SDK version as authoritative for an existing project. For new projects, use the current official documentation and starter templates.

## Check Current Sources

Before generating or changing SDK code, identify the installed versions:

```bash
render --version
python -m pip show render
npm ls @renderinc/sdk
```

Use the source that matches the question:

- [Python SDK reference](https://render.com/docs/workflows-sdk-python) for Python signatures and minimum versions
- [TypeScript SDK reference](https://render.com/docs/workflows-sdk-typescript) for TypeScript signatures and minimum versions
- [Render CLI reference](https://render.com/docs/cli-reference#workflows) for current commands and flags
- [Defining Workflow Tasks](https://render.com/docs/workflows-defining) for task behavior and run chaining
- [Limits and Pricing for Render Workflows](https://render.com/docs/workflows-limits) for compute plans, quotas, retention, and pricing
- [Python examples](https://github.com/render-examples/render-workflows-examples-python) and [TypeScript examples](https://github.com/render-examples/render-workflows-examples-ts) for current scaffolding patterns
- [Render SDK repository](https://github.com/render-oss/sdk) for the implementation and changelogs

Do not silently upgrade an existing project's SDK major version. If an upgrade is part of the request, read the relevant SDK changelog and migrate all task definitions and callers together.

## SDK 1.x Invariants

Preserve these rules in generated code:

- Python installs and imports the `render` package, not `render_sdk`.
- Every task function accepts a `TaskContext` as its first positional parameter. Render supplies it; callers do not include it in task input.
- Chain a task run with `await ctx.run(task_definition, ...args)`. A task definition is not directly callable.
- All tasks in a `ctx.run` chain belong to the same workflow service. Trigger another workflow through the SDK client or Render API.
- Python calls `app.start()` from its workflow entrypoint. TypeScript task registration auto-starts in the workflow environment.
- Task arguments and return values must be JSON-serializable.

Read [references/quick-reference.md](references/quick-reference.md) when writing SDK code. Read [references/task-patterns.md](references/task-patterns.md) for chaining, fan-out, retries, scheduled triggers, or cross-workflow calls.

## Choose a Starting Point

Do not require `render workflows init`. In an existing codebase, add the SDK and task definitions directly, preserve the project's dependency and build conventions, and validate them with the local development server. For a minimal project or direct integration guidance, read [references/manual-scaffolding.md](references/manual-scaffolding.md).

### Optional starter scaffolding

Use the CLI scaffolder when the user wants a quick example project or a new standalone workflow service:

```bash
render workflows init
```

For non-interactive setup, pass the language and destination explicitly:

```bash
render workflows init --confirm --language python --dir workflows --git=false
render workflows init --confirm --language node --dir workflows --git=false
```

Use `--git=false` when scaffolding inside an existing Git repository to avoid creating a nested repository. Other useful options include `--template`, `--install-deps`, and `--install-agent-skill`; verify current behavior in the CLI reference.

## Define and Validate Tasks Locally

After adding or changing SDK task definitions, use the local development server as the primary validation loop. Start it with the workflow's actual start command:

```bash
render workflows dev -- <workflow-start-command>
```

In another terminal, list and run tasks against the local server:

```bash
render workflows tasks list --local
render workflows start <task-name> --local --input='[]' -o json
render workflows tasks runs show <task-run-id> --local -o json
```

Use a JSON array for positional input. Python tasks can also receive a JSON object for named input. Read [references/local-development.md](references/local-development.md) for environment files, custom ports, run inspection, cancellation, and application-client configuration.

In non-interactive mode, `workflows start` returns the new task run ID before the run necessarily finishes. Use that ID with `tasks runs show`, polling for a bounded period until the run completes, fails, or is canceled. Verify the result or error, not only that the run was created.

When safe and practical, run this verification yourself and stop the dev server afterward. Do not claim a task registers successfully based only on source inspection.

## Deploy After Local Validation

Creating or releasing a workflow changes the user's Render account. Only do it when deployment is part of the request and the tasks have been validated locally when local execution is supported.

Before deploying, authenticate and confirm the target workspace:

```bash
render whoami
render workspace current
```

If authentication is missing, use `render login` for an interactive local session or provide `RENDER_API_KEY` through the environment for automation. Never print the key.

Confirm the intended branch and region, and ensure the exact code to deploy is committed and pushed to GitHub, GitLab, or Bitbucket. `--repo .` resolves the repository's remote; it does not deploy uncommitted local changes.

Create the workflow with the CLI. Interactive mode prompts for configuration:

```bash
render workflows create
```

For a non-interactive deployment, use the build and run commands from the selected starter template or the project's existing configuration:

```bash
render workflows create \
  --name my-workflow \
  --repo . \
  --branch <branch> \
  --region <region> \
  --runtime python \
  --build-command "pip install -r requirements.txt" \
  --run-command "python main.py"
```

For a workflow in a subdirectory, add `--root-directory`. Environment variables can be supplied with repeatable `--env-file` and `--env-var` flags. Never expose secrets in commands or output.

The Render Dashboard is also a supported creation path and is currently required for Docker-based workflow creation. Before recommending a Blueprint, check the current [Workflows limitations](https://render.com/docs/workflows#faq) and [Blueprint specification](https://render.com/docs/blueprint-spec); support is evolving.

For later explicit releases, use:

```bash
render workflows versions release <workflow-id> \
  --commit <pushed-commit-sha> \
  --wait
```

Do not report deployment success from command acceptance alone. Wait for the release, then verify that the expected tasks registered and that one safe task reaches a terminal state with the expected result:

```bash
render workflows versions list <workflow-id> -o text
render workflows tasks list <ready-workflow-version-id> -o text
render workflows start <workflow-slug>/<task-name> --input='[]' -o json
render workflows tasks runs show <task-run-id> -o json
```

Poll `tasks runs show` for a bounded period if the run is still queued or running. Stop and report the release or task error instead of retrying an external mutation indefinitely.

If deployment fails, read [references/troubleshooting.md](references/troubleshooting.md). Use the `render-debug` skill for broader Render deployment diagnosis when it is available.

## Trigger Deployed Tasks

Task slugs use `{workflow-slug}/{task-name}`.

Python, synchronous:

```python
from render import Render

render = Render()
result = render.workflows.run_task("my-workflow/hello", ["world"])
print(result.results)
```

Python, asynchronous:

```python
from render import RenderAsync

render = RenderAsync()
started = await render.workflows.start_task("my-workflow/hello", ["world"])
finished = await started
print(finished.results)
```

TypeScript:

```typescript
import { Render } from "@renderinc/sdk";

const render = new Render();
const started = await render.workflows.startTask("my-workflow/hello", ["world"]);
const finished = await started.get();
console.log(finished.results);
```

These clients use `RENDER_API_KEY` unless a token is passed explicitly. Prefer environment variables or a secret manager over hard-coded credentials.

Workflows do not currently provide native scheduled triggers. Use a Render cron job that invokes the SDK client or API when scheduling is required.

## Limits, Compute Plans, and Pricing

Do not hard-code compute-plan IDs, resource specifications, quotas, retention periods, or prices in this skill or its references.

Before setting a task's `plan`, estimating cost, or advising on capacity, consult [Limits and Pricing for Render Workflows](https://render.com/docs/workflows-limits). Keep only API-shape constraints needed to write correct code in the skill.

## References

- [references/quick-reference.md](references/quick-reference.md): SDK 1.x task and client surface
- [references/task-patterns.md](references/task-patterns.md): chaining, fan-out, retries, cron triggers, and cross-workflow calls
- [references/local-development.md](references/local-development.md): local server, CLI runs, environment configuration, and limitations
- [references/troubleshooting.md](references/troubleshooting.md): common setup, registration, execution, and client failures
- [references/manual-scaffolding.md](references/manual-scaffolding.md): direct SDK project setup without `workflows init`

## Related Skills

- `render-deploy`: deploy other Render service types
- `render-debug`: diagnose deployment and runtime failures
- `render-monitor`: inspect service health and performance
