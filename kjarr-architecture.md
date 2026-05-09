# 

Kjarr  
*kyarr · Old Norse, thicket · a distributed agent system for production operations*  
**`Document`**` Architecture overview Status Design complete · pre-implementation Audience Engineering, Security, SRE`  
*A network of small, capability-bounded agents distributed across production infrastructure, coordinating through an encrypted peer-to-peer mesh, with cognition mediated by an audited proxy. The thicket as substrate.*  
Kjarr is built around a simple thesis: agents in production should be jailed by default, communicate only through audited channels, and act only within signed capability manifests. Where existing agent orchestration assumes a developer's laptop and trusts the agents that run there, Kjarr inverts the posture — every agent is treated as potentially compromised at all times, every action is authorized by a manifest signed at provisioning, and every cognition request flows through a single chokepoint that enforces budgets, validates capabilities, and writes the audit log.  
The system has eight components, divided into *coordinators* (the Architect), *observers* (mon), *operators* (pg-prod, with siblings to follow), *services* (Refinery, Notification Service, Proxy, Witness), and a *substrate* (the harness, which every agent runs inside). Each is documented below with its capability profile, role, and trust posture.

## ---

The system

`Substrate & control flow`  
Three planes run in parallel. The *data plane* is the p2panda mesh — encrypted peer-to-peer, append-log, decentralized authentication, every event signed by its publishing agent. The *cognition plane* is the proxy — every model request from every agent, validated against the agent's manifest, audited, budget-enforced. The *integrity plane* is the Witness — observing health events, cross-checking manifest hashes, issuing quarantine commands when something is wrong.  
Domain agents (pg-prod, mon) sit as sidecars next to the infrastructure they manage. They speak only through the mesh and reason only through the proxy. The Architect coordinates from the mesh; the Refinery serializes mutations; the Notification Service translates mesh events to a configurable set of human-facing channels — in the reference deployment, Zulip for the running team narrative, Matrix for incident commentary, ActivityPub for casual broadcast, and Signal as the team's existing paging mechanism (which Kjarr cannot write to). Humans never interact with agents directly — they read the channels, they receive pages through the team's own pipeline, they approve through the dashboard.  
Production infrastructure · external to Kjarr Postgres Aurora cluster Prometheus \+ AlertManager \+ Grafana Application services EKS, Go services, etc. EXTERNAL CHANNELS Zulip Matrix Signal Mastodon On-call engineer Sidecar layer · agents next to the things they manage pg-prod OPERATOR · POSTGRES mon OBSERVER · METRICS future agents REDIS · K8S · TF-STATE Notification service TEMPLATE-DRIVEN · NO LLM p2panda mesh · encrypted control plane append-log events · group access control · offline-tolerant · signed by publisher DATA PLANE Coordinators · running on the mesh Architect FEDERATED · SIGNS MANIFESTS Refinery LEASE SERVICE · NO LLM Witness INTEGRITY · NO LLM Cognition plane · separate from mesh LLM proxy AUDIT · CAPABILITY · BUDGETS · MCP ALLOWLIST · KILL SWITCH cognition (every model call) Model providers · external to Kjarr Anthropic · OpenAI · others ↑ subscribers and publishers ↓ subscribers and publishers all events durable in append-log SOLID LINES → MESH (encrypted, signed) DASHED ORANGE → COGNITION (audited proxy) DOTTED → INFRASTRUCTURE (sidecar to thing managed) System overview · solid lines indicate mesh traffic · orange dashed lines indicate cognition flow · dotted lines indicate sidecar relationships to managed infrastructure  
The substrate is the most important thing in this picture. Every interaction between Kjarr components — work assignments, hypotheses, status reports, capability disputes, quarantine commands — flows through the p2panda mesh as a signed, durable event. The mesh is the source of truth; everything else is a view over it. A new component subscribing to a topic gets a complete history; a restarted component recovers state by replaying. There is no shared mutable state outside the mesh.  
The cognition plane is deliberately separate. Mesh events are agent-to-agent communication; model traffic is agent-to-LLM. Routing both through the same channel would conflate two very different trust profiles. The proxy sees every model request, validates against the calling agent's manifest, enforces budgets, writes audit, and can revoke an agent's cognition by setting its kill-switch flag — all without being able to see or interfere with the agents' coordination traffic.

## ---

The components

