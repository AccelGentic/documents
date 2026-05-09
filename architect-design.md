# Architect — agent design

*The Architect is Kjarr's central coordinator: the agent that decomposes work, routes it to domain agents, signs capability grants, and is the point of contact between humans and the rest of the system.*

## What the Architect is, and what it isn't

The Architect is a router with judgment. Most of what it does is structural: receive a request (from a human or from the mesh), decompose it into work units, decide which domain agents are competent to handle each unit, sign the capability grants those agents need, watch the convoy execute, and close it out. This is the dispatcher half of its job, and it's the part that should be boring, predictable, auditable.

A smaller and more interesting part of its job is cross-component judgment. When the use case shows `mon` raising an anomaly and `pg-prod` proposing a hypothesis, somebody has to decide whether to approve the intervention, whether the approval envelope applies, and whether the proposed action is the right one given other things happening on the fleet. No individual domain agent has the context to make that call. The Architect does, because every event flows through topics it subscribes to.

The Architect is not a domain expert. It does not reason about Postgres internals or Prometheus queries or Kubernetes scheduling. When it needs that knowledge, it asks the agent that has it. This separation is load-bearing: if the Architect started reasoning about specific infrastructure, it would need capabilities matching every domain it reasons about, and its blast radius would explode. The Architect's authority comes from the structural role it plays, not from accumulated expertise. The expertise lives at the edges, where it can be capability-bounded.

The Architect is not a notification service, not a paging service, not a dashboard. It publishes events to mesh topics; other components consume those events and translate to vendor APIs or rendered UIs. This separation lets the Architect be replaced, federated, or upgraded without touching the integration surface, and lets the integration surface be tightened down to a minimum capability scope without affecting the Architect's reasoning.

## Capability profile

The Architect's capability manifest grants more authority than any other agent in the system, which makes pinning it precisely especially important. The grants:

**Mesh subscriptions.** The Architect subscribes to every topic in the `prod-platform` group: anomaly events at all tiers, hypotheses, intervention status, agent health, capability disputes, and human input topics. It is the only agent that needs this much breadth, because it is the only agent doing cross-component reasoning. No other agent should subscribe to all topics, and the manifest enforces that asymmetry.

**Mesh publishing.** The Architect publishes convoy lifecycle events (started, approval pending, approval granted, escalated, closed), capability-grant events (issuing manifests for task-scoped agents), and human-facing notification events (the topics the notification service consumes). It does not publish to domain-specific topics — those are owned by the domain agents.

**Manifest signing.** The Architect holds an Ed25519 signing key (in p2panda-auth terms) used to issue capability manifests for task-scoped agents and to attest convoy decisions. The key is held in a sealed local store, requires harness attestation to use, and every signing operation is logged at the proxy as a distinct event-type beyond ordinary tool calls. There is exactly one Architect signing key per deployment; federation shares this key across instances via the harness's identity bootstrap.

**Model access.** The Architect uses a higher-quality model than domain agents typically need. Domain agents do bounded, narrow reasoning; the Architect does open-ended decomposition and judgment. Default to the strongest available model for the Architect, with per-call cost limits but a higher per-convoy budget than domain agents.

**No tool execution against infrastructure.** The Architect cannot call Postgres, Prometheus, terraform, kubernetes, or anything else. Its tools are mesh-publish, manifest-sign, convoy-close, request-hypothesis, and a small set of meta-operations on its own state. If the Architect needs something done in the world, it asks a domain agent. This is the cleanest possible boundary: the Architect's blast radius is "things the Architect can publish to the mesh," and nothing else.

**No direct human channel access.** The Architect does not call the Slack API, the PagerDuty API, or any other vendor messaging surface. It publishes notification events, the notification service translates. Same logic.

## Behavior

The Architect runs continuously and reacts to mesh events. There are five primary behaviors, each with a defined input and output shape.

