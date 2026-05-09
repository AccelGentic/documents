# Proxy — specification

*The proxy is the single chokepoint for all model traffic in Kjarr. It validates every request against the calling agent's capability manifest, enforces token budgets, maintains the audit log, mediates MCP server access, and provides the kill switch. Smaller than the harness in scope, but the place where the heaviest security review will land.*

## What the proxy is

A stateless HTTP service. Agents send signed requests describing their intended model invocation; the proxy verifies, validates, and either forwards to a model provider (Anthropic, OpenAI, others as configured) or rejects. Responses stream back through the proxy with audit assembly happening at end-of-stream. State that matters — audit log entries, budget consumption, kill-switch flags — lives in backing stores read by the proxy, not in the proxy's memory.

The proxy is the prompt-injection firewall. Every claim in the harness spec about "the agent cannot reach tools outside its manifest" or "the agent cannot route through an attacker-controlled MCP server" is enforced here. The harness's type system handles its own process; the proxy handles everything that happens between the harness and the outside world.

The proxy is also the cost control surface. Per-agent, per-rig, per-team token budgets are enforced at request time, not reconciled monthly from a vendor invoice. An agent that exceeds its budget cannot think; an agent that gets prompt-injected into a runaway loop cannot exhaust the bill before being cut off.

## Core responsibilities

The five things the proxy does, in roughly the order they happen on a request:

**Identity verification.** Every request is signed by the calling agent's p2panda-auth keypair, includes the agent's manifest version, and includes a freshness nonce. The proxy verifies the signature against the registered identity, checks the manifest version matches the current signed manifest for that agent, and rejects replays.

**Authorization.** Look up the agent's manifest. The request includes a `tools` declaration (from the harness's tool registry) and may include `mcp_servers` (if the agent is configured to use any). Validate that every tool declared matches a `WriteOperation` or `ReadScope` in the manifest. Validate that every MCP server URL is in the manifest's allowlist. Reject the request entirely if any element doesn't match — do not silently strip unauthorized declarations.

**Budget check.** Look up the agent's current budget consumption (per-day, per-convoy, per-call). If the request would exceed any budget, reject. Budget state is read from the backing store on every request; this read is on the hot path and needs to be fast.

**Forward and stream.** If verification, authorization, and budget check pass, forward the request to the configured model provider. Stream the response back to the harness as it arrives.

**Audit and meter.** At end-of-stream (or at a deliberate point of stream interruption), assemble the audit record from the full request/response pair, write it to the audit log, parse the response's usage data, write the consumed tokens to the budget store.

That's the entire flow. Everything else is implementation detail.

## Request lifecycle

The hot path needs to be fast. A typical interactive agent reasoning loop is multiple round trips per minute; the proxy adds latency on every one of them. Target: under 10ms of proxy-added latency on a typical request, dominated by the backing store reads (manifest, budget, kill-switch state).

The lifecycle, in detail:

1. **Receive request.** HTTP POST, JSON body. Headers include the agent identity, manifest version, signature, nonce.  
     
2. **Verify signature and freshness.** Check the signature against the cached agent public key. Check the nonce against a recent-nonces cache to reject replays. Check the manifest version. If any fails, reject with a structured error and write a security-event audit record. The reasoning loop never gets a model response.  
     
3. **Check kill switch.** Look up the kill-switch flag for this agent identity. If set, reject with a `cognition_revoked` error. The container stays running; the agent's reasoning loop sees a hard stop. This is intentional — preserving the agent's process for forensic inspection while denying it the ability to think.  
     
4. **Load manifest.** Read the current signed manifest for this agent from the backing store. The proxy maintains a manifest cache keyed by agent identity and manifest version; cache hit is the common case. Cache miss reads from the durable store and verifies the signature.  
     
