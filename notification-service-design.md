# Notification service — design

*The notification service is the thin shim between Kjarr's mesh events and the vendor messaging APIs (Slack, PagerDuty) the team already uses. It is intentionally boring, narrowly scoped, and the smallest possible component that bridges Kjarr's internal world to the team's external world.*

## Role

Every other agent publishes mesh events. The notification service is the first concrete consumer of those events on the human-facing side. When the Architect publishes a `convoy_started` event, somebody has to translate that into a Slack message in `#sre-prod`. When a convoy enters fallback (post-page assistance) mode, somebody has to comment on the existing PagerDuty incident. The notification service does that work, and only that work.

The motivation for separating it from the Architect — and from every other agent — is the same motivation that runs through the rest of the design: the agent doing reasoning shouldn't also be doing vendor SDK plumbing, and the component talking to vendor APIs shouldn't also be doing reasoning. Different jobs, different blast radii, different capability profiles, different failure modes.

The notification service is the Kjarr analog of OpenClaw's Gateway, stripped down. Where OpenClaw's Gateway is a feature-rich, plugin-driven channel orchestrator, Kjarr's notification service is a small, opinionated event-to-API translator that knows about exactly the channels the deployment has configured and nothing else. There is no skill marketplace. There are no plugins loaded at runtime. The service does what its capability manifest says it does, and nothing more.

## Capability profile

The notification service is not an LLM-driven agent. It does not have model access. It is a small Rust service (or could be — see implementation notes below) that subscribes to mesh topics, processes events deterministically, and calls vendor APIs.

The grants:

**Mesh subscriptions.** Specifically the `notify.*` topics published by the Architect, plus the convoy lifecycle topics it needs context from. No subscription to domain agent topics, anomaly topics, or hypothesis topics. The notification service should never need to reason about why a convoy is what it is — it just needs to translate convoy state into messages.

**Slack API access, narrow scope.** The token grants permission to post messages and reactions in a configured allowlist of channels. No permission to read channel history, no permission to modify channel configuration, no permission to invite or remove users, no permission to act in any channel outside the allowlist. The harness validates every API call against the allowlist; out-of-scope calls are rejected before reaching Slack.

**PagerDuty API access, narrower scope.** The token grants permission to post comments and notes on existing incidents in configured services. *No* permission to create incidents, *no* permission to acknowledge incidents, *no* permission to resolve incidents, *no* permission to modify incident severity or assignment. This is the most security-relevant scope in Kjarr's entire design, because PagerDuty's discipline is what teams rely on for paging integrity. The harness rejects any PagerDuty operation outside comment-on-incident.

**No mesh publishing other than its own health.** The notification service publishes `health_status` events for its own observability, but does not publish anything else to the mesh. It is a pure consumer.

**No model access, no proxy access.** This is what makes the notification service structurally different from every other Kjarr component. The translation from event payload to message text is deterministic — templates, not LLM generation. There is no path from a compromised notification service to model exfiltration or prompt injection of any other agent.

## Why no LLM?

This is the design choice that takes the most explanation. Every other component in Kjarr involves LLM reasoning. The notification service does not.

The reason is asymmetric risk. The notification service has direct write access to Slack and PagerDuty — channels the team relies on for actual on-call workflow. A compromised LLM-driven notification service could craft messages designed to mislead, to suppress, or to spoof. A template-driven service can only write what its templates produce, with the variables filled in from the event payload. The blast radius of a compromised template-driven notifier is "messages with corrupted variable substitution"; the blast radius of a compromised LLM-driven notifier is "anything the LLM can be persuaded to say."

There is genuine value loss in this choice. An LLM-driven notifier could write more natural-sounding messages, summarize convoys more eloquently, adapt tone to context. The Slack messages from Kjarr will read a little stilted because they're filled-in templates. That's a fair trade. The team can read terse template messages just fine; they cannot recover trust if Kjarr's Slack output one day gets prompt-injected into something embarrassing or harmful.

If natural-language summaries are wanted, the right pattern is for the *Architect* to produce a structured summary as part of the convoy close event (under its own audit and capability constraints), and for the notification service to use that summary verbatim in its template. The reasoning happens behind the proxy with full audit; the writing-to-Slack happens with no model in the path.

## Templates

Each event topic the notification service consumes has one or more templates associated with it, configured per-deployment. A typical setup for the canonical convoy lifecycle:

event: convoy\_started

