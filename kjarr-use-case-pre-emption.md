# Use case: 1:47 AM

*A pair of agents notices a problem brewing and resolves it before it can page anyone.*

## Setup

Threshold Goods runs an e-commerce platform on AWS: a dozen Go services on EKS, an Aurora Postgres cluster for orders and customers, a Prometheus and Grafana stack for monitoring with PagerDuty downstream of AlertManager. Kjarr is deployed across this infrastructure with two domain agents in scope:

**`pg-prod`** runs as a sidecar to the Aurora cluster. Its capability manifest authorizes read access to the `pg_stat_*` and `pg_class` system views, query plan introspection via `EXPLAIN`, and a small set of low-impact reversible maintenance operations: `ANALYZE`, `REINDEX CONCURRENTLY`, killing queries by PID. No DDL, no `DROP`, no application data reads.

**`mon`** runs as a sidecar to the Prometheus stack. Its manifest authorizes the Prometheus query API and the Grafana annotation API, plus read access to the deploy event stream and AlertManager rules — but no write permission on AlertManager. The team's paging discipline is untouched. `mon` runs an additional, parallel, lower-threshold watch loop on top of the existing alert rules, looking for trends that aren't yet violations but are heading toward them.

A separate notification service consumes Kjarr's mesh events and translates them to Slack messages and PagerDuty incident comments. It cannot create or resolve PagerDuty incidents. The team's on-call workflow is unchanged whether Kjarr is running or not.

It is 1:47 AM on a Tuesday. The on-call engineer is asleep. PagerDuty is quiet.

## What happens

**1:47:12** — `mon` is computing burn-rate forecasts on the checkout service's p99 latency every minute. The metric isn't violating SLO — it's at 480ms against a 2s ceiling — but the trajectory is steepening. The 15-minute slope, projected forward against current traffic, crosses the SLO threshold in roughly 14 minutes. AlertManager won't fire until the threshold is actually breached. `mon` flags the trend as `anomaly_brewing` rather than `anomaly_observed`.

**1:47:28** — `mon` publishes the brewing event:

topic: brewing.checkout

payload:

  service: checkout

  metric: http\_request\_duration\_seconds{quantile="0.99"}

  observed: 480ms

  baseline: 280ms

  trend\_15m: \+0.31σ/min

  projected\_slo\_violation: 14m

  burn\_rate\_projected: 4.8x

  correlated\_deploys:

    \- service: checkout

      version: 2.41.0

      shipped\_at: 2025-04-29T01:02:14Z

      diff\_summary: "added loyalty\_points calculation, new query path on orders table"

  dependency\_graph:

    \- postgres-prod (orders, customers)

    \- redis-prod (sessions)

  page\_imminent: false

**1:47:30** — `pg-prod` is subscribed to `brewing.*` events for any service in its dependency graph. It picks the event up, scopes its analysis to the same time window, and starts asking its own questions. It pulls `pg_stat_statements` for the last 45 minutes and finds a new query fingerprint appearing at growing volume — a join on `orders` filtering by `loyalty_tier`. It checks `pg_stat_user_tables.last_analyze` for `orders`: four days, seven hours ago. It runs `EXPLAIN` on a representative query and sees a sequential scan where it expects an index scan. It checks `pg_class.reltuples` against the actual row count: stats are off by a factor of 30\.

**1:48:02** — `pg-prod` publishes a `hypothesis_proposed` event:

topic: hypothesis.checkout

in\_response\_to: \<brewing event id\>

payload:

  candidate\_cause: "stale planner statistics on orders table"

  evidence:

    \- new query fingerprint shipped 01:02 UTC, filtering on orders.loyalty\_tier

    \- last\_analyze on orders: 4d 7h ago

    \- EXPLAIN shows seq scan, expected index scan on orders\_loyalty\_tier\_idx

    \- pg\_class.reltuples diverges from actual count by 30x

  proposed\_intervention:

    operation: ANALYZE

    target: orders

    estimated\_duration: 60-90s

    reversibility: trivial (idempotent)

    expected\_outcome: plan flips to index scan, latency trend reverses within 2 minutes

    confidence: 0.86

  capability\_check: authorized (low-impact reversible maintenance)

  tier: pre-page

