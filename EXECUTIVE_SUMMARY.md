# Kjarr v1 — Executive Summary

*A distributed agent system for production operations. Designed for the security posture of regulated environments. Roadmap to v1 release attached.*

## The thesis

SRE teams are being asked to evaluate AI agents that run autonomously in production, and the existing options put security teams in an impossible position: agents that share the trust boundaries of a developer's laptop, with implicit access to credentials, infrastructure, and outbound network. Kjarr inverts the posture. Every agent runs jailed, communicates only through an audited mesh, reasons only through a single proxy that enforces capability manifests, and is monitored by an external integrity check. The design is complete; this document is the plan to ship a v1 that demonstrates the approach end-to-end.

## What v1 is

A working system that an SRE team interested in the design can deploy in their staging environment, run against real Postgres and real Prometheus, and evaluate honestly. Both canonical flows work end-to-end:

**Pre-emption (the primary value):** the monitoring agent notices a metric trending toward an SLO ceiling well below the team's existing alert threshold. A domain agent forms a hypothesis, the coordinator approves under a strict envelope, an intervention runs, the trend reverses. No page fires. The on-call engineer reads about it the next morning. *This is the case the team's existing tooling cannot do — the page that didn't happen because something below the threshold got addressed.*

**Post-page assistance (the fallback):** the team's existing alerting pipeline fires a real page through whatever mechanism they use. Kjarr arrives in parallel, investigates, comments on the existing incident with what it tried, and lets the engineer wake up to a partially-investigated problem rather than a cold one. Kjarr never owns the paging path; the team's alerting remains untouched.

The eight components are specified end-to-end (architect, two domain agents, three services, a Rust harness substrate, a witness for integrity). The trust story is enforced at five independent layers: container egress firewall, mesh group membership, proxy capability validation, harness type system, and external witness. A compromised agent's blast radius is bounded by capability manifest at each layer.

## Timeline and resourcing

| Phase | Goal | Estimate |
| :---- | :---- | :---- |
| 0 — Specs | Lock the wire-format and manifest schemas | 2 weeks |
| 1 — Foundation | Harness and proxy skeletons | 4–6 weeks |
| 2 — Vertical slice | First flow with one domain agent | 6–8 weeks |
| 3 — Horizontal expansion | Second agent, both flows working | 4–6 weeks |
| 4 — Notification surface | Three open-protocol channels | 2–3 weeks |
| 5 — Hardening | Real-environment exercise, failure modes | 4–6 weeks |
| 6 — Release | Documentation, deploy story | 2 weeks |

**Sequential: \~6 months. Aggressive parallelism: \~4–5 months.** Team size: 3–5 engineers with mixed Rust, distributed-systems, and SRE experience. Cross-cutting security review and dependency audit run continuously.

## What's deliberately *not* in v1

Threshold signing for the architect, federation for the witness, additional domain agents (Redis, Kubernetes, Terraform-state), Slack and PagerDuty adapters, multi-region deployment, LLM-augmented integrity checks, and Zulip-reaction approvals all defer to v2 or later. Each is called out explicitly in the roadmap with rationale. The deferrals are what make v1 shippable in 6 months instead of 18\.

## Risk posture

The three risks worth executive attention:

**Schema lock-in (Phase 0).** The mesh event format and capability manifest schema are the contract between every component. Slipping here ripples through every later phase. Mitigation: budget three weeks if there is any sign of disagreement during review, and require sign-off from at least two engineers and one security person before locking.

**Calibration (Phase 5).** False-positive brewing-tier anomalies erode team trust faster than almost anything else. If the monitoring agent fires too often without real underlying issues, the team will route around Kjarr. Mitigation: ship conservative defaults, design the calibration update loop into Phase 5, and make the first-90-days guide explicit about what to expect.

**Single-signer architect (v1 limitation).** The architect's manifest signing key is a single point of compromise in v1; threshold signing arrives in v2. Deployments using v1 should be considered staging-grade, with the migration path to threshold signing documented from day one. This is the one explicit limitation that gates promotion from v1 to a true production-grade deployment.

## The asks

To proceed, three decisions are needed:

**Engineering investment.** A 3–5 person team for 4–6 months. Skill mix matters more than headcount; one experienced Rust systems engineer is worth more than two who have to learn it.

**Security review involvement.** Security review is continuous, not one-time. We need a designated reviewer engaged from Phase 0 (schema review) through Phase 5 (penetration testing of the prompt-injection paths and MCP allowlist enforcement).

**Pilot deployment commitment.** Phase 5's "real-environment exercise" presumes a place to exercise against. Lining up a willing internal team or pilot customer should already be in motion by Phase 3, ideally earlier. This is upstream of the engineering work; without it, v1 ships into a vacuum.

The detailed roadmap with checkable tasks, dependencies, open decisions, and risk registry is in [TODO.md](http://./TODO.md). The complete design corpus (architecture document, eight component specs, two flow walkthroughs) is in the companion documents.  