channels: \[slack:\#sre-prod\]

template: |

  🔬 \*\*Kjarr — pre-emptive intervention proposed\*\*

  

  Watching \`{{service}}\` {{metric\_human}} {{trend\_description}}.

  

  Hypothesis: {{candidate\_cause}}. Confidence {{confidence}}.

  

  Plan: \`{{operation}} {{target}}\` (\~{{duration\_estimate}}, {{reversibility}}).

  

  {{approval\_window\_message}}. Convoy \`{{convoy\_id}}\`.

The `{{...}}` substitutions are the typed fields from the event payload. The template engine has no access to anything not in the payload, no ability to call out to other systems, no escape hatches. If a template references a field that isn't in the event payload, the template fails to render and the notification service publishes a health event flagging the misconfigured template — better to fail loudly than to send a message with `null` or `undefined` in it.

Templates are configuration, signed by the Architect (same trust path as approval envelopes). Changing a template requires a manifest revision. The team can edit Kjarr's voice without touching code; an attacker who compromises the configuration plane cannot inject arbitrary template content because the template files are signed.

For PagerDuty incident comments, the template is similar but renders to PagerDuty's incident-note format. The deployment configures which PagerDuty service IDs are addressable and the mapping from convoy events to comment templates.

## Channel routing

A single event can fan out to multiple channels depending on routing rules. A convoy starting in pre-page mode might post only to Slack; a convoy escalating after a falsified hypothesis with imminent SLO violation might post to Slack *and* comment on the relevant PagerDuty incident (if one exists; if not, just Slack). Routing rules are signed configuration in the manifest, with conditions matching event fields:

event: convoy\_started

when: tier \== "pre-page" && approval\_status \== "pending"

route:

  \- slack: \#sre-prod

    template: pre-page-intervention-proposed

event: convoy\_escalated

when: tier \== "pre-page" && reason \== "hypothesis\_falsified"

route:

  \- slack: \#sre-prod

    template: brewing-falsified-imminent

  \- pagerduty: 

    when: incident\_exists\_for\_service

    service\_id\_from\_topology: true

    template: brewing-falsified-pd-comment

Routing is deterministic, configured, and inspectable. There is no "smart" routing that might surprise the team.

## Failure modes

**Channel API outage.** Slack or PagerDuty is unreachable. The notification service queues events with bounded retention (the bound matters — unbounded queues are a denial-of-service vector if the API stays down) and retries with backoff. If the queue fills, the service publishes a `health_status: degraded` event, drops events with the lowest priority first, and surfaces the situation through the dashboard. Kjarr's internal mesh and convoy state are unaffected; only the human-facing channel is affected.

**Template misconfiguration.** A template references a field that doesn't exist in the event payload, or has malformed substitution syntax. The service fails to render the message, publishes a health event with the template ID and the missing field, and falls back to a minimal "convoy state changed, see dashboard" message so the team isn't left without any notification. Better to send something terse than to send something corrupted.

**Compromised credentials.** The Slack or PagerDuty token is leaked. The blast radius is "an attacker can post messages or comments using Kjarr's bot identity, in the configured channels." This is bad but recoverable: rotate the token, audit the messages Kjarr posted during the compromise window. The token's narrow scope (no incident creation, no channel modification) means the attacker cannot use Kjarr's identity to suppress real pages or fabricate fake ones — the worst they can do is post misleading messages, which a human reading carefully will catch. Combined with the Architect's signed convoy events being separately verifiable, an attacker cannot use Kjarr's notification path to forge convoy outcomes.

**Compromised notification service binary.** Worst case for this component. The service can post arbitrary template-rendered messages with arbitrary substituted content within the templates, but cannot escape the templates, cannot call APIs outside its allowlist, cannot read mesh topics outside its subscription. Recovery is to redeploy from a verified binary build and rotate the API tokens. The Witness's behavioral checks should detect a compromised notification service emitting events at anomalous rates or with anomalous content distributions.

**Slack rate limiting.** Slack imposes per-channel rate limits, and Kjarr could plausibly hit them during incident floods. The service buffers and respects rate limits; events that exceed sustained capacity get dropped (lowest-priority first) with a health event flagging the drop. Operators tune the message volume by adjusting which events route to which channels.

## Implementation notes

This is the part of Kjarr most likely to be implemented as something other than a Rust binary — a small Go or Python service, or even a Lambda function, would be defensible because the security model doesn't depend on the language guarantees the way the agent harness does. There's no model access, no capability manifest enforcement at the language level, no state machine that benefits from typed enums.

That said, my recommendation is still Rust. The reasons:

The notification service is in the trust chain — its outputs reach the team's actual on-call workflow. Implementing it in the same language as the harness with the same build discipline (reproducible builds, signed binaries, version-pinned dependencies) means the Witness can verify it the same way it verifies any other Kjarr component. Mixing languages adds a verification surface that needs separate tooling.

The mesh subscription and event-handling code is naturally typed, and Rust's serde plus the same event-schema crate the rest of Kjarr uses means the notification service consumes events with the same type safety the publishers produce them with. Schema mismatches are compile errors, not runtime template failures.

The service is small enough that Rust's ergonomic costs aren't a real burden. We're talking about an event consumer with template rendering and HTTP POSTs. Tokio for async, reqwest for HTTP, handlebars or askama for templates. Maybe 1500 lines.

If a team has a strong preference for Go or Python here, the design works either way; the binding requirement is that the implementation is verifiable by the Witness and respects the manifest scoping at runtime.

## What the notification service doesn't do

**No reasoning about message content.** Templates render structured event payloads into text. The service does not summarize, paraphrase, or interpret.

**No bidirectional channels.** Inbound Slack messages or PagerDuty acknowledgments do not flow back through the notification service to Kjarr. Approval flows go through the dashboard or signed CLI; Slack reactions to approval-pending messages are the *one exception* and only because the reaction-handling can be made deterministic and narrow (the harness validates that the reacting user is on the on-call list, validates that the reaction is on a current approval-pending message, validates the reaction emoji against the allowed set). v1 can defer even this; the safer option is to require all approvals to come through the dashboard.

**No inference about which channel to use.** Routing is configured. If the team wants different channels for different services, that's a routing rule the operator writes.

**No history.** The notification service does not maintain a record of past notifications. The mesh log is the source of truth; the service is a stateless transformer (modulo its retry queue).

**No vendor support beyond Slack and PagerDuty in v1.** Adding email, Teams, Discord, Telegram, and so on is conceptually straightforward (each is a new template family and a new API client) but it's deliberately out of scope for v1. Two channels handle 95% of SRE workflows; expanding the surface area dilutes the security review that v1 needs.

## What's next

Refinery next — the boring lease-service version, deferring the LLM-driven smart-arbiter version to v2 or beyond. After that, the harness specification, which is where the design moves from prose into types and the implementation can actually start.  