**Decomposition.** A human or another agent posts a high-level intent: "deploy v2.41.0 to the EU region," "investigate the latency on checkout," "run the monthly reconciliation." The Architect parses the intent, identifies the rigs and domain agents involved, breaks the work into ordered or parallel units, and emits a Convoy with assigned work for each. For routine intents (deploys, scheduled maintenance, defined runbooks) the decomposition is template-driven and mostly deterministic. For ad-hoc intents the Architect uses the model to reason about decomposition, with the prompt structured to require explicit listing of the agents involved, the capability profile each will need, and the failure-tolerance shape (parallel, sequential, must-succeed-before-next). Decomposition output is a Convoy bead and a sequence of work-assignment events.

**Routing.** When a domain agent emits an event the Architect is subscribed to (anomaly, hypothesis, blocked, falsified), the Architect determines whether other agents should be pulled in, whether human attention is needed, whether the approval envelope is satisfied, and what the next step is. Routing is closer to deterministic than decomposition — the routing rules are mostly policy lookups against the team's configured envelopes — but the model is invoked when the routing decision genuinely requires judgment about cross-component implications.

**Approval gating.** Every state-mutating action proposed by a domain agent passes through an approval check. The check evaluates: the action's reversibility, the agent's confidence, whether the action is in the auto-approval envelope for the rig and tier (pre-page vs post-page), whether other in-flight Convoys have conflicting state assumptions, whether the proposing agent has been recently flagged for any kind of capability dispute. Most checks are deterministic. The few that require judgment — "is this hypothesis sufficiently distinct from the failed one we just tried, or is it the same thing in different words" — invoke the model.

**Escalation.** When the approval envelope is exceeded, when a hypothesis is falsified, when an agent reports being blocked, when a convoy crosses a duration threshold without progress, when health events indicate an agent is misbehaving — the Architect publishes escalation events. The notification service consumes these and translates to Slack messages or PagerDuty comments. Escalation events carry full convoy context so that whoever picks them up has the running investigation, not a cold notification.

**Closure.** When a convoy completes (success or failure), the Architect attests the outcome, computes the page-avoided judgment, signs the closure event, and emits the human-readable summary that lands in Slack. The page-avoided attestation is one of the most consequential things the Architect emits — it's the metric that justifies Kjarr's cost — and it's worth being explicit about how the judgment is made: the Architect compares the convoy's evidence trail against the AlertManager rules that would have fired without intervention, and attests `page_avoided=true` only when the rules would have tripped and the intervention demonstrably reversed the condition. False positives on this metric erode trust faster than almost anything else; the Architect should be conservative about claiming credit.

## Federation

A single Architect is a single point of failure for everything that flows through it: convoy decisions, capability grants, human notifications. The system needs Architect federation, and the federation model has to be defined before deployment, not retrofitted.

The clean answer is **multi-instance, leader-elect-for-decisions, distributed-for-reads**. Every region or availability zone runs an Architect instance. All instances subscribe to the full mesh and maintain a convergent view of convoy state. For any given convoy, one instance is elected leader and is the one that signs decisions; the others observe and stand ready to take over if the leader becomes unreachable.

Leader election runs via p2panda-auth group consensus, scoped to the `architect` role. Liveness is monitored by heartbeat events; if the leader misses heartbeats beyond a configured threshold, election re-runs. Convoy state is durable in the mesh, so a new leader picking up an in-flight convoy reads its state from the log and continues — same restart-as-replay semantics that domain agents use.

Manifest signing is more delicate. The Architect signing key cannot be casually replicated across instances; the harness's sealed-store guarantees would be undermined. The right model is **threshold signing**: the signing key is split via Shamir's Secret Sharing across N Architect instances, signing requires K of them to cooperate, and the threshold operation is what produces a valid signature. This means no single compromised Architect instance can issue capability grants, and federation is the security property rather than just the availability property. K of N is the kind of decision that should be set per deployment based on the threat model — typical defaults might be 3-of-5 for production, 2-of-3 for staging, 1-of-1 for development.

