# Kjarr v1 — Roadmap to Release

A roadmap from the current design corpus (eight component specs, two flow walkthroughs, the architecture document) to a v1 release: a working system that an SRE team can deploy in staging, run against real Postgres and real Prometheus, and evaluate against their own infrastructure. The two canonical flows (pre-emption and post-page assistance) work end-to-end. The audit log is complete. The notification surface is real.

This is *not* "production-ready for arbitrary deployment" — that's v2+. v1 is "a serious team can take this seriously."

## Definition of done for v1

A team can clone the repository, follow a one-page deployment guide, and have:

- A running harness binary attached to a real Postgres instance (`pg-prod`)  
- A running harness binary attached to a real Prometheus instance (`mon`)  
- An Architect, a Refinery, a Notification Service, a Witness, and an LLM proxy  
- Notifications flowing to Zulip, Matrix, and Mastodon (Signal explicitly absent — that's the whole point)  
- Both flows demonstrably working: pre-emptive ANALYZE on stale stats, and post-page commentary on a fired AlertManager incident  
- A complete audit log of every model call, every tool execution, every lease grant, every notification  
- Documentation sufficient for someone who didn't build it to run it

That's v1. Anything more is v2.

## Milestones overview

| Phase | Goal | Estimate | Blocks |
| :---- | :---- | :---- | :---- |
| 0 — Specs | Finalize wire-format schemas | 2 weeks | Everything |
| 1 — Foundation | Harness \+ proxy skeletons running | 4–6 weeks | Phases 2–4 |
| 2 — Vertical slice | First end-to-end flow with `pg-prod` | 6–8 weeks | Phase 3 |
| 3 — Horizontal expansion | `mon`, Refinery, Witness, both flows | 4–6 weeks | Phase 4 |
| 4 — Notification surface | Zulip, Matrix, Mastodon | 2–3 weeks | Phase 5 |
| 5 — Hardening | Real-environment exercise, failure modes | 4–6 weeks | Phase 6 |
| 6 — Release | Documentation, deploy story, polish | 2 weeks | — |

Sequential total: roughly **6 months for a 3–5 person team**, with parallelism reducing this if Phase 4 starts during Phase 3 and Phase 5 absorbs concurrent fixes.

---

## Phase 0 — Specification finalization

**Goal:** Lock the schemas that everything else depends on. Mistakes here ripple through every later phase.

**Duration:** 2 weeks

### Event schema

The mesh event format is the contract between every component. Get it right once.

- [ ] Define the canonical event envelope (id, topic, publisher identity, signature, timestamp, schema version, payload type)  
- [ ] Define payload types for: `anomaly_brewing`, `anomaly_observed`, `hypothesis_proposed`, `hypothesis_confirmed`, `hypothesis_falsified`, `intervention_started`, `intervention_completed`, `intervention_failed`, `convoy_started`, `convoy_closed`, `convoy_escalated`, `approval_pending`, `approval_granted`, `lease_request`, `lease_granted`, `lease_denied`, `lease_released`, `lease_expired`, `health_status`, `dispute`, `quarantine`, `manifest_updated`, `notification_request`  
- [ ] Choose serialization format — strong recommendation: CBOR with a JSON-Schema-style validation file. Rust serde supports both; CBOR is more compact and unambiguous.  
- [ ] Versioning strategy for schema evolution — additive fields, explicit `schema_version`, hard rejection on unknown required fields  
- [ ] Topic naming convention and group membership — `prod.<rig>.<event-type>` etc.  
- [ ] Reference event vocabulary documentation

### Manifest schema

The capability manifest is the single most security-relevant document in the system.

- [ ] Define the manifest envelope (version, agent\_id, role, issued\_at, expires\_at, signature, signing\_key\_id)  
- [ ] Define `Capabilities` variants for every read scope and write operation in `pg-prod` and `mon`  
- [ ] Define `MeshConfig` (subscriptions, publications) as topic patterns  
- [ ] Define `ModelConfig` (allowed models, per-call budget, per-day budget, per-convoy budget)  
- [ ] Define `ApprovalEnvelope` schema (per-tier confidence thresholds, allowed operations, delay windows)  
- [ ] Define `Topology` schema (dependency graph for `pg-prod`, service topology for `mon`)  
- [ ] Define MCP allowlist format (URL or URL pattern, per-server scope notes)  
- [ ] Write five reference manifests covering `pg-prod`, `mon`, Architect, Notification Service, Witness

### Configuration schemas

Configuration that gets signed and loaded at runtime.

- [ ] Approval envelope file schema (per-rig, per-tier policies)  
- [ ] Notification template schema (per-event, per-channel, with allowed substitution fields)  
- [ ] Notification routing schema (which events go to which channels with what conditions)  
- [ ] Brewing-query library schema (PromQL templates with parameter sets, thresholds, slope policies)  
- [ ] Witness rule schema (which checks, what thresholds, alert vs. quarantine)  
- [ ] Trust manifest schema (allowed binary hashes, Architect public key, deployment-wide signing keys)

### Build and release infrastructure

- [ ] Cargo workspace layout (harness crate, proxy crate, refinery crate, notification crate, witness crate, shared schemas crate)  
- [ ] Reproducible build setup (`cargo --frozen`, locked dependencies, deterministic linking)  
- [ ] SBOM generation as part of the build  
- [ ] Binary signing process — offline release key, separate from runtime keys  
- [ ] CI pipeline scaffold (build, test, sign, publish hash)

### Exit criteria

- [ ] All schemas reviewed by at least two engineers and at least one security person  
- [ ] Reference manifests parse cleanly and round-trip through CBOR  
- [ ] An example event flows through the full schema validator  
- [ ] Build infrastructure produces a signed binary with verified hash

---

## Phase 1 — Foundation

**Goal:** A harness binary that loads identity, verifies a manifest, connects to the mesh, and heartbeats. A proxy that verifies a signed request and forwards to a model. Skeleton, not features.

**Duration:** 4–6 weeks

### Identity and bootstrap

- [ ] Sealed local store for the agent's keypair (TPM-backed where available, kernel keyring fallback)  
- [ ] Identity loading and verification at startup  
- [ ] Architect bootstrap signature verification on the agent's identity record  
- [ ] Manifest loading from signed file  
- [ ] Manifest signature verification against Architect public key

### p2panda mesh integration

- [ ] Pick p2panda crate version, lock it, document the choice  
- [ ] Group membership setup for the deployment's prod-platform group  
- [ ] Subscription handling per the agent's manifest  
- [ ] Publish path with signature, schema validation, and durable append  
- [ ] Replay path on agent startup — reconstruct state from recent events  
- [ ] Test harness for mesh integration against a local p2panda node

### Harness state machine

- [ ] State enum with all variants from the harness spec  
- [ ] Supervisor task that owns transitions  
- [ ] State persistence via mesh `health_status` events at every transition  
- [ ] Restart-as-replay logic  
- [ ] Quarantine state and the signed-command path that enters it

### Tool registry scaffolding

- [ ] `Tool` trait and `ExecutionContext` types  
- [ ] Manifest-driven tool registry construction  
- [ ] Type-validated argument parsing (the `AllowedTable`, `AllowedDashboard` patterns)  
- [ ] Tool invocation flow: reasoning task produces `ToolCall`, sends to tool task, tool task validates against manifest, executes, returns result  
- [ ] Idempotency token mechanism for tool execution  
- [ ] One stub tool (`heartbeat_echo` or similar) implementing the trait end-to-end

### Proxy skeleton

- [ ] HTTP server (axum or similar)  
- [ ] Signature verification on incoming requests  
- [ ] Manifest fetch from durable store  
- [ ] Stub validation (accept-everything for now, real validation in Phase 2\)  
- [ ] Forward path to Anthropic and OpenAI APIs  
- [ ] Streaming response handling  
- [ ] Audit log writer (append to file or initial Postgres table — production backing store decided in Phase 5\)

### Exit criteria

- [ ] A harness binary can be deployed with a manifest, connect to a local mesh, and heartbeat for 24 hours without crashing  
- [ ] The proxy can receive a signed request, verify it, forward to a real model, stream the response, and write an audit entry  
- [ ] An end-to-end "hello world": harness sends a model request through the proxy, receives a response, publishes a health event with the round trip recorded

---

## Phase 2 — Vertical slice

**Goal:** `pg-prod` end-to-end. One agent, one tool, one notification, one demo. This is the moment Kjarr becomes real.

**Duration:** 6–8 weeks

### `pg-prod` agent implementation

- [ ] Real Postgres connection management (connection pool, retry, timeout) in the tool task  
- [ ] `read_pg_stat` tool with allowed-views enforcement  
- [ ] `read_catalog` tool  
- [ ] `get_query_fingerprints` tool against `pg_stat_statements`  
- [ ] `explain_fingerprint` tool with read-only enforcement  
- [ ] `analyze_table` tool with allowed-tables enforcement (the first real write)  
- [ ] Hypothesis formation prompt with structured output validation  
- [ ] Confidence calibration table (initially seeded with reasonable defaults)  
- [ ] Dependency graph loading from manifest  
- [ ] Self-watch mode with low-frequency timer  
- [ ] All five reasoning modes wired up

### Architect (basic)

- [ ] Convoy lifecycle management (started, approval\_pending, approval\_granted, executing, closed)  
- [ ] Approval envelope evaluation (rules-based, no LLM judgment in v1)  
- [ ] Convoy state persistence in mesh  
- [ ] Closure attestation including `page_avoided` judgment  
- [ ] Decomposition for the trivial case: one anomaly event → one hypothesis request → one approval → one execution → one closure  
- [ ] No federation in v1; single instance with hot standby for availability  
- [ ] No threshold signing in v1; single signing key in sealed store

### Proxy (real validation)

- [ ] Real capability validation against the manifest  
- [ ] Tool declaration matching against capability variants  
- [ ] MCP allowlist enforcement — most security-critical check, get this right first  
- [ ] Model selection validation  
- [ ] Per-request budget projection and rejection  
- [ ] Kill-switch lookup and enforcement  
- [ ] End-of-stream audit assembly with full request/response capture

### Notification (Zulip only, for now)

- [ ] Zulip API client with allowlisted streams enforcement  
- [ ] Template rendering engine (handlebars or askama)  
- [ ] One template family covering convoy lifecycle events  
- [ ] Backpressure on rate limits

### Demo scenario

- [ ] Synthetic Postgres workload that produces stale-stats latency degradation  
- [ ] AlertManager rule below SLO that `mon` would normally watch (but `mon` doesn't exist yet, so simulate)  
- [ ] Manual injection of an `anomaly_brewing` event  
- [ ] Full convoy: investigation → hypothesis → approval → ANALYZE → recovery → closure → Zulip post  
- [ ] Recorded demo video showing the flow

### Exit criteria

- [ ] The full pre-emption flow works against a real Postgres database  
- [ ] An `ANALYZE orders` actually executes and the audit log proves it  
- [ ] A Zulip post lands in a real Zulip workspace at convoy closure  
- [ ] The convoy completes in under 4 minutes (per the 1:47 AM walkthrough)  
- [ ] Restarting the agent mid-convoy resumes correctly

---

## Phase 3 — Horizontal expansion

**Goal:** `mon`, the Refinery, the Witness. Both canonical flows working end-to-end.

**Duration:** 4–6 weeks

### `mon` agent implementation

- [ ] Prometheus query API client in the tool task  
- [ ] AlertManager rule loader  
- [ ] Grafana annotation API client with tag-prefix and rate-limit enforcement  
- [ ] PromQL template library with parameter substitution  
- [ ] `compute_trend_projection` tool — typed slope-fitting and forward projection  
- [ ] Brewing query library schema and loader  
- [ ] Self-watch as primary reasoning mode (60-second cadence)  
- [ ] Hypothesis formation pointing at services and root causes (not interventions)  
- [ ] Hypothesis verification mode (watch for predicted recovery after `pg-prod` interventions)  
- [ ] Health event for AlertManager rule drift detection

### Refinery (lease service)

- [ ] Lease state machine (granted, expired, revoked, released)  
- [ ] FIFO grant policy  
- [ ] TTL enforcement and auto-release  
- [ ] Renewal mechanism with renewal-count limits  
- [ ] Mesh event publishing for every state transition  
- [ ] Restart from mesh log replay  
- [ ] Per-resource policy hooks (TTL per operation type, multi-reader allowance)

### Witness (basic checks)

- [ ] Manifest hash verification on every health event  
- [ ] Binary hash verification against allowed-builds list  
- [ ] Heartbeat monitoring with configurable thresholds  
- [ ] Tool-call rate anomaly detection (basic histogram baseline)  
- [ ] Hypothesis pattern anomaly detection (rate, distribution)  
- [ ] MCP allowlist violation forwarding from proxy  
- [ ] Quarantine command issuance with signing  
- [ ] Operator-key escalation path for lifting quarantine

### Architect (post-page mode)

- [ ] Post-page envelope evaluation (more permissive than pre-page)  
- [ ] Post-page convoy lifecycle (no approval window, immediate execution for trivial reversibles)  
- [ ] Coordination with `mon`'s `anomaly_observed` events (vs. `anomaly_brewing`)

### Both flows demonstrable

- [ ] Pre-emption flow with real `mon` triggering on real Prometheus metrics  
- [ ] Post-page assistance flow with simulated AlertManager firing into Matrix  
- [ ] Synthetic-incident harness that can produce both scenarios on demand

### Exit criteria

- [ ] Both canonical flows work against real Postgres and real Prometheus  
- [ ] The Refinery serializes a deliberate concurrent-mutation test scenario  
- [ ] The Witness catches a deliberately mismatched manifest hash and quarantines  
- [ ] A second demo video covering Flow B

---

## Phase 4 — Notification surface

**Goal:** All three Kjarr-writable channels working. Signal *explicitly* not implemented.

**Duration:** 2–3 weeks

### Zulip (already done in Phase 2, polish only)

- [ ] Topic-per-convoy organization  
- [ ] @stream mention support for URGENT events  
- [ ] Reaction-based approval (deferred to v2 — dashboard-only approval in v1)

### Matrix integration

- [ ] Matrix client (matrix-rust-sdk or equivalent)  
- [ ] Bot identity setup, allowlisted-rooms enforcement  
- [ ] AlertManager → Matrix bot pattern documented (this is team infrastructure, not Kjarr — but document the integration pattern for the demo)  
- [ ] Comment-on-existing-thread template family  
- [ ] End-to-end test: alert lands in Matrix → Kjarr comments on the thread

### ActivityPub / Mastodon integration

- [ ] ActivityPub client or Mastodon API client (start with Mastodon API for simplicity)  
- [ ] Bot account setup with a clear public profile  
- [ ] Templates for `convoy_closed` (page-avoided celebration) and `convoy_summary` (morning digest)  
- [ ] Posting cadence and rate limits  
- [ ] Decision on public vs. follower-locked default — recommendation: follower-locked by default, public toggle per deployment

### Signal — explicit non-implementation

- [ ] Document the team-side AlertManager → signal-cli pattern  
- [ ] Document explicitly that Kjarr has no Signal client  
- [ ] Document the AlertManager-watching-Kjarr pattern for teams that want Kjarr-internal events to escalate to paging  
- [ ] Add a test that fails if Kjarr ever attempts a Signal write (paranoid but warranted)

### Notification routing

- [ ] Signed routing rules per the schema from Phase 0  
- [ ] Per-event channel selection logic  
- [ ] Cross-channel coordination (e.g., `convoy_started` post-page goes to both Matrix and Zulip)

### Exit criteria

- [ ] Notifications land in real Zulip, real Matrix, and real Mastodon for the right events  
- [ ] No code path in the notification service can call Signal  
- [ ] Templates render cleanly with no `null`/`undefined` substitutions in any test scenario

---

## Phase 5 — Hardening

**Goal:** Make it not embarrassing. Real-environment exercise, real failure modes, real backing stores.

**Duration:** 4–6 weeks

### Failure mode coverage

- [ ] Proxy unreachable → harness pauses gracefully, dashboard shows degraded state  
- [ ] Mesh peer disconnection → recovery without data loss  
- [ ] Tool execution timeout → escalation path works  
- [ ] Lease leak via agent crash → TTL recovery works  
- [ ] Manifest version mismatch during in-flight convoy → graceful handling  
- [ ] Architect signing key rotation (manual procedure, documented)  
- [ ] Witness false positive on a healthy agent → operator can lift the quarantine  
- [ ] Falsified hypothesis under pre-page → escalation to humans without acting on a second hypothesis

### Real backing stores

- [ ] Audit log: production-grade append-only store (recommendation: start with a single Postgres with appropriate partitioning; defer Kafka-based pipelines to v2)  
- [ ] Budget store: low-latency reads and durable writes (Redis with AOF persistence is fine for v1; Postgres also works)  
- [ ] Manifest store: signed file directory with version cache  
- [ ] Backup and recovery procedures for each

### Calibration

- [ ] Initial confidence calibration table for `pg-prod` based on synthetic-load results  
- [ ] Initial brewing-query precision baselines for `mon`  
- [ ] Witness threshold tuning based on observed agent behavior over a 1-week soak test  
- [ ] Documentation on how operators should re-tune for their environments

### Operational primitives

- [ ] CLI for operator actions (issue manifest, sign quarantine-lift, rotate Architect key, set kill switch)  
- [ ] Dashboard for system status (which agents are healthy, recent convoys, current quarantines, budget consumption)  
- [ ] Approval interface in the dashboard for envelope-exceeding requests  
- [ ] Log export for compliance / audit reviewers

### Security review

- [ ] Internal threat model review against the trust boundaries diagram  
- [ ] Penetration test focused on prompt injection paths (database content, metric labels, dashboard text)  
- [ ] MCP allowlist enforcement test (deliberately attempt to declare an unauthorized server)  
- [ ] Manifest tampering test (deliberately deploy an agent with a manifest hash mismatch)  
- [ ] Audit log tampering attempt (verify integrity primitives work)

### Exit criteria

- [ ] 7-day soak test in staging with no unhandled errors  
- [ ] All failure modes from the design docs have explicit tests that pass  
- [ ] Security review signs off  
- [ ] Operator can perform every documented administrative action from the CLI

---

## Phase 6 — Release

**Goal:** Documentation, deploy story, polish. Make it possible for someone who didn't build Kjarr to run it.

**Duration:** 2 weeks

### Deployment story

- [ ] One-page quickstart: clone, configure, deploy  
- [ ] Reference Helm chart or Compose file for staging deployment  
- [ ] Bootstrap script that generates initial manifests, signing keys, deployment trust manifest  
- [ ] Documentation for the AlertManager → Matrix bot integration (so teams can wire their existing alerting into the Matrix lane)

### Documentation

- [ ] Polish all eight component design docs to release quality  
- [ ] Polish the architecture HTML document  
- [ ] Operator runbook: incident response, credential rotation, harness upgrade, witness tuning, common issues  
- [ ] First-90-days guide: what to expect during calibration, false-positive rates, when to graduate from alert-only to quarantine-on-trigger  
- [ ] Failure-mode reference: each known failure, how to diagnose, how to recover  
- [ ] FAQ: the questions a CISO will ask, answered honestly

### Demo materials

- [ ] Recorded walkthrough of both flows  
- [ ] Sample audit-log queries showing what compliance can extract  
- [ ] Architecture talk slides (matching the HTML document's structure)  
- [ ] One-pager for VP-Eng / CISO audiences

### Release artifacts

- [ ] Tagged v1.0.0 release with signed binaries  
- [ ] Release notes describing what's in scope, what's deferred, what's known limitations  
- [ ] Public repository (if open-sourcing) or internal release announcement  
- [ ] Migration plan documentation for moving from v1 to future versions

### Exit criteria

- [ ] An engineer who hasn't seen the codebase can deploy it in staging following the quickstart in under 2 hours  
- [ ] All documents pass a "would I be embarrassed by this in front of a serious team" review  
- [ ] Release tagged and announced

---

## Cross-cutting concerns

These run throughout all phases.

### Testing

- [ ] Unit tests for every typed boundary (manifest variants, tool argument validation, state transitions)  
- [ ] Integration tests for the harness ↔ proxy ↔ mesh round-trip  
- [ ] End-to-end tests for both canonical flows in CI  
- [ ] Property-based tests on schema serialization (round-trip, version compatibility)  
- [ ] Soak tests for memory leaks, channel backpressure, lease leaks  
- [ ] Adversarial tests for prompt injection (the proxy must reject unauthorized tool declarations even if the model produces them)

### Observability of Kjarr itself

- [ ] Structured logging across all components  
- [ ] Metrics endpoint per component (Prometheus exposition format — yes, Kjarr observes its own infrastructure with metrics, recursion is fine)  
- [ ] Tracing for cross-component flows  
- [ ] Dashboard for system health that operators check daily

### Security review (continuous, not one-time)

- [ ] Threat model document maintained as the implementation evolves  
- [ ] Dependency audit (`cargo audit`) in CI  
- [ ] Supply chain attestation (SBOM, signed builds)  
- [ ] Periodic review of MCP allowlist policy and per-deployment overrides

### Documentation discipline

- [ ] Every design decision that gets implemented updates the corresponding spec  
- [ ] Every spec divergence from the design is called out explicitly with rationale  
- [ ] Decision log maintained alongside code (lightweight ADRs are fine)

---

## Open decisions

Things we have not yet settled and should before or during the relevant phase.

### Schema/protocol

- [ ] **Mesh event serialization:** CBOR vs. MessagePack vs. JSON. Recommendation is CBOR; needs validation against p2panda's existing patterns.  
- [ ] **Idempotency token format:** UUID v7 (time-ordered) vs. content hash. Recommendation is UUID v7 for ergonomics.  
- [ ] **Schema versioning policy:** allow additive evolution within a major version, hard break across major versions. Need to write this down.

### Implementation language

- [ ] **Notification service:** Rust (consistent with rest) vs. Go (faster to write for HTTP+templates). Recommendation: Rust, for verification consistency. Decision needs sign-off.  
- [ ] **Refinery:** same question. Same recommendation.

### Federation & signing

- [ ] **Threshold signing for Architect:** v1 ships single-signer, v2 adds threshold. Need explicit "this deployment is dev-grade until threshold is operational" warning in the deploy guide.  
- [ ] **Witness federation:** primary/standby in v1, multi-instance with quorum in v2. Same warning pattern.

### Approval mechanism

- [ ] **Slack-reaction approval (in our case, Zulip-reaction):** deferred to v2 per the notification service spec. Confirm this stays deferred — every team will ask for it.  
- [ ] **Dashboard approval interface:** must be in v1 (it's the only approval path). Needs UX work — specifically, what does the engineer see when an approval window is open and what's the click that approves.

### Calibration

- [ ] **Initial confidence calibration:** seeded from synthetic load or shipped with conservative defaults? Recommendation: ship conservative defaults (everything below 0.7 confidence requires explicit human approval), let teams tune up.  
- [ ] **Brewing-query library:** ship with examples or require teams to author their own? Recommendation: ship a small "starter pack" for common Postgres and Prometheus patterns, document the authoring process.

### Hosting

- [ ] **Reference deployment topology:** Kubernetes (most common in target audience), Compose for local dev, bare VMs as alternative. Need to document all three or commit to one.  
- [ ] **p2panda node deployment:** sidecar to each component vs. shared mesh node per host. Performance vs. operational simplicity tradeoff.

---

## Risk registry

What could go wrong and how we'd notice.

### Technical risks

- **p2panda maturity:** the project is alive but small. If a critical issue surfaces in the mesh layer, we have limited support. Mitigation: contribute upstream, maintain a fork as fallback.  
- **Calibration false positives:** if `mon`'s brewing queries fire too often without real underlying issues, the team's trust erodes fast. Mitigation: ship conservative thresholds, design the calibration update loop into Phase 5\.  
- **Audit log scale:** every model call writes to audit. At scale this is a lot of data. Mitigation: design partitioning into the schema from Phase 0; defer "real" pipeline (Kafka, etc.) to v2.  
- **Restart-as-replay correctness:** the most subtle part of the harness. Mitigation: extensive testing in Phase 1, deliberate crash-injection testing in Phase 5\.

### Adoption risks

- **Approval envelope complexity:** if teams find the envelope schema impenetrable, they'll either run with permissive defaults (security risk) or refuse to deploy (commercial risk). Mitigation: invest in the first-90-days guide, ship sensible defaults, build the dashboard's approval UI to be obvious.  
- **Demo dependence on synthetic load:** if the demo only works with our synthetic Postgres workload, it doesn't generalize. Mitigation: by Phase 5, exercise against a real production-shaped workload that someone outside the team contributes.  
- **Notification channel switchover friction:** if a team is already on Slack/PagerDuty, asking them to add Zulip/Matrix for a demo might be friction. Mitigation: keep the notification service abstract enough that drop-in adapters for Slack/PagerDuty are a one-day port. Document this as the v1.x roadmap.

### Operational risks

- **Secrets handling for the proxy's model API keys:** if these leak, attackers can run up bills on our infrastructure. Mitigation: sealed store for proxy credentials, separate from agent identity stores, with explicit rotation procedure.  
- **Manifest signing key compromise:** with single-signer in v1, this is catastrophic. Mitigation: keep the signing key offline except during signing operations; document the compromise-response procedure; treat "we lost the key" as a known scenario with a rebuild path.  
- **Witness false positives causing self-DoS:** an over-eager Witness can quarantine all agents. Mitigation: alert-only thresholds for the first weeks of operation, conservative defaults, easy operator override.

### Schedule risks

- **Phase 0 underestimated:** schemas always take longer than expected. If we slip here, everything slips. Mitigation: budget 3 weeks instead of 2 if there's any sign of disagreement during the schema review.  
- **Phase 5 hardening expanding indefinitely:** "real-environment exercise" can become a tarpit. Mitigation: time-box it strictly. If the failure-mode list isn't complete in 6 weeks, ship v1 with a known-issues list and address in v1.1.

---

## Post-v1 deferrals (explicit)

What we are *not* building for v1, with reasons.

| Item | Why deferred | Phase target |
| :---- | :---- | :---- |
| Threshold signing for Architect | Single-signer is enough for staging eval; threshold is the v2 hardening pass | v2 |
| Witness federation with quorum | Same logic as Architect | v2 |
| Smart-arbiter Refinery (LLM-driven priority) | FIFO is enough for v1 workloads; smart arbitration earns its keep only at scale | v2+ |
| Additional domain agents (`redis-prod`, `k8s-prod-eu`, `terraform-state`) | Pattern is set by `pg-prod`; new agents are content, not infrastructure | v1.1+ |
| Multi-region deployment | Single region is sufficient for v1 evaluation | v2 |
| Slack/PagerDuty adapters | The whole point of v1's notification surface is open protocols; enterprise adapters as a v1.x port | v1.x |
| Zulip-reaction approvals | Approval discipline is dashboard-only in v1 to avoid the auth complexity | v2 |
| LLM-augmented Witness checks | Rules-based is adequate; LLM-augmentation is a separate analysis layer | v2+ |
| Persistent learning of team preferences | Configuration is signed, not learned | v3+ |
| Operator-facing IDE / authoring tools for envelopes and templates | Plain files plus signed config plus dashboard is the v1 surface | v2 |

---

## What's *not* on this list

Worth being explicit about gaps.

- **A go-to-market plan.** This is engineering scope, not product scope. If Kjarr is being released as a product (vs. internal infrastructure or an open-source project), the GTM work runs in parallel and isn't tracked here.  
- **Pricing or licensing decisions.** Same.  
- **Hiring plans.** Same. The estimates in this document assume a 3–5 person team with mixed Rust/distributed-systems/SRE experience; staffing decisions are upstream.  
- **Customer / pilot recruiting.** Phase 5's "real-environment exercise" presumes there's somewhere to exercise against. Lining up the first pilot deployment is upstream of this roadmap and should already be in motion by Phase 3\.

---

## A note on parallelism

The phases above are written sequentially for clarity, but several can overlap:

- **Phase 0 → 1:** the schema work in Phase 0 can finalize while Phase 1 foundation work is starting on the mesh integration and identity bootstrap. Schemas need to be locked before tool registry construction begins.  
- **Phase 2 → 3:** `mon` work can begin in parallel with `pg-prod` polishing once the harness skeleton is in place. The tools differ; the substrate is the same.  
- **Phase 3 → 4:** the notification service can be expanded with Matrix and Mastodon support while the Witness and Refinery are being built. They share the mesh substrate but otherwise don't block each other.  
- **Phase 5 hardening overlaps everything.** Calibration data starts accumulating from the moment Phase 2's vertical slice is running. Failure-mode tests can be written and run concurrently with feature work.

A team with the right skill mix should be able to ship v1 in **4–5 months** with aggressive parallelism rather than 6 months sequentially. The risk of moving fast is integration debt; mitigate by keeping the integration test suite green as the gating criterion across all phases.

---

## How this document evolves

Update it as you go. When a task completes, check the box. When an estimate proves wrong, write down the new estimate and the reason. When a deferred item gets pulled forward, move it. When a new risk appears, add it to the registry.

This is a living artifact, not a planning ceremony.  
