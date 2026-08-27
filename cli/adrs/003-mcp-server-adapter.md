# ADR 003 - MCP Server as a Second Primary-Adapter Family

**Status**: Accepted
**Date**: 2026-07-05

## Context

Every command shipped by a Cobra-based CLI adopting this constitution is reached through exactly one primary (driving) adapter family: the Cobra command tree under `/cmd` ([ADR 001](001-system-architecture.md)). ADR 001 already anticipates a second family in prose — "a `serve` subcommand exposing the same use-cases over HTTP... itself just another primary adapter calling the same use-case root package, never a parallel implementation" — but that sentence is not, by itself, a worked decision an agent can implement from: it says a second primary-adapter family is *permitted*, not how one should be shaped, wired, and bounded.

This ADR generalizes the pattern for a project's first non-Cobra primary adapter: a Model Context Protocol (MCP) server exposing selected use-cases as MCP Tools, so an LLM client can read/act on the tool's domain directly, the same way a human does through the command line. Adopting MCP support is typically also the first time a project takes on a third-party dependency purely to *speak a driving protocol*, as opposed to a secondary/driven integration (a cloud SDK, `git`, a REST client).

## Decision

1. **MCP is a second primary-adapter family, alongside Cobra.** An MCP tool handler MUST be a thin wrapper — decode the tool's JSON arguments, call the identical `internal/app/<domain>.Component` primary-port function every equivalent Cobra command already calls, render the result, nothing more. It is never a parallel reimplementation of business logic already expressed for the Cobra path. A project's MCP tools MUST delegate verbatim into the same `internal/app/<domain>` functions its sibling Cobra subcommands call — e.g. an MCP tool exposing "fetch one item" and "search" calls the exact functions the CLI's own `get`/`grep`-style subcommands call, never a second, divergent implementation of "how do we fetch X" or "how do we search Y."

2. **Transport**: `github.com/modelcontextprotocol/go-sdk`, the official Go SDK for MCP, MUST be the project's sole MCP transport dependency (see Neglected for what this rules out). `mcp.StdioTransport` is the default transport; `mcp.NewStreamableHTTPHandler` (Streamable HTTP/SSE) is used when an explicit `--http <addr>` flag is given. The SDK's `Transport` abstraction is consumed directly by the primary-adapter command that wires it up (e.g. `cmd/<tool>/serve.go`) — it is not re-wrapped in a project-private port, since (per ADR 001 port isolation rule 2) a port narrower than what the SDK already expects would add indirection with no second implementation to justify it.

3. **Loopback-by-default bind address**: `--http <addr>`'s value is a `[host]:port` address spec. A bare port or `:port` (no explicit host) binds `127.0.0.1` only; an explicit host binds exactly that host. An operator must opt in to a non-loopback bind by naming a host explicitly — the default is never reachable from another machine.

## Neglected

- **A hand-rolled JSON-RPC 2.0 transport** is rejected outright: it would mean re-implementing MCP's initialize/list-tools/call-tool handshake and schema-validation logic from scratch — the exact "second, divergent client for the same external system" constitution Principle VII forbids — for a protocol that already has an official, well-tested SDK.
- **`github.com/mark3labs/mcp-go`**, a popular third-party MCP server library, is rejected as the default choice: it predates the official SDK and is not maintained by the protocol's own org. A project with a pre-existing dependency on it MAY document a project-specific exception per ADR 001's amendment procedure, but MUST NOT adopt it fresh alongside the official SDK.
- **A project-private port wrapping `mcp.Transport`** is rejected: there is no second use-case that could need a *different* MCP transport implementation, so the indirection would exist for its own sake.

## To Achieve

A second primary-adapter family that reaches the same use-case roots every Cobra command already reaches, so an LLM client and a human operator are guaranteed to see identical behavior for the same underlying operation — never two diverging code paths for "how do we fetch a resource" or "how do we search content."

## Accepting

An MCP server command produces no `stdout` document of its own and uses none of the CLI's human-facing output renderer/theme machinery (ADR 002 DS-04, DS-05) — its "output" is MCP tool-call content (typically markdown text) and one stderr log line per call. Where a project already has a documented deviation for another machine/LLM-consumable output surface (e.g. a `--json`-only or scriptable subcommand), this ADR extends that same deviation to a second adapter family entirely. We accept a long-running `RunE` (blocks until its context is canceled) as a structural property of a server command — not a deviation this ADR could design away. A project MUST record how it adapts its E2E-testing pattern (ADR 001, constitution Principle VIII) for a long-running command in that feature's own design/research notes.

## Implementation notes

- `cmd/<tool>/serve.go` (or `cmd/<tool>/<domain>/serve.go` for a multi-domain CLI) is the sole new primary-adapter code for this feature: it registers the MCP tools, translating MCP args → domain calls → response content.
- A new domain method needed only to satisfy an MCP-specific access pattern (e.g. a single-item "get" alongside an existing "list"/"search") SHOULD reuse existing internal helpers already defined for sibling use-case methods in the same package — no new traversal or parsing logic duplicated for the MCP path.
- No `cobra` or MCP SDK import appears anywhere below `cmd/`, preserving ADR 001's hexagonal boundary for this second adapter family exactly as it already holds for Cobra.
- Implement MCP endpoints only when explicitly asked as part of requirements
