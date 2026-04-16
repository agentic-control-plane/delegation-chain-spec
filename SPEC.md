# Agent Delegation Chain Specification (ADCS)

**Version:** 0.1.0
**Status:** Draft (open for comment)
**Editor:** David Crowe ([@davidcrowe](https://github.com/davidcrowe)), Reducibl, Inc.
**Reference implementation:** [agentic-control-plane/gatewaystack-connect](https://github.com/davidcrowe/gatewaystack-connect) (`apps/tenant-gateway/src/integrations/agents/delegation.ts`)

## Abstract

This specification defines a normative data structure and runtime semantics for **agent delegation chains** — the chain of agent-to-agent (A2A) hops that begins with a human principal and extends through one or more autonomous agents acting on that principal's behalf.

ADCS specifies (1) the wire shape of a delegation chain, (2) the semantics for narrowing scopes and tools when one agent invokes another, (3) propagation of cost budgets through the chain, (4) cycle prevention, (5) preservation of the originating human identity at every depth, and (6) the audit-emission shape that makes a chain reconstructable from logs.

ADCS is transport-neutral. It composes with the [A2A Protocol](https://a2a-protocol.org/) for transport, the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) for tool invocation, [OAuth 2.0 Token Exchange (RFC 8693)](https://www.rfc-editor.org/rfc/rfc8693.html) for actor claims, and the [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) for telemetry emission.

## 1. Conformance language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) when, and only when, they appear in all capitals, as shown here.

## 2. Definitions

**Origin principal**
A natural person, service account, or otherwise non-agent identity that initiated the work that an agent is performing. The origin principal is identified by `originSub` in the chain and MUST NOT change for the lifetime of the chain.

**Agent**
A software process that consumes input, decides on actions, and invokes tools or other agents on behalf of an origin principal. An agent MAY have a stable type identity (`agentProfileId`) and MUST have a per-execution runtime identity (`agentRunId`).

**Agent type identity (`agentProfileId`)**
A stable identifier for the configuration that defines an agent — its system prompt, tool permissions, model, scope and budget caps. Multiple runs of the same logical agent share an `agentProfileId`.

**Agent runtime identity (`agentRunId`)**
A unique identifier for a single execution of an agent. Distinct from `agentProfileId` so audit logs can correlate the calls made by one specific run, even if the agent type ran multiple times in parallel.

**Delegation**
The act of an agent invoking another agent (the **child**) to perform a sub-task. The invoking agent is the **parent**.

**Effective scopes / effective tools**
The set of OAuth scopes and the set of tool identifiers a given link in the chain is currently authorized to use. Computed by intersection (see §6).

**Remaining budget**
A non-negative integer (in cents) representing the maximum LLM cost a given link in the chain MAY incur. Computed by minimum-propagation (see §7).

## 3. Data model

A delegation chain is a JSON document with the following shape:

```json
{
  "originSub": "string",
  "originClaims": { },
  "links": [ DelegationLink ],
  "depth": "integer (>= 0)"
}
```

A `DelegationLink` is:

```json
{
  "agentProfileId": "string",
  "agentRunId": "string",
  "agentName": "string",
  "effectiveScopes": [ "string" ],
  "effectiveTools": [ "string" ],
  "remainingBudgetCents": "integer (>= 0)",
  "delegatedAt": "string (RFC 3339 timestamp)"
}
```

The complete JSON Schema is at [`schema/chain.schema.json`](schema/chain.schema.json).

## 4. Field semantics

**`originSub`** — REQUIRED. Stable identifier of the origin principal. Implementations MAY use any URI-shaped identifier (e.g. an OIDC `sub`, a `urn:` identifier, an email, a service-account ARN). Once a chain is created, `originSub` MUST NOT change at any depth.

**`originClaims`** — OPTIONAL. The principal's identity claims at chain creation, preserved verbatim. When present, this enables downstream authorization decisions to use the human's full claim set without a fresh authentication round-trip.

**`links`** — REQUIRED. An ordered array of `DelegationLink` objects. `links[0]` is the first agent invoked by the origin principal. `links[N-1]` is the currently-executing agent. The array MAY be empty in a freshly minted chain that has not yet entered an agent context.

**`depth`** — REQUIRED. The number of links in the chain. Implementations MUST keep `depth === links.length`.

For each `DelegationLink`:

**`agentProfileId`** — REQUIRED. The stable type identifier of the agent at this hop. Two different runs of the same agent share this value.

**`agentRunId`** — REQUIRED. A unique-per-execution identifier. SHOULD be a UUID or an opaque random string. MUST be distinct between any two runs, even of the same `agentProfileId`.

**`agentName`** — REQUIRED. A human-readable identifier suitable for dashboards and audit logs. MAY equal the `agentProfileId`.

**`effectiveScopes`** — REQUIRED. The set of OAuth scopes this link is currently authorized to exercise. MUST be the result of `intersectScopes` (see §6) of all parents' effective scopes and this agent's profile scopes.

**`effectiveTools`** — REQUIRED. The set of tool identifiers this link is currently authorized to invoke. MUST be the result of `intersectTools` (see §6).

**`remainingBudgetCents`** — REQUIRED. Non-negative integer. MUST be the result of `computeChildBudget` (see §7) at delegation time. MAY be decremented by the runtime as the agent makes LLM calls.

**`delegatedAt`** — REQUIRED. RFC 3339 timestamp of when this link was added to the chain.

## 5. The originSub invariant

The `originSub` and `originClaims` fields of a chain MUST NOT change after creation. Specifically:

- A new chain MAY be created when an origin principal first invokes an agent.
- Every delegation MUST preserve the parent chain's `originSub` and `originClaims` exactly.
- An implementation MUST NOT permit a delegation operation that would alter, replace, or remove `originSub`.
- An implementation MUST reject a chain whose audit log shows a changed `originSub` as malformed or tampered with.

This invariant is the single most important property of ADCS. It is what makes auditability of multi-agent systems possible: every action taken at any depth can be traced back to a single accountable human.

## 6. Scope and tool narrowing (intersection)

When an agent (the parent, with `effectiveScopes_parent` and `effectiveTools_parent`) delegates to another agent (the child, with `profileScopes_child` and `profileTools_child`), the child's effective permissions are computed by **intersection**:

```
effectiveScopes_child = intersectScopes(effectiveScopes_parent, profileScopes_child)
effectiveTools_child  = intersectTools(effectiveTools_parent,  profileTools_child)
```

`intersectScopes` is defined as:

```
function intersectScopes(parent: string[], childProfile: string[]): string[]:
  if parent is empty or childProfile is empty: return []
  return [ s for s in childProfile if matchesAny(s, parent) ]

function matchesAny(value: string, patterns: string[]): boolean:
  for p in patterns:
    if p == value: return true
    if p ends with ".*":
      prefix = p without trailing "*"
      if value starts with prefix: return true
  return false
```

`intersectTools` is identical to `intersectScopes` with one exception: an empty `parent` array means "unrestricted" (the parent inherited no tool restriction), and the child gets its full profile tool set. An empty `childProfile` means "no tools requested" and the child gets none.

**Permissions MUST only narrow.** A child's effective scopes or tools MUST be a subset of the parent's effective scopes or tools (modulo the unrestricted-empty-parent exception for tools). Implementations MUST NOT permit a delegation operation that would widen permissions.

Wildcard support (e.g. `github.*` matching `github.repos.create`) is REQUIRED for both scopes and tools. The wildcard pattern is the dot-star (`.` followed by `*`) at the end of a string only.

## 7. Budget propagation

When an agent delegates to a child, the child's `remainingBudgetCents` is computed as:

```
remainingBudgetCents_child = min(
  remainingBudgetCents_parent,
  maxBudgetCents_childProfile
)
```

The child MAY consume up to `remainingBudgetCents_child` and MUST stop attempting LLM calls when its remaining budget reaches zero. When that happens, the runtime MUST return a structured error to the parent containing:

```json
{ "error": "BUDGET", "code": -32002, "remainingBudgetCents": 0 }
```

The parent MAY then decide how to continue (retry with a different agent, return partial results, surface the budget exhaustion to the origin principal).

**Budget MUST monotonically decrease as the chain deepens.** A child's `remainingBudgetCents` MUST be less than or equal to its parent's at the time of delegation. Implementations MUST NOT permit budget propagation that would increase the child's budget above the parent's remaining.

## 8. Cycle prevention

An agent MUST NOT delegate to a target whose `agentProfileId` already appears in the current chain. Specifically, before adding a new link with `agentProfileId == targetProfileId`, the implementation MUST check:

```
function detectCycle(chain: DelegationChain, targetProfileId: string): boolean:
  return any link in chain.links where link.agentProfileId == targetProfileId
```

If `detectCycle` returns true, the delegation MUST be rejected with:

```json
{ "error": "CYCLE", "code": -32003 }
```

This prevents agents from creating infinite loops by re-invoking themselves through intermediaries. It is a stricter rule than depth-limiting alone — depth-limiting bounds compute, cycle prevention bounds also reasoning loops.

Implementations MAY additionally enforce a depth limit (e.g. maximum 10 levels) for compute-cost reasons. Depth-limiting is RECOMMENDED but OPTIONAL.

## 9. Audit emission

Every tool call made within a chain context MUST emit an audit log entry containing the following fields:

```json
{
  "timestamp": "RFC 3339 string",
  "originSub": "string",
  "agent": {
    "profileId": "string",
    "runId": "string",
    "name": "string"
  },
  "delegation": {
    "depth": "integer (>= 0)",
    "chain": [ "agentName", ... ],
    "runChain": [ "agentRunId", ... ],
    "parentProfileId": "string | null",
    "remainingBudgetCents": "integer (>= 0)"
  },
  "tool": {
    "name": "string",
    "ok": "boolean"
  }
}
```

These fields are the minimum required for chain reconstruction. Implementations MAY add framework-specific fields (e.g. PII findings, decision reasons, latency).

When emitting telemetry to OpenTelemetry, implementations SHOULD map the `delegation.*` fields to span attributes namespaced as `acp.delegation.depth`, `acp.delegation.origin_sub`, `acp.delegation.chain` (joined with `>` for readability), etc., until a normative GenAI semconv extension is defined.

## 10. Security considerations

**Trust boundary.** ADCS does not specify how the origin principal is authenticated. Implementations MUST authenticate the origin principal at chain creation using a mechanism appropriate to their deployment (OIDC, mTLS, signed JWT, etc.). All subsequent narrowing happens server-side, so the agent's claimed identity at each hop is never trusted — the gateway maintains the authoritative chain.

**Replay.** An audit log entry containing a chain MUST be considered a record of a past event, not a credential. Replaying a chain document does not re-authorize anything.

**Cryptographic envelope.** ADCS specifies a JSON data shape, not a wire format. Implementations MAY transmit chains in any envelope (HTTP header, JWT claim, [Biscuit](https://doc.biscuitsec.org/) capability, custom protobuf). When transport over an untrusted channel is required, implementations SHOULD use a signed envelope (JWT or Biscuit) so each link can be cryptographically verified against the previous.

**Privacy.** `originClaims` MAY contain personally-identifying information. Implementations MUST handle audit logs containing `originClaims` according to the privacy regime applicable to their deployment.

## 11. Extension namespace

Implementations MAY add fields beyond those defined in §3 and §9. Such extensions MUST be namespaced under a `vendor.` prefix or in a separate `vendorExtensions` object to avoid collision with future ADCS revisions:

```json
{
  "agentProfileId": "...",
  "vendorExtensions": {
    "myvendor.classification": "internal-only"
  }
}
```

Future ADCS revisions MAY promote widely-adopted vendor extensions into the normative spec.

## 12. Relationship to other specifications

- **[A2A Protocol](https://a2a-protocol.org/)** — A2A defines transport, discovery, and capability negotiation between agents. ADCS layers on top: an A2A invocation MAY carry an ADCS chain in a header, message-part, or extension. The two specs are complementary.
- **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)** — MCP defines tool invocation between an agent and a tool. ADCS chains wrap MCP calls — every MCP `tools/call` made within a chain context emits an audit entry per §9.
- **[RFC 8693 OAuth Token Exchange](https://www.rfc-editor.org/rfc/rfc8693.html)** — RFC 8693's nested `act` claim is conceptually similar to `links[]`. An ADCS chain MAY be expressed as a JWT with nested `act` claims plus ADCS-specific claims (`effectiveScopes`, `effectiveTools`, `remainingBudgetCents`). This is left to implementations.
- **[OAuth Transaction Tokens](https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/)** — Transaction Tokens enforce monotonic scope reduction across a call chain. ADCS's intersection rule (§6) is a stricter version (set-intersection rather than scope-string-subset) and adds budget propagation.
- **[Authenticated Delegation paper (arxiv 2501.09674)](https://arxiv.org/abs/2501.09674)** — This work motivates the need for delegation credentials. ADCS provides one concrete answer.
- **[OpenTelemetry GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/)** — ADCS audit emission MAY be expressed as OpenTelemetry spans with the attributes described in §9.

## 13. Conformance

An implementation conforms to ADCS v0.1.0 if, given the test vectors in [`conformance/`](conformance/), it produces the documented output for every operation:

- `intersectScopes` and `intersectTools` produce the expected sets
- `computeChildBudget` produces the expected integer
- `detectCycle` returns the expected boolean
- A chain emitted after delegation has `depth === links.length` and an unchanged `originSub`
- An audit log entry contains all REQUIRED fields from §9

A conformance test runner is intentionally not specified — implementations are free to integrate the test vectors however they choose.

## 14. Versioning

ADCS uses [Semantic Versioning](https://semver.org/). v0.x is unstable; field names and semantics MAY change between v0.x releases. v1.0 will commit to backwards-compatibility guarantees.

## 15. References

### Normative

- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) — Key words for use in RFCs to indicate requirement levels
- [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) — Date and Time on the Internet: Timestamps

### Informative

- [A2A Protocol](https://a2a-protocol.org/latest/specification/) — Agent2Agent Protocol specification
- [Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25) — MCP specification
- [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693.html) — OAuth 2.0 Token Exchange
- [draft-ietf-oauth-transaction-tokens](https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/) — OAuth Transaction Tokens
- [Authenticated Delegation (arxiv 2501.09674)](https://arxiv.org/abs/2501.09674)
- [OIDC-A: OpenID Connect for Agents (arxiv 2509.25974)](https://arxiv.org/abs/2509.25974)
- [Agent Identity Protocol (arxiv 2603.24775)](https://arxiv.org/abs/2603.24775)
- [Biscuit Authorization Token](https://doc.biscuitsec.org/)
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Cloud Security Alliance — Fixing AI Agent Delegation](https://cloudsecurityalliance.org/blog/2026/03/25/control-the-chain-secure-the-system-fixing-ai-agent-delegation)

## 16. Acknowledgments

ADCS draws on prior art from RFC 8693, OAuth Transaction Tokens, Macaroons (Birgisson et al., 2014), Biscuit, the Authenticated Delegation paper (Chan et al., 2025), and OIDC-A (Subramanya, 2025). Where this spec opinionates differently from those works, the divergence is intentional and documented in the comparison table at [README.md](README.md#how-adcs-relates-to-other-specs).

## License

This specification is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Reference implementations are MIT-licensed.