**1:48:04** — The Architect receives the hypothesis. Pre-page interventions go through a stricter envelope than post-page work. The pre-approval policy reads: *for pre-page hypotheses meeting confidence \> 0.85 and reversibility \= trivial, post a Slack notice and proceed in 90 seconds unless someone reacts with a stop signal. Otherwise, post and wait.*

The hypothesis meets the threshold. The Architect publishes a `convoy_started` event and pings the notification service.

**1:48:06** — Slack message in `#sre-prod`:

🔬 **Kjarr — pre-emptive intervention proposed**

Watching `checkout` p99 trending toward SLO ceiling (projected violation in \~14m).

Hypothesis: stale planner statistics on `orders` table after 1:02 AM deploy. Confidence 0.86.

Plan: `ANALYZE orders` (\~90s, fully reversible).

Proceeding in 90 seconds unless someone reacts ✋. Convoy `bd-c41`.

**1:49:36** — No reactions. Convoy proceeds. `pg-prod` executes `ANALYZE orders`. Proxy logs the model call that produced the decision, the tool call that executed it, the duration. The mesh logs the state transition.

**1:50:31** — `ANALYZE` completes. `pg-prod` re-runs `EXPLAIN`, confirms the plan flipped to an index scan, publishes `intervention_completed`.

**1:51:00** — `mon` is watching for the predicted recovery. p99 on the checkout service drops from 480ms to 295ms over the next two scrape intervals. The brewing trend reverses. Burn-rate projection clears. `mon` publishes `hypothesis_confirmed`.

**1:51:48** — The Architect closes the convoy. Slack message in `#sre-prod`:

✅ **Convoy `bd-c41` resolved**

`checkout` p99 returned to 295ms (baseline 280ms). Trend reversed. PagerDuty was not paged.

Root cause: stale stats after 1:02 AM deploy.

Recommended follow-up filed: add `ANALYZE` to the post-deploy hook for any deploy that introduces a new query fingerprint.

Page avoided. Full report: .

The on-call engineer never woke up. PagerDuty stayed quiet. The team reads the convoy summary in Slack the next morning, alongside three other quiet pre-emptions from overnight.

## What flowed through the mesh

1:47:28  mon         anomaly\_brewing

1:48:02  pg-prod     hypothesis\_proposed

1:48:04  architect   convoy\_started

1:48:06  architect   approval\_pending (90s window)

1:49:36  architect   approval\_granted (window expired, no objection)

1:49:38  pg-prod     intervention\_started

1:50:31  pg-prod     intervention\_completed

1:51:00  mon         hypothesis\_confirmed

1:51:48  architect   convoy\_closed (resolved, page\_avoided=true)

Nine events, durably written to the p2panda log, signed by the publishing agent's keypair, replicated across all peers in the `prod-platform` group. The `page_avoided=true` attestation feeds the running tally of pre-emptions, which is the metric that justifies Kjarr's cost.

## What got recorded

For each agent, every model call routed through the proxy in this window is in the audit log: timestamp, agent identity, conversation id, model, prompt, response, tool declarations, tool calls, token cost, capability manifest version, outcome. For this convoy that's roughly 14 model calls between the two agents, totaling around 95k tokens. Every tool call against the database or against Prometheus is also recorded with arguments, agent identity, and result.

The PagerDuty integration was not exercised because no incident was created. If the engineer reviews their morning queue, there is no incident to comment on — only a Slack message confirming a problem was caught early.

## Where this fits

This is Kjarr's primary mode and primary value. Pre-emptive intervention catches the cost curve before customer impact starts, before PagerDuty fires, before the team's night gets disrupted. Most of what Kjarr does, when it's working well, looks like this: a Slack message at 1:47 AM that the team reads at 8:00 AM with their coffee, alongside the morning summary of overnight convoys.

The 3:14 AM scenario — where Kjarr arrives after PagerDuty has already fired and contributes commentary on a live incident — is the *fallback* mode. It's the case where Kjarr was too slow to pre-empt, or didn't have a hypothesis, or the situation was a step change rather than a trend. In that mode, Kjarr annotates the existing PagerDuty incident with what it sees and what it's attempting, and the engineer wakes up to a partially-investigated problem instead of a cold one. Useful, but secondary.

