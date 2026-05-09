# mon — agent design

*`mon` is the domain agent that sits next to the monitoring stack. It watches Prometheus, reads AlertManager configuration, annotates Grafana dashboards, and produces most of the brewing and anomaly events that the rest of Kjarr reasons about. This document inherits the domain agent pattern from `pg-prod` and specifies what's different.*

## Role

`mon` plays a different role in the system than the other domain agents. Where `pg-prod` is a deep expert on one database that gets pulled into a convoy when something Postgres-shaped is suspected, `mon` is the agent that *does the suspecting*. Most convoys originate from a `mon` observation. The other domain agents hear about most problems through `mon`'s events.

This makes `mon` the primary noise source in the system. If `mon` is over-sensitive, the team gets brewing events for things that aren't really brewing, and confidence in Kjarr erodes. If `mon` is under-sensitive, problems pre-empt past it and show up as PagerDuty pages instead of pre-emptions, and Kjarr's main value proposition is undermined. Calibrating `mon` is the most important calibration in the entire system.

`mon` is also the agent with the strangest authority surface. It doesn't intervene against infrastructure the way `pg-prod` does — it doesn't have an ANALYZE-equivalent. Its "actions" are observations and annotations. The state-mutating part of its capability is small, bounded, and deliberately separated from the team's paging discipline.

The pattern this sets: an *observer* agent has different shape than an *operator* agent. Other observer agents in future versions (a `logs` agent watching Loki/ELK, a `traces` agent watching Tempo/Jaeger) will follow `mon`'s pattern. Other operator agents (`redis-prod`, `k8s-prod-eu`, `terraform-state`) will follow `pg-prod`'s pattern.

## Capability profile

The grants, with the differences from `pg-prod` flagged:

