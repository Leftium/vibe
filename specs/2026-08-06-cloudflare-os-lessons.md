# What Vibe Can Learn From Cloudflare OS

**Date:** 2026-08-06  
**Status:** Research note  
**Cloudflare OS revision reviewed:** `aedcda8b3066ff666f57ae28ecef7341d6c2dee7`  
**Cloudflare OS starter revision reviewed:** `9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee`  
**Related:** [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md), [Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md), [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md)

## One Sentence

Cloudflare OS demonstrates several mechanisms Vibe should adopt or adapt, especially typed capabilities shared by apps and agents, resource brokers, isolated app instances, draft-to-mainline editing, provenance-aware sharing, and blueprint distribution, without requiring Vibe to inherit Cloudflare's hosted runtime or source model.

## First-Screen Contract

This document is a source-backed analysis of Cloudflare OS as a reference architecture for Vibe.

The current Vibe foundation defines an independent, local-first-capable system with readable files, Git history, an optional Builder, a stable shell, typed actors, configuration-first customization, and last-good artifacts. No Vibe implementation exists yet.

Cloudflare OS is an existing open-source system for agent workspaces and modifiable personal apps. It already implements many adjacent mechanisms, but with a different center of gravity: a hosted Workshop, Cloudflare Workers, Durable Objects, Yjs-managed source, and Gadgets that run inside the Workshop.

The target outcome of this note is not a substrate decision. It is a precise list of:

- Mechanisms Vibe should adopt
- Mechanisms Vibe should adapt to its own contracts
- Mechanisms worth testing
- Cloudflare-specific assumptions Vibe should reject
- Changes that may be needed in the Vibe foundation

This research is complete when each material recommendation points to an upstream source and states whether it changes Vibe's product contract, implementation, or roadmap.

## Scope

This note examines:

- Cloudflare OS product and process boundaries
- Gadget runtime isolation
- Cap'n Web and agent-callable APIs
- Gatekeepers, bindings, credentials, approvals, and observations
- Agent draft changes and mainline code
- Live sharing and Blueprint distribution
- Collaboration roles and per-user authority
- Core-versus-deployment customization
- Yjs, workerd, Durable Objects, and other implementation choices
- Direct reuse opportunities and coupling risks

It does not:

- Select Cloudflare OS as Vibe's substrate
- Change the Vibe foundation specification
- Validate Cloudflare OS by running it
- Perform a security review
- Promise compatibility with Cloudflare OS
- Design Vibe's complete RPC, package, or connector schema

The companion [Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md) document handles the substrate proposal and go/no-go experiment.

## Research Basis

The analysis is grounded in the following primary sources:

