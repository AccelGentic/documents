# Harness — specification

*The harness is the Rust binary every Kjarr agent runs inside. It owns the p2panda identity, owns the model client, owns the tool execution boundary, and enforces the capability manifest. This document specifies the harness in enough detail that a Rust engineer can begin implementation without inventing primitives along the way.*

## What the harness is, again

The harness is the trustworthy boundary between an LLM-driven reasoning loop (which we assume can be hijacked at any time) and the rest of the world (which must not be). Every claim in the preceding designs about capabilities being scoped, credentials being protected, tools being typed, and identity being durable is a claim about the harness. If the harness is correct, the system's security properties hold. If the harness is wrong, the agents are running with the same effective trust as a developer's laptop.

The harness is small on purpose. Every feature added to it expands the trust boundary. The discipline is: anything that doesn't strictly need to be in the harness should live elsewhere — in the proxy, in the control plane, in the Witness, in operator agents.

## The single binary

One Rust binary, configured at startup with a manifest file and an identity. The same binary runs every agent — `pg-prod`, `mon`, future agents — with role-specific behavior coming entirely from the manifest and the role-specific prompt. There is no per-agent harness fork. There are no per-agent compile-time customizations. This matters because the Witness's job becomes "verify the running harness binary matches a known-good build hash"; if there were per-agent harnesses, the verification surface multiplies and the supply-chain story weakens.

The binary's responsibilities, in dependency order:

1. Load and verify identity (sealed local store, p2panda keypair, signed by Architect's bootstrap)  
2. Load and verify manifest (signed by Architect, schema-versioned)  
3. Connect to p2panda mesh (subscriptions per manifest)  
4. Establish proxy connection (authenticated with identity, scoped to manifest)  
5. Build tool registry from manifest  
6. Initialize state machine in `Provisioning` state  
7. Begin the reasoning loop

If any step fails, the harness exits with a structured error event published to the mesh on a best-effort basis (because it may not be connected yet). The Witness sees the failure and the deployment knows the agent didn't come up.

## Type system

The harness's correctness rests on Rust's type system more than on runtime checks. The type design is therefore load-bearing.

### Core identity types

// The agent's durable identity, loaded from sealed store

pub struct AgentIdentity {

    pub keypair: Ed25519Keypair,        // p2panda-auth keypair

    pub agent\_id: AgentId,              // stable identifier

    pub role: AgentRole,                // pg-prod, mon, etc.

    pub provisioned\_at: Timestamp,

    pub bootstrap\_signature: Signature, // Architect's signature on this identity

}

pub struct AgentId(Uuid);

pub enum AgentRole { 

    PgProd, 

    Mon, 

    RedisProd, 

    K8sProd,

    /\* extensible per deployment \*/

}

`AgentRole` is an enum, not a string. A new role requires a harness rebuild. This is deliberate — adding a role is a security-relevant decision and should require the same discipline as any other harness change.

### Capability manifest

The manifest is the document that defines everything the agent is allowed to do. It is loaded from a signed file at startup and verified against the Architect's signing key.

pub struct Manifest {

    pub version: ManifestVersion,

    pub agent\_id: AgentId,

    pub role: AgentRole,

    pub issued\_at: Timestamp,

    pub expires\_at: Option\<Timestamp\>,

    pub capabilities: Capabilities,

    pub mesh: MeshConfig,

    pub model: ModelConfig,

    pub envelope: ApprovalEnvelope,

    pub topology: Topology,           // role-specific dependency / service graph

    pub signature: Signature,

}

pub struct Capabilities {

    pub read\_scopes: Vec\<ReadScope\>,

    pub write\_operations: Vec\<WriteOperation\>,

    pub mesh\_publications: Vec\<TopicPattern\>,

    pub mesh\_subscriptions: Vec\<TopicPattern\>,

}

pub enum ReadScope {

    PgStatView { view: PgStatView, columns: Vec\<String\> },

    PgCatalog { catalog: PgCatalog },

    PrometheusQuery { template\_set: TemplateSetId },

    AlertManagerRules,

    GrafanaAnnotations { dashboards: Vec\<DashboardId\> },

    /\* extensible \*/

}

pub enum WriteOperation {

    PgAnalyze { tables: Vec\<TableId\> },

    PgReindexConcurrently { indexes: Vec\<IndexId\> },

    PgKillQuery { constraints: KillQueryConstraints },

    PgVacuumAnalyze { tables: Vec\<TableId\> },

    GrafanaAnnotationCreate { dashboards: Vec\<DashboardId\>, tag\_prefix: String },

    /\* extensible \*/

}

The key property: every variant of `ReadScope` and `WriteOperation` corresponds exactly to a tool in the registry. Adding a tool requires adding a variant; the type system makes it impossible for a tool to exist that isn't represented in the manifest schema.

### Tool registry

pub trait Tool: Send \+ Sync {

    type Args: DeserializeOwned \+ Validate;

    type Output: Serialize;

    

    fn name(\&self) \-\> &'static str;

    fn description(\&self) \-\> &'static str;

    fn schema(\&self) \-\> serde\_json::Value;

    

    fn validate(\&self, args: \&Self::Args, manifest: \&Manifest) \-\> Result\<(), ValidationError\>;

    

    async fn execute(

        \&self, 

        args: Self::Args, 

        ctx: \&ExecutionContext,

    ) \-\> Result\<Self::Output, ToolError\>;

}

Each tool implements this trait once. The harness builds the active tool set at startup by consulting the manifest:

fn build\_tool\_set(manifest: \&Manifest) \-\> ToolSet {

    let mut tools \= ToolSet::new();

    for op in \&manifest.capabilities.write\_operations {

        match op {

            WriteOperation::PgAnalyze { tables } \=\> {

                tools.register(PgAnalyzeTool::new(tables.clone()));

            },

            // ...

        }

    }

    for scope in \&manifest.capabilities.read\_scopes {

        // analogous for read tools

    }

    tools

}

A tool that isn't in the manifest is never constructed, never registered, never callable. The reasoning loop sees only the tools the harness built; the proxy sees only the same tools (declared via the model API's `tools` parameter generated from this same registry). The model literally cannot invoke a tool that isn't in the manifest because the model never has a name for one.

