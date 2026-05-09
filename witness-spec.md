# Witness — specification

*The Witness is the external integrity check. It watches the events the harness and proxy publish, cross-checks claims against the Architect's signed records, detects behavioral anomalies, and issues quarantine commands when something is wrong. The smallest of Kjarr's trust-bearing components, and the one whose existence catches a compromised harness.*

## What the Witness is, and why it exists

The harness self-reports its health and self-enforces its manifest. The proxy validates every model request against the manifest and rejects unauthorized calls. Together those two components handle most of the security model. So why have a Witness at all?

Because a compromised harness's self-reports are unreliable, and the proxy can only catch what comes through it. If a harness binary has been replaced with a malicious one — through supply-chain compromise, through a privilege escalation in the host OS, through an attacker who got root on the container — its health events become whatever the attacker wants them to be. The proxy still enforces the manifest at the cognition boundary, but the harness might be doing things outside cognition that the proxy doesn't see.

The Witness is the answer to "who watches the watcher." It is independent of the harness, runs separately, and verifies harness claims against external sources of truth. If the harness lies, the Witness catches the lie by comparing what the harness says to what other components have observed.

The Witness is deliberately small. It has no model access, no infrastructure access, no agent role. It is a watcher and an alarm — its only output is health observations and quarantine commands. Every responsibility added to the Witness expands its blast radius and weakens its credibility as an independent check.

## Core responsibilities

Five things the Witness does:

**Manifest verification.** The harness publishes the hash of the manifest it loaded at startup as part of every health event. The Witness compares this hash against the Architect's signed manifest record for the same agent identity. If they diverge, the Witness has caught either a manifest tampering or a stale-cached harness; either way, quarantine is the right response.

**Binary attestation.** The harness publishes its binary build hash as part of every health event. The Witness compares this against the deployment's allowed-build-hashes list. A harness running an unrecognized build is quarantined immediately — this catches supply-chain compromise where an attacker has replaced the binary on disk.

**Heartbeat monitoring.** The harness publishes health events on a 30-second heartbeat. The Witness flags missed heartbeats and, after a configurable threshold (default 3 missed heartbeats \= 90 seconds of silence), publishes an alert. Sustained silence triggers quarantine.

**Behavioral anomaly detection.** The Witness observes the rate, shape, and content distribution of events the harness publishes — hypotheses produced, tool calls executed, confidence levels asserted. It compares these against expected baselines for the agent role. Sustained deviation triggers an alert; severe deviation triggers quarantine. This is the most sophisticated of the Witness's checks and the one most likely to evolve over time.

**Quarantine authority.** The Witness can publish signed quarantine commands. The harness, the proxy, and the Architect all respect quarantine commands from the Witness — the harness suspends its reasoning loop, the proxy blocks the agent's cognition requests, and the Architect routes work elsewhere.

That's the entire job. Everything else lives in other components.

## What the Witness watches for

The specific checks, with the corresponding signal each represents.

**Manifest hash mismatch.** Harness claims to be running manifest version V with hash H; the Architect's signed record for this agent has manifest V with hash H'. Either the harness is running a tampered manifest or its claim is fabricated. Action: quarantine, alert.

**Binary hash mismatch.** Harness reports binary hash B; allowed-build-hashes list does not include B. Either supply-chain compromise or an unauthorized build has been deployed. Action: quarantine, alert with high severity.

**Manifest version stale.** Harness claims to be running manifest version V; the current signed manifest for this agent is version V+N. The harness hasn't picked up a recent manifest update. Less alarming than hash mismatch (could be a normal in-progress rollout) but worth flagging if sustained beyond a configured grace period.

**Heartbeat missed.** No health event within the configured window. After three misses, alert. After ten misses, quarantine — the agent is either dead, network-partitioned, or compromised in a way that prevents heartbeats. Quarantine is safe in all three cases.