Threshold signing is implementation work that v1 might defer, in which case the fallback is **single-instance Architect with a hot standby that has its own signing key**, and an explicit failover ceremony when the primary is unreachable. Less elegant, less secure, faster to ship. The migration path from "single-with-standby" to "threshold federated" should be designed in from day one even if v1 only implements the simpler version, so the manifest schema and the convoy event format don't have to change later.

## Decomposition behavior in detail

Decomposition is the most prompt-dependent part of the Architect's job and the place where the model's quality shows up most. Worth specifying the behavior carefully.

The Architect maintains, per rig, a registry of the domain agents available, their declared capability surfaces, and their recent track record (success rate per intervention type, false-positive rate on hypotheses, average convoy duration). The registry is built from mesh events, not configured separately — agents announce themselves and their manifests on registration, and the Architect's view is a CRDT-converged read.

For a routine intent like "deploy v2.41.0 to checkout in EU region," decomposition is a template lookup: deploy convoys have a known shape, the involved agents are known, the order is known, the approval gates are known per stage. The model is barely invoked except to validate that the intent matches the template and to handle any rig-specific quirks.

For an ad-hoc intent like "investigate the latency on checkout," decomposition is more open. The Architect prompts the model with the intent, the rig context, the agent registry, and recent relevant events from the mesh, and asks for a structured decomposition: which agents to engage, what to ask each one, how to combine their outputs, what success looks like, what failure looks like. The output is constrained to a typed structure (so the model can't invent agent names or capability scopes that don't exist), and any decomposition that references agents or capabilities outside the registry is rejected at the type level before being acted on.

The decomposition prompt is one of the parts of the system that benefits most from being editable post-deployment. Operators should be able to add domain context, rig-specific patterns, and known-bad decompositions to the system prompt without touching code. This implies prompt-as-config, with version-pinned prompts that are signed by the Architect's signing key (so a compromised config plane can't slip in a malicious decomposition prompt) and audited at the proxy.

## Approval envelopes

The team's approval envelope is configured per-rig and per-tier. The Architect reads the envelope on every decision; envelopes are versioned and signed, just like manifests.

A typical envelope might look like:

rig: postgres-prod

pre-page tier:

  auto-approve:

    confidence: \> 0.85

    reversibility: trivial

    operations: \[ANALYZE, REINDEX\_CONCURRENTLY, kill\_query\_by\_pid\]

    impact\_max: low

    delay\_window\_seconds: 90  (slack-stoppable)

  require-confirmation:

    confidence: \> 0.70

    reversibility: trivial-or-bounded

    requires: slack-thumb-up

  reject:

    operations: \[DDL, data-write, schema-change\]

    confidence: \< 0.70

post-page tier:

  auto-approve:

    confidence: \> 0.70

    reversibility: trivial-or-bounded

    operations: \[ANALYZE, REINDEX\_CONCURRENTLY, kill\_query\_by\_pid, restart\_replica\]

    impact\_max: medium

    delay\_window\_seconds: 30

  require-confirmation:

    confidence: \> 0.50

  reject:

    operations: \[DROP, data-mutation, schema-change\]

The asymmetry between tiers is deliberate. Pre-page work needs higher confidence and longer stoppable windows because humans aren't yet aware of the situation. Post-page work can be more permissive because the team is already converging on the incident.

Envelopes are the primary lever for tuning Kjarr's autonomy. New deployments start with restrictive envelopes and earn their way to broader ones over time, as the page-avoided tally and false-positive rate establish trust. The default envelope shipped with Kjarr should be deliberately conservative — tighter than what any experienced team will eventually want — because the cost of starting too loose is asymmetric.

## Failure modes

Worth being explicit about how the Architect can fail and what happens when it does.

