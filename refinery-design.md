# Refinery — design

*The Refinery serializes state-mutating operations against shared infrastructure. v1 is a lease service with a small policy layer; the LLM-driven smart-arbiter version is deferred. This is the right kind of boring.*

## Role

The problem the Refinery solves is concrete: two domain agents try to mutate the same shared resource at the same time. `pg-prod` wants to ANALYZE the orders table during a convoy; a sibling `pg-prod-migrations` agent (in some future version with that role) wants to add an index on the same table; a human runs a maintenance script through the team's existing tooling. Without coordination, these can collide. With coordination, they serialize.

In the current v1 design only `pg-prod` does mutations against the database, so the immediate need is small. But the architecture has to support multiple operator agents per resource eventually, has to coordinate with operations the team performs through their existing channels, and has to make the coordination story coherent before the system grows past one operator agent per resource. Designing in the lease service now makes the rest of the agent design honest about how concurrent mutations are handled.

The Refinery is named after Yegge's Gas Town role, but it does much less than his version. Yegge's Refinery is a smart agent that reasons about merge queue ordering and resolves rebase conflicts. Kjarr's v1 Refinery is a lease manager. It does not reason. It serializes.

## What v1 is

A small service (similar in shape to the notification service — non-LLM, narrow capability profile) that holds typed leases on declared resources, grants leases on request, releases them on completion or timeout, and refuses concurrent mutations of the same resource. That's the entire job.

A lease has:

- A resource identifier (e.g., `postgres-prod:orders`, `postgres-prod:idx_orders_loyalty_tier`)  
- A holder identity (the agent's p2panda-auth identity)  
- An operation type (one of a fixed enum: ANALYZE, REINDEX, KILL\_QUERY, etc., extensible per-resource)  
- A grant timestamp  
- A TTL (expiry, after which the lease is auto-released)  
- An optional renewal counter

Resources are typed and configured. The deployment manifest declares which resources require leasing for which operation types. ANALYZE on `orders` requires a lease on `postgres-prod:orders`; REINDEX requires a lease on the specific index. Read operations don't require leases (they don't conflict with each other). The lease taxonomy is part of the deployment configuration, signed by the Architect, the same trust path as the rest of the system's configuration.

## Lease lifecycle

When a domain agent receives convoy approval to perform a state-mutating operation, it doesn't immediately execute. It first requests a lease from the Refinery:

1\. Agent: request\_lease(resource, operation, holder=self\_identity)

2\. Refinery: checks current leases on resource

   \- if no conflicting lease: grant, return lease\_id and TTL

   \- if conflicting lease held by another agent: deny, return holder identity and remaining TTL

   \- if conflicting lease expired: reclaim, grant new lease

3\. Agent (on grant): execute operation, then release\_lease(lease\_id)

4\. Agent (on deny): publish a wait event, retry after the indicated TTL, or escalate

Lease grants and releases are mesh events, signed by the Refinery. The full history is in the durable mesh log; replaying it gives a complete audit of every mutation against every leased resource.

If the agent crashes or the convoy stalls without releasing the lease, the TTL eventually expires and the resource becomes available again. The default TTL is operation-specific (90 seconds for ANALYZE, longer for REINDEX which can run for hours, much shorter for KILL\_QUERY which is near-instant). An agent can renew a lease before TTL expiry if its operation legitimately needs more time, with renewals logged separately.

## Conflict policy

The default policy is simple: one mutator at a time per resource, FIFO on lease requests. No prioritization, no preemption, no smart ordering. If `pg-prod` is holding a lease on `postgres-prod:orders` and another agent requests the same lease, the second request is denied with the holder's identity and TTL, and the requesting agent decides whether to wait or escalate.

This simplicity is deliberate. Smart prioritization (which lease request is more urgent, which can be safely interleaved, which can preempt) is exactly the kind of thing the LLM-driven smart-arbiter version of Refinery would handle. v1 doesn't need it. Real workloads at v1 scale will have so few concurrent mutation requests on the same resource that FIFO is fine.

A small set of policy hooks is configurable per resource:

- TTL per operation type  
- Whether the resource permits multiple concurrent readers (yes for most, but worth being explicit)  
- Renewal limits (how many times can a lease be renewed before forced release)  
- Preemption authority (who can revoke a lease — typically only the Architect, and only with explicit operator approval)

Anything more sophisticated than this — priority queues, dependency-aware ordering, batched coordination — waits for v2.

## Capability profile

Like the notification service, the Refinery is not an agent and does not have model access. It's a small Rust service with:

**Mesh subscriptions.** `lease_request.*`, `lease_release.*`, `lease_renew.*`, plus the resource-configuration topic for live updates to the lease taxonomy.

**Mesh publications.** `lease_granted`, `lease_denied`, `lease_released`, `lease_expired`, `lease_revoked`, `health_status`. Every lease state transition is a signed mesh event for full auditability.

**No external API access at all.** The Refinery doesn't touch infrastructure. It doesn't call Postgres, doesn't call Kubernetes, doesn't call anything external. It only manages its internal lease state and publishes events about that state. This is the cleanest capability profile in Kjarr.

**Persistent state.** The current set of held leases must survive Refinery restart — otherwise an agent holding a lease at the moment of restart would suddenly find its lease unprotected, and a sibling agent could start a conflicting operation. Persistence is the mesh log itself: lease state is reconstructed from replaying the lease event topic on startup. This makes the Refinery effectively stateless from a deployment perspective; restart is just catch-up.