**Heartbeat liveness mismatch.** Heartbeats arriving but with stale or duplicate timestamps. The harness might be replaying old health events rather than producing new ones. This is a sophisticated compromise pattern but worth detecting; mitigation is to require the heartbeat to include the most recent mesh event ID the harness has acknowledged, which an attacker can't easily fabricate without keeping up with the live mesh.

**Tool-call rate anomaly.** Agent of role R typically executes between L and U tool calls per hour at this time of day. Sustained rate outside \[L, U\] is flagged. This catches both runaway loops (rate too high) and silent compromise where the agent stops doing useful work (rate too low). The bounds are learned from per-role historical data and tightened over time.

**Hypothesis pattern anomaly.** For roles that produce hypotheses (`pg-prod`, `mon`, etc.), the distribution of hypothesis types, confidence levels, and outcomes is observed. Sudden shifts — every hypothesis is suddenly high-confidence, or every hypothesis points at the same root cause regardless of evidence, or hypothesis rate spikes without corresponding mesh events that would justify it — are flagged.

**Confidence calibration drift.** The harness's confidence-calibration table maps the model's structured rationales to actual confidence numbers based on track record. The Witness independently maintains its own observed track record and compares: if the calibration table is producing confidence numbers that are systematically higher (or lower) than the actual outcomes warrant, that's drift. Sustained drift is flagged for operator review; severe drift triggers quarantine because it suggests the calibration table or the prompt has been tampered with.

**MCP allowlist violations forwarded from the proxy.** When the proxy rejects a request because the agent declared an MCP server outside its allowlist, the proxy publishes a security event. The Witness sees this and treats it as high-confidence prompt-injection signal — quarantine the agent immediately, alert with high severity. The threshold for "this happened more than zero times" is low; legitimate operation should never produce these events.