`Eight components, four kinds`  
Each component has a tightly-bounded role and a capability profile that the harness, proxy, and Witness all enforce independently. The pattern is consistent: every agent runs inside the same Rust harness binary; every cognition request goes through the proxy; every health event gets observed by the Witness. Differences between agents come from their manifests and prompts, not from the substrate they run on.  
The *kind* field below distinguishes coordinators (orchestration), observers (read-only domain analysis), operators (read plus narrow reversible writes), services (boring, non-LLM, structurally important), and the substrate (the harness itself).

`Architect`  
`COORDINATOR · LLM-DRIVEN · FEDERATED`  
Decomposes intents into convoys, routes work to domain agents, signs capability manifests for task-scoped agents, and is the point of contact between the work graph and humans. Holds the deployment's manifest signing key (via threshold signing in production deployments). Federated for HA — multiple instances share the work graph through the mesh and elect leadership for signing decisions.  
ReasoningDecomposition, routing, approval gating, escalation, closure attestation  
AuthoritySign manifests · close convoys · publish notifications · revoke kill switches  
CannotExecute against infrastructure · talk to vendor APIs · accumulate persistent preferences  
Documentarchitect-design.md

`pg-prod`  
`OPERATOR · LLM-DRIVEN · ONE PER DATABASE`  
Domain expert on a specific Postgres cluster. Reads pg\_stat\_\* and system catalogs, runs EXPLAIN against fingerprinted queries, executes a small fixed set of reversible maintenance operations (ANALYZE, REINDEX CONCURRENTLY, kill query). Forms hypotheses about latency or contention, proposes interventions, executes when approved. The harness owns the database connection — credentials never reach the agent's reasoning loop.  
ReasoningHypothesis formation, intervention proposal, plan verification, self-watch  
AuthorityANALYZE · REINDEX CONCURRENTLY · KILL\_QUERY · VACUUM ANALYZE — typed args, allowlisted targets  
CannotRead application data · DDL · cross-database queries · raw SQL of any kind  
Documentpg-prod-design.md

`mon`  
`OBSERVER · LLM-DRIVEN · ONE PER PROMETHEUS`  
Watches the monitoring stack. Runs a brewing-tier query library on top of the team's existing AlertManager rules, looking for trends below page threshold. Most convoys originate here. Hypotheses point at problems and likely root causes; intervention proposals come from operator agents. The only state-mutating capability is creating Grafana annotations under a fixed tag prefix — three writes per convoy, on allowlisted dashboards only.  
ReasoningSelf-watch (primary), hypothesis verification, brewing detection, anomaly classification  
AuthorityPrometheus query · AlertManager read · Grafana annotation create (rate-limited, tag-prefixed)  
CannotModify AlertManager rules · silence alerts · push metrics · access application data  
Documentmon-design.md

`Refinery`  
`SERVICE · NO LLM · LEASE MANAGER`  
Serializes state-mutating operations against shared infrastructure. Domain agents request leases before executing; the Refinery grants or denies based on current holders and FIFO ordering. Lease grants and releases are signed mesh events, durable, fully audited. Stateless from a deployment perspective — restart replays the lease topic to reconstruct current state. Smart-arbiter LLM version deferred to v2.  
ReasoningNone — deterministic policy lookup  
AuthorityGrant, deny, expire, revoke leases on declared resources  
CannotTouch any infrastructure · reason about priority · preempt without explicit authority  
Documentrefinery-design.md

`Notification service`  
`SERVICE · NO LLM · TEMPLATE-DRIVEN`  
The thin shim between Kjarr's mesh events and the team's human-facing channels. Subscribes to Architect notification topics, renders templates, calls vendor APIs. Templates are signed configuration. No LLM in the path — message text is variable substitution, not generation. Reference deployment uses four open-protocol channels: **Zulip** for running narrative, **Matrix** for incident commentary, **ActivityPub** for casual broadcast, and **Signal** as the team's pager (which Kjarr *cannot write to*). Each channel has its own narrow capability scope.  
ReasoningNone — template substitution from typed payloads  
AuthorityZulip post (allowlisted streams) · Matrix message (allowlisted rooms) · ActivityPub status post · no Signal access  
CannotWrite to Signal · trigger pages of any kind · summarize freely · add new channels without manifest revision  
Documentnotification-service-design.md

`LLM proxy`  
`SERVICE · NO LLM IN PATH · COGNITION CHOKEPOINT`  
Single entry point for every model request. Verifies the calling agent's signature and manifest version, validates declared tools against the manifest, validates MCP server URLs against the agent's allowlist, checks budget, forwards to the model provider, streams the response back, writes the audit log at end-of-stream, meters consumption. Stateless on the hot path; state lives in backing stores (manifest cache, budget store, audit log, kill-switch state).  
ReasoningNone — deterministic validation and routing  
AuthorityForward, reject · revoke cognition (kill switch is a budget set to zero)  
CannotModify requests · manipulate prompts · cache responses · retry transparently · operate without audit  
Documentproxy-spec.md