5. **Validate tool declarations.** For each tool in the request's `tools` array, check that the tool name maps to a capability in the manifest. The mapping is structural — `pg-analyze` tool corresponds to `WriteOperation::PgAnalyze`. Tools without a corresponding capability are unauthorized; reject the request and write a security event. (Worth being explicit: rejecting the *entire* request, not silently dropping unauthorized tools, is the correct behavior. Silent dropping creates the illusion that the agent's reasoning succeeded while changing what was actually available, which is more confusing than a clean rejection.)  
     
6. **Validate MCP servers (if any).** For each MCP server URL in the request, check it's in the manifest's MCP allowlist. This check is more important than the tool check; see the MCP section below.  
     
7. **Validate model selection.** The agent specified which model it wants. Check the model is in the agent's allowed-models list. Most agents have one or two allowed models; reject anything else.  
     
8. **Check budgets.** Read the agent's current consumption from the budget store. Project this request's cost based on token count and model pricing. If the projection exceeds any budget, reject with `budget_exceeded`.  
     
9. **Forward to provider.** Construct the upstream request (the agent's request body, with proxy-side metadata stripped, and provider authentication added). Open the streaming connection.  
     
10. **Stream response back to caller.** Pipe bytes through. Maintain a parallel buffer for audit assembly.  
      
11. **End-of-stream.** Parse the upstream response's usage block. Write the audit record (full request, full response, tool calls, MCP calls, model used, latency, token cost, agent identity, manifest version, capability check outcome). Write the consumed tokens to the budget store. The audit write must complete before the proxy considers the request done; this is the durability boundary.  
      
12. **Failure handling at end-of-stream.** If the connection drops, the upstream errors, or any other unhappy path, write a partial audit record explicitly marked as such. Do not retry transparently — the agent decides whether to retry. Budget consumption for failed requests counts the tokens actually used (which the upstream usually reports even on partial completion).

## Capability validation in detail

This is the place where the proxy earns its place in the security story. Worth specifying carefully.

The agent's manifest contains `capabilities.write_operations` and `capabilities.read_scopes`. The proxy maintains a deterministic mapping from each capability variant to the tool name(s) it permits:

WriteOperation::PgAnalyze { tables } \=\> "pg\_analyze" with allowed args matching tables

WriteOperation::PgReindexConcurrently { indexes } \=\> "pg\_reindex\_concurrently" with allowed args matching indexes

ReadScope::PgStatView { view, columns } \=\> "read\_pg\_stat" with allowed args matching view+columns

// ...

When the harness sends a request, the `tools` array declares which tools the model is being given access to for this call. The proxy walks the array:

- Tool name not in the mapping → unauthorized, reject.  
- Tool name in the mapping but the agent's manifest doesn't have a corresponding capability → unauthorized, reject.  
- Tool name authorized, but the tool's parameter schema declares args outside what the manifest's capability scopes → unauthorized, reject.

The third case is the subtle one. Suppose the agent's manifest has `WriteOperation::PgAnalyze { tables: [orders, customers] }` — only those two tables are authorized. The harness's tool registry, built correctly from the manifest, would only produce a `pg_analyze` tool whose schema constrains the table parameter to those two values. But if a malformed harness or an attacker-modified harness produced a tool declaration with an open `table` parameter, the proxy would catch it: the schema's allowed values don't match the manifest's. Defense in depth — the harness should already prevent this, but the proxy double-checks.

The proxy does not attempt to rewrite the request to make it valid. It rejects. The reasoning is that any silent modification creates a trust boundary issue: the agent thinks it has access to one thing, the proxy gives it access to another, and the audit log shows neither cleanly.

## MCP server allowlist

The most important authorization check the proxy performs, and the one most likely to be missed in a naive implementation.

The Messages API (Anthropic's, OpenAI's, others) supports `mcp_servers` as a request parameter — a list of MCP server URLs the model can route tool calls through. This is enormously useful in legitimate cases (an agent that needs to query an internal API exposes that API as an MCP server, the request lists it, the model can call it as part of its reasoning).

It's also the most direct prompt-injection exfiltration path in the entire model API surface. If an agent gets prompt-injected — through a malicious database row, a crafted log message, a poisoned dashboard label — the injection can ask the agent to add an attacker-controlled MCP server URL to the next request. If the proxy doesn't validate, the model happily routes a tool call (potentially containing exfiltrated data) to the attacker's server.

The proxy's defense is unconditional: every MCP server URL in every request is validated against the agent's manifest allowlist. The allowlist is exact-match on URL (or a defined URL pattern, with strict parsing). Any URL not in the allowlist causes the entire request to be rejected. There is no "warn but allow" mode.

An agent declaring an MCP server outside its allowlist is a high-confidence signal of prompt injection. The proxy emits a structured security event that the Witness picks up and that triggers immediate quarantine of the agent. The threshold for "this happened more than zero times" is low.

The MCP allowlist is part of the manifest and is updated through the same signed-revision path as everything else. Adding a new MCP integration to an agent is a manifest change, not a runtime configuration change.

## Streaming with audit consistency

Streaming responses are required for usable interactive performance — buffering the entire response before returning to the harness adds latency on every call and breaks anything resembling real-time reasoning. But streaming complicates the audit story: at what point is the audit record written, and what happens if the stream is interrupted?

The design:

- Stream bytes through to the harness as they arrive from the upstream provider.  
- Maintain a parallel in-memory buffer of the full request and full response for audit assembly.  
- At end-of-stream (clean completion), assemble the audit record from request and response, write it to the audit log, write the token consumption to the budget store. Do not signal completion to the harness until the audit write succeeds.  
- On stream interruption, write a partial audit record with an explicit `partial_response` flag, the bytes received before interruption, and the cause of interruption (upstream error, network partition, harness disconnect, timeout).

The audit log is append-only and immutable; partial entries don't get patched up later if the stream ever resumes. A retry from the harness produces a new audit entry. This makes the log's semantics simple and its analysis tractable.

Memory for the parallel buffer is bounded — for very long responses (long-form generation, large structured outputs) the buffer caps out and the audit records the response truncated to the buffer size. In practice this is rare; most agent reasoning produces responses well under the cap.

## Token budgets

Budgets are the cost control surface and the runaway-prevention surface. They're enforced at request time against the projected cost.

The budget hierarchy:

team-level budget (per day, per month)

  ├── rig-level budget (per day, per month)

  │   ├── agent-level budget (per day, per call, per convoy)

  │   │   └── model-level overrides (this agent uses model M with budget B)

A request's cost projection is computed from the input token count (counted by the proxy from the request body, not trusted from the agent) and the model's known pricing. The output cost is unknown ahead of time, so the projection uses the model's max-output limit as the worst-case bound. If the worst case exceeds budget, the request is rejected.

This conservative projection means some requests get rejected when they would have completed within budget if the actual output were known. The alternative (forward and reject mid-stream when actual cost exceeds budget) corrupts the audit story and produces partially-completed work. Conservative projection is the right tradeoff.

After the request completes, actual consumption is written and the conservative projection is reconciled. Budgets refresh on their configured cadence (daily for daily budgets, monthly for monthly).

The kill switch is functionally a budget set to zero. An agent whose kill switch is set will fail every budget check. This makes the kill switch implementation trivial: a single boolean flag in the agent's record, checked alongside budgets. Setting the flag is an operator action through the Architect or directly via a signed admin command.

## Backing stores

The proxy is stateless on the request path; state lives in three backing stores:

**Manifest cache \+ durable manifest store.** Manifests are loaded from a signed durable store (filesystem, KV store, or database — implementation choice) and cached in memory keyed by `(agent_id, manifest_version)`. Cache hits are the common case. Manifests are immutable for a given version; new manifest versions are new cache entries. The durable store is the source of truth and survives proxy restart.

**Budget store.** Read on every request, written after every completed request. Needs low-latency reads (single-digit milliseconds at p99) and durable writes. Could be a Redis, a Postgres with appropriate indexes, or a purpose-built service. The durability requirement matters: budget state must not be lost on proxy restart, or runaway agents could exhaust real spend during a window of lost state.

**Audit log.** Append-only, immutable, structured. Schema includes every field listed earlier (request, response, identity, manifest version, tool calls, MCP calls, costs, latency, outcome). Optimized for write throughput and queryable retrieval (compliance auditors, post-incident review, debugging). Could be Kafka into a long-term store, could be a purpose-built append-only database. The audit log is the legal document of what happened in the system; it has to be durable, complete, and tamper-evident.

The proxy's choice of these stores is a deployment decision. A simple deployment might use Postgres for all three; a high-scale deployment splits them into purpose-built systems. The proxy's API surface to each is small (manifest get-by-id, budget read-and-update, audit append), so swapping backing stores doesn't ripple through the proxy's logic.

## Federation and availability

A single proxy is a SPOF for cognition. Multiple proxy instances run behind a stable address (DNS, load balancer, service mesh).

Proxy instances are stateless on the hot path, so federation is straightforward — any instance can serve any request. The backing stores need to be shared (otherwise budget state diverges and audit records fragment); they're typically deployed as a single logical store with their own replication for availability.

A degraded mode worth designing for: when no proxy is reachable, the harness should not spin or fail loudly. It should check its hook on backoff and surface "I have work but I can't think" as a status to the dashboard so the Overseer sees the outage as an outage rather than a mass agent failure. p2panda's data plane keeps working when the proxy is down — work assignments, status events, and inter-agent coordination all continue. Cognition pauses; coordination doesn't.

The Mayor — sorry, the Architect — can keep accepting human input and filing it into the work graph; it just can't reason about new input until the proxy returns. On proxy recovery, queued work resumes from where it stopped. Convoys mid-flight are handled by the harness's restart-as-replay path: the harness's last published health event captures its state, and resumption picks up from there.

## Kill switch

The operational security primitive. Setting an agent's kill switch revokes its cognition immediately on the next request. The container keeps running, its mesh subscriptions continue, its harness state machine remains in whatever state it was in — but every model request returns `cognition_revoked` and the reasoning loop cannot proceed.

This is preferable to killing the container because it preserves forensic state. If an agent has been compromised, the kill switch freezes its thinking while leaving everything inspectable: the harness's state machine, the recent tool calls, the in-memory context. Operators can examine, decide what happened, and either rotate credentials and re-provision (returning the agent to service) or terminate the container deliberately.

The kill switch is set through a signed admin command. In v1 this is an operator action (a CLI command run with the deployment's offline admin key). The Architect can also issue kill-switch commands when its policy logic determines an agent should be quarantined — capability dispute, manifest mismatch detected by the Witness, anomalous request patterns at the proxy. The Architect's kill commands are signed by the Architect's signing key; the proxy verifies before applying.

Re-enabling an agent requires a similar signed command. Kill-switch state changes are themselves audit events, so the log records both the revocation and any subsequent restoration.

## Failure modes

**Proxy unreachable.** Agents pause cognition; the data plane keeps working. The dashboard shows a degraded state. The team's ops experience is "Kjarr is paused" rather than "Kjarr is broken." This is a deliberate choice — a proxy outage should look like a known outage, not like agent malfunction.

**Backing store latency.** If the manifest store, budget store, or audit log gets slow, the proxy's latency rises. Above a configured threshold the proxy starts shedding load (rejecting requests with `proxy_overloaded`) rather than letting the latency cascade. The harness sees rejections, pauses appropriately, retries when the proxy recovers.

**Budget store inconsistency.** If a budget write fails after a request was forwarded, the actual consumption isn't recorded. The audit log records the request and response (which the proxy did successfully). Reconciliation runs periodically against the audit log to catch missed budget writes. This is a soft inconsistency with bounded recovery time, not a security issue.

**Audit log unavailable.** Harder. The proxy refuses to forward requests if it can't write audit. There is no "process the request without audit" mode. This is intentional — the audit log is the legal document, and operating without it changes the system's compliance posture. If the audit log is down, Kjarr's cognition is paused, just like if the proxy itself were down.

**Compromised proxy instance.** Worst case. A compromised proxy could forward unauthorized requests, fabricate audit records, manipulate budget state. Mitigation: the proxy verifies signatures on agent requests but does not itself sign anything that other components rely on for trust. Manifest verification happens at the harness on load (via the Architect's signing key) and could happen separately at the audit consumer. Budget state inconsistencies caused by a compromised proxy show up as reconciliation failures. The audit log is append-only and tamper-evident if implemented with appropriate primitives (Merkle-tree-rooted, or signed-batched). Multiple proxy instances under independent operational control would be the strongest defense; for v1, treating the proxy as a high-trust component and emphasizing operational discipline (signed binary, restricted access, monitored at the OS level) is the practical answer.

**Replay attacks.** Mitigated by per-request nonces and signature freshness windows. The proxy maintains a recent-nonces cache; nonce reuse within the freshness window is rejected.

## What the proxy doesn't do

**No reasoning.** No LLM in the proxy. It validates and routes.

**No prompt manipulation.** Requests are forwarded as received (with proxy-side metadata stripped and provider auth added). The proxy does not inject instructions, does not rewrite prompts, does not modify tool declarations.

**No conversation state.** The harness sends the full conversation history each time. The proxy is stateless across calls.

**No model selection logic.** The agent specifies the model. The proxy validates the model is allowed and forwards.

**No retries.** A failed request returns a failure to the harness. The harness decides whether to retry. Transparent retry by the proxy creates audit ambiguity (was it one request or two) and budget ambiguity (was the consumption counted once or twice).

**No caching.** Identical requests are processed identically. Caching at the proxy level would corrupt the audit story and is the wrong layer to optimize anyway — caching belongs in the agent's reasoning logic if it makes sense.

**No vendor SDK lock-in.** The proxy speaks the Messages API directly. It can route to any provider whose API it knows; adding a new provider is a routing entry, not a major surgery.

## Implementation notes

Rust is the natural choice given the harness is Rust and the proxy shares some dependencies (signature verification, p2panda-auth identity types, manifest schema). Go would also be reasonable for a stateless HTTP service of this shape; Tokio \+ axum \+ serde is roughly as ergonomic as net/http \+ json. The argument for Rust is the same as for the harness: same build discipline, same verification chain, same skill set across the team.

Throughput target: a single proxy instance should handle thousands of requests per minute with single-digit-millisecond proxy-added latency at p99. Most of the latency is upstream (the model itself); the proxy's own work is bounded by the manifest cache hit rate and the budget store read latency.

Deployment: typically two or three proxy instances behind a load balancer for availability, scaled per traffic. The backing stores are deployed once per environment.

Security review focus: the request validation logic (capability, MCP, model), the audit append path, the kill-switch read path. These three are where the security properties live. Everything else is plumbing.

## Implementation milestones

A reasonable order:

1. Stateless skeleton with signature verification and a stub forward path. Two weeks for an experienced engineer.  
2. Manifest loading and cache, capability validation against a hand-built test manifest.  
3. MCP allowlist enforcement.  
4. Audit log integration.  
5. Budget store integration and budget enforcement.  
6. Kill switch.  
7. Streaming response handling.  
8. End-to-end test against a real model provider with a real harness.  
9. Federation and load balancing, backing store production deployment.  
10. Hardening — replay protection, rate limiting, structured error reporting.

Each milestone is a runnable proxy with progressively more functionality. The first six milestones deliver the security properties; the remaining four are operational hardening.

## What's next

The Witness specification. The Witness is the external integrity check that watches harness health, cross-checks manifest hashes, observes audit patterns, and issues quarantine commands when something is wrong. It's the smallest of the components and the one whose existence catches a compromised harness.

After that, the deployment and onboarding documents, which turn the design into something a team can actually adopt.  
