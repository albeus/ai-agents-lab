# AI Agents Lab

Small, hands-on projects for understanding how AI agents use tools and how
those tools can be exposed, governed and observed. I built these exercises
while preparing for AGNTCon + MCPCon Europe 2026, with an
infrastructure/DevOps focus: explicit boundaries, testable access controls,
useful diagnostics and honest failure reporting.

This is a **learning lab**, not a production reference architecture or a set
of deployable infrastructure services. Each directory can be explored
separately; the projects are related but do not form a single, fully
integrated agent platform.

## Projects

| Directory | Purpose | Start here |
| --- | --- | --- |
| [`miniagent/`](https://github.com/albeus/ai-agents-miniagent) | A small Python agent harness using a local Ollama model. Makes the model → tool request → validation → execution → result loop visible. | Run it against non-sensitive local notes and inspect which requests its three tools accept or reject. |
| [`mcp-itsops/`](https://github.com/albeus/ai-agents-mcp-itsops) | A narrow MCP server offering read-only-style operations tools and a lab-only timeout probe. Supports stdio and Streamable HTTP. | Discover and call its tools with MCP Inspector; follow the workflow for the selected transport. |
| [`litellm/`](https://github.com/albeus/ai-agents-litellm-gateway) | A LiteLLM Proxy and PostgreSQL lab for routing a local model and registering the MCP server, with separate ITS Operations and Service Desk identities. | Read the results and known issue before relying on any access-control behaviour. |
| [`trace_probe/`](https://github.com/albeus/ai-agents-trace-probe) | A standalone observability experiment correlating a model call and MCP calls with one trace ID, including a deliberately timed-out tool call. | Read the example JSONL trace and reproduce the failure in an isolated lab. |

## How the pieces fit

```text
miniagent ────────────────────────────────→ Ollama
   │  local agent loop and its own tools
   └── not currently an MCP client in this lab

MCP Inspector / other MCP client ─────────→ mcp-itsops
                                            stdio (client launches process)
                                            or Streamable HTTP (/mcp)

Client ───────────────────────────────────→ LiteLLM Proxy ─→ Ollama
                                                 │
                                                 └──────────→ mcp-itsops
                                                             (HTTP lab setup)

trace_probe ──────────────────────────────→ LiteLLM Proxy (model call)
           └──────────────────────────────→ mcp-itsops (direct MCP calls)
           └── writes correlated local JSONL events
```

The key distinction: `miniagent` owns an agent loop and executes its own local
tools; `mcp-itsops` publishes tools for MCP clients. LiteLLM is a separate
gateway experiment, and `trace_probe` is a scripted probe rather than a full
agent or distributed-tracing stack. The probe calls the MCP server directly;
it does not establish end-to-end tracing through LiteLLM's MCP gateway.

## Suggested learning path

1. Start with [`miniagent/`](miniagent/README.md) to see the model propose a tool call and the Python harness decide whether to execute it.
2. Move to [`mcp-itsops/`](mcp-itsops/README.md) to expose similarly narrow capabilities through MCP and inspect tool discovery and invocation.
3. Read and, if appropriate, reproduce [`litellm/`](litellm/README.md) to explore model routing, team keys and the difference between registering a tool server and proving its access policy.
4. Use [`trace_probe/`](trace_probe/README.md) to see what a shared trace ID can reveal about model latency, MCP calls and a client-side timeout.

Each subproject README contains its own prerequisites, setup steps, test
commands and limitations. There is no root-level installer or one-command
deployment.

## MCP transport quick reference

Run the following from the MCP server repository root after `uv sync`. All
server commands use the module invocation `uv run python -m mcp_itsops.server`.

### Inspector over stdio

The default server transport is stdio. Let Inspector launch the process:

```bash
MCP_TRANSPORT=stdio \
npx @modelcontextprotocol/inspector \
  uv run python -m mcp_itsops.server
```

Do not start a separate stdio server first. Inspector discovery and calls do
not require a model; use runbook search or the lab-only `slow_probe` if you
want a check without model or cluster dependencies.

### Inspector over HTTP

Start the server in one terminal:

```bash
MCP_TRANSPORT=streamable-http \
MCP_HOST=127.0.0.1 \
MCP_PORT=8000 \
uv run python -m mcp_itsops.server
```

Start Inspector in another terminal:

```bash
npx @modelcontextprotocol/inspector
```

Select **Streamable HTTP** and use `http://127.0.0.1:8000/mcp`. Use the proxy
connection mode if offered. The server runs independently of Inspector in
this workflow.

### LiteLLM container upstream

Use the same HTTP mode with `MCP_HOST=0.0.0.0` for the documented container
setup. LiteLLM's upstream URL is `http://host.docker.internal:8000/mcp`, with
`transport: "http"` in its configuration. The wider bind is not loopback-only:
restrict it with the host firewall. See the subproject READMEs for full steps.

## What has and has not been shown

| Area | Observed in the lab | Not established |
| --- | --- | --- |
| Agent loop | `miniagent` can ask a local model for tool calls and execute constrained local tools. | Production-grade command isolation, hard run-level time limits, multi-user security or an MCP-connected agent. |
| MCP | `mcp-itsops` exposes a small, constrained tool surface. Its transport switch was confirmed working during review on 4 October 2026; the README now documents stdio and HTTP workflows. | Production authentication, audit, sandboxing or safe access to real infrastructure. Client/server version compatibility still needs checking in each environment. |
| Gateway | Both team identities could reach the configured model; the LiteLLM master key could list the registered MCP tools. | Per-team MCP access was **not** proven. In the recorded test, grants were not persisted as expected and non-master keys received 403 responses. Do not treat the intended ITS Operations/Service Desk tool split as enforced. |
| Observability | One local JSONL trace correlated a model call, MCP discovery and a deliberately timed-out MCP tool call. | Automatic, cross-service distributed tracing or proof of server-side cancellation when the client times out. |

These are results from the documented experiments, not guarantees about
another installation or a later software version. Updating the transport
instructions does not establish a new gateway authorisation result.

## Safety before publishing or running

- **Inspect the repository for secrets before pushing it to GitHub.** The earlier review flagged files named `*key*.json`, `*team*.json` and tool-response text under `litellm/`. Some may contain live bearer tokens, identifiers or sensitive responses. Do not assume they are harmless from their filenames; check the current tree.
- Do not commit `.env`, generated keys, real credentials, kubeconfigs, confidential runbooks, trace logs or database volumes. Add appropriate `.gitignore` rules and sanitised examples instead. If a secret has already been committed or shared, revoke/rotate it and address the repository history; deleting the working-tree copy is not enough.
- `miniagent` currently uses `shell=True` with text-based checks. Its prefix filtering and metacharacter rejection are learning controls, **not** a robust command-execution boundary. Use only in a disposable, non-sensitive environment.
- `mcp-itsops` is designed for narrow inputs, but its server process still has the permissions of the local user and any configured Kubernetes credentials. Use lab-only, least-privilege credentials and non-sensitive data.
- In the gateway experiment, making Ollama or the MCP server reachable from a container may require binding to `0.0.0.0`. That does **not** by itself restrict access to loopback. Check host firewall/network exposure; do not expose these unauthenticated lab endpoints to an untrusted network.
- The trace probe deliberately creates a slow operation and a client timeout. A client timeout does not imply that the server stopped its work. Avoid write-capable or consequential tools in this experiment.

## Next improvements

- Replace `miniagent`'s shell-string execution with fixed executable/argument lists, and verify HTTP and run-level timeouts alongside its existing iteration and repeated-call guards.
- Add automated tests for allowed and denied MCP tool inputs and for both transport startup modes.
- Re-test LiteLLM MCP permissions against a pinned version, verifying stored grants and effective behaviour for **both** teams before claiming policy enforcement.
- Add trace context at the agent, gateway and MCP layers if pursuing genuine end-to-end observability.

The subproject READMEs are the source of truth for detailed setup and the
recorded outcomes of each exercise.