- [Cloudflare OS announcement](https://blog.cloudflare.com/cloudflare-os/)
- [Cloudflare OS repository at the reviewed revision](https://github.com/cloudflare/cloudflare-os/tree/aedcda8b3066ff666f57ae28ecef7341d6c2dee7)
- [Repository README](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/README.md)
- [Repository agent instructions and architecture map](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/AGENTS.md)
- [Shared Workshop API](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-shared/src/api.ts)
- [Gatekeeper API](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-shared/src/gatekeeper.ts)
- [Blueprint design](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/blueprints.md)
- [Sharing design](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/sharing.md)
- [Observer and read-through permission design](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/observers.md)
- [Cloudflare OS starter at the reviewed revision](https://github.com/cloudflare/cloudflare-os-starter/tree/9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee)

The repository identifies itself as early-access software. These findings describe the reviewed revisions, not a stable compatibility promise from Cloudflare.

## Cloudflare OS In Vibe Terms

Cloudflare OS describes itself using an operating-system analogy. Translated into Vibe's vocabulary:

| Cloudflare OS | Responsibility | Closest Vibe concept |
| --- | --- | --- |
| Workshop frontend | Chat, editor, app UI, sharing and connection controls | Stable shell plus Builder UI |
| Workshop backend / Overseer | App lifecycle, source state, agent tasks, access, sharing, runtime orchestration | Builder core plus app supervisor |
| Gadget | One private, modifiable app instance | Installed Vibe app |
| Gadget client | Sandboxed browser UI | App canvas runtime |
| Gadget server | Isolated stateful server code | Optional service actor |
| Blueprint | Code template used to create an independent Gadget | Portable `.vibeapp` or app template |
| Gatekeeper | Credential-holding, policy-enforcing external-service adapter | Capability Broker actor |
| Binding | A scoped capability supplied to app or agent code | Binding grant |
| Observation | A protected resource read through a Gatekeeper | Observation receipt |
| Workspace agent | Code Mode agent with scoped resources | Builder agent backend |
| Chat draft | Proposed code and binding changes scoped to one chat | Task draft worktree |
| Mainline | Committed Gadget code used by ordinary users | Active source revision |
| Workshop core | Security-sensitive platform kernel | Vibe shell and Builder core |
| Starter deployment | Branding, configuration, integrations and pinned core | Vibe installation profile |

This mapping is close enough to reuse architectural thinking, but not exact enough to substitute names mechanically.

## Verified Architectural Findings

### Each App Is An Isolated Instance

Cloudflare OS does not treat a Gadget as a shared SaaS frontend over one central application database. Each Gadget is an application instance with its own code and state.

The server side runs in an isolated Dynamic Worker facet and receives its own SQLite-backed state. The browser side runs in a sandboxed iframe. Direct outbound network access is denied; external access arrives through explicit bindings.

This validates several Vibe assumptions:

- The unit of installation should own an isolated state namespace.
- Generated code should not share ambient credentials or network.
- Per-app isolation makes user modification less dangerous.
- A shared host or Builder does not require shared app state.

It does not prove that Dynamic Workers or Durable Objects are the right local implementation.

### The UI And Agent Share One Typed API

Gadget client and server code communicate through Cap'n Web RPC. The API a client calls is also available to the agent's Code Mode environment.

This is more than a transport choice. It means app behavior is agent-operable by construction:

```text
human uses generated UI
        |
        v
typed app capability <--- agent invokes the same capability
        |
        v
app state and effects
```

The agent does not need a separately maintained MCP server for every generated app. It can inspect the same TypeScript interface and call it from code.

Cloudflare OS's shared API also uses capability-returning RPC objects. A caller receives only the interface appropriate to its role. This avoids one global service object whose methods all perform manual authorization.

### External Access Is A Binding, Not Ambient Authority

A Gadget or agent begins with no access to external services. A user introduces a specific resource through a Gatekeeper. The resulting binding is a capability, not a raw credential.

Gatekeepers:

- Wrap a service's native API with a smaller typed API
- Hold OAuth credentials
- Restrict access to the intended resource and operations
- Record reads and actions
- Mediate externally visible side effects
- Ask for approval when policy requires it

Generated code can receive a binding resembling `env.PROJECT` without receiving the underlying token or account-wide API.

This is stricter than declaring a broad capability such as `network: github.com`. It answers three different questions:

1. Which connector implementation is trusted?
2. Which account or resource has the user selected?
3. Which methods may this app or agent invoke?

### Source Changes Are Drafted Per Chat

The Workshop API distinguishes mainline code from chat-scoped proposed changes. Code edits, new Gadgets, and binding additions can remain provisional to a chat. The API includes merge, revert, finalize, and discard operations.

This model supports:

- Previewing one task without changing the shared mainline
- Multiple independent conversations
- Discarding failed work
- Reviewing an accumulated change set
- Making binding changes part of the same proposal

Cloudflare OS uses Yjs as its live code representation and history substrate. Vibe can adopt the draft/mainline semantics while using files, worktrees, and Git.

### Live Sharing And Code Sharing Are Different

Cloudflare OS distinguishes:

- Collaborating on the same Gadget, which shares identity and state
- Publishing a Blueprint, which lets another user create an independent instance

A Blueprint contains a source snapshot, binding requirements, and metadata. It excludes Gadget SQLite data, chat and edit history, credentials, and live connections.

This is a useful product distinction. "Share this app" is otherwise ambiguous:

```text
live share
  -> same identity
  -> same state
  -> ongoing authorization

blueprint share
  -> copied code
  -> new identity
  -> new state
  -> recipient's bindings and credentials
```

Vibe's current portable package is closer to a Blueprint than a live share.

### Blueprints Are Templates, Not Upgrade Channels

The Blueprint design documents an important limit: an instantiated Gadget is independent from its Blueprint source, and existing instances do not receive automatic Blueprint updates. Updating a Blueprint helps future installations but does not propagate improvements to its source Gadget, parent, or sibling instances.

This is the right default for templates. Recipients get independent identity, state, bindings, and ownership. It is weak for behavior that should improve across many apps. Copying a renderer, editor, importer, or security-sensitive helper into every Gadget turns each copy into a maintenance fork.

Vibe should distinguish:

- A **template or Blueprint**, which creates an independent copy
- A **versioned module**, which remains an explicit dependency
- **Vendored source**, which is editable but retains its upstream base and local patch
- A **configuration recipe**, which changes supported settings without forking code

Module updates should be offered through exact locks, compatibility checks, capability deltas, migrations, preview, and rollback. They should not be live mutations of every dependent app. Shared module code still receives separate state, configuration, bindings, and grants in each app.

The companion [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md) specification defines this boundary.

### A Collaborator Brings Their Own Authority

Cloudflare OS currently distinguishes `build` and `use` collaboration roles.

A build collaborator can edit code and use AI, but uses their own model and connected accounts. They do not inherit the owner's LLM billing identity or broad connector credentials. A use collaborator receives a restricted interface that exposes the deployed UI but not Builder operations.

The implementation makes the restricted capability implement the full interface and default-deny newly added methods. Adding a method therefore forces an explicit authorization decision at compile time.

This is directly relevant to Vibe's requirement that every user modify an app with their own model credentials.

### Policy Follows Observed Data

Cloudflare OS tracks resources read through Gatekeepers. This addresses a problem that ordinary capability checks miss:

1. An app is authorized to read private data.
2. The app stores or displays a derived result.
3. The app is shared with someone who cannot read the original data.
4. The shared output becomes an unintended disclosure.

The Observer design requires a collaborator to connect their own account for each relevant Gatekeeper. The Gatekeeper verifies whether that account may observe everything the Gadget has read. Future observations may be blocked when an existing observer would not be allowed to see them.

The important principle is:

> Authorization must cover both the current operation and the data lineage of persistent outputs.

Vibe's current foundation tracks capability requests and local grants, but not resource observations or derived-output provenance.

### Core And Deployment Customization Are Separate

Cloudflare OS publishes a core repository and a starter deployment repository. The starter pins an upstream revision and owns branding, identity, routes, storage, integrations, observability, connector policy, and upgrades.

The deployment can customize common behavior without modifying core. Product behavior that the extension boundaries do not expose still requires a pinned fork or upstream change.

This is a concrete form of Vibe's no-fork configuration ladder:

```text
admin setting
  -> deployment configuration
    -> custom Gatekeeper
      -> wrapper-owned code
        -> pinned core patch
```

It also makes upgrade ownership explicit. A deployment chooses when to adopt a new core revision.

### Repeatable Work Becomes Deterministic

Cloudflare OS distinguishes interactive agent work from persistent apps and workflows. A known sequence can become code, with model calls retained only where judgment is useful.

This reinforces Vibe's runtime-independence contract:

- A prompt may create behavior.
- Accepted behavior becomes code or configuration.
- Re-running the behavior should not replay the authoring conversation.
- A workflow may still contain explicit model-backed actors where the application actually requires inference.

### Side Effects Can Be Proposed Before Approval

Gatekeepers can queue actions that require approval and may return a simulated result so an agent can continue planning. The user can later approve or reject a batch.

This solves a real UX failure: an unattended task should not stop at its first predictable approval. It also creates risk if a simulation is mistaken for committed reality.

Vibe should not adopt generic simulated success. A future broker may support deferred effects only when it can provide:

- A typed plan
- A bounded simulation
- Explicit speculative status
- Idempotent commit
- Cancellation
- Reconciliation when the result differs

Synchronous approval remains the safe default.

## Comparison With The Vibe Foundation

| Concern | Cloudflare OS | Vibe foundation |
| --- | --- | --- |
| Product center | Organization workspace and hosted personal apps | Portable app with integrated modification UX |
| Runtime host | Workshop on Workers or local workerd | Browser-compatible shell, later native |
| Builder | Integrated into Workshop | Logically separate and optionally installed |
| App unit | Gadget instance | Source-bearing Vibe app |
| Client runtime | Sandboxed iframe | Sandboxed app canvas |
| Server runtime | Dynamic Worker facet | Optional actor in Worker, process, or native host |
| Source truth | Yjs document and Workshop mainline | Conventional files and Git |
| Candidate model | Per-chat draft | Candidate worktree or overlay |
| App data | Per-Gadget SQLite | IndexedDB initially, adapters later |
| External access | Gatekeeper bindings | Runtime and authoring capabilities |
| App API | Cap'n Web capability interfaces | Generic actor messages in current draft |
| Distribution | Blueprint or live Gadget share | `.vibeapp` with source, history, and artifact |
| Configuration | Deployment settings and app code | Typed, layered, config-first public API |
| Version history | Yjs changes and Blueprint versions | Standard Git plus task log |
| Offline/standalone | Local workerd possible; Gadget normally needs Workshop | Last-good app should run without Builder |
| Native goal | Not central | Native shell is a target |
| Collaboration | First-class | Deferred |

The overlap is strongest in security, app isolation, agent integration, and sharing semantics. The largest differences are source ownership, portability, configuration, and runtime independence from the hosting platform.

## Lessons For Vibe

### 1. Use One Typed API For UI And Agent Operations

**Classification:** Adopt.

Every stateful actor should expose a typed capability interface. The generated UI and agent receive scoped handles to the same interface.

Example:

```ts
interface ReminderCapability {
  list(): Promise<Reminder[]>;
  create(input: CreateReminder): Promise<Reminder>;
  updateSchedule(input: ReminderSchedule): Promise<void>;
}
```

Benefits:

- One semantic contract
- Less adapter code
- Better agent discoverability
- Smaller tool catalogs
- Easier tests
- Capability-based authorization

Vibe should retain structured event envelopes for broadcasts, persistence, telemetry, and loose coupling. It should not make a stringly typed global message bus the only way to call an actor.

### 2. Separate A Binding Requirement From A Binding Grant

**Classification:** Adopt.

An app package should state what kind of resource it needs. A local installation chooses the actual account and resource.

```ts
type BindingRequirement = {
  name: string;
  protocol: string;
  operations: string[];
  required: boolean;
  description?: string;
  suggestedResource?: string;
};

type BindingGrant = {
  requirementName: string;
  brokerId: string;
  localResourceId: string;
  grantedOperations: string[];
};
```

Requirements travel with Blueprints or `.vibeapp` packages. Grants, accounts, tokens, and local policy do not.

### 3. Add Capability Brokers

**Classification:** Adopt.

Vibe's `PermissionActor` decides whether access is allowed. A new logical `CapabilityBrokerActor` should mediate the access itself.

```text
app or agent
  -> typed capability
    -> broker policy
      -> credential
        -> external service
```

The broker owns:

- Credentials
- Resource scoping
- API normalization
- Input and output filtering
- Rate limits
- Approval policy
- Audit records
- Observation receipts

This prevents generated code from needing raw fetch or broad service tokens.

### 4. Make Draft And Mainline Product Concepts

**Classification:** Adapt.

Use Cloudflare OS's semantics with Vibe's source model:

```text
Git mainline
  -> task worktree
    -> incremental draft
      -> draft preview
        -> discard or merge
          -> build and promote last-good
```

The task draft should include source, configuration, actor graph, and binding changes. A task may show live preview updates before final promotion.

### 5. Define Two Sharing Verbs

**Classification:** Adopt.

Use explicit product language:

- **Collaborate:** grant access to the same app identity and data.
- **Instantiate:** create an independent app from a Blueprint or package.

The initial Vibe scope implements instantiate. Collaboration remains deferred but receives compatible metadata and authority concepts.

### 6. Track Observation Provenance Before Collaboration

**Classification:** Adapt, implement incrementally.

Add an `ObservationReceipt` contract early enough that brokers can emit it:

```ts
type ObservationReceipt = {
  brokerId: string;
  bindingName: string;
  resourceDescriptor: string;
  operation: string;
  policyVersion: string;
  observedAt: string;
  contentFingerprint?: string;
};
```

The first version may only use receipts for audit, export warnings, and diagnostics. Live collaborator enforcement can come later.

Receipts should not contain the sensitive response body.

### 7. Give Each Collaborator Their Own Builder Authority

**Classification:** Adopt for future sharing.

A collaborator who can modify an app uses:

- Their own Builder connection
- Their own model credentials and budget
- Their own external-service accounts
- Their own local grants

Sharing the ability to build does not share the owner's credentials.

Initial roles:

- `owner`: delete, export, transfer, and manage all sharing
- `build`: use Builder, edit, configure bindings, run app
- `use`: run app through restricted runtime capabilities

Chat-only and read-only roles can wait for evidence.

### 8. Split Platform Core From Installation Profile

**Classification:** Adopt.

```text
Vibe core
  builder protocol
  shell
  runtime SDK
  permission kernel
  recovery

Installation profile
  branding
  model providers
  connector catalog
  default policies
  shared skills and context
  featured app templates
```

An organization or person should customize the installation without patching core. A source patch is an explicit escalation with an upgrade cost.

### 9. Make Model Use Explicit In Runtime Graphs

**Classification:** Adopt.

Most generated behavior should be deterministic. If an app needs inference at runtime, that dependency should appear as a declared model binding or actor:

```text
scheduled workflow
  -> deterministic fetch
  -> deterministic filter
  -> model-backed classification
  -> deterministic notification
```

This makes cost, privacy, offline behavior, and failure visible.

### 10. Treat Deferred Effects As Transactions

**Classification:** Defer.

If Vibe later supports non-blocking approval, require a connector-specific transactional interface:

```ts
interface TransactionalCapability<Input, Preview, Result> {
  plan(input: Input): Promise<Preview>;
  commit(planId: string): Promise<Result>;
  cancel(planId: string): Promise<void>;
}
```

The agent and UI must know that `Preview` is speculative. A broker without a sound simulation blocks for approval.

### 11. Keep Git As Vibe's Canonical Source History

**Classification:** Reject Yjs as the initial source of truth; study it later.

Cloudflare OS uses Yjs to synchronize code and replay changes. That is valuable for multiplayer editing. It is not required for:

- One user
- One Builder task
- Conventional editors
- Standard source checkout
- Git diffs and history

Vibe can later layer collaborative editing over task drafts. It should not introduce a second canonical source model before a collaboration requirement exists.

### 12. Evaluate Cap'n Web And workerd Independently

**Classification:** Study.

Cap'n Web and workerd solve different layers:

- Cap'n Web: object-capability RPC over browser and network boundaries
- workerd: an isolated runtime for Worker-compatible server code

Vibe can adopt Cap'n Web without adopting Cloudflare OS. It can also use workerd as one actor placement or hosted runtime while retaining another local runtime.

## Adoption Matrix

| Mechanism | Disposition | Initial timing | Notes |
| --- | --- | --- | --- |
| One typed API for UI and agent | Adopt | Phase 0 | Test with one toy capability |
| Object-capability RPC | Study, likely adopt | Phase 0 | Compare Cap'n Web with a smaller local interface |
| Binding requirements and local grants | Adopt | Phase 1 schema | Required for portable packages |
| Gatekeeper-style brokers | Adopt | Before first external API | Credentials never reach generated code |
| No ambient network | Adopt | First sandbox | Allow only explicit bindings |
| Chat draft and mainline | Adapt | First source task | Back with Git worktrees or overlays |
| Blueprint versus live share | Adopt | Package design | Implement independent instantiation first |
| Build/use collaboration roles | Adapt | Before collaboration | Use caller's Builder and accounts |
| Observation provenance | Adapt | Broker contract early, enforcement later | Prevent derived-data leaks |
| Core/deployment split | Adopt | Repository and config design | Preserve no-fork customization |
| Model bindings in runtime graph | Adopt | Actor manifest | Make runtime inference explicit |
| Speculative queued effects | Defer | After stable brokers | Only with reliable plan/commit semantics |
| Yjs canonical source | Reject for v0 | Revisit for collaboration | Git remains canonical |
| Dynamic Workers and Durable Objects | Optional backend | Substrate spike | Not a portable contract |
| React Workshop frontend | Do not adopt as Vibe contract | Cloudflare edition only | Vibe UI remains independently specified |
| Cloudflare account requirement | Reject as universal requirement | Hosted edition only | Local runner remains a target |
| Whole-repository fork | Avoid | Revisit if extension points fail | Prefer pinned adapter and bounded patches |

## Proposed Vibe Primitive Set

The Cloudflare OS findings suggest a tighter set of primitives than a generic actor bus alone:

```ts
interface Capability {
  readonly protocol: string;
  readonly version: number;
}

type BindingRequirement = {
  name: string;
  protocol: string;
  operations: string[];
  required: boolean;
};

type BindingGrant = {
  requirementName: string;
  brokerId: string;
  localResourceId: string;
  grantedOperations: string[];
};

type ObservationReceipt = {
  brokerId: string;
  bindingName: string;
  resourceDescriptor: string;
  operation: string;
  policyVersion: string;
  observedAt: string;
};

type TaskDraft = {
  taskId: string;
  baseRevision: string;
  workspaceId: string;
  status: "editing" | "validating" | "ready" | "failed";
};

type AppBlueprint = {
  source: SourceSnapshot;
  bindingRequirements: BindingRequirement[];
  metadata: AppMetadata;
};

type InstallationProfile = {
  providers: ProviderPolicy[];
  brokers: BrokerRegistration[];
  defaults: ConfigLayer;
  featuredBlueprints: string[];
};
```

These types are illustrative. Their purpose is to expose the missing conceptual boundaries before selecting serialization or RPC libraries.

## Implications For The Foundation Spec

The [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md) should remain unchanged until the substrate experiment finishes. Regardless of that result, a later revision should consider:

1. Demoting generic `ActorMessage` envelopes from primary call interface to event and transport representation.
2. Adding typed capability interfaces to every externally callable actor.
3. Adding binding requirements, grants, and brokers.
4. Adding observation receipts to the security model.
5. Naming task drafts and mainline explicitly.
6. Distinguishing independent instantiation from live collaboration.
7. Adding owner, build, and use authority.
8. Adding installation profiles to the configuration hierarchy.
9. Adding a Phase 0 capability-RPC spike.
10. Declaring runtime model use as a manifest dependency.

The Cloudflare OS substrate decision may additionally change SvelteKit, Git, IndexedDB, and packaging milestones. Those are conditional and belong in the companion proposal.

## Direct Reuse Candidates

### Strong Candidates

- **Cap'n Web:** Evaluate as a direct dependency for typed capability RPC.
- **Public interface patterns:** Learn from the separation between shared API, backend kernel, and restricted caller capabilities.
- **Blueprint semantics:** Reuse the distinction among code snapshot, binding shape, metadata, data, and credentials.
- **Gatekeeper contract ideas:** Reuse resource scoping, credential isolation, observations, and proposed effects.
- **Sharing algorithms:** Study default-deny role capabilities, live role computation, reversible revocation, and observer verification.
- **Starter layout:** Reuse the pinned-core plus deployment-owned customization pattern.

### Conditional Candidates

- **workerd:** Evaluate for hosted or local service actors.
- **Yjs:** Evaluate only when concurrent source editing becomes active scope.
- **Existing Gatekeepers:** Potentially reuse in a Cloudflare-hosted edition, subject to their Worker and Durable Object dependencies.
- **Agent harness:** Study its Code Mode environment and binding presentation; do not make it Vibe's only backend.

### Poor Candidates

- The complete React Workshop frontend
- The entire Workshop backend as Vibe's portable runtime
- Durable Objects as the only storage API
- Cloudflare-specific deployment configuration in app source
- Yjs documents as Vibe's only editable source format

The reviewed repositories use the Apache-2.0 license. Any copied code must retain the required license and notices. This note is an architecture recommendation, not legal advice.

## Risks And Countermeasures

### Mistaking Similar UX For Identical Product Boundaries

Cloudflare OS and Vibe both create modifiable apps through chat, but Cloudflare OS centers a shared hosted Workshop. Vibe centers a portable app that should survive without its Builder.

Countermeasure: evaluate every borrowed mechanism against runtime independence and portable source.

### Designing Around Early-Access Internals

Cloudflare OS explicitly warns that its current version is early access and a rewrite of its first version.

Countermeasure: pin research and experiments to revisions. Depend on Vibe-owned interfaces rather than undocumented internal paths.

### Overbuilding Collaboration

Observers, Yjs, permission graphs, and real-time multiplayer solve important problems but can overwhelm a local first version.

Countermeasure: shape portable metadata now, implement collaboration only after the single-user source transaction works.

### Creating Two Sources Of Truth

Combining Yjs, files, and Git without a clear authority can lose edits or produce misleading history.

Countermeasure: keep files and Git canonical in the independent Vibe architecture. If a Cloudflare edition uses Yjs, define deterministic import and export boundaries.

### Treating A Virtual Shell As A Sandbox

Cloudflare OS combines runtime isolation, no ambient network, capabilities, and brokers. A command facade alone does not provide equivalent security.

Countermeasure: keep authoring command UX, process/runtime isolation, and capability policy as separate layers.

### Leaking Derived Data

Exporting source safely does not guarantee that saved app data or generated outputs are safe to share.

Countermeasure: separate source, state, credentials, grants, transcripts, observations, and derived artifacts in the package model.

## Open Questions

### Should Cap'n Web Replace The Proposed Actor Call Model?

Options:

- Use Cap'n Web for all cross-boundary capabilities.
- Use it only in the Cloudflare edition.
- Keep a Vibe RPC abstraction with Cap'n Web as one adapter.

Recommendation: define Vibe-owned TypeScript capability interfaces and test Cap'n Web as the first adapter. Do not expose Cap'n Web-specific types in app domain interfaces unless the spike proves the coupling is useful.

Decision trigger: the Phase 0 capability-RPC spike.

### How Much Provenance Does A Local App Need?

Options:

- Audit records only
- Resource-level observation receipts
- Full derived-data lineage

Recommendation: resource-level receipts from brokers, without content bodies. Add collaborative enforcement later.

Decision trigger: first external-service app and first live-sharing design.

### Can A Broker Safely Simulate Side Effects?

Options:

- Never
- For explicitly transactional APIs only
- For any connector using best-effort mocks

Recommendation: only for connectors that implement typed plan, commit, cancel, and reconciliation.

Decision trigger: first workflow that needs multiple delayed approvals.

### Should Vibe Support Live Sharing Or Only Blueprints?

Recommendation: independent Blueprint/package instantiation first. Preserve roles, bindings, and provenance so live sharing can be added without redefining the app.

Decision trigger: validated demand for multiple users sharing one stateful instance.

### Is workerd A Local Native Runtime Candidate?

Recommendation: evaluate it independently from the Cloudflare OS adoption decision. A useful service-actor runtime does not require adopting the Workshop product model.

Decision trigger: local process packaging, startup, filesystem, native bridge, and update experiments.

## Conclusions

Cloudflare OS provides strong evidence for the following shape:

```text
isolated app instance
  + typed capabilities
  + no ambient authority
  + credential-holding brokers
  + task drafts
  + independent blueprints
  + provenance-aware sharing
  + installation-level customization
```

Vibe should absorb that shape.

Vibe should preserve its distinct commitments:

- Portable source
- Standard Git
- Configuration-first customization
- Optional Builder
- Last-good standalone behavior
- Local and native paths
- A runtime contract not owned by one host

The next question is not whether Cloudflare OS contains useful ideas. It does. The companion proposal asks whether its implementation can serve as Vibe's first kernel without making those Vibe commitments impossible.

## References

- [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md)
- [Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md)
- [Cloudflare OS announcement](https://blog.cloudflare.com/cloudflare-os/)
- [Cloudflare OS repository, reviewed revision](https://github.com/cloudflare/cloudflare-os/tree/aedcda8b3066ff666f57ae28ecef7341d6c2dee7)
- [Cloudflare OS README](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/README.md)
- [Cloudflare OS architecture instructions](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/AGENTS.md)
- [Workshop shared API](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-shared/src/api.ts)
- [Gatekeeper API](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-shared/src/gatekeeper.ts)
- [Blueprint design](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/blueprints.md)
- [Sharing design](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/sharing.md)
- [Observer design](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/observers.md)
- [Cloudflare OS starter, reviewed revision](https://github.com/cloudflare/cloudflare-os-starter/tree/9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee)