**Capability dispute resolution.** Two agents disagreeing on the state of shared infrastructure (`pg-prod` says the analyze worked, `mon` says the metric trend didn't reverse) publish dispute events. The Witness logs the dispute and surfaces it to the Architect, which makes the routing decision. The Witness itself doesn't resolve disputes — it's an observer, not an adjudicator.

## Detection mechanisms

The Witness's checks are mostly rules-based, not LLM-driven. This is deliberate.

Rules-based checks are deterministic, debuggable, and not themselves susceptible to prompt injection. The Witness is supposed to be a stable cross-check; introducing an LLM into its decision logic would create the same prompt-injection surface the rest of the system is trying to avoid.

The rules are configurable per deployment, with reasonable defaults. Per-role baselines (tool-call rates, hypothesis distributions) are learned from observed history and re-baselined periodically. Per-deployment thresholds (heartbeat windows, drift tolerances) are configured in a signed Witness configuration file, loaded at startup, updated through the same signed-revision path as other configuration.

The role of LLM augmentation, if any, comes later. A v2 enhancement might use an LLM to characterize "is this hypothesis pattern weird in a way that suggests prompt injection" — a deeper semantic check than rate-based rules can achieve. But that LLM would be running outside the live decision path, producing observations for human review rather than triggering automated quarantine. The automated quarantine path stays rules-based for v1 and v2 because the cost of a false positive is operational disruption and the cost of false negative is bounded by the harness and proxy's primary defenses.

## Quarantine

Quarantine is the Witness's only state-mutating action. Issuing one is significant; the design has to make it both available (when needed) and safe (against false positives that disrupt operations).

A quarantine command is a signed mesh event addressed to a specific agent identity. It contains:

- The target agent's identity  
- The reason for quarantine (manifest mismatch, heartbeat lost, behavioral anomaly, MCP violation, etc.)  
- The evidence — the specific events or conditions that triggered the command  
- A timestamp and the Witness's signature

The harness, on receiving a quarantine command for itself, transitions to the `Quarantined` state immediately and suspends its reasoning loop. The proxy, on seeing the quarantine event in its mesh subscription, marks the agent's identity as cognition-revoked. The Architect routes work away from the quarantined agent.

Lifting quarantine requires explicit operator action — a signed admin command countersigning the quarantine and authorizing return to service. The Witness cannot lift its own quarantines; that asymmetry is intentional. Quarantines are easy to issue, hard to lift, and fully audited.

Threshold for quarantine vs. alert: the Witness uses two levels.

- **Alert** is for conditions that are concerning but not yet conclusive. Single missed heartbeat, mild rate anomaly, manifest version stale within grace period. Alerts go to the dashboard and Slack via the notification service; the agent continues operating.  
- **Quarantine** is for conditions where the agent should not be trusted. Manifest hash mismatch, binary hash mismatch, sustained heartbeat loss, MCP allowlist violation, severe behavioral anomaly. Quarantine is automatic when the rules trigger, and the agent is offline until the operator countersigns.

The split between alert and quarantine is configurable, with conservative defaults. New deployments err toward alert (quarantine is disruptive, and false positives during initial calibration are bad UX); mature deployments tighten toward quarantine as the team's confidence in the rule set grows.

## Capability profile

Deliberately small. The Witness has:

**Mesh subscriptions.** All health events from all agents in the deployment, security events from the proxy, manifest update events from the Architect, the Witness's own configuration topic. No subscription to domain agent topics (anomaly events, hypothesis events, convoy lifecycle) — the Witness watches metadata about agents, not the work they do. The work-content separation matters because it limits the Witness's blast radius if compromised.

**Mesh publications.** Quarantine commands, alert events, the Witness's own health events. No other publications.

**No model access.** Rules-based, deterministic. Adding LLM access to the Witness would undermine its credibility as an independent check.

**No infrastructure access.** No Postgres, no Prometheus, no kubernetes. The Witness watches mesh events, period.

**No human-channel access.** Alerts are mesh events; the notification service translates them to Slack. Same pattern as the rest of the system.

**Read access to the Architect's signed records.** Manifest registry, binary hash list, allowed-models list, deployment configuration. These are the sources of truth the Witness compares observed reality against.

**Write access to its own quarantine command authority.** Signed by a Witness key, distinct from the Architect's signing key. The Witness key is held in the Witness's own sealed store, used only for quarantine commands.

## Federation

The Witness is a critical component but not a hot-path component — it watches and decides on a longer time scale than the proxy. Federation requirements are correspondingly looser.

A typical deployment runs two Witness instances, one as primary and one as standby. Both subscribe to the full event stream and maintain the same view of agent health. The primary issues quarantine commands; the standby observes and is ready to take over on primary failure. Failover is detected through a heartbeat between Witness instances and through quorum-confirmed leader election via p2panda-auth.

Threshold-signing for quarantine commands is a v2 hardening item. v1 ships with single-instance signing and operator-key escalation if the Witness itself becomes unavailable.

## Failure modes

**Witness is wrong.** False positive on a rule, agent gets quarantined incorrectly. Detection: the operator notices the agent is offline when it shouldn't be. Recovery: operator signs the lift-quarantine command, the agent returns to service. The Witness's logged evidence is reviewed; if the rule that triggered the false positive is too sensitive, the rule is tuned. False positives during the early operating period are common and expected; they're how the rule set gets calibrated.

**Witness is compromised.** A compromised Witness could quarantine arbitrary agents (denial of service) or fail to quarantine actual compromises (silent failure). Detection: operator review of Witness's recent quarantine pattern; any of the agents the Witness *is* watching observe and flag if the Witness's behavior is itself anomalous (yes, this is recursive — see below). Mitigation: the Witness's mesh activity is itself observable, and a separate alert path watches for Witness-quarantining-too-many-agents or Witness-going-silent-on-real-anomalies. Operator escalation paths can override the Witness directly.

**Witness watching itself.** The recursion problem. Briefly worth addressing: the Witness's own health is monitored by the Architect and by sibling Witness instances in a federated deployment. If the Witness goes silent or behaves anomalously, the Architect can quarantine the Witness through the operator-key path (admin command, not the Witness's own authority). This is intentionally awkward — quarantining the Witness should be a real event that requires operator involvement, not something automated.