## What this demonstrates

**Two-tier monitoring with strict separation.** AlertManager's thresholds are untouched and remain the single source of truth for paging. `mon` runs a parallel, lower-threshold trend-watch on top of the same metrics, with no permission to modify AlertManager. The team's paging discipline cannot be undermined by Kjarr — Kjarr can only act below the page threshold, never reshape it.

**Asymmetric approval envelopes.** Pre-page interventions face stricter approval gates than post-page work. Acting before humans are aware of a problem is more authority than acting after they've already converged on it. The 90-second Slack window is the minimum conservative form: high-confidence reversible interventions are auto-approved with a *stoppable* delay rather than a synchronous *go-ahead*. Tunable per team — some teams will want synchronous approval for any pre-page action, others will allow auto-execution for trivial reversible operations within tight bounds.

**Notification service as the thin shim to vendor APIs.** The Architect publishes mesh events; the notification service translates to Slack messages and (in post-page mode) PagerDuty incident comments. The notification service has its own narrow capability manifest — comment-on-PD-incident only, post-to-specific-Slack-channels only, no model access. A compromised notification service can spam Slack but cannot create false PagerDuty incidents, suppress real ones, or read mesh topics outside its subscription.

**The success metric is page-avoidance, not MTTR.** Convoy closures attest whether a page would have fired without intervention. Over time the team gets a running tally of how many pages Kjarr prevented, which is the number that justifies the system's cost. MTTR remains the right metric for fallback (post-page) mode, but the primary value lives in events that never reached MTTR's denominator.

## When pre-emption is wrong

The honest version: `pg-prod`'s hypothesis is plausible but wrong. Stats were a little stale, but they're not what's driving the latency trend — there's a connection-pool issue introduced by the same deploy that the agents don't see yet. `pg-prod` runs `ANALYZE orders` at 1:49:38. `mon` watches for the predicted reversal at 1:51:00 and observes none — the trend keeps climbing.

`mon` publishes `hypothesis_falsified`. The Architect doesn't close the convoy; it escalates. Now Kjarr is in a different posture: it has acted on a wrong hypothesis, the trend is still climbing, and the page is still imminent. The escalation path posts to Slack with higher salience:

⚠️ **Convoy `bd-c41` — hypothesis falsified, trend continuing**

Attempted `ANALYZE orders` (hypothesis: stale stats). Did not reverse trend.

`checkout` p99 still climbing, projected SLO violation in \~6m. PagerDuty will likely fire.

Current state: stats now fresh, plan now index scan. Bottleneck is downstream of query planning.

No further auto-actions. Standing by. Convoy: .

If the team has opted into pre-page direct paging for falsified-hypothesis-with-imminent-violation, Kjarr can also direct-page through PagerDuty. Most teams won't enable that initially — they'll let AlertManager fire the page in the normal way, with Kjarr's investigation already attached as context. Either way, the engineer wakes up (or AlertManager fires shortly) to a partially-investigated incident with a ruled-out hypothesis and a starting point: not stats, look downstream of the planner.

The pre-page envelope's bias toward inaction is what makes this safe. Kjarr ran one reversible intervention and stopped. It did not chain hypotheses, did not escalate its own authority, did not act faster as the trend worsened. The default for pre-page work is conservative: try the high-confidence reversible thing once, observe, hand off to humans if it didn't work. Aggressive autonomous remediation is for environments where the team has explicitly raised the envelope after watching Kjarr earn the trust.

## What this does not demonstrate

This is one Convoy on one quiet night. It does not address: simultaneous brewing anomalies across multiple services, conflicting hypotheses from multiple domain agents, agents disagreeing about the state of shared infrastructure, prompt injection attacks via Slack reactions, the case where the pre-page intervention causes a different incident, or the long tail of subtle trend signals that don't have clean hypotheses attached.

Each of those is a real design problem. None of them is unsolvable on this substrate, but each requires its own treatment. This document covers the simplest valuable case: one brewing trend, one domain hypothesis, one reversible intervention executed under a Slack-stoppable approval window, complete audit trail, no human disruption, no page fired. Everything else is an extension of these primitives.  