The `validate` method on each tool runs at every invocation, against the live manifest. This is defense in depth — even if somehow a tool ended up in the registry that shouldn't have, validation against the manifest at execution time would reject the call.

### State machine

The agent's lifecycle is a Rust enum. Illegal transitions are compile errors.

pub enum AgentState {

    Provisioning,

    Registering { manifest: Manifest },

    Idle { since: Timestamp },

    Triggered { event: MeshEvent, started\_at: Timestamp },

    Reasoning { event: MeshEvent, model\_call\_id: ModelCallId },

    Proposing { hypothesis: HypothesisDraft },

    AwaitingApproval { convoy: ConvoyId, hypothesis: Hypothesis },

    Executing { convoy: ConvoyId, operation: ScheduledOperation },

    Reporting { convoy: ConvoyId, result: OperationResult },

    SelfWatch { cycle\_id: CycleId },

    Confirming { intervention: InterventionRef, predicted\_outcome: Outcome },

    Blocked { reason: BlockedReason, since: Timestamp },

    Errored { error: AgentError, since: Timestamp },

    Quarantined { reason: QuarantineReason },

    Handoff { to: HandoffTarget },

}

pub struct AgentSupervisor {

    state: AgentState,

    // ...

}

impl AgentSupervisor {

    pub async fn transition(\&mut self, transition: StateTransition) \-\> Result\<(), TransitionError\> {

        // matched on (current\_state, transition); illegal pairs are exhaustive errors

    }

}

The supervisor is the only component that can transition state. Every transition emits a `health_status` event to the mesh. Restarts begin in `Provisioning` and use mesh sync to reconstruct the most recent state — restart-as-replay.

### Concurrency model

The harness is a Tokio application with a small fixed set of long-running tasks coordinated by bounded channels:

pub struct Harness {

    network\_task: NetworkTask,        // owns p2panda node, mesh I/O

    reasoning\_task: ReasoningTask,    // owns model client, drives prompt loop

    tool\_task: ToolTask,              // executes tools, owns external connections

    reporter\_task: ReporterTask,      // consumes events, publishes to mesh

    supervisor\_task: SupervisorTask,  // owns state, coordinates shutdown

    

    // bounded channels between tasks

    network\_to\_reasoning: mpsc::Sender\<MeshEvent\>,

    reasoning\_to\_tool: mpsc::Sender\<ToolCall\>,

    tool\_to\_reasoning: mpsc::Sender\<ToolResult\>,

    everything\_to\_reporter: mpsc::Sender\<StatusEvent\>,

}

Channels are bounded. If the reporter can't keep up with status events, the reasoning task blocks. Backpressure, not unbounded buffering. A misbehaving agent that produces events faster than the mesh can ingest them slows itself down; the failure is visible.

## The credential boundary

Owned by the harness, never visible to the reasoning loop. Concretely:

pub struct ExternalConnections {

    postgres: Option\<PgPool\>,         // pg-prod harnesses only

    prometheus: Option\<PromClient\>,   // mon harnesses only

    grafana: Option\<GrafanaClient\>,   // mon harnesses only

    // ...

}

pub struct ExecutionContext\<'a\> {

    connections: &'a ExternalConnections,

    manifest: &'a Manifest,

    proxy: &'a ProxyClient,

    // ...

}