**Witness offline.** No quarantine commands can be issued. Existing quarantines remain in effect. New compromises go undetected by the Witness's own checks, but the harness's self-enforcement and the proxy's request validation continue to function. The deployment is in a degraded state but not catastrophically broken; the team has a window to restore the Witness before risk accumulates.

**Sustained anomaly without quarantine.** The Witness sees something concerning but stops short of quarantine because the rule is configured for alert-only. The alert flows to Slack and the dashboard; if humans don't act, the situation continues. This is the right failure mode for ambiguous cases — the Witness shouldn't act unilaterally when the evidence is mixed, but it should be loud about what it's seeing.

## What the Witness doesn't do

**No deep inspection of agent reasoning.** The Witness sees mesh events, not prompt content or model responses. The proxy has those; if deep semantic checks on prompts are needed, they belong at the proxy or in a separate audit consumer, not in the Witness.

**No application data access.** Same as every other Kjarr component.

**No tool execution.** The Witness watches; it doesn't act against infrastructure.

**No authority over the proxy or the Architect.** The Witness can issue quarantine commands that the proxy and Architect respect, but it cannot modify their configuration or override their decisions in any other way.

**No human messaging.** Alerts go through the notification service. The Witness is mesh-only.

**No automated rule learning.** Per-role baselines are learned from observed history (with operator approval before they take effect), but the rule set itself — what to check, what triggers what threshold — is configuration. The Witness does not adapt its rules autonomously, because autonomous rule adaptation in a security check is exactly the kind of thing that would introduce the prompt-injection surface the Witness exists to defend against.

## Implementation notes

Rust again, for the same reasons as the harness and proxy: shared dependencies (signature verification, p2panda-auth identity, manifest schema), shared build discipline, shared verification chain.

Smaller binary than the harness or proxy. The core logic is event subscription, rule evaluation, and command publishing. A small state store for tracking heartbeat history per agent, a configuration file with the rules and thresholds, the signing key in a sealed store. Maybe 1500 lines of Rust for v1.

Throughput requirement is modest. The Witness consumes events at the rate the rest of the system produces them; for a typical deployment with tens of agents, that's hundreds of events per minute, easily handled by a single instance.

Security review focus is small but pointed: the rule evaluation logic, the quarantine command path, the signing key handling. The rest is mesh subscription plumbing.

## Implementation milestones

1. Mesh subscription, identity setup, signing key in sealed store. Heartbeat monitoring as the simplest rule. One week.  
2. Manifest hash and binary hash verification.  
3. Tool-call rate and hypothesis pattern anomaly detection.  
4. Quarantine command issuance and harness/proxy/Architect integration with quarantine.  
5. Calibration drift detection.  
6. Federation — primary/standby with leader election.  
7. Operational hardening — alert routing, dashboard integration, documentation.

The Witness is the last of the major specifications. The remaining documents are deployment, onboarding, and operational runbooks — turning the design into something a team can actually use.

## What's next

Deployment and onboarding documents. These are operational rather than architectural — how a team gets Kjarr running in their environment, how they migrate from existing tooling, how they tune the approval envelopes for their context, how they handle the first weeks of operation when calibration is rough. These documents are smaller and more concrete than the design specs, but they're where the rubber actually meets the road for adoption.

The architectural design is now complete in prose form. Eight components are specified end-to-end: Architect, `pg-prod`, `mon`, notification service, Refinery, harness, proxy, Witness. With the Witness, the integrity story closes — every claim about capability scoping, credential isolation, identity verification, and behavioral monitoring has a designed enforcement point.  
