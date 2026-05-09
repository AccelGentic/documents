# **Bastion vs. Gas Town**

*Why fleet-deployed agents need a different substrate.*

## **What Gas Town is**

Steve Yegge's Gas Town is a real piece of work. In seventeen days he built a coordination layer for fleets of coding agents — Mayor, Polecats, Witness, Refinery, Deacon — sitting on top of a git-backed issue tracker (Beads) and tmux. It works. Yegge runs twenty-plus Claude Code instances against his own repos and ships at speeds nobody else seems to match. The MEOW stack — Molecules, Wisps, Patrols, Convoys — is a genuinely interesting model for durable workflow execution against nondeterministic agents, and the role taxonomy is the cleanest decomposition anyone has published for swarm-shaped agent work.

But the assumptions encoded in its design are workstation assumptions. Gas Town is a developer's tool, deployed to a developer's laptop, operated by a developer who is simultaneously the security boundary, the audit log, and the kill switch. Yegge says this himself: it "can wreck your shit in an instant," and the right answer if you don't like that is "go back to your IDE and shelter in place." That posture is acceptable when the blast radius is one repo and the developer is in the chair. Enterprise infrastructure cannot accept that posture, because the blast radius is everything.

Bastion takes the same intellectual lineage — supervised swarms of LLM-driven workers, durable work expressed as a graph, role-specialized agents coordinating through a shared substrate — and rebuilds it for the inverse deployment posture: distributed, untrusted, audited, and operationally sovereign.

## **Where Bastion diverges, structurally**

### **Distribution, not colocation**

Gas Town colocates every worker on a single host, sharing a tmux session and a filesystem. The control plane is a directory of JSONL files watched by a daemon. This is elegant on a laptop and impossible across infrastructure.

Bastion distributes workers to where the workloads they manage actually run — production clusters, customer regions, isolated VPCs, edge sites. Each agent lives where its work lives. Communication happens over an encrypted peer-to-peer mesh built on p2panda, with decentralized authentication, offline tolerance, and append-log semantics. The control plane works identically whether agents are next door or on another continent, and the mesh itself enforces who can talk to whom through cryptographic group membership.

### **Zero trust, not peer process**

Gas Town agents are trusted process peers. They share the developer's credentials, the developer's shell, the developer's outbound network, and the developer's API keys. Compromise of any one agent — through prompt injection, malicious tool output, or an upstream model issue — gives an attacker the developer's full operating context.

Bastion treats every agent as potentially compromised at all times. Each runs in its own container, firewalled to a tiny set of allowed destinations: its application's port, the control mesh, and the model proxy. Nothing else. The harness verifies a signed capability manifest at startup; the Witness verifies the same manifest from the outside; the proxy verifies the agent's identity on every request and refuses any operation outside its declared capabilities. Compromise of one agent gives an attacker nothing it didn't already have at the application layer.

### **Cognition through a single audited path**

Gas Town agents call model providers directly using the developer's API key. There is no per-agent audit log, no policy enforcement, no budget enforcement, and no way to revoke a single agent's cognition without nuking the shared credential and breaking everyone else.

Bastion forces all model traffic through an audited proxy. The proxy is the only outbound path for cognition, and it is the single chokepoint where token budgets, MCP allowlists, capability validation, and kill-switch revocation are enforced. Every prompt, every response, every tool declaration, every tool call is recorded against the agent that made it. Revoking a compromised agent is one operation at the proxy; the container keeps running for forensics, but it cannot think.

### **Cryptographic identity, not file paths**

Gas Town identities are file paths and tmux session names, durable only for as long as the developer's machine is. Bastion identities are cryptographic: each agent has a p2panda-auth keypair, a signed capability manifest issued by the Architect at provisioning time, and a verifiable position in the group permission graph. This integrates cleanly with existing enterprise IAM through the proxy and the Architect, gives you a real chain of custody for every action, and survives container restarts because identity lives in the mesh, not in local state.

### **Operations, not code generation**

Gas Town optimizes for code generation: polecats produce merge requests, the Refinery lands them, convoys ship features. Bastion is built for deployment and operations: agents apply infrastructure changes, run remediations, manage rollouts, respond to alerts, execute runbooks. The work model is recognizable — durable workflows, supervisor patterns, ephemeral workers — but the consequences of action are categorically different. A bad MR can be reverted; a bad terraform apply against production cannot. The trust model and the Refinery's role are calibrated to that asymmetry.