## Why not just use the database's locking?

A reasonable question. Postgres has its own locking; ANALYZE and REINDEX acquire appropriate locks; concurrent operations are already serialized at the database level.

The reason to layer the Refinery on top is twofold. First, the Refinery coordinates *before* the database lock is acquired. Two agents requesting leases that would conflict at the database level can be denied at the Refinery layer without ever reaching the database, which is faster and avoids waiting on database lock acquisition timeouts. Second, the Refinery coordinates across systems that don't share a locking primitive. Once Kjarr deploys agents against terraform state, kubernetes resources, Redis clusters, etc., the database's locking is irrelevant — the Refinery is the cross-system coordinator. v1 only deploys `pg-prod` so the cross-system case is hypothetical, but designing in the Refinery layer now means the architecture supports it later without a refactor.

There's also an audit argument. Database lock acquisition is invisible in any meaningful operational sense — the team can see that ANALYZE was running, but not that it was *competing* with another operation. Refinery events make competition explicit and auditable: the mesh log records every lease request, grant, denial, and release.

## Federation

Like the Architect, the Refinery is a coordinator and benefits from federation for availability. Unlike the Architect, the Refinery's state can be reconstructed from the mesh log alone, so the federation model is simpler.

Multiple Refinery instances run, all subscribing to the lease topic, all maintaining the same view of held leases. Lease grants require a leader-elected quorum to prevent the split-brain case (two Refinery instances granting conflicting leases simultaneously). The leader election runs through p2panda-auth group consensus; failovers are seconds. During leader transitions, lease grant requests are queued briefly; in-flight leases continue to apply.

The threshold-signing model used for the Architect's manifest signing is overkill for the Refinery — leases are short-lived and revocable, where capability manifests are durable trust grants. Simple leader election with quorum-confirmed grants is sufficient. v1 can ship with single-instance Refinery and the standard hot-standby pattern; multi-instance with quorum is a v1.5 hardening item.

## Failure modes

**Refinery unreachable.** Domain agents can't get leases. State-mutating operations queue or fail. Read operations are unaffected. The Architect publishes a degraded-mode notification to Slack so the team knows that Kjarr can't currently coordinate mutations. If the Refinery comes back, queued operations resume; if it stays down, in-flight convoys requiring mutations escalate to humans (who can perform the operation through their existing tooling, with the team's existing change-management discipline).

**Lease leak.** An agent acquires a lease, then crashes before releasing, and the TTL is too long. The resource is unavailable for the TTL duration. Mitigation: TTLs are tuned per operation type to reasonable bounds, the Architect can revoke leases on operator request, the Witness flags agents holding leases longer than expected for an investigation.

**Conflicting lease grant.** A bug or a federation split-brain causes two leases to be granted on the same resource. Detection: the mesh event log shows the conflict on replay, the Refinery (or any peer subscribed to the lease topic) flags the inconsistency. Recovery is to revoke one of the leases and let the affected agent retry. This is a serious bug if it happens in production and would warrant a postmortem; the federation design with quorum-confirmed grants is intended to make it impossible.

**Coordination with non-Kjarr operations.** A human runs ANALYZE through psql at 2 AM during a convoy. Kjarr has no way to know. The lease wasn't acquired through the Refinery; the conflict happens at the database lock level; the agent's ANALYZE call waits or fails with a Postgres-level error. Mitigation: this is fundamentally an integration problem with the team's existing operations, and the right answer is for the team's existing tooling to also acquire Refinery leases when performing mutations on resources Kjarr coordinates. Until that integration exists, the failure mode is "Kjarr's operations might collide with human operations and one of them will lose to database locking." Annoying but recoverable.

## What v1 doesn't do

**Smart ordering.** No prioritization, no batching, no LLM judgment about which lease request is more important. FIFO.

**Preemption based on judgment.** Leases are not preempted unless the operator explicitly revokes them through the Architect.

**Cross-resource transactions.** The Refinery grants leases on individual resources, not on transactional sets. If an operation needs to mutate three resources atomically, the agent acquires three separate leases (in a defined order to avoid deadlock) and holds all of them until the operation completes. v2 might add transaction-set leases.

**Auto-resolution of conflicts.** When a conflict is detected (somehow), the Refinery flags it and waits for human intervention. It does not attempt to resolve.

**Coordination across multiple Kjarr deployments.** The Refinery is per-deployment. A multi-deployment scenario (a single team running multiple Kjarrs against the same shared infrastructure) is out of scope.

## When the smart-arbiter version becomes worth building

The right time to upgrade the Refinery from a lease service to an LLM-driven arbiter is when the FIFO policy starts producing operational frustration: "the migration was queued behind a routine ANALYZE for 40 minutes," "a brewing-tier intervention was blocked by a low-priority maintenance task," "the reorder of these three operations could have completed in half the time." When the team starts manually reordering lease requests through the Architect to work around FIFO, that's the signal that smart arbitration would earn its keep.

Until then, FIFO is fine. Most teams' production workloads simply don't have enough concurrent mutations against the same resources to need anything cleverer.

## What's next

The harness. This is the document that turns all of this prose into Rust types and the place where the implementation can actually start. The harness is the trust boundary that underwrites every capability claim in every preceding document — if the harness is correct, the design is buildable; if the harness is wrong, the design is theatre.

Harness specification next.  
