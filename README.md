# MCP Gateway: What It Is and How It Works

**TL;DR**

- An MCP gateway is a control layer between MCP clients and MCP servers that centralizes authentication, tool access policy, credentials, and audit logging for every tool call.
- An MCP server exposes tools for one system, an MCP proxy relays messages for a connection, and an MCP gateway applies one policy across many clients and many servers.
- The MCP authorization specification forbids passing tokens to servers they were not issued for, so an MCP gateway must broker upstream credentials instead of forwarding client tokens.
- Loading every tool definition into the prompt is expensive; Anthropic reported that having agents call tools through code cut one workflow from 150,000 tokens to 2,000.
- MCP gateways differ most in upstream authentication modes, tool filtering granularity, audit depth, deployment model, added latency, and whether model and agent traffic share the same policy.

An MCP gateway is a single entry point that sits between MCP clients, such as coding agents, IDEs, and chat assistants, and the [Model Context Protocol](https://modelcontextprotocol.io/specification/latest) servers that give those clients tools and data. Without one, every client keeps its own server list and credentials, and no one has a complete record of which tools AI applications call. An MCP gateway moves authentication, tool permissions, and logging into one place that a platform team operates. This guide covers how it works, how it differs from a proxy and a server, and how to evaluate one.

## What Is an MCP Gateway?

An MCP gateway is infrastructure that accepts MCP requests from many clients, authenticates each caller, decides which tools that caller may see and invoke, and forwards approved calls to the correct upstream MCP server. To clients, the gateway looks like one MCP server. To upstream servers, it looks like one client holding the right credentials.

![Three MCP clients connect to one MCP gateway, which applies authentication, tool filters, and audit logging before calling three upstream MCP servers](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-mcp-gateway/ai-gateways-mcp-gateway-placement.png)

*Figure 1: Clients configure one endpoint; the gateway decides which upstream tools each one can reach.*

The Model Context Protocol defines two roles: clients inside AI applications, and servers that expose tools, resources, and prompts. As Figure 1 shows, an MCP gateway plays both: a server toward clients and a client toward each upstream server.

The current specification revision (2026-07-28) defines [two standard transports](https://modelcontextprotocol.io/specification/latest/basic/transports): stdio, where the client launches the server as a local subprocess, and Streamable HTTP, where each message is an HTTP POST to one endpoint. A gateway typically runs stdio servers itself, connects to remote Streamable HTTP servers, and exposes one HTTP endpoint to clients. Earlier revisions used a session handshake, so gateways serving older clients also bridge revisions.

## Why Teams Use an MCP Gateway

Teams adopt an MCP gateway when the number of MCP servers and clients grows past what per-machine configuration can manage. The usual triggers are credential sprawl, no inventory of which servers are in use, no audit trail for tool calls, and security reviews that require one enforcement point for tool permissions.

![Top lane: two agents each connect directly to both MCP servers with their own tokens. Bottom lane: both agents call one MCP gateway instead](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-mcp-gateway/ai-gateways-mcp-gateway-without-vs-with.png)

*Figure 2: Without a gateway, every new client or server adds another set of credentials and another unlogged path.*

- **Credential sprawl.** Each client configuration stores a token for each server. Rotating one credential means finding every machine that holds a copy.
- **Unknown servers.** The [OWASP MCP Top 10](https://owasp.org/projects/mcp-top-10) lists shadow MCP servers and missing audit telemetry among the most critical risks in MCP deployments.
- **Untrusted tool descriptions.** [Invariant Labs](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) described tool poisoning in April 2025: instructions hidden in a tool description that the model reads and the user does not see. The [MCPTox benchmark](https://arxiv.org/abs/2508.14925), built on 45 live MCP servers, measured attack success rates as high as 72.8% across 20 LLM agents.

## How an MCP Gateway Works

An MCP gateway processes each tool call as a short pipeline: it authenticates the caller, checks the requested tool against that caller's allowlist, attaches the credential the upstream server expects, forwards the call, and records the result. Tool listings pass through the same filter.

![A tool call passes through authentication, a tool access check, and credential attachment before forwarding to the MCP server and an audit log; denied calls stop early](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-mcp-gateway/ai-gateways-mcp-gateway-tool-call-pipeline.png)

*Figure 3: Access checks run before any upstream credential is used, so a denied call never reaches the server.*

1. **Authenticate the caller** from an API key, OAuth token, or single sign-on session.
2. **Resolve visible tools.** For `tools/list`, the gateway merges tool lists from permitted servers and namespaces names, since two servers can both expose a tool called `search`.
3. **Check tool access.** For `tools/call`, the gateway confirms the tool is on the caller's allowlist; denied calls are rejected and logged (Figure 3).
4. **Attach the upstream credential** the target server accepts for this user: a secret, an OAuth token, or an exchanged token.
5. **Forward and record** the call, writing an audit entry with caller, tool, status, and latency.

## MCP Gateway vs MCP Proxy vs MCP Server

An MCP server implements tools, resources, and prompts for one system. An MCP proxy relays MCP messages for a connection, often to make a local stdio server reachable over HTTP. An MCP gateway includes proxying but adds identity, per-client tool policy, credential brokering, and audit logging across many servers and clients.

| Aspect | MCP server | MCP proxy | MCP gateway |
|---|---|---|---|
| Primary job | Expose tools and data from one system | Relay messages, often across transports | Govern tool access across many servers |
| Scope | One backend | One connection or server | Every client and server in an organization |
| Authentication | Validates tokens issued for itself | Usually none of its own | Authenticates callers and brokers upstream credentials |
| Tool policy | Defines its own tools | None | Allowlists per client, team, or key |
| Audit trail | Server-specific, if any | Minimal | Central record of every tool call |

Products use these labels loosely, so test for identity, policy, and audit features rather than trusting the name.

## Core MCP Gateway Capabilities

A production MCP gateway covers identity, tool governance, credential brokering, and observability. Discovery and transport handling are common to all of them. Gateways differ in how finely they scope tools, how they handle per-user credentials, and what they record about each call.

| Capability | What it does |
|---|---|
| Caller authentication | Verifies users, services, or API keys before any tool is listed |
| Tool filtering | Limits visible and callable tools per client, team, key, or request |
| Credential brokering | Stores upstream secrets, runs OAuth flows, and issues per-user or exchanged tokens |
| Human approval | Requires explicit approval, or an allowlist, before sensitive tools run automatically |
| Audit logging | Records caller, tool, arguments, status, and latency for each call |
| Server catalog | Keeps a registry of approved servers and the tool definitions that were reviewed |

## MCP Gateway Security

MCP gateway security centers on credentials and tool descriptions. The MCP authorization specification requires servers to accept only tokens issued for them, which rules out passing client tokens upstream. Tool descriptions are untrusted input, so a gateway should record approved definitions and limit which tools each client can reach.

The [MCP authorization specification](https://modelcontextprotocol.io/specification/latest/basic/authorization) builds HTTP authorization on OAuth 2.1 and treats a protected MCP server as an OAuth resource server; stdio servers read credentials from their environment instead. That leaves an MCP gateway three patterns for upstream access:

- **Shared service credentials.** One secret per server, stored in the gateway; simple, but every user acts under the same upstream identity.
- **Per-user OAuth.** Each user authorizes their own account once, and the gateway stores and refreshes the token, so upstream actions are attributed to the requester.
- **Token exchange.** The gateway swaps the caller's token for one scoped to the upstream server using [OAuth 2.0 Token Exchange (RFC 8693)](https://datatracker.ietf.org/doc/html/rfc8693), so no long-lived user token sits in the gateway.

Tool poisoning and "rug pulls," where a server changes a tool description after approval, call for recording each reviewed tool definition and flagging changes. Short tool lists per client and approval steps for destructive tools limit damage when a bad description passes review.

## Reducing Token Cost from Large Tool Sets

Every tool exposed to a model adds its name, description, and input schema to the prompt. With dozens of connected servers, tool definitions can fill a large share of the context window before the model reads the task. MCP gateways reduce this cost by filtering tools per client and by letting models call tools through code.

Filtering is the first control: a support assistant does not need a platform agent's deployment tools. The second is code-based tool orchestration. In November 2025, Anthropic described agents that [call MCP tools through code](https://www.anthropic.com/research/code-execution-with-mcp), loading definitions on demand and processing intermediate results in a sandbox. In the example Anthropic gave, token use dropped from 150,000 to 2,000, a 98.7% reduction. Implemented in the gateway, the pattern benefits every client without client-side changes.

## How to Evaluate an MCP Gateway

Evaluate an MCP gateway against the traffic it will carry in production, not a demo with one server. The checks that separate MCP gateways are upstream authentication modes, tool filtering granularity, audit depth, deployment model, added latency under load, and whether model calls and agent calls fall under the same policy.

- **Upstream authentication.** Look for per-user OAuth and token exchange in addition to static headers, so actions in third-party systems run under the requesting user's identity.
- **Tool filtering granularity.** Filtering per client, per API key, and per request lets one gateway serve a coding agent and a support bot with different tool sets.
- **Token cost controls.** Code-based tool orchestration or on-demand tool loading matters once tool counts reach the hundreds.
- **Audit and observability.** Tamper-resistant audit logs and OpenTelemetry traces put tool calls into the monitoring stack a team already runs.
- **Deployment model.** Tool arguments and results often contain source code and customer data; a self-hosted gateway running in a VPC, on-premises, or air-gapped keeps them inside the network.
- **Added latency.** Agents chain many tool calls per task, so per-call overhead compounds. Ask for benchmarks measured at the request rate you expect.
- **One policy across AI traffic.** Agents call models, tools, and other agents within one task. A gateway that also governs [LLM traffic](https://github.com/ai-gateways/llm-gateway) and [agent-to-agent calls](https://github.com/ai-gateways/agent-gateway) applies one set of keys, budgets, and logs to all three.
- **Protocol coverage.** Confirm support for stdio and Streamable HTTP servers, the current specification revision, and clients that still use earlier revisions.

## Open Source MCP Gateway Options

Several open-source projects implement the MCP gateway pattern with different scopes. Some focus on running MCP servers in containers, some on Kubernetes-native routing, and some combine MCP governance with model and agent traffic. The table summarizes what each project publishes in its repository.

| Project | Language, license | Transports | Authentication | Tool filtering | Also governs | Deployment |
|---|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost) | Go, Apache 2.0 | Exposes `/mcp` over HTTP and SSE; connects to servers over STDIO, HTTP, or SSE | Headers, OAuth, per-user OAuth, per-user headers, token exchange | Per request, per MCP client connection, per virtual key | LLM calls across 25+ providers; A2A agent traffic | npx, Docker, Kubernetes, or Go SDK; in-VPC, on-prem, or air-gapped |
| [ContextForge](https://github.com/IBM/mcp-context-forge) (IBM) | Python, Apache 2.0 | HTTP, JSON-RPC, WebSocket, SSE, Streamable HTTP; stdio server-side | JWT, Basic, or SSO for callers; user-scoped OAuth tokens upstream | Virtual servers that bundle selected tools | A2A agents; REST and gRPC APIs | PyPI, Docker, or Kubernetes with Helm |
| [Docker MCP Gateway](https://github.com/docker/mcp-gateway) | Go, MIT | stdio, SSE, streaming | OAuth flows; secrets through Docker Desktop | Enable or disable tools per profile | Not published | Docker CLI plugin; each local server runs in its own container |
| [Microsoft MCP Gateway](https://github.com/microsoft/mcp-gateway) | C#, MIT | Streamable HTTP (MCP 2026-07-28 clients) | Entra ID authentication | App-role authorization per server and tool | Not published | Kubernetes StatefulSets; Azure deployment option |

Details reflect each repository in October 2026; confirm current capabilities in each project's documentation before choosing.

## Frequently Asked Questions

### What is the difference between an MCP gateway and an API gateway?

An API gateway routes and secures HTTP APIs using paths, methods, and headers. An MCP gateway understands MCP messages: it merges tool lists from several servers, filters tools by name for each caller, brokers OAuth credentials according to the MCP authorization specification, and logs tool-level events.

### Do I need an MCP gateway for a single MCP server?

One developer using one trusted MCP server rarely needs a gateway. An MCP gateway becomes useful when several clients share several servers, when credentials need central rotation, when security teams ask for an audit trail of tool calls, or when different teams need different tool sets.

### Is an MCP gateway part of the MCP specification?

Not as a named role. The MCP specification defines clients, servers, transports, and authorization; its Streamable HTTP binding mirrors request metadata into headers so intermediaries can route requests. An MCP gateway is an architecture pattern built on those roles: a server toward clients and a client toward upstream servers. It must follow the specification on both sides, including token audience rules.

### How does an MCP gateway handle authentication?

An MCP gateway authenticates in two directions. Inbound, it verifies callers through API keys, OAuth, or single sign-on. Outbound, it presents whatever each upstream server accepts: static headers, a shared OAuth client, per-user OAuth tokens, or tokens obtained through token exchange. The MCP authorization specification forbids forwarding a caller's token to a server it was not issued for.

### Can an MCP gateway prevent tool poisoning?

An MCP gateway reduces tool poisoning risk but does not remove it. A gateway can restrict clients to approved servers, record reviewed tool definitions and flag changes, keep tool lists short per client, require approval for sensitive tools, and log every call for review. It cannot detect every malicious instruction inside a description or a tool result, so upstream servers still need their own review.

### Does an MCP gateway add latency?

Every MCP gateway adds some overhead, because each call passes through authentication, policy checks, and logging. The amount depends on the implementation language, connection reuse, and whether policy lookups run in memory or hit a database. Agents make many tool calls per task, so measure overhead at your expected concurrency.

## Related

- [LLM gateway](https://github.com/ai-gateways/llm-gateway)
- [Agent gateway](https://github.com/ai-gateways/agent-gateway)
- [AI Gateways organization](https://github.com/ai-gateways)
