# Kjarr? \- for John Mark

A working document for you to take in. Take this as seriously as you want. Written peer-to-peer, not as a pitch. The pitch lives in the executive summary and the architecture HTML; this is our conversation behind those docs.

## Where things actually stand

The design is complete. Eight components specified end-to-end, two canonical flows walked through, a full architecture document with diagrams, a comparison against Gas Town that explains why this approach inverts the existing trust posture, and a six-phase implementation roadmap with explicit risk registry and post-v1 deferrals. None of it is hand-wave. Anyone with serious engineering experience can read the corpus and form a real opinion about whether it'll work; that's the bar I held the design to, and I think it clears it.

What does *not* exist yet is the implementation. That's the next 3–4 months.

## The aggressive timeline

The TODO.md says 6 months for a 3–5 person team. That estimate is honest for a normal team scenario. The reality I'm planning for is different: **team of one, focused, until v1**. I expect to cut the timeline roughly in half — call it 3–4 months sequentially, with a real chance of landing closer to 3 if Phase 0 (schemas) goes cleanly.

The math for one focused engineer:

| Phase | Team-of-5 estimate | Solo-focused estimate |
| :---- | :---- | :---- |
| 0 — Specs | 2 weeks | 1 week |
| 1 — Foundation | 4–6 weeks | 2–3 weeks |
| 2 — Vertical slice | 6–8 weeks | 3–4 weeks |
| 3 — Horizontal expansion | 4–6 weeks | 2–3 weeks |
| 4 — Notification surface | 2–3 weeks | 1 week |
| 5 — Hardening | 4–6 weeks | 2–3 weeks |
| 6 — Release | 2 weeks | 1 week |
| **Total** | **24–33 weeks** | **12–16 weeks** |

Why this isn't fantasy: solo dev cuts communication overhead to zero, eliminates parallelism coordination cost, and lets a single person hold the whole system in their head — which for a system of this design density is actually faster than splitting across a team. The component specs are detailed enough that I'm not inventing as I go; I'm translating prose to types. I’m also going to burn a pile of tokens at Claude to help drive this faster. The biggest unknown is Phase 5 (hardening against real environments), and that's the phase where solo doesn't help — but it also doesn't hurt much, because the work is mostly investigative, not throughput-bound.

The biggest *risk* of solo-dev is the bus factor of one. If you’re serious about this, I’m in. For realzies. 

## The math for investors

This is what holds up if you go looking for reference customers and need to explain why the engineering side isn't the bottleneck.

**Capital efficiency.** A single experienced systems engineer shipping v1 in 3–4 months is roughly $60–$100k of fully-loaded engineering cost, depending on how you count. Compared to the AI agent companies raising at unicorn valuations to pay teams of 30, the spend gap is two orders of magnitude. This is not a feature; it's the consequence of starting with a complete design and a tight scope. I’m going to burn a LOT of tokens to get there this fast. I’m ok with that ***IF*** you are in. 

**De-risked execution.** The single biggest investor question on early-stage technical bets is "can they actually build it?" The complete design corpus answers that question before the conversation starts. A serious engineering reviewer can read the harness spec and the proxy spec and form an opinion in an afternoon. That changes the diligence dynamic entirely.

**Parallel commercial work.** While I'm building, you're talking. You own it all. I work for you. This is a new dynamic for us, but honestly it would be a relief to actually trust someone I work for again. Reference customer recruitment, pilot deployment commitments, OSS community engagement, design partner conversations, I’m there where you need me, but all of that can runs in parallel with implementation, and Phase 3 (≈8 weeks in) is when the first demo flow is real enough to walk a customer through. By the time v1 ships, we can have friendly committed pilots waiting.

**Open source as GTM accelerant.** The notification surface is deliberately built around open protocols (Zulip, Matrix, ActivityPub), not enterprise SaaS. This is partly principled — the trust story holds up to CISO review — and partly tactical: it removes the "we need to onboard a vendor" friction from initial evaluation. A team can clone, deploy in staging, and have a working demo without a procurement conversation. That's the difference between a 4-week eval cycle and an 8-month eval cycle.

## The reference customer pitch

Three audiences, three angles:

**For SRE/Platform leads:** "It catches problems before they page you. It runs alongside whatever you already have — your AlertManager, your existing pager, your existing dashboards. Nothing about your on-call workflow changes. The agent investigates the same metrics you would have, faster, and either fixes the small things or documents what it found and stops. You evaluate it for a month, you keep it if it earned its keep."