**Prometheus query API access, read-only.** The harness owns the connection (same discipline as `pg-prod`'s database connection). `mon` cannot write to Prometheus, cannot push metrics, cannot modify the TSDB.

**AlertManager configuration access, read-only.** `mon` reads the team's existing alert rules and routes. This is what gives it knowledge of where the page-tier thresholds live, so it can compute its own brewing-tier thresholds in relation. `mon` cannot modify AlertManager configuration. *This is the most important "no" in `mon`'s manifest:* the team's paging discipline is the source of truth for what counts as urgent, and Kjarr does not get to second-guess it. AlertManager fires pages exactly when AlertManager would have fired them with or without Kjarr running.

**Grafana annotation API access, read and write.** This is the only write capability `mon` has, and it's narrow. The agent can create annotations on dashboards in a configured allowlist, with a fixed tag prefix (`kjarr:` by default) that distinguishes Bastion annotations from human-created ones. The annotation content is a typed payload — not free-text — so the agent cannot inject arbitrary content into the dashboard. The harness rejects annotation writes on non-allowlisted dashboards or with non-prefixed tags.

**Deploy event stream, read-only.** Same source the team's existing observability uses (CI events, GitOps commit stream, kubernetes events). `mon` correlates deploy timing with anomaly trends, which is most of how it identifies likely causes.

**Service topology, configured.** This is `mon`'s analog of `pg-prod`'s dependency graph, but bigger and weirder. `mon` watches many services across many rigs; the topology tells it which services are in scope, which metrics matter for each, which downstream domain agents to notify when an anomaly hits a service in their dependency graph. The topology is in the manifest, signed, updated only via Architect-signed manifest revisions. There is no in-band topology learning.

**Mesh subscriptions.** `mon` subscribes to far less than the Architect but more than `pg-prod`. It needs to see deploy events on the mesh, hypothesis events from other domain agents (so it can verify hypothesis predictions against metrics), convoy lifecycle events (to know when interventions are happening so it can watch for predicted recovery), and capability/policy updates for itself.

**Mesh publications.** This is where `mon` is busiest in the system. It publishes `anomaly_brewing` (the pre-page tier), `anomaly_observed` (the post-page tier — when something has crossed an SLO threshold and the agent recognizes it), `hypothesis_confirmed` and `hypothesis_falsified` (verifying other agents' predictions against actual metric behavior), `health_status` (its own), and `dispute` (rare, when `mon` thinks a sibling agent's hypothesis doesn't match what `mon` is observing).

**Model access.** Through the proxy. `mon` runs more inferences per day than most agents because of its observer role, so per-day budgets are typically higher. Per-call cost matters more for `mon` than for `pg-prod`; using a faster cheaper model for routine trend analysis and reserving the strongest model for hypothesis-confirmation-against-prediction is a sensible v1 optimization.

**No notification capabilities, no direct human contact.** Same as `pg-prod`.

## Tool registry

query\_prometheus(query: PromQLTemplate, params: Params, time\_range: TimeRange) \-\> MetricResult

get\_metric\_metadata(metric: AllowedMetric) \-\> MetricMetadata

get\_alertmanager\_rules() \-\> AlertRuleSet

get\_alertmanager\_status() \-\> AlertManagerStatus

compute\_trend\_projection(query: PromQLTemplate, params: Params, threshold: Threshold) \-\> Projection

get\_deploys(time\_range: TimeRange, service\_filter: ServiceFilter) \-\> DeployEvents

get\_service\_topology() \-\> Topology

create\_grafana\_annotation(dashboard: AllowedDashboard, payload: AnnotationPayload) \-\> AnnotationId

read\_grafana\_annotations(dashboard: AllowedDashboard, time\_range: TimeRange, tag\_filter: TagFilter) \-\> Annotations

publish\_anomaly\_brewing(payload: BrewingPayload) \-\> EventId

publish\_anomaly\_observed(payload: AnomalyPayload) \-\> EventId

publish\_hypothesis\_confirmed(payload: ConfirmationPayload) \-\> EventId

publish\_hypothesis\_falsified(payload: FalsificationPayload) \-\> EventId

publish\_health(payload: HealthPayload) \-\> EventId

publish\_dispute(payload: DisputePayload) \-\> EventId

`PromQLTemplate` is the most important type in this registry. Like `pg-prod`'s fingerprint approach, `mon` does not construct novel PromQL — it has a library of templates and parameterizes them. Templates come from two sources: the team's existing AlertManager rules (read-only, copied at agent startup) and a curated brewing-tier query set provided by the team in the manifest. Both sources are signed configuration. The agent can parameterize templates (adjust time ranges, label filters within a defined set), but it cannot construct queries that aren't grounded in a template. This is the equivalent of `pg-prod`'s "EXPLAIN against fingerprints, never raw SQL" discipline, and it serves the same purpose.

`compute_trend_projection` is the most distinctive tool. PromQL by itself is good at "what is the value now" and weaker at "where is this heading." The trend projection tool wraps a templated query with a slope-fitting and forward-projection computation, returning a typed result like "this metric is projected to cross threshold T at time \+14m at current trend rate, with confidence interval \[11m, 17m\]." This is the computation that drives most brewing events. Putting it in a typed tool rather than asking the model to do PromQL math is a major reliability gain.

## Behavior

The same five reasoning modes from `pg-prod`, but the relative weighting is inverted.

**Self-watch is the primary mode for `mon`.** Where `pg-prod` is mostly idle and reactive, `mon` is mostly self-watching. On a configurable cadence (default every 60 seconds), `mon` runs its brewing-tier query library against the current metric state, computes trend projections, and decides whether anything is brewing. Most cycles produce nothing — the system is healthy and metrics are stable. When a brewing trend is detected, the agent moves into hypothesis-formation mode and publishes `anomaly_brewing` if confidence meets threshold.

**Triggered investigation is rare for `mon` and means something specific.** When other domain agents publish hypothesis events, `mon` is the verifier — it watches whether the predicted metric change occurs after an intervention. The "triggered" mode for `mon` is mostly "I've been asked to confirm or falsify this hypothesis," and it's structured around watching specific metrics for specific predicted changes within specific time windows.

**Intervention execution is roughly nil for `mon`.** The only actions `mon` takes against external systems are Grafana annotations. There is no ANALYZE-equivalent. When a convoy is approved, `mon` doesn't execute anything — it stays in observation mode and watches for the predicted effect of whatever the operator agent does.

**Outcome confirmation is `mon`'s most important non-self-watch mode.** When an intervention completes, `mon` is the agent that determines whether it actually worked. The hypothesis-confirmed and hypothesis-falsified events that flow back to the rest of the system are mostly `mon`'s output. This is also where `mon`'s annotation writes happen: an intervention completing successfully gets an annotation on the relevant dashboards, so a human looking at the dashboard later sees "Kjarr ran ANALYZE here at 1:50 AM, latency recovered."

**Idle observation is brief.** `mon` is rarely truly idle because the self-watch cadence is short. Between cycles the reasoning loop is suspended, but the suspension windows are seconds, not minutes.

## Brewing-tier semantics

The brewing-tier watch loop is `mon`'s primary value-add and the most carefully-bounded part of its design.

The library of brewing-tier queries is configured, not learned. Operators define them by adapting their existing AlertManager rules: where AlertManager fires at "5× burn rate sustained for 10m," the brewing query is "burn rate trending toward 5× over the next 15m at current slope." Templates and thresholds are explicit. The team controls the brewing-tier sensitivity directly, just as they control AlertManager's sensitivity directly.

The brewing watch is parallel to AlertManager and never replaces it. AlertManager continues to fire pages on its own thresholds; `mon` runs an additional, lower-threshold watch on top. If `mon` is broken, removed, or quarantined, the team's paging behavior is unchanged. This separation is the property that makes Kjarr safe to deploy alongside an existing observability stack — there's no risk that introducing Kjarr degrades the team's existing alerting, because Kjarr's watch is structurally additive and cannot modify AlertManager.

When a brewing trend is detected, `mon` does not act on it directly. It publishes the event and the Architect routes it. The Architect's approval envelope determines whether any operator agent is permitted to intervene. `mon` is the *finder*, not the *fixer*.

## Hypothesis formation

`mon` forms hypotheses too, but they look different from `pg-prod`'s. A `pg-prod` hypothesis says "I think the cause is stale stats and the fix is ANALYZE." A `mon` hypothesis says "I think this metric trend will continue and will cross threshold T at time T+14m, and the most likely root cause is service X based on deploy correlation."

The structured output: brewing trend description (metric, trajectory, projected threshold crossing), candidate root cause (a service identifier from the topology, with attribution to the deploy event or other correlation that suggests it), suggested investigators (which other domain agents should be pulled in based on the topology), confidence (0–1, calibrated through the harness like `pg-prod`'s).

`mon` does not propose interventions. It points at problems. The intervention proposals come from operator agents (`pg-prod`, `redis-prod`, etc.) that know what to do about the problems `mon` flags.

This separation is structural. An observer agent that can both observe and propose interventions has a much wider blast radius than one that can only observe. By splitting the roles, the worst case for a compromised `mon` is "raises false anomalies that distract sibling agents," not "claims a false anomaly and proposes a destructive remediation."

## Tier-aware behavior

`mon` operates at both tiers but emits different events:

- Pre-page tier: `anomaly_brewing` events, generated by self-watch against the brewing query library. Higher confidence threshold (0.85+) for publication. Conservative posture; "brewing" doesn't mean "act," it means "investigate."  
- Post-page tier: `anomaly_observed` events, generated when AlertManager fires (so Kjarr knows what the team is being paged about) or when self-watch sees a metric already past SLO. Lower confidence threshold (0.70+); the situation is acute, more agents should be pulled in.

The tier flag is set by `mon` based on which threshold was crossed (brewing query template vs AlertManager rule).

## Annotation discipline

The one place `mon` writes to a third-party system. Worth being careful here.

The agent creates Grafana annotations at three points in a convoy:

1. When `anomaly_brewing` or `anomaly_observed` is published — the dashboard gets a marker showing "Kjarr noticed this here." Tag: `kjarr:noticed:<convoy-id>`.  
2. When the Architect approves an intervention — the dashboard gets a marker showing "Kjarr is attempting X." Tag: `kjarr:intervention:<convoy-id>`.  
3. When the convoy closes — the dashboard gets a marker with the resolution and a link to the full convoy report. Tag: `kjarr:resolved:<convoy-id>` or `kjarr:escalated:<convoy-id>`.

That's it. Three annotation writes per convoy, all on the same set of dashboards (the ones in the allowlist for the affected service), all with the agreed tag prefix. The harness rejects any other annotation operation. Humans reading the dashboards get a clean overlay of Kjarr's involvement; the team can filter or hide Kjarr annotations entirely if they want, by their tag prefix.

There's a subtle privilege here: annotations are visible to anyone with dashboard read access, and Kjarr's annotation content is structured but readable. If sensitive incident detail shouldn't be visible on shared dashboards, the annotation payload should be terse — "investigating" rather than "stale stats on orders table." The detailed incident report stays in the convoy log and Slack; the dashboard annotation is the breadcrumb pointing to it.

## Failure modes

Most failure modes carry over from `pg-prod` with the obvious substitutions. The `mon`\-specific ones:

**Brewing-query false positives.** This is the most operationally important failure mode. `mon`'s brewing-tier queries are calibrated by the team initially, and their thresholds drift over time as traffic patterns change. If `mon` is producing brewing events that consistently don't pan out, the calibration table updates, the agent's confidence scoring downshifts, and over time those query templates fall below threshold and stop firing. Operators get a periodic report of brewing-query precision (how often each template's brewing events resulted in confirmed hypotheses) and can retire templates that aren't earning their place.

**AlertManager rule drift.** `mon`'s brewing thresholds are computed in relation to AlertManager's page thresholds. If the team modifies AlertManager rules without updating the brewing queries, the relationship breaks — `mon` might be running brewing watches on rules that no longer exist, or missing brewing watches for rules that newly exist. Mitigation: `mon` reports a configuration health event when it detects a discrepancy between AlertManager rules and its brewing query library, and the team gets a Slack message asking them to sync. This is annoying ops work that could be automated later but should be explicit and human-confirmed in v1.

**Prompt injection via metric labels.** A malicious or compromised application could emit metrics with carefully-crafted label values intended to inject instructions when `mon` reads the metadata. Mitigation: same discipline as `pg-prod`'s query text — labels and label values are treated as untrusted data, wrapped in delimiters, never interpreted as instructions. The typed event output structure prevents an injected prompt from causing the agent to publish events outside its declared shape.

**Annotation flood.** A compromised `mon` could spam Grafana annotations, polluting dashboards. Mitigation: rate limits at the harness (max N annotations per minute, per dashboard, per convoy), and the tag prefix lets operators bulk-delete Kjarr annotations if a flood happens. The blast radius is "annoying dashboard noise," which is recoverable.

**Disagreement with sibling agents.** `mon` says the trend is reversing; `pg-prod` says its intervention worked. They might both be right (the intervention reversed the trend, all is well) or `mon` might be measuring noise (the trend was going to reverse anyway and `pg-prod`'s intervention was unnecessary). The dispute mechanism handles this — `mon` can publish a `dispute` event explaining its observation, and the Architect mediates. Most of the time these disputes reveal something genuine and worth surfacing to humans.

## What `mon` doesn't do

**No write access to AlertManager.** Architectural, not just operational. The team's paging discipline must be untouchable.

**No write access to Prometheus.** No remote\_write, no metric injection.

**No execution against application infrastructure.** `mon` does not call kubectl, terraform, AWS APIs, or anything else. Observation only.

**No correlation across multiple monitoring stacks.** Each Prometheus instance gets its own `mon` agent. If a deployment has multiple Prometheus instances (for example, one per region), there are multiple `mon` agents and the Architect coordinates between them.

**No authority to silence alerts.** Even during a known intervention, `mon` cannot tell AlertManager "this firing alert is expected, suppress it." That decision belongs to humans (or to the team's existing silence-management tooling), not to Kjarr.

## What's mon-specific vs. the domain agent pattern

The pattern carried over from `pg-prod`:

- Sidecar deployment, harness-owned credentials  
- Read scopes are explicit allowlists, write scopes are typed and narrow  
- Tool registry derived from manifest with strongly-typed argument validation  
- Five reasoning modes, with the relative weighting adjusted for an observer role  
- Hypotheses are typed structures with required evidence and harness-calibrated confidence  
- Tier-aware behavior with stricter pre-page envelopes  
- Mesh-mediated communication, no direct vendor or human contact  
- Topology in manifest, no in-band learning  
- Self-monitoring via health events with Witness cross-checks

The mon-specific parts:

- Self-watch is the primary mode rather than triggered investigation  
- The PromQL template library and `compute_trend_projection` tool  
- Read-only access to AlertManager rules with no modification authority  
- Grafana annotation as the only state-mutating capability  
- Hypotheses point at problems and root causes; they do not propose remediations  
- Brewing-tier watch separated from AlertManager's page-tier watch  
- Annotation discipline (three writes per convoy, with a tag prefix and rate limits)  
- The configuration-drift health check between AlertManager rules and brewing queries

## What's next

With the Architect, `pg-prod`, and `mon` specified, the rest of the agent design is largely the pattern applied with substitutions. Worth doing in this order:

1. The notification service — small, narrow, structurally important. Need to specify it before getting deeper into other agents because everything publishes events and the notification service is the first concrete consumer.  
2. Refinery — the boring lease-service version first.  
3. The harness — Rust binary specification, the place where the design moves from prose to types.  
4. Other domain agents (`redis-prod`, `k8s-prod-eu`, `terraform-state`) — quick documents because the pattern is fully set.

The notification service next. After that, the harness specification — that's the document that turns all of this prose into something a Rust engineer can implement.  