`ExecutionContext` is what tools receive. It exposes connection access only through methods that take typed arguments (a table name, a query template ID) — never raw connection access, never the credentials themselves. The reasoning task never has access to `ExecutionContext`; it only sends `ToolCall` messages to the tool task, which holds the context.

// reasoning task: produces tool calls, never touches connections

async fn reason(\&mut self, event: MeshEvent) \-\> Result\<Vec\<ToolCall\>, ReasoningError\> {

    let response \= self.model\_client.invoke(/\* ... \*/).await?;

    let tool\_calls \= self.extract\_tool\_calls(\&response)?;

    Ok(tool\_calls)

}

// tool task: executes tool calls, holds connections

async fn execute(\&mut self, call: ToolCall) \-\> Result\<ToolResult, ToolError\> {

    let tool \= self.registry.get(\&call.tool\_name)?;

    tool.validate(\&call.args, \&self.manifest)?;  // re-validate at execution time

    let result \= tool.execute(call.args, \&self.ctx).await?;

    Ok(result)

}

A compromised reasoning task — through prompt injection, model error, or any other means — can produce malformed `ToolCall` values, but cannot produce a `ToolCall` for a tool that isn't in the registry, cannot invoke a tool with arguments that don't pass validation, cannot reach connections directly. The worst case is "produces tool calls that the tool task rejects." The connections are unreachable.

## The proxy boundary

All model traffic goes through the proxy. The harness's proxy client is the only HTTP client in the binary configured to talk to anything resembling a model API. Direct calls to model providers are not permitted at the OS level (egress firewall) and not permitted at the type level (the binary has no client for them).

pub struct ProxyClient {

    endpoint: ProxyEndpoint,

    identity: AgentIdentity,         // for authenticated requests

    manifest\_version: ManifestVersion, // sent with each request

}

impl ProxyClient {

    pub async fn invoke(

        \&self, 

        prompt: Prompt, 

        tools: ToolDeclarations,

        // ...

    ) \-\> Result\<ModelResponse, ProxyError\> {

        let signed\_request \= self.sign(/\* ... \*/);

        let response \= self.http.post(\&self.endpoint).json(\&signed\_request).send().await?;

        // ...

    }

}

Every request to the proxy is signed by the agent's identity and includes the manifest version. The proxy verifies both, looks up the agent's authorization, validates that the requested tools match the manifest, applies token budget checks, logs the request and response, returns the result. A compromised harness cannot bypass this — there is no other path to a model.

The proxy itself is specified separately. From the harness's side, the contract is: signed request goes in, signed response comes out, audit happens at the proxy.

## Tool execution detail

Tools execute in the tool task's process or, for tools that shell out, in scoped subprocesses with explicit env, no inherited file descriptors, dropped privileges, and resource limits.

pub async fn execute\_pg\_analyze(

    args: PgAnalyzeArgs, 

    ctx: \&ExecutionContext\<'\_\>,

) \-\> Result\<AnalyzeResult, ToolError\> {

    // args.table is type AllowedTable, validated against manifest at construction

    // ctx provides the connection, never visible to caller

    

    let started\_at \= Timestamp::now();

    let pool \= ctx.connections.postgres.as\_ref()

        .ok\_or(ToolError::ConnectionUnavailable)?;

    

    let client \= pool.get().await?;

    

    // SQL is constructed by the tool, not by the agent

    // args.table.as\_sql\_identifier() is a validated identifier (no injection)

    let sql \= format\!("ANALYZE {}", args.table.as\_sql\_identifier());

    

    // execute with timeout

    let result \= tokio::time::timeout(

        Duration::from\_secs(args.timeout\_seconds.unwrap\_or(120)),

        client.execute(\&sql, &\[\]),

    ).await??;

    

    Ok(AnalyzeResult {

        table: args.table,

        started\_at,

        completed\_at: Timestamp::now(),

        rows\_analyzed: result,

    })

}

The thing to notice: SQL is constructed by `format!` from a validated identifier type, not from a string. `AllowedTable::as_sql_identifier()` returns a type that escapes/validates the identifier; you cannot construct an `AllowedTable` from arbitrary input — the only constructor is `AllowedTable::from_manifest_entry()`. Even if the agent's reasoning produced "orders; DROP TABLE users;" as the tool argument, deserialization into `AllowedTable` would reject it because that string isn't in the manifest's allowed-table list.

The same pattern applies to every tool. Subprocess invocations use typed args, never raw command lines. HTTP calls use typed clients with allowlisted endpoints.

## Restart and replay

The harness must restart cleanly. Crash, network partition, intentional restart for upgrade — the agent must come back with the same identity and the same view of the world.

On startup:

1. Sealed store unlocked (out-of-band, using the harness's local secret — typically TPM-backed or kernel keyring).  
2. Identity loaded and verified.  
3. Manifest loaded, signature verified against Architect public key.  
4. Mesh connected, identity authenticated.  
5. Subscribe to relevant topics, including the agent's own historical event stream.  
6. Replay recent events to reconstruct state. The most recent `health_status` published by the agent contains its last known state machine position; if the agent was mid-tool-execution, the harness queries the tool's idempotency log to determine what happened.  
7. Resume: idle if there's no in-flight work, or pick up where the previous instance left off.

Tool execution is idempotent where possible. ANALYZE is idempotent. REINDEX CONCURRENTLY is idempotent (it can be re-run safely if the previous run crashed mid-execution). KILL\_QUERY is one-shot but querying `pg_stat_activity` tells you whether the previous attempt succeeded. Every tool's `execute` method records an idempotency token before performing the operation, so restart can disambiguate "the operation didn't run" from "the operation ran and we crashed before reporting."

For tools that aren't naturally idempotent and don't have a queryable post-state, the harness publishes a `Blocked` state event with a `needs_reconciliation` reason and waits for a human to confirm the operation's outcome. Better to pause than to risk double-execution.

## The Witness's view

The Witness is the external observer that verifies harness integrity. From the harness's side, the contract is:

- Publish `health_status` on every state transition and on a heartbeat (default 30s).  
- Include the manifest version, the binary build hash (computed at startup), and the current state in every status event.  
- Respond to Witness queries (a small set of typed mesh requests: "what's your current state," "what's your manifest hash," "list your active tool calls").  
- Accept `quarantine` instructions from the Witness or Architect. Quarantine moves the state to `Quarantined` and suspends the reasoning loop pending operator review.

The Witness's actual implementation is its own document — for harness purposes, treat the Witness as an entity that subscribes to health events and may issue quarantine commands. The harness must respect quarantine commands; it must not be quarantine-able if the command isn't signed by an authorized signer (Architect or operator key).

## What the harness doesn't do

**No business logic for any agent role.** The harness is the same for `pg-prod` and `mon` and any future agent. Role-specific behavior comes from the manifest, the prompt, and the tool implementations.

**No prompt construction.** Prompts are loaded from signed configuration. The harness fills in the role-specific context (current event, current state, available tools) but does not author the prompts. Prompt versions are pinned and signed.

**No model selection.** The proxy decides which model to use based on the agent's manifest and the call's metadata. The harness sends a request and receives a response.

**No vendor SDK code.** Slack, PagerDuty, Grafana — those are the notification service's problem. The harness has Postgres, Prometheus, Grafana (annotation only) clients depending on role, and that's it.

**No persistent state outside identity.** All durable state is in the mesh log. The harness's local persistence is the sealed identity store and a small idempotency log for in-flight tool calls. Both are recoverable from operator action if corrupted.

## Build and verification

Reproducible builds. `Cargo.lock` pinned. Dependencies audited. The binary's hash is published alongside the Architect's manifest signing key and included in the deployment's trust manifest. The Witness verifies that any registered harness reports a binary hash matching a known-good build.

A new harness build is a real event: it requires regenerating the trust manifest, distributing the new binary hash, and updating the Witness's allowed-hash list. This friction is intentional. Casual harness changes are not desirable; the harness is the thing whose correctness underwrites the system, and rebuilding it should feel important.

The build process produces three artifacts: the binary, the SBOM, and the signature over both. All three are required for deployment. The signature is produced by the deployment's release-signing key (held offline, used rarely), not by the Architect's runtime signing key — different trust roots for different operations.

## Implementation milestones

A reasonable implementation order:

1. Identity loading, manifest verification, mesh connection. The agent that does nothing but register and heartbeat. Two weeks for an experienced Rust engineer.  
2. Tool registry and the `Tool` trait, with one stub tool implementation. Validates the trait shape, the registry construction, the manifest binding.  
3. The state machine and supervisor. Validates the type-driven lifecycle.  
4. The proxy client and a complete reasoning loop with one role (`pg-prod`) and a minimal set of tools. End-to-end test: agent receives a mock anomaly event, produces a hypothesis event.  
5. Restart-as-replay. Validates the idempotency story.  
6. Real tool execution against a real Postgres test instance.  
7. The full `pg-prod` tool set, the full prompt, calibration.  
8. The same for `mon`.

Each milestone is a working agent with progressively more capability. Each is a place to stop, audit, write tests, and validate the design against real behavior.

## What's next

The next document is the proxy specification, which is the other half of the cognition-control story. The proxy is much smaller than the harness in scope, but it's where the audit log lives, where token budgets are enforced, where the kill switch is implemented, and where most of the security review's attention will be focused. After that, the Witness specification, then the deployment and onboarding documents.

This is the design at a level where a Rust engineer can begin work. The remaining specs refine the surrounding components, but the harness is the trust-bearing core.  
