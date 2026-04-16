# Agent Delegation Chain Specification (ADCS)

> An open spec for **agent-to-agent delegation chains** — the chain of A2A hops that begins with a human and extends through autonomous agents. Defines a JSON data structure plus normative semantics for scope narrowing, budget propagation, cycle prevention, identity preservation, and audit emission.

[**Read the spec → `SPEC.md`**](./SPEC.md)
&nbsp; · &nbsp; [JSON Schema](./schema/chain.schema.json)
&nbsp; · &nbsp; [Examples](./examples/)
&nbsp; · &nbsp; [Conformance test vectors](./conformance/)
&nbsp; · &nbsp; [Reference implementation (Apache-2)](https://github.com/davidcrowe/gatewaystack-connect/blob/main/apps/tenant-gateway/src/integrations/agents/delegation.ts)

---

## Why this exists

Agent-to-agent (A2A) is now a real piece of infrastructure. Google's [A2A Protocol](https://a2a-protocol.org/) (50+ launch partners, donated to Linux Foundation) covers transport, discovery, and capability negotiation. CrewAI ships [first-class A2A delegation](https://docs.crewai.com/en/learn/a2a-agent-delegation). Anthropic's Agent SDK has subagents. OpenAI's Agents SDK has handoffs. AWS Bedrock AgentCore models delegated identity with `Act: Agent` claims.

What's missing is a **shared, vendor-neutral data shape** for the chain itself: who started this work, which agents touched it, what permissions were narrowed at each hop, how much budget remains, was a cycle prevented? Today every product invents its own answer. Audit logs don't compose. Multi-vendor tracing breaks at the agent boundary. There's no single artifact a procurement officer can ask for to verify that "agent X acted on behalf of person Y under constraints Z."

ADCS proposes one.

## What it specifies

A normative JSON data structure for the chain, plus runtime rules that make the structure actually useful:

| Rule | What it says | Why it matters |
|---|---|---|
| **`originSub` invariant** | The human at the root of the chain MUST NOT change at any depth | Auditability: every action traces to one accountable human |
| **Scope intersection** | A child agent's permissions = `intersect(parent, childProfile)`. Permissions can only narrow. | Least-privilege at every hop, by construction |
| **Budget propagation** | `child_budget = min(parent_remaining, child_max)` | Runaway loops in deep chains are mathematically bounded |
| **Cycle prevention** | An agent MUST NOT delegate to a profile already in the chain | No mutual recursion, no infinite loops |
| **Type vs runtime identity** | Distinct `agentProfileId` (stable type) and `agentRunId` (per-execution) | Audit logs can correlate the calls of one specific run, even if the agent type ran in parallel |
| **Audit emission shape** | Every governed tool call MUST emit an entry containing the full chain | Logs are reconstructable; SIEM integration is a query, not a custom pipeline |

The spec is ~2,500 words. Read it: [`SPEC.md`](./SPEC.md).

## How ADCS relates to other specs

| Spec | What it covers | What ADCS adds | Relationship |
|---|---|---|---|
| **[A2A Protocol](https://a2a-protocol.org/)** | Transport, discovery, auth handoff between agents | The chain semantics A2A explicitly leaves to a higher layer | **Layers on top** — an A2A invocation MAY carry an ADCS chain |
| **[MCP](https://modelcontextprotocol.io/)** | Tool invocation between agent and tool | The chain context that wraps each MCP `tools/call` | **Wraps MCP** — every governed MCP call emits an ADCS audit entry |
| **[RFC 8693 OAuth Token Exchange](https://www.rfc-editor.org/rfc/rfc8693.html)** | Nested `act` claims for delegation | Scope intersection + budget + tool sets + runtime identity | **Profile of** — ADCS chains MAY be expressed as JWTs with nested `act` |
| **[OAuth Transaction Tokens (draft)](https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/)** | Scope monotonic reduction across call chains | Set-intersection (stricter), budget, agent-typed identity | **Strictly stronger** intersection rule |
| **[Macaroons / Biscuits](https://doc.biscuitsec.org/)** | Capability tokens with cryptographic attenuation | A standardized JSON shape for the chain payload | **Compatible envelope** — ADCS chains MAY be wrapped in Biscuit |
| **[OIDC-A (arxiv 2509.25974)](https://arxiv.org/abs/2509.25974)** | `delegator_sub` + `delegation_chain` claim names | Concrete data shape, budget, runtime identity, conformance vectors | **Acknowledges as prior art**; chooses different vocabulary to avoid OIDC namespace collision |
| **[Agent Identity Protocol (arxiv 2603.24775)](https://arxiv.org/abs/2603.24775)** | Invocation-Bound Capability Tokens with Biscuit + Datalog | JSON-first, no Datalog; budget propagation; simpler audit shape | **Sister spec** — different design choices for different audiences |
| **[OpenTelemetry GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/)** | Span attributes for agent invocation | The chain attributes the semconv doesn't yet cover | **Emits to** — ADCS audit entries map cleanly onto OTel attributes |

ADCS is **deliberately one piece** of the agent governance puzzle. It does not specify how agents authenticate, how they discover each other, or how they execute. It specifies the chain that records what happened.

## Non-normative design opinions

The spec has opinions. Here are the load-bearing ones, with the alternatives we considered and rejected:

**Most-restrictive wins (intersection), not last-actor wins.** RFC 8693's `act` claim is "current actor wins, prior actors recorded." We chose intersection because least-privilege at every hop is a stronger security posture than last-actor-replaces-prior. A delegation chain isn't a stack of credentials; it's a narrowing of authority.

**Budget propagation is a first-class field.** No competing spec we surveyed has it. We added it because runaway loops in deep multi-agent chains are the #1 production failure mode, and bounding them at the data-structure level is cheap and load-bearing.

**Runtime identity (`agentRunId`) is distinct from type identity (`agentProfileId`).** Audit logs need to correlate the calls of *this specific execution*, not just "calls by any instance of agent X." Two parallel runs of the same agent type get distinct chains.

**JSON-first, not Datalog-first.** The Agent Identity Protocol uses Biscuit + Datalog policies for chained delegations. That's mathematically powerful and audit-ready for advanced users. We chose JSON as the primary shape because it's readable by every CISO, every SOC analyst, and every junior engineer in the room — and a Biscuit envelope is still possible as an optional layer (§10).

**`originSub` invariant, not actor-update semantics.** Some delegation models replace `actor` at each hop. We chose to preserve the human at the root absolutely, because *who is accountable* is a different question from *who is currently executing* — and the second question is already answered by the last link in the chain.

We're explicit about these because **opinions are how a spec creates leverage.** A spec without opinions describes data; a spec with opinions describes behavior. Behavior is what makes things compose.

## Reference implementation

The reference implementation is the [Agentic Control Plane gateway](https://github.com/davidcrowe/gatewaystack-connect):

- [`delegation.ts`](https://github.com/davidcrowe/gatewaystack-connect/blob/main/apps/tenant-gateway/src/integrations/agents/delegation.ts) — `intersectScopes`, `intersectTools`, `computeChildBudget`, `detectCycle`, `buildChildChain`
- [`hookGovernance.ts`](https://github.com/davidcrowe/gatewaystack-connect/blob/main/apps/tenant-gateway/src/govern/hookGovernance.ts) — chain merge into the policy evaluator
- Audit emission via `emitLogEvent` in `mcp/logging.ts`

Used in production by Reducibl, Inc. for [Agentic Control Plane](https://agenticcontrolplane.com/).

## Status and process

ADCS is **v0.1.0 — Draft**. Field names and semantics MAY change before v1.0. Once v1.0 ships, backwards-compatibility commitments apply.

We're seeking implementer feedback. Open an issue with:

- A use case ADCS doesn't yet cover
- An ambiguity in the spec
- A field name you'd argue against
- A normative semantic you'd argue for
- An implementation report ("we built ADCS in [language] — here's what was hard")

Pull requests welcome for spec text, schema, examples, and conformance vectors.

## Roadmap

- **v0.1.x** — early adopter implementations across at least 3 frameworks (Claude Code, CrewAI, LangGraph). Extension namespace stable. Conformance suite runnable.
- **v0.2** — JWT + Biscuit envelope profiles. OpenTelemetry semconv extension drafted.
- **v0.3** — Submitted to [CSAI Foundation](https://cloudsecurityalliance.org/) for review. IETF individual draft as a profile of RFC 8693 + Transaction Tokens.
- **v1.0** — Stable. Backwards-compatibility commitments. Multi-vendor implementer registry.

## License

Specification text: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — copy, adapt, redistribute with attribution.
JSON Schema, examples, conformance vectors: [MIT](https://opensource.org/licenses/MIT) — use freely.

## Contact

- Editor: David Crowe — [david@reducibl.com](mailto:david@reducibl.com)
- GitHub: [agentic-control-plane/delegation-chain-spec](https://github.com/agentic-control-plane/delegation-chain-spec)
- Reducibl: [agenticcontrolplane.com](https://agenticcontrolplane.com)
