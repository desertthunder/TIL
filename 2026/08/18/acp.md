---
title: The Agent Client Protocol
description: A JSON-RPC interface between code editors and coding agents
tags:
  - artificial-intelligence
  - agents
  - protocols
  - developer-tools
---

The Agent Client Protocol (ACP) standardizes communication between a code editor and a
coding agent. It is analogous to LSP: the Agent owns the model-and-tools loop, while the
Client owns the user interface, environment, and access decisions. Neither side needs the
other’s private integration API.[^1]

ACP messages use JSON-RPC 2.0. Local agents use newline-delimited UTF-8 over stdin/stdout,
and stdout must contain only valid ACP messages. A connection starts with `initialize`,
where the Client advertises its latest major protocol version and capabilities and the Agent
responds with the negotiated version and its capabilities. The Client then creates
`session/new` with an absolute `cwd` and optional MCP servers. The returned session ID
scopes the conversation and its state.[^2]

A prompt turn starts with `session/prompt`. The Agent streams text, thoughts, plans, and
tool-call progress through `session/update`. It may ask the Client for permission with
`session/request_permission`. The Client can interrupt with `session/cancel`. The Agent
eventually reports a stop reason. In v1, that reason is the response to the still-pending
prompt request.

As of August 2026, v1 is stable and v2 is a draft. v2 acknowledges `session/prompt`
immediately and moves foreground lifecycle into `state_update`. An `idle` update carries
the completion reason. It also uses upsert-style updates and removes v1’s Client filesystem
and terminal-execution APIs. Implementers should negotiate the version per connection and
support v1 alongside v2 during the transition.[^3]

[^1]: Agent Client Protocol, [Introduction](https://agentclientprotocol.com/get-started/introduction).

[^2]: Agent Client Protocol, [Initialization](https://agentclientprotocol.com/protocol/v1/initialization), [Session Setup](https://agentclientprotocol.com/protocol/v1/session-setup), and [Transports](https://agentclientprotocol.com/protocol/v1/transports).

[^3]: Agent Client Protocol, [Migrating from v1](https://agentclientprotocol.com/protocol/v2/migration).