**For CISOs:** "Every AI agent vendor will eventually need to answer for their security posture. We're answering that question first. Capability manifests, audited cognition, external integrity check, type-system enforcement. Every claim is backed by an enforcement layer. We can't fire pages we're not authorized to fire — that's a property of the system, not a promise."

**For CTOs/VPs Eng evaluating the broader market:** "This is what the post-hype agent infrastructure looks like. Open protocols, self-hostable, capability-bounded by design. If your team is going to run autonomous agents in production over the next 24 months, the security model is the question that hasn't been answered yet by anyone else."

## We need a company. Legit, this thing has legs.

You're the face. The design ships under your sponsorship, your company, your direction. Community engagement, comms, the open-source narrative arc, design partner conversations, investor introductions, and eventually the hire/raise decision when it's time to scale beyond solo. You'll know better than I do when that moment arrives. I’m here to support and drive what you want to offload. You know me, so this doesn’t have to be a whole thing. I want 20%, 80% is yours to do with what you will. I trust the process and want to legit do something amazing. 

What we need, concretely:

**Pilot deployment commitments by Phase 3 (≈8 weeks in).** This is the single highest-leverage thing on your side. Phase 5 (hardening) is materially better if there's a real production-shaped workload to exercise against. One committed pilot is enough to start; two is great. Without one, Phase 5 is harder and v1's "real-environment exercise" is more theoretical than it should be. We can do some of this work in AWS if we have too, honestly, it just won’t be as good without real workloads.

**OSS license and project home decision before Phase 6 (≈12 weeks in).** Apache 2.0, BSL, custom \- your call. I have opinions but defer. Same for repo home: GitHub vs. self-hosted Forgejo vs. something else. I’ll advise, but it’s your call.

**Active investor pipeline by month 4\.** You already know all of this. 

**Periodic friction-reducing nudges.** Solo dev gets stuck on small things sometimes \- I am a walking ADHD stereotype. Maybe weekly 30-minute syncs are right; maybe async-but-responsive on Signal is right. We'll figure out the cadence, but I ***need*** external feedback. I am fire when I’m focused. I am a dumpster fire when I’m not. 

## The artifacts

Everything is in the google drive. In the order you'll probably want to walk through:

| For | Document |
| :---- | :---- |
| 30-second pitch | EXECUTIVE\_SUMMARY.md |
| Visual walkthrough (the show-don't-tell version) | kjarr-architecture.html |
| The two scenarios in detail | kjarr-use-case-pre-emption.md, bastion-use-case-3am.md |
| Why this is different from Gas Town | bastion-vs-gastown.md |
| Component design specs | architect-design.md, pg-prod-design.md, mon-design.md, refinery-design.md, notification-service-design.md, harness-spec.md, proxy-spec.md, witness-spec.md |
| Implementation roadmap | TODO.md |

Two of those documents still say "Bastion" in places — the project was renamed during design and a few files predate the rename. I'll do a corpus-wide rename pass before the public release; for internal use it's fine. Also, we can change it to something else. Names are hard.

## The honest version

Things that could go wrong, so you can speak to them when they come up:

- **Solo work leads to burn out.** Real risk. Mitigation: design corpus is complete, code is checked in continuously, work is described in commit messages well enough that a second engineer could continue. But there's no substitute for the one person who's been holding the system in their head, and we should be honest about that.  
- **Phase 0 (schemas) takes longer than a week.** Possible. If it slips to two, the whole timeline shifts but the math still works. If it slips to three, we should talk.  
- **Calibration in Phase 5 surfaces real problems.** The brewing-tier query library and the confidence calibration table are the two parts most likely to need iteration after first contact with real workloads. We may end up shipping v1.0 with conservative defaults and tuning aggressively in v1.1 over the following month. Worth setting expectations accordingly.  
- **The "AI agent" market gets ugly before we ship.** Either by an embarrassing public incident from a bigger vendor (which actually helps us) or by market fatigue (which doesn't). We're positioned to benefit from the first and to weather the second because the design speaks to durable problems, not the current hype cycle. But timing matters and we should pay attention.

## Closing

The conversation today was freaking awesome. You could do this with someone else. I could do this with someone else. I don’t want to. The design holds up. The execution path is short. The market need is durable and getting clearer every month as the agent-security gap widens. The aggressive timeline is realistic, not ambitious. I start burning tokens tomorrow. Think on it.

Test the market. Find the folks, I'll ship and come sales engineer the shit out of this when needed.

Holler with questions.  
\-theron