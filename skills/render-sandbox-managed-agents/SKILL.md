---
name: render-sandbox-managed-agents
description: Sets up Claude Managed Agents to run tool calls in Render Sandboxes, as a self-hosted sandbox environment. Covers creating the self-hosted environment, deploying the orchestrator to Render with a Blueprint, building the snapshot, locking sandbox egress, sending a first session, and troubleshooting. Use when the user wants Claude Managed Agents, Anthropic's agent API, or a self-hosted agent sandbox to run on Render.
license: MIT
compatibility: Requires a Render workspace with Sandboxes enabled, Claude Managed Agents access, and the Render CLI 2.28.0+. The environment key is created in the Claude Console.
metadata:
  author: Render
  version: "1.0.0"
  category: sandboxes
---

# Claude Managed Agents on Render Sandboxes

Claude Managed Agents can run an agent's tool calls in your own infrastructure.
On Render, an orchestrator on a background worker gives every session its own Render sandbox.

## How it works

1. Anthropic runs Claude and queues sessions that target a self-hosted environment.
2. The orchestrator claims each session and creates a Render sandbox for it.
3. Inside the sandbox, Anthropic's `ant beta:worker run` executes the tool calls.
4. The sandbox is deleted after the session has been idle for a minute.

The sandbox receives only that session's one-time secret. The long-lived environment key stays on the orchestrator.

## Steps

Do the parts you can, and tell the user clearly which parts only they can do.

1. **Check prerequisites.** `render ea sandbox-groups list` should print one group.
2. **Create the environment (user).** Claude Console > **Workspace > Environments > New > Self-hosted**. Note the `env_...` ID.
3. **Generate the environment key (user).** Open the environment and click **Generate environment key**.
4. **Get the orchestrator code** and run its offline tests.
5. **Deploy** with the repo's `render.yaml` Blueprint. Render asks for `ANTHROPIC_ENVIRONMENT_ID`, `ANTHROPIC_ENVIRONMENT_KEY`, `RENDER_API_KEY`, and `RENDER_WORKSPACE_ID`.
   Never put the user's Claude API key on Render.
6. **Build the snapshot** by triggering the snapshot cron job once.
7. **Send a first session** from the user's machine. The agent should report Debian 12, which proves its tools ran on Render.

## Lock down egress

The worker only needs Anthropic's API, plus GitHub if it downloads `ant` at startup.
Create sandboxes with an `allow-list` network policy:

```json
{"networkPolicy": {"default": "allow-list", "allowedDomains": ["api.anthropic.com", "github.com", "*.githubusercontent.com"]}}
```

With a snapshot that already contains `ant`, `api.anthropic.com` alone is enough.
Add the domains the agent's own tasks need, such as package registries.
Don't use `deny-all`: it blocks Anthropic's API and the worker can't run.

## Rules to keep

- **Keep the Claude API key off Render.** Only the environment key goes on the orchestrator.
- **Don't forward the environment key into sandboxes** unless the worker reports "no sessions token" and the user accepts the risk.

## Troubleshooting

| What you see | What to do |
| --- | --- |
| `401` from Anthropic | Check the environment key and ID, or generate a new key. |
| Sessions stay queued | Check the background worker is running and polling. |
| `the work secret carries no sessions token` | Contact Anthropic support. As a stopgap, forward the environment key. |
| The worker can't reach Anthropic | Check `api.anthropic.com` is in `allowedDomains`. |
| Files are gone on the next turn | The sandbox stopped after going idle. Raise the idle limit for longer conversations. |
