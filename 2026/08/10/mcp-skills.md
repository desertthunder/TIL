---
title: MCP and Agent Skills
description: Connecting agents to capabilities and teaching them how to use those capabilities
tags:
  - artificial-intelligence
  - agents
  - model-context-protocol
  - agent-skills
---

The Model Context Protocol (MCP) and Agent Skills extend an agent at different layers.
MCP connects an agent application to external data and actions through a standard
protocol. A skill packages instructions, references, scripts, and assets that teach the
agent how to perform a kind of work.

In short:

- MCP supplies **what the agent can access or do**.
- A skill supplies **how the agent should approach the work**.

Neither makes the model more intelligent, and neither guarantees that it will choose the
right action but they both give the agent better interfaces and better procedural context.

## Model Context Protocol

MCP is an open protocol for exchanging context between an AI application and external
programs. An MCP **host**, such as an editor or agent application, creates a client for
each MCP server. A server can run as a local process over standard input and output or as
a remote service over Streamable HTTP.[^1]

Servers expose three main primitives:

- **Tools** are callable operations, such as searching an issue tracker, querying a
  database, or creating a calendar event.
- **Resources** are readable data, such as file contents, database records, or an API
  response.
- **Prompts** are reusable message templates that a user can select.

The client discovers these primitives through methods such as `tools/list` and invokes a
tool through `tools/call`. The host combines tools from its connected servers into the
set available to the model, routes the model's call to the correct server, and returns
the result to the conversation.[^1]

The current `2026-07-28` protocol uses JSON-RPC 2.0 and a stateless request model. Each
request carries its relevant protocol and client information, while the optional
`server/discover` call lets a client inspect a server's versions and capabilities in
advance. The protocol defines the exchange; it does not prescribe how the host chooses
context, prompts the model, or runs its agent loop.[^1][^2]

MCP is therefore an integration boundary rather than a workflow description. A GitHub
server might expose tools for reading and changing issues, but it does not by itself tell
an agent how a team triages a bug, which evidence to collect, or when to ask for approval.

## Agent Skills and `SKILL.md`

An Agent Skill is a directory whose required entry point is `SKILL.md`. That file begins
with YAML frontmatter containing at least a `name` and `description`, followed by Markdown
instructions. The directory can also include `scripts/`, `references/`, and `assets/` for
code, detailed documentation, templates, and other supporting material.[^3]

Skills use progressive disclosure:

1. The agent initially receives each installed skill's name and description.
2. When the task matches that description, it loads the complete `SKILL.md`.
3. It reads supporting files or runs bundled scripts only when the instructions call for
   them.[^4]

Activating from a description match keeps the initial context cost small while allowing
a skill to carry a substantial playbook. The description is important because it is both
documentation and a routing hint. A poorly described skill may never be selected, or may
activate for unrelated work.

A skill can encode a review checklist, house style, migration procedure, output schema,
or a sequence for using several tools, any domain knowledge. Bundled scripts are useful
where ordinary code is cheaper and more reliable than asking the model to reproduce an
operation, but the skill still depends on the host's filesystem, execution environment,
permissions, and available tools.[^3][^5]

## Advantages and disadvantages

|                  | MCP                                                                    | Agent Skill                                                          |
| ---------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Primary job      | Connect the host to live data and actions                              | Give the agent reusable procedures and domain context                |
| Interface        | Structured client/server protocol                                      | Filesystem package led by Markdown instructions                      |
| Use Case         | APIs, databases, SaaS products, changing or private state              | Stable guidance, examples, templates, and local reference material   |
| Execution        | Server tools with declared input schemas                               | Existing agent tools and optional bundled scripts                    |
| Operational cost | Server lifecycle, transport, authorization, latency, and compatibility | Authoring, triggering, context use, maintenance, and model adherence |
| Trust boundary   | A local or remote service with data and action permissions             | Instructions and code loaded into the agent's environment            |

### MCP