`Witness`  
`SERVICE · RULES-BASED · INTEGRITY CHECK`  
External observer. Cross-checks manifest hashes against Architect records, verifies harness binary hashes against allowed-builds list, monitors heartbeats, detects behavioral anomalies (rate, distribution, calibration drift). Issues signed quarantine commands when something is wrong. Deliberately small and rules-based — no LLM, because an LLM in the integrity path would inherit the prompt-injection surface the Witness exists to defend against.  
ReasoningNone — rule evaluation against deployment thresholds  
AuthorityQuarantine any agent · alert through notification service  
CannotLift its own quarantines · reason about agent work content · execute against infrastructure  
Documentwitness-spec.md

`Harness`  
`SUBSTRATE · RUST BINARY · EVERY AGENT RUNS INSIDE`  
The single Rust binary that every agent runs inside. Owns the p2panda identity, owns the model client (talks only to the proxy), owns the tool execution boundary (holds DB and Prometheus connections, validates every tool call against the manifest, executes typed operations). The state machine is a Rust enum — illegal transitions are compile errors. Reproducible builds, signed binaries, the build hash verified by the Witness. The trust boundary that underwrites every other component's claims about capability scoping.  
RoleIdentity, manifest enforcement, tool execution, mesh I/O, model client, state machine  
Type systemManifest variants ↔ tool registry ↔ proxy authorization — same enum across all three layers  
Same binarypg-prod, mon, future agents — all run the same harness build, differentiated by manifest and prompt  
Documentharness-spec.md

## ---

Interaction flows

`Two scenarios`  
Kjarr operates in two modes that differ in posture and approval discipline. *Pre-emption* is the primary mode and the source of most of Kjarr's value: mon notices a trend below SLO, a domain agent forms a hypothesis, the Architect approves under a strict envelope, an intervention runs, the trend reverses, no page fires. *Post-page assistance* is the fallback: the team's existing AlertManager has already paged the on-call via Signal, Kjarr arrives in parallel with the engineer, attempts a remediation, comments on the existing Matrix incident thread with what it tried.  
Both flows share the same components and the same trust posture. What differs is the approval envelope (stricter pre-page), the visibility surface (Zulip-only pre-page, Matrix-comments-plus-Zulip post-page), and the failure consequence (a wrong pre-page intervention can cause a page; a wrong post-page hypothesis just rules out a lead).

### Flow A · Pre-emption (1:47 AM)

mon sees checkout p99 trending toward SLO ceiling but well below page threshold. pg-prod picks up the brewing event, finds stale planner statistics, proposes ANALYZE. Architect approves under the pre-page envelope's stoppable window. ANALYZE runs, plan flips to index scan, trend reverses. Zulip-only notification (with a celebratory cross-post to the Mastodon broadcast feed at closure). No page fires. The on-call engineer reads the convoy summary in the morning over coffee.

### mon OBSERVER pg-prod OPERATOR Architect COORDINATOR Proxy COGNITION Refinery LEASES Notif. svc CHANNELS Zulip \#convoys 01:47:12 self-watch 01:47:28 anomaly\_brewing checkout p99 trending toward SLO 01:47:30 investigate model invocations · audited · budgeted 01:48:02 hypothesis\_proposed stale stats · ANALYZE · 0.86 01:48:04 envelope 01:48:06 approval\_pending · 90s window zulip post · stream \#convoys no objection received in 90 seconds 01:49:36 request\_lease(orders) lease\_granted · TTL 90s 01:49:38 ANALYZE 01:50:31 intervention\_completed 01:51:00 hypothesis\_confirmed trend reversed · p99 back to baseline 01:51:48 convoy\_closed · page\_avoided=true zulip \+ mastodon resolution \+ report link Total convoy duration: 3 minutes 6 seconds · No Signal page · No human disruption Flow A · pre-emption · solid arrows are mesh events · dashed orange is cognition through the proxy · green denotes lease coordination