### **Engineered, not vibe-coded**

Yegge is explicit that Gas Town is vibe-coded, three weeks old, and may not survive twelve months. He's never read the source. That's the right tradeoff for his use case and his audience.

The Bastion harness is the thing whose correctness underwrites the security of the whole system. It is a typed Rust binary with a statically-verified state machine, signed and version-pinned manifests, reproducible builds, and explicit version-skew handling between the control plane and deployed harnesses. Rebuilding it should feel like a real event, not a casual git push. The Witness verifies that any running harness matches a known-good build hash before admitting it to work topics.

## **Why this matters for the enterprise**

### **Audit and compliance**

The proxy log is a complete, structured record of every cognitive action any agent has taken — request, response, tool calls, costs, timing, identity. The p2panda event log is a complete, durable record of every work assignment, claim, status update, and result. Together they form the evidence base SOC 2, HIPAA, PCI, and GDPR auditors expect: who did what, when, on whose behalf, with what authority. Gas Town's evidence base is git history plus tmux scrollback.

### **Cost governance**

Token spend is visible and bounded per-agent, per-rig, per-team, and per-cost-center at the proxy. Budgets are enforced at request time, not reconciled monthly from a vendor invoice. An agent that exceeds its budget cannot think; an agent that gets prompt-injected into a runaway loop cannot exhaust the bill before being cut off. Finance can see real-time burn against allocated budgets in the same dashboard the SRE team uses.

### **Security posture defensible to a CISO**

Zero trust between agents, defense in depth at the proxy and harness, least privilege via signed capability manifests, an explicit kill switch, and tenant isolation through cryptographic group membership are not aftermarket additions. They are how the system is constructed. The capability manifest is the agent's authorization document, the harness refuses to operate outside it, the proxy refuses to forward requests that violate it, and the Witness refuses to register a runtime that doesn't match it. The blast radius of any one compromise is bounded by design and verifiable in code, not by operator vigilance.

### **Operational sovereignty**

Bastion can be operated by an SRE team, not by one wizard who built it. Reproducible builds, versioned manifest schemas, federated Architect, durable state in the mesh, and clean restart-as-replay semantics mean the system survives the loss of any single component, region, or operator. Runbooks for upgrading the harness, rotating credentials, and recovering from a regional outage are real artifacts the on-call rotation can execute. Gas Town in its current form requires Yegge — or someone willing to operate the way Yegge operates — at the wheel.

### **Change management and multi-tenancy**

The Refinery serializes state-mutating operations against shared infrastructure, so two agents cannot simultaneously apply conflicting changes to the same production system. Human approvals are wired in at the proxy or Architect for any state transition above a configured impact threshold. Rollbacks are first-class because the work graph is durable in the mesh and reversible operations are typed as such in the capability manifest. Different teams run agents under different p2panda-auth groups, different proxy policies, different cost centers, and different audit scopes, sharing the substrate without sharing the blast radius. This is the integration surface enterprises actually need with their existing change-management, IAM, and FinOps processes.

## **What Bastion gives up**

Bastion is heavier to deploy than Gas Town. There is a control plane to stand up, a proxy to operate, a harness to build and verify, and an Architect to keep available. For one developer working on one repo on one laptop, Gas Town is the right tool. Bastion does not try to replace it for that audience.

The iteration loop is also slower. Gas Town's tmux-driven feedback is real-time and intimate; Bastion's mesh-driven feedback is structured but asynchronous. This is a feature for production work — slow, deliberate, audited — and a tax on exploratory work. The two systems solve adjacent problems, not the same problem.

## **Summary**

Gas Town is the right tool for a developer running twenty coding agents on their laptop, optimizing for personal throughput in a single repo. Bastion is the right tool for an enterprise running operational agents across production infrastructure, where the blast radius matters, the audit log matters, the cost matters, and the security posture has to survive a CISO review. Same intellectual lineage. Completely different deployment posture, trust model, and operational story.

Yegge built the first viable agent orchestrator. Bastion is what the same idea has to become to ship into production.