MCP's main advantage is interoperability. One server can present a consistent interface
to several compatible hosts, while discovery lets the available operations and data
change without hard-coding them into every prompt. It is well suited to current state
because the server can query the source system when the agent needs it instead of copying
that state into instructions. Structured tool schemas also give the host a clearer place
to validate arguments, request consent, log calls, and enforce authorization.

That interface carries real operational weight because someone must build, deploy, version,
authenticate, observe, and keep the server available. Remote calls add latency and a new
failure dependency. Large or overlapping tool catalogs can also make tool selection
harder and consume context, although clients can use progressive discovery rather than
load every tool at once.[^1]

The security/permission structure is substantial because tools may read private data or
cause side effects. The MCP security guidance covers confused-deputy attacks, token passthrough,
server-side request forgery, excessive OAuth scopes, and unsafe local-server privileges.
Hosts and servers still need least-privilege access, meaningful consent, credential
validation, sandboxing where appropriate, and auditable actions, as using the protocol does
not supply those controls automatically.[^6]

### Agent Skills

Skills are simpler to create and inspect. They are ordinary files that can live beside a
project, can be reviewed in version control, and be shared without operating a service.
They are a good fit for institutional knowledge and repeatable work because they can
combine instructions with examples, references, templates, and tested scripts. On-demand
loading lets an agent keep many skills available without placing every full playbook in
its initial context.[^4]

Their weakness is that most of the playbook remains natural-language guidance. The model
can misunderstand, skip, or inconsistently apply it. Activation also depends on the
description and on host-specific skill support. Portability of the package does not mean
every client provides the same tools, sandbox, model behavior, or support for experimental
fields such as `allowed-tools`.[^3]

Skills can go stale when they duplicate changing product facts or API behavior. They are
also executable supply-chain inputs. A repository-level skill can inject instructions,
and a bundled script can do whatever its granted environment permits. Skill-capable
clients should require trust for project skills, while users should review instructions
and scripts before granting sensitive tools or credentials.[^7]

## When to use each

Use MCP when the agent needs to:

- retrieve live, private, or frequently changing information
- take actions in an external system
- use one maintained integration from several compatible agent hosts
- cross a service boundary that needs explicit schemas, authentication, permissions, and
  audit logs.

Use a skill when the agent needs to:

- follow a repeatable procedure or organization-specific policy
- produce a consistent format, apply a checklist, or use a house style
- consult stable reference material, examples, or templates only when relevant
- combine tools already available to the agent into a higher-level workflow

Use both when a workflow requires external capabilities and procedural judgment. For
incident triage, an MCP server could read alerts, query deployment history, and update an
issue. A skill could tell the agent which evidence to collect, how to rank severity, when
to request human approval, and how to write the incident summary. Anthropic describes
this as a complementary design: skills teach workflows that involve MCP tools and other
software.[^8]

Use neither for a simple one-off task that a short prompt and existing tool can handle.
Avoid turning a small local script into an MCP server unless a standard, discoverable
service boundary is useful. Avoid putting credentials or a changing copy of an external
system into `SKILL.md`. Let the skill describe the procedure and let a properly scoped
tool retrieve the current state.

[^1]: Model Context Protocol. “Architecture overview.” <https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture>.

[^2]: David Soria Parra and Den Delimarsky. “The 2026-07-28 Specification.” Model Context Protocol, 28 July 2026. <https://blog.modelcontextprotocol.io/posts/2026-07-28/>.

[^3]: Agent Skills. “Specification.” <https://agentskills.io/specification>.

[^4]: Agent Skills. “Agent Skills Overview.” <https://agentskills.io/home>.

[^5]: Agent Skills. “Using scripts in skills.” <https://agentskills.io/skill-creation/using-scripts>.

[^6]: Model Context Protocol. “Security Best Practices.” <https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices>.

[^7]: Agent Skills. “How to add skills support to your agent.” <https://agentskills.io/client-implementation/adding-skills-support>.

[^8]: Barry Zhang, Keith Lazuka, and Mahesh Murag. “Equipping agents for the real world with Agent Skills.” Anthropic, 16 October 2025. <https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills>.