Flow B · Post-page assistance (3:14 AM)  
AlertManager fires through the team's existing alerting; the alert lands in a Matrix \#incidents room and the on-call engineer is paged via Signal through the team's own AlertManager → signal-cli pipeline. Kjarr arrives in parallel — mon sees the SLO violation, pg-prod investigates, the Architect approves under the post-page envelope (more permissive, no stoppable window for trivial reversible operations), the intervention runs. mon confirms recovery. Kjarr posts a comment on the existing Matrix incident thread with what it tried. The engineer wakes up to a partially-investigated problem with the answer attached — or, if Kjarr was wrong, to a ruled-out hypothesis and a starting point. *Kjarr never writes to Signal* — it cannot fire a page even if it wanted to.

## AlertManager \+ SIGNAL · UNCHANGED mon \+ pg-prod PARALLEL Architect COORDINATOR Matrix \#incidents Notif. svc COMMENT-ONLY Zulip \#convoys On-call SRE 03:14:08 SLO violation · alert lands in \#incidents 03:14:14 Signal page · phone · sound · team's existing pipeline, not Kjarr 03:14:24 anomaly\_observed Kjarr arrives in parallel, does not own the paging path cognition · audited 03:14:51 hypothesis\_proposed stale stats · 0.86 03:14:52 auto-approve 03:15:02 convoy\_started · post-page matrix message "Kjarr investigating, hypothesis: stale stats..." zulip post 03:15:08 ANALYZE 03:16:30 hypothesis\_confirmed trend reversed 03:17:14 convoy\_closed · resolved matrix message "resolved; engineer can ack & close" 03:18+ arrives Engineer wakes to a Matrix \#incidents thread with full Kjarr investigation already attached Flow B · post-page · the warning-orange path is the team's existing alerting (AlertManager fires alert into Matrix and pages on-call via Signal — both untouched) · Kjarr operates in parallel with comment-only authority on the existing incident

---

Trust boundaries

`Defense in depth`  
Five enforcement layers, each independent, each capable of catching what the others miss. The harness's type system prevents the agent's reasoning from invoking unauthorized tools at the language level. The proxy validates the same constraints at the network level. The Witness cross-checks both at the behavioral level. The mesh's group-membership cryptography enforces who can publish what at the substrate level. The container's egress firewall enforces network-reachability at the OS level.  
A single layer can fail and the system's security properties hold. Two layers failing simultaneously requires correlated compromise, which dramatically raises the attacker's bar. The intent is not to prevent every attack but to make any successful attack expensive enough that the operational cost of the attempt exceeds the value of the target.

## 5 · Container egress firewall · OS network policy Allow: application port · proxy endpoint · mesh peers · Deny: everything else 4 · p2panda group membership · cryptographic publish/subscribe authorization Each topic has a defined subscriber/publisher set · membership signed by Architect · revocable 3 · Witness · external integrity check Manifest hash · binary hash · heartbeat · rate & distribution anomalies · MCP violations · drift 2 · LLM proxy · request-time capability validation Signature · manifest version · tool declarations · MCP allowlist · model selection · budget · kill switch 1 · Harness type system · compile-time enforcement Tool registry from manifest · typed argument validation · state machine as enum · idempotency tokens Agent reasoning · assumed potentially compromised Five enforcement layers · the agent's reasoning is the assumed-hostile core · each outer layer is an independent check on what the inner layers permit

---

Where humans are notified

`Notification surface`  
Humans interact with Kjarr through four open-protocol channels, each chosen for a specific tier and a specific role. **Zulip** for the running team narrative — its stream-and-topic model fits SRE workflows better than flat threading, and each convoy gets its own topic. **Matrix** for incident commentary — the team's existing alert pipeline lands in a Matrix \#incidents room, and Kjarr has comment permission on those threads (the role PagerDuty plays in enterprise deployments). **ActivityPub** for casual broadcast — page-avoided summaries, morning digests, the running tally of what Kjarr is doing, all federated and followable. **Signal** for actual paging — but operated by the team's existing AlertManager → signal-cli pipeline, *not* by Kjarr. Kjarr has no write access to Signal at all.  
This separation is the most important property in the entire notification surface: the team's paging mechanism (Signal, in this deployment) is reachable only by the team's existing alerting, never by Kjarr. If the team wants Kjarr-internal urgent events to escalate to Signal, AlertManager watches Kjarr's mesh events and fires its own alerts when thresholds are exceeded. AlertManager remains the single source of paging.  
The escalation discipline matters: *FYI* events are background information the team reads at their convenience. *Attention* events surface in Zulip with light salience — a thread mention, not a page. *Urgent* events are when Kjarr has detected something that should make a human stop what they're doing — a Witness quarantine for serious misbehavior, an MCP allowlist violation, a manifest hash mismatch. These are loud Zulip with @stream mentions, never Signal pages.  
EVENTS convoy\_closed · pre-empt OK convoy\_summary · morning report approval\_pending · 90s window convoy\_started · post-page convoy\_escalated · falsified witness\_quarantine · agent mcp\_violation · proxy manifest\_mismatch · witness CHANNELS Mastodon · ActivityPub followable broadcast, page-avoided summaries URGENCY · CASUAL · PUBLIC OR FOLLOWER-LOCKED Zulip · \#convoys (FYI) running narrative, topic-per-convoy, read at leisure URGENCY · BACKGROUND Zulip · \#convoys (attention) approval window, escalation, thread mention URGENCY · ATTENTION Matrix · \#incidents comment on existing incident threads only POST-PAGE · COMMENT-ONLY Zulip · \#convoys (URGENT) @stream mention, integrity violations URGENCY · STOP-WHAT-YOU-ARE-DOING Signal · pager (NO KJARR ACCESS) team's AlertManager → signal-cli pipeline PAGING · NEVER WRITTEN BY KJARR ↛ no path from any Kjarr event reaches Signal On-call engineer Human notification map · green \= FYI · copper \= attention · orange \= urgent · all paths converge on the on-call engineer · Signal (paging) reachable only via the team's AlertManager pipeline, never by Kjarr Notification triggers and routing