**The Architect makes a wrong decomposition.** It assigns work to agents that can't handle it, or misses a domain agent that should have been involved, or ordering is wrong. Detection: domain agents reject work-assignments outside their capability scope (the harness refuses), and the Architect's convoy stalls. Recovery: the Architect re-decomposes with the rejection feedback in context. If re-decomposition keeps failing, the convoy escalates to a human. This is a recoverable failure mode and shouldn't be alarming.

**The Architect's model has been prompt-injected.** A domain agent under attack might inject malicious content into hypothesis events that, when read by the Architect, manipulates its decomposition or approval logic. Mitigation: the Architect's prompt strictly distinguishes between attested mesh content (signed by the publishing agent) and arbitrary text content (which is treated as untrusted). All decision-making prompts are structured to require typed outputs against the agent registry, so an injected event can't trick the Architect into inventing capabilities or routing to non-existent agents. The Witness (the harness's external observer) cross-checks Architect decisions against expected patterns and flags anomalous routing.

**The Architect signs a manifest it shouldn't.** Either through prompt injection, model error, or compromise of the signing path. Mitigation: signing requires both the model's decision *and* the harness's policy check (which is deterministic and inspectable). The harness rejects signing requests that exceed configured limits — manifests with capabilities that no agent in the registry holds, manifests with TTLs longer than allowed, manifests issued at suspicious rate. The proxy logs every signing operation as a distinct event, and a separate audit consumer alerts on signing rate anomalies.

**The Architect goes down.** Federation handles this. The threshold-signing model means K-1 down Architects degrade signing latency but don't break it; below K, signing is paused but reads continue. The deployment's K-of-N choice determines the tradeoff. During a paused-signing period, in-flight convoys can complete (closure events don't require new signatures, only attestations), but new task-scoped agents can't be provisioned and new convoys requiring new manifests are queued.

**The Architect is compromised.** Worst case. Mitigation depends on threshold signing being in place — without it, a compromised single Architect can issue arbitrary capabilities and the system's security model collapses. With threshold signing, a single compromised instance is detected by the other Architects (whose signing requests it can't satisfy and whose decisions it can't unilaterally override), and revoked via group consensus. The compromised instance's signing share is rotated out via standard p2panda-auth member-removal flow.

The threshold-signing model is therefore not optional for a production deployment. v1 can ship without it, but the deployment should be considered dev-grade until threshold signing is operational.

## What the Architect doesn't do, and why

A few capabilities the Architect could plausibly own but shouldn't:

**Direct alerting.** Tempting because the Architect has the most context. Wrong because it would put a model-driven agent in the paging path, and AlertManager's discipline shouldn't be bypassed by an LLM's judgment. The Architect comments on incidents the team's existing alerting fires; it does not fire pages itself.

**Direct execution.** Tempting because the Architect knows what should happen across the fleet. Wrong because granting it execution capabilities collapses the boundary between coordination and action, and concentrates blast radius. Domain agents execute; the Architect coordinates.

**Long-term memory of human preferences.** Tempting because teams will want their Architect to learn their patterns. Wrong because the Architect is federated and ephemeral-state-friendly — durable preferences belong in versioned configuration the team controls, not in agent memory. If the team wants their Architect to know that they prefer ANALYZE-during-business-hours-only, that goes in the envelope file, not in the model's context.

**Capacity planning, billing, fleet topology decisions.** Tempting because the Architect could reason about all of this. Wrong because these are slower-tempo decisions that benefit from human deliberation, and the Architect's role is fast-tempo coordination of in-flight work. A separate planning surface (probably another agent type) can take these on later.

## What's next

This document fixes the Architect's role and bounds. The next step is the actual design of one domain agent — `pg-prod` is the obvious choice, since it appears in both use cases and has a tractable scope. After that, the notification service (boring but structurally important), then Refinery (lease service first, agent later if needed), then the harness specification that all of these run inside.

The Architect is the last agent that should be designed in this order, paradoxically. It's the most consequential, but its design depends on the shape of what it coordinates — and the shape of what it coordinates depends on individual domain agents being clearly specified. So the next document is `pg-prod`.  
