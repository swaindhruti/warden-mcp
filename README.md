# Warden
A distributed, security-focused **MCP (Model Context Protocol) gateway** written in Go — stateless routing per the [2026-07-28 MCP spec](https://modelcontextprotocol.io/specification/2026-07-28/changelog), with per-tool rate limiting and cost attribution, audit logging, and tool-call anomaly detection.

> **Status: early build.** Architecture is settled, implementation in progress. This README will be updated with real benchmarks and usage instructions as pieces land — see [Roadmap](#roadmap).

## What is this?
MCP standardizes how AI agents call tools across servers, but production deployments need what every API gateway needs: rate limiting, auth, audit trails, and abuse detection — none of which the protocol itself provides. Warden sits between an MCP host (Claude Desktop, an agent framework, your own app) and one or more real MCP servers, and from the host's side looks like a single MCP server — while underneath, it fans out to and manages many.

```
MCP Host → [Warden] → Backend MCP Server A
                    → Backend MCP Server B
                    → Backend MCP Server C
```
Built stateless per the 2026-07-28 spec: every request self-describes its protocol version and capabilities via `_meta`, so any Warden instance can handle any request with zero coordination — run N instances behind a plain round-robin load balancer, no sticky sessions, no shared session store.

## Why
MCP servers are increasingly the thing agents talk to in production, and the [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) exists because that surface is already being attacked — over 30 CVEs filed against MCP infrastructure in Jan–Feb 2026 alone. Warden is the control-plane layer that threat model calls for: a single, auditable choke point in front of tool calls, instead of every backend server reinventing its own rate limiting and logging.

## Core features (target)

- **Stateless routing** — dispatches `tools/call` to the correct backend based solely on the incoming request; no session state held anywhere
- **Per-tool rate limiting** — token-bucket limits scoped per caller and per tool, not just per connection
- **Cost attribution** — tracks usage per caller/tool for accountability
- **Audit logging** — structured, queryable log of every tool call, correlated by `tool_use_id`
- **Anomaly detection** — rule-based detection of abnormal call-rate or call-sequence patterns per caller

## Architecture

```
warden/
├── cmd/warden/             # entrypoint — wiring only
├── internal/
│   ├── transport/          # Streamable HTTP, _meta parsing, protocol version checks
│   ├── mcp/                # MCP types, server/discover, protocol primitives
│   ├── router/             # stateless dispatch to backend servers
│   ├── middleware/
│   │   ├── auth/           # OAuth 2.1-aligned token validation/forwarding
│   │   ├── ratelimit/      # per-tool, per-caller rate limiting
│   │   ├── audit/          # async structured audit logging
│   │   └── anomaly/        # tool-call anomaly detection
│   ├── backend/            # registry + client logic for real MCP servers
│   └── config/
├── test/                   # integration + load test scripts
├── Dockerfile
├── docker-compose.yml
└── warden.yaml.example
```
Built on Go's standard library (`net/http`, `httputil.ReverseProxy`) — no web framework. Warden has a handful of routes and is already building on `ReverseProxy` as its core primitive; a framework adds dependency surface without solving a problem Warden has, and for a security-focused gateway, minimizing third-party dependency surface is itself a design goal.

Audit logging and anomaly detection run off the synchronous request path (via a channel + worker pool) so they don't add latency to the actual tool call. Rate limiting runs synchronously, since the decision has to be made before forwarding.

## Request lifecycle

1. **Transport** parses the incoming request envelope and `_meta` (protocol version, client capabilities, client identity). Unsupported protocol versions are rejected here, before any other work runs.
2. **Middleware chain** (composed `func(http.Handler) http.Handler`): auth check → rate limit check → route decision → forward to backend as an MCP client → collect result.
3. **Audit + anomaly detection** happen off the hot path — the completed request/response is pushed onto a channel and drained by a separate worker pool, so they never add latency to the tool call itself.

## Roadmap

- [ ] Transport layer + `_meta` parsing + `server/discover`
- [ ] Stateless router, single hardcoded backend
- [ ] In-process rate limiter (token bucket)
- [ ] Auth passthrough
- [ ] Async audit logger
- [ ] Anomaly detection (rule-based)
- [ ] Table-driven tests + `-race` + `testing.B` benchmarks
- [ ] Dockerfile + docker-compose example
- [ ] Distributed (Redis-backed) rate limiter — stretch goal
Target: working MVP with tests and measured benchmarks (latency p50/p95/p99, req/sec — not "I benchmarked it") by **November 2026**.

## Running it
Not yet runnable — check back once the transport + router skeleton lands.

Once available, Warden ships as a standalone binary or Docker image, configured via YAML — not imported as a library. It's infrastructure a host points traffic *at*, the same way you'd run nginx or Envoy in front of a service.

## License
MIT