| Event | Channel(s) | Urgency | What the human sees   |
| :---- | :---- | :---- | :---- |
| convoy\_closed (pre-empt) | Zulip \+ Mastodon | `FYI` | "Convoy bd-c41 resolved. Page avoided. Full report: *link*." Mastodon followers see the same as a federated post. |
| convoy\_summary | Zulip \+ Mastodon | `FYI` | Daily morning digest: night's convoys, pages avoided, anomalies investigated. Cross-posted to ActivityPub. |
| approval\_pending | Zulip | `Attention` | "Pre-emptive intervention proposed. Plan: ANALYZE orders. Proceeding in 90s unless ✋." Topic-threaded under the convoy. |
| convoy\_started (post-page) | Matrix \+ Zulip | `Attention` | Matrix \#incidents thread gains a comment from Kjarr. Zulip topic for the convoy is created in parallel. |
| convoy\_escalated | Matrix \+ Zulip | `Attention` | "Hypothesis falsified, trend continuing. Page may fire shortly. Standing by." Matrix thread \+ Zulip topic. |
| witness\_quarantine | Zulip (URGENT) | `Urgent` | "@stream — Kjarr quarantined *agent-id* for *reason*. Operator review needed." High-salience but not Signal. |
| mcp\_violation | Zulip (URGENT) | `Urgent` | "@stream — agent declared an MCP server outside its allowlist. Quarantined. Likely compromise." |
| manifest\_mismatch | Zulip (URGENT) | `Urgent` | "@stream — running manifest hash differs from Architect record. Quarantined." |
| *any Kjarr event* | Signal (paging) | — | *Kjarr cannot write to Signal.* If the team wants Kjarr-internal events to escalate to Signal, AlertManager watches Kjarr's mesh events and fires its own alerts. |

One more property worth being explicit about: **Kjarr never has write access to the paging channel.** A compromised agent, a quarantined Witness, a manifest mismatch — these surface in Zulip with high urgency, but they cannot fire Signal pages. The Signal pipeline is reserved for production incidents the team's existing AlertManager fired; Kjarr's own integrity issues are an operational concern that surfaces in Zulip for whoever's reading, not whoever's holding the pager. This separation keeps the paging channel uncluttered with Kjarr-internal noise and protects Signal's integrity as the production-paging system of record. The choice of channels is a deployment decision — replace Zulip with Mattermost, Matrix with IRC, Mastodon with Pleroma, Signal with PagerDuty — but the structural rule does not change: *the team's paging channel is reachable only by the team's existing alerting*.  
The four-protocol mix in this reference deployment is also a flexibility demonstration. Kjarr's notification service speaks to whatever the team uses, with each channel under a tightly-scoped capability bound. Adding a new channel is a new template family and a new API client behind a new manifest entry — not a major surgery. The value of this composability is most visible in demos: nothing in the reference deployment requires a paid SaaS subscription, every protocol is open and self-hostable, and the team's existing tooling stays the team's existing tooling.  
Kjarr · Architecture overview  
Companion specs: architect-design · pg-prod-design · mon-design · refinery-design · notification-service-design · harness-spec · proxy-spec · witness-spec  
Set in Newsreader & IBM Plex Mono