# AI Agents Lab

Small, hands-on projects exploring how AI agents use tools, how those tools can be exposed through MCP, and how gateway behaviour and failures can be tested and observed. I built these exercises while preparing for AGNTCon + MCPCon Europe 2026, with an infrastructure/DevOps focus: explicit boundaries, testable controls, useful diagnostics and honest failure reporting.

> **Learning labs, not a production reference architecture.** These are four independent repositories, not one integrated agent platform. The recorded outcomes describe specific experiments; they are not guarantees about other installations or software versions.

## Projects

| Repository | What it explores | Start here |
| --- | --- | --- |
| [`ai-agents-miniagent`](https://github.com/albeus/ai-agents-miniagent) | A small Python agent harness using a local Ollama model. Shows the model → tool request → validation → execution → result loop. | Run it with non-sensitive local notes and examine which requests its three local tools accept or reject. |
| [`ai-agents-mcp-itsops`](https://github.com/albeus/ai-agents-mcp-itsops) | A narrow MCP server offering read-only-style operations tools: runbook search, host health and bounded Kubernetes pod status. | Discover and call its tools with MCP Inspector. |
| [`ai-agents-litellm-gateway`](https://github.com/albeus/ai-agents-litellm-gateway) | A LiteLLM Proxy and PostgreSQL lab for routing a local model and registering the MCP server, with separate ITS Operations and Service Desk identities. | Read the [results and known issue](https://github.com/albeus/ai-agents-litellm-gateway#results) before relying on any access-control behaviour. |
| [`ai-agents-trace-probe`](https://github.com/albeus/ai-agents-trace-probe) | A scripted observability experiment correlating a model call and direct MCP calls with one trace ID, including a deliberate client-side timeout. | Read the example JSONL trace and reproduce the failure in an isolated lab. |

## How the pieces fit

```text
miniagent ────────────────────────────────→ Ollama
   │  local agent loop and its own tools
   └── not currently an MCP client in this lab

MCP Inspector / other MCP client ─────────→ mcp-itsops
                                            narrow MCP tools

Client ───────────────────────────────────→ LiteLLM Proxy ─→ Ollama
                                                 │
                                                 └──────────→ mcp-itsops
                                                             (HTTP lab setup)

trace_probe ──────────────────────────────→ LiteLLM Proxy (model call)
           └──────────────────────────────→ mcp-itsops (direct MCP calls)
           └── writes correlated local JSONL events
```

`miniagent` owns an agent loop and executes its own local tools; it does not call `mcp-itsops`. The MCP server publishes tools for MCP-compatible clients. LiteLLM is a separate gateway experiment. `trace_probe` is a scripted probe, not a full agent or a distributed-tracing stack: it calls the MCP server directly, so it does not establish end-to-end tracing through LiteLLM's MCP gateway.

## Suggested learning path

1. Start with [`ai-agents-miniagent`](https://github.com/albeus/ai-agents-miniagent) to see the model propose a tool call and the Python harness decide whether to execute it.
2. Move to [`ai-agents-mcp-itsops`](https://github.com/albeus/ai-agents-mcp-itsops) to expose similarly narrow capabilities through MCP and inspect discovery and invocation.
3. Read and, if appropriate, reproduce [`ai-agents-litellm-gateway`](https://github.com/albeus/ai-agents-litellm-gateway) to explore model routing and team keys—and the difference between registering a tool server and proving its access policy.
4. Use [`ai-agents-trace-probe`](https://github.com/albeus/ai-agents-trace-probe) to see what a shared local trace ID can reveal about model latency, direct MCP calls and a client-side timeout.

Each repository has its own README with setup steps, recorded results and limitations. There is no root-level installer or one-command deployment.

## What has and has not been shown

| Area | Observed in the lab | Not established |
| --- | --- | --- |
| Agent loop | `miniagent` can ask a local model for tool calls and execute constrained local tools. | Production-grade command isolation, run-level limits, multi-user security or an MCP-connected agent. |
| MCP | `mcp-itsops` exposes a small, constrained tool surface. The server was also exercised over HTTP for the gateway lab. | Production authentication, audit, sandboxing or safe access to real infrastructure. Check the server code and README for the transport configuration you plan to run. |
| Gateway | Both team identities could reach the configured model; the LiteLLM master key could list the registered MCP tools. | Per-team MCP access was **not** proven. In the recorded test, grants were not persisted as expected and non-master keys received 403 responses. Do not treat the intended ITS Operations/Service Desk tool split as enforced. |
| Observability | One local JSONL trace correlated a model call, MCP discovery and a deliberately timed-out MCP tool call. | Automatic cross-service distributed tracing or proof of server-side cancellation when the client times out. |

## Safety and publication

- The lab READMEs describe experiments, not hardened services. Use disposable or isolated lab environments with non-sensitive data and least-privilege credentials.
- Before publishing or copying any lab, inspect **files and Git history** for secrets and internal information. In particular, review LiteLLM key/team JSON output and tool-response files. Do not publish `.env`, generated keys, real credentials, kubeconfigs, confidential runbooks, trace logs or database volumes. Use sanitised examples instead. If a credential has already been exposed, revoke or rotate it; deleting the latest copy does not remove it from Git history.
- `miniagent` uses `shell=True` with text-based checks. Its prefix filter and metacharacter rejection are learning controls, **not** a robust command-execution boundary.
- `mcp-itsops` runs with the permissions of its local user and any configured Kubernetes credentials. Use lab-only, least-privilege credentials and non-sensitive material.
- To reach host services from the gateway container, Ollama or the MCP server may need to bind to `0.0.0.0`. That does **not** mean loopback-only access. Check the host firewall and do not expose unauthenticated lab endpoints to an untrusted network.
- The trace probe deliberately creates a slow operation and a client timeout. A client timeout does not establish that the server stopped working; avoid write-capable or consequential tools.

## Next improvements

- Replace `miniagent`'s shell-string execution with fixed executable/argument lists, and add loop and wall-clock limits.
- Document the MCP server's actual stdio and HTTP launch paths consistently, and add automated tests for allowed and denied inputs.
- Re-test LiteLLM MCP permissions against a pinned release, checking stored grants and effective behaviour for **both** teams before claiming policy enforcement.
- Add trace context at the agent, gateway and MCP layers if pursuing genuine end-to-end observability.

The individual lab READMEs are the source of truth for detailed setup and the recorded outcome of each experiment.
