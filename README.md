# MCP Gateway

An MCP gateway is a single entry point that sits between MCP clients, such as coding agents, IDEs, and chat assistants, and the Model Context Protocol servers that give those clients tools and data. Without one, every client keeps its own server list and credentials, and no one has a complete record of which tools AI applications call. An MCP gateway moves authentication, tool permissions, and logging into one place that a platform team operates.

This guide explains how an MCP gateway works, how it differs from an MCP proxy and an MCP server, how it handles security and token cost, and which open-source projects implement it.

## Contents

- [What is an MCP gateway?](#what-is-an-mcp-gateway)
- [Why teams use an MCP gateway](#why-teams-use-an-mcp-gateway)
- [How an MCP gateway works](#how-an-mcp-gateway-works)
- [MCP gateway vs MCP proxy vs MCP server](#mcp-gateway-vs-mcp-proxy-vs-mcp-server)
- [Core MCP gateway capabilities](#core-mcp-gateway-capabilities)
- [MCP gateway security](#mcp-gateway-security)
- [Reducing token cost from large tool sets](#reducing-token-cost-from-large-tool-sets)
- [How to evaluate an MCP gateway](#how-to-evaluate-an-mcp-gateway)
- [Open source MCP gateway options](#open-source-mcp-gateway-options)
- [Related guides](#related-guides)

## What is an MCP gateway?

An MCP gateway is infrastructure that accepts MCP requests from many clients, authenticates each caller, decides which tools that caller may see and invoke, and forwards approved calls to the correct upstream MCP server. To clients, the gateway looks like one MCP server. To upstream servers, it looks like one client holding the right credentials.

```text
coding agent    --+                                  +--> MCP server: code hosting
IDE             --+--> MCP gateway                --+--> MCP server: database
chat assistant  --+    (auth, tool filters, audit)   +--> MCP server: ticketing
```

The Model Context Protocol defines [hosts, clients, and servers](https://modelcontextprotocol.io/specification/2026-07-28/architecture#core-components): clients run inside AI applications, and servers expose tools, resources, and prompts. An MCP gateway plays both roles, a server toward clients and a client toward each upstream server.

The current specification revision (2026-07-28) defines two standard transports: [stdio](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio#:~:text=the%20client%20launches%20the%20MCP%20server%20as%20a%20subprocess), where the client launches the server as a local subprocess, and [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#sending-messages), where each message is an HTTP request to one endpoint. A gateway typically runs stdio servers itself, connects to remote Streamable HTTP servers, and exposes one HTTP endpoint to clients. Earlier revisions opened a session with an `initialize` handshake, so gateways that serve older clients also [bridge protocol revisions](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports#backward-compatibility).

## Why teams use an MCP gateway

Teams adopt an MCP gateway when the number of MCP servers and clients grows past what per-machine configuration can manage. The usual triggers are credential sprawl, no inventory of which servers are in use, no audit trail for tool calls, and security reviews that require one enforcement point for tool permissions.

- **Credential sprawl.** Each client configuration stores a token for each server. Rotating one credential means finding every machine that holds a copy.
- **Unknown servers and missing logs.** The OWASP MCP Top 10 lists [shadow MCP servers](https://github.com/OWASP/www-project-mcp-top-10/blob/165fe0f78ef104459237b4a8e0f6e78db9b02391/2025/MCP09-2025%E2%80%93Shadow-MCP-Servers.md#description) (MCP09) and [lack of audit and telemetry](https://github.com/OWASP/www-project-mcp-top-10/blob/165fe0f78ef104459237b4a8e0f6e78db9b02391/2025/MCP08-2025%E2%80%93Lack-of-Audit-and-Telemetry.md#description) (MCP08) among the main risks in MCP deployments.
- **Untrusted tool descriptions.** Invariant Labs described [tool poisoning attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks#:~:text=We%20have%20discovered%20a%20critical%20vulnerability%20in%20the%20Model%20Context%20Protocol) in April 2025: instructions hidden in a tool description that the model reads and the user does not see. The [MCPTox benchmark](https://arxiv.org/abs/2508.14925#:~:text=MCPTox%20is%20constructed%20upon%2045%20live), built on 45 live MCP servers, measured attack success rates as high as 72.8% across 20 LLM agents.

One developer using one trusted MCP server rarely needs a gateway. It becomes useful when several clients share several servers, when credentials need central rotation, when security teams ask for an audit trail of tool calls, or when different teams need different tool sets.

## How an MCP gateway works

An MCP gateway processes each tool call as a short pipeline: it authenticates the caller, checks the requested tool against that caller's allowlist, attaches the credential the upstream server expects, forwards the call, and records the result. Tool listings pass through the same filter.

```text
tools/call from client
  -> authenticate caller ........ unknown caller: reject
  -> check tool allowlist ....... tool not allowed: reject and log
  -> attach upstream credential
  -> forward to MCP server
  -> write audit entry (caller, tool, status, latency)
result to client
```

1. **Authenticate the caller** from an API key, OAuth token, or single sign-on session.
2. **Resolve visible tools.** For [`tools/list`](https://modelcontextprotocol.io/specification/2026-07-28/server/tools#listing-tools), the gateway merges tool lists from permitted servers and namespaces [tool names](https://modelcontextprotocol.io/specification/2026-07-28/server/tools#tool-names), since two servers can both expose a tool called `search`.
3. **Check tool access.** For [`tools/call`](https://modelcontextprotocol.io/specification/2026-07-28/server/tools#calling-tools), the gateway confirms the tool is on the caller's allowlist; denied calls are rejected and logged before any upstream credential is used.
4. **Attach the upstream credential** the target server accepts for this user: a secret, an OAuth token, or an exchanged token.
5. **Forward and record** the call, writing an audit entry with caller, tool, status, and latency.

## MCP gateway vs MCP proxy vs MCP server

An MCP server implements tools, resources, and prompts for one system. An MCP proxy relays MCP messages for a connection, often to make a local stdio server reachable over HTTP. An MCP gateway includes proxying but adds identity, per-client tool policy, credential brokering, and audit logging across many servers and clients.

| Aspect | MCP server | MCP proxy | MCP gateway |
|---|---|---|---|
| Primary job | Expose tools and data from one system | Relay messages, often across transports | Govern tool access across many servers |
| Scope | One backend | One connection or server | Every client and server in an organization |
| Authentication | Validates tokens issued for itself | Usually none of its own | Authenticates callers and brokers upstream credentials |
| Tool policy | Defines its own tools | None | Allowlists per client, team, or key |
| Audit trail | Server-specific, if any | Minimal | Central record of every tool call |

The specification does not name a gateway role, but it anticipates one: the Streamable HTTP transport [mirrors request metadata into HTTP headers](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http#request-metadata) so that load balancers, gateways, and observability tools can route requests without parsing the body. A general API gateway can carry that traffic, but it routes by paths and headers; an MCP gateway understands MCP messages, merges tool lists, and filters tools by name for each caller. Products use these labels loosely, so test for identity, policy, and audit features rather than trusting the name.

## Core MCP gateway capabilities

A production MCP gateway covers identity, tool governance, credential brokering, and observability. Discovery and transport handling are common to all of them. Gateways differ in how finely they scope tools, how they handle per-user credentials, and what they record about each call.

| Capability | What it does |
|---|---|
| Caller authentication | Verifies users, services, or API keys before any tool is listed |
| Tool filtering | Limits visible and callable tools per client, team, key, or request |
| Credential brokering | Stores upstream secrets, runs OAuth flows, and issues per-user or exchanged tokens |
| Human approval | Requires explicit approval, or an allowlist, before sensitive tools run automatically |
| Audit logging | Records caller, tool, arguments, status, and latency for each call |
| Server catalog | Keeps a registry of approved servers and the tool definitions that were reviewed |

## MCP gateway security

MCP gateway security centers on credentials and tool descriptions. The MCP authorization specification requires servers to accept only tokens issued for them, which rules out passing client tokens upstream. Tool descriptions are untrusted input, so a gateway should record approved definitions and limit which tools each client can reach.

The [MCP authorization specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization#standards-compliance) builds HTTP authorization on OAuth 2.1, and its [token handling rules](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization#token-handling) require MCP servers to validate that each access token was issued for them as the intended audience. The MCP security best practices name [token passthrough](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices#token-passthrough) as an anti-pattern and describe the [confused deputy problem](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices#confused-deputy-problem) that proxies to third-party APIs must avoid. That leaves an MCP gateway three patterns for upstream access:

- **Shared service credentials.** One secret per server, stored in the gateway; simple, but every user acts under the same upstream identity.
- **Per-user OAuth.** Each user authorizes their own account once, and the gateway stores and refreshes the token, so upstream actions are attributed to the requester.
- **Token exchange.** The gateway swaps the caller's token for one scoped to the upstream server using an [OAuth 2.0 Token Exchange request (RFC 8693)](https://datatracker.ietf.org/doc/html/rfc8693#section-2.1), so no long-lived user token sits in the gateway.

Tool poisoning and "rug pulls," where a server changes a tool description after approval, call for recording each reviewed tool definition and flagging changes. A gateway reduces this risk but does not remove it: short tool lists per client and approval steps for destructive tools limit the damage when a malicious description or tool result gets through, and upstream servers still need their own review.

## Reducing token cost from large tool sets

Every tool exposed to a model adds its name, description, and input schema to the prompt. With dozens of connected servers, tool definitions can fill a large share of the context window before the model reads the task. MCP gateways reduce this cost by filtering tools per client and by letting models call tools through code.

Filtering is the first control: a support assistant does not need a platform agent's deployment tools. The second is code-based tool orchestration. In November 2025, Anthropic described agents that call MCP tools through code, loading definitions on demand and processing intermediate results in a sandbox; in its example, [token use dropped from 150,000 to 2,000](https://www.anthropic.com/research/code-execution-with-mcp#:~:text=150%2C000%20tokens%20to%202%2C000%20tokens), a 98.7% reduction. Implemented in the gateway, the pattern benefits every client without client-side changes.

## How to evaluate an MCP gateway

Evaluate an MCP gateway against the traffic it will carry in production, not a demo with one server. The checks that separate MCP gateways are upstream authentication modes, tool filtering granularity, audit depth, deployment model, added latency under load, and whether model calls and agent calls fall under the same policy.

- **Upstream authentication.** Look for per-user OAuth and token exchange in addition to static headers, so actions in third-party systems run under the requesting user's identity.
- **Tool filtering granularity.** Filtering per client, per API key, and per request lets one gateway serve a coding agent and a support bot with different tool sets.
- **Token cost controls.** Code-based tool orchestration or on-demand tool loading matters once tool counts reach the hundreds.
- **Audit and observability.** Tamper-resistant audit logs and OpenTelemetry traces, using standard attributes such as [`gen_ai.tool.name`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-tool-name), put tool calls into the monitoring stack a team already runs.
- **Deployment model.** Tool arguments and results often contain source code and customer data; a self-hosted gateway running in a VPC, on-premises, or air-gapped keeps them inside the network.
- **Added latency.** Every gateway adds some overhead through authentication, policy checks, and logging. The amount depends on the implementation language, connection reuse, and whether policy lookups run in memory or hit a database. Agents chain many tool calls per task, so ask for benchmarks at the request rate you expect.
- **One policy across AI traffic.** Agents call models, tools, and other agents within one task. A gateway that also governs [LLM traffic](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works) and [agent-to-agent calls](https://github.com/ai-gateways/agent-gateway#how-the-a2a-protocol-works) applies one set of keys, budgets, and logs to all three.
- **Protocol coverage.** Confirm support for stdio and Streamable HTTP servers, the current specification revision, and clients that still use earlier revisions.

## Open source MCP gateway options

Several open-source projects implement the MCP gateway pattern with different scopes. Some focus on running MCP servers in containers, some on Kubernetes-native routing, and some combine MCP governance with model and agent traffic. The table summarizes what each project publishes in its repository.

| Project | Language, license | Transports | Authentication | Tool filtering | Also governs | Deployment |
|---|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost/blob/dev/docs/mcp/overview.mdx) | Go, Apache 2.0 | Exposes `/mcp` over HTTP and SSE; connects to servers over STDIO, HTTP, or SSE | Headers, OAuth, per-user OAuth, per-user headers, token exchange | Per request, per MCP client connection, per virtual key | LLM calls across 25+ providers; A2A agent traffic | npx, Docker, Kubernetes, or Go SDK; in-VPC, on-prem, or air-gapped |
| [ContextForge](https://github.com/IBM/mcp-context-forge#overview--goals) (IBM) | Python, Apache 2.0 | HTTP, JSON-RPC, WebSocket, SSE, Streamable HTTP; stdio server-side | JWT, Basic, or SSO for callers; user-scoped OAuth tokens upstream | Virtual servers that bundle selected tools | A2A agents; REST and gRPC APIs | PyPI, Docker, or Kubernetes with Helm |
| [Docker MCP Gateway](https://github.com/docker/mcp-gateway#features) | Go, MIT | stdio, SSE, streaming | OAuth flows; secrets through Docker Desktop | Enable or disable tools per profile | Not published | Docker CLI plugin; each local server runs in its own container |
| [Microsoft MCP Gateway](https://github.com/microsoft/mcp-gateway#features) | C#, MIT | Streamable HTTP (MCP 2026-07-28 clients) | Entra ID authentication | App-role authorization per server and tool | Not published | Kubernetes StatefulSets; Azure deployment option |

Details reflect each repository in October 2026; confirm current capabilities in each project's documentation before choosing.

## Related guides

- [LLM gateway vs API gateway](https://github.com/ai-gateways/llm-gateway#llm-gateway-vs-api-gateway): how model traffic differs from general HTTP traffic
- [Agent gateway vs MCP gateway vs LLM gateway](https://github.com/ai-gateways/agent-gateway#agent-gateway-vs-mcp-gateway-vs-llm-gateway): the three hops a single agent task crosses
- [The three kinds of AI traffic](https://github.com/ai-gateways/ai-gateway#the-three-kinds-of-ai-traffic): how LLM, MCP, and agent gateways fit under one AI gateway
- [Guardrails for tools and agents](https://github.com/ai-gateways/ai-guardrails#guardrails-for-tools-and-agents): checking tool arguments and tool results
- [Shadow AI](https://github.com/ai-gateways/ai-governance#shadow-ai): unreviewed MCP servers and AI tools outside the gateway
- [AI Gateways](https://github.com/ai-gateways): all guides in this organization
