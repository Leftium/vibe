# Building Vibe on Cloudflare OS

**Date:** 2026-08-06  
**Status:** Proposed substrate experiment - architecture decision pending  
**Updated:** 2026-09-29  
**Cloudflare OS August baseline:** `aedcda8b3066ff666f57ae28ecef7341d6c2dee7`  
**Cloudflare OS current-main revision targeted:** `687aab049cf084030a42093c10c4dde3d9c33fb4`  
**Cloudflare OS starter revision targeted:** `9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee`  
**Related:** [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md), [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md), [Vibe Runtime Providers and Placement](./2026-08-06-vibe-runtime-providers.md), [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md)

**Runtime placement note:** The companion runtime-provider specification refines this document's hosted-first assumption. Cloudflare-hosted, remote self-hosted, managed local, system local, and bundled local execution are provider profiles behind the same Vibe contracts. Where this document recommends hosted v0, treat that as a substrate candidate to test rather than a settled placement decision.


## One Sentence

Build the first hosted Vibe edition as a malleability layer over a Cloudflare OS workspace: Vibe owns Resources, Tools, Compositions, Recipes, modules, capability contracts, and last-good semantics, while Cloudflare OS supplies Git-backed workpieces, agent/task execution, Gatekeepers, collaboration, and sandboxed placement behind a provider adapter.


## First-Screen Contract

Vibe can plausibly use Cloudflare OS as its first serious runtime provider, but it should not become "Cloudflare OS with different branding."

The September 2026 reassessment materially improves the fit. Since the August proposal, Cloudflare OS has generalized one-Gadget workspaces into multi-workpiece workspaces, moved accepted Gadget source to real Git commits, replaced active Yjs editing with CodeMirror operational transforms, and added Git-backed Worktrees for arbitrary repositories.

The proposed split is now:

- **Vibe supplies the semantic malleability layer:** Resources, Tools, Compositions, Recipes, versioned modules, a malleability ladder, provider-neutral runtime capabilities, package semantics, and last-good recovery.
- **Cloudflare OS supplies one execution/authoring provider:** workspace identity, Gadget and Worktree workpieces, Git-backed source history, chat-scoped proposals, the agent harness, Gatekeepers, storage, collaboration, and Blueprint machinery.
- **A narrow adapter owns placement:** Vibe semantic objects may co-locate in one Gadget or span several workpieces when authority, state ownership, lifecycle, failure isolation, scaling, or reuse justify it.
- **App and tool code imports Vibe contracts, not Workshop internals.** The Cloudflare-specific mapping stays below the adapter boundary.

The experiment should therefore stop testing whether Vibe can force a Git-shaped source model onto Yjs or force a multi-component product into one Gadget. Upstream has largely removed those mismatches. The remaining question is whether Vibe's semantic layer can stay cleanly above Cloudflare workpieces while preserving portability to a local/native provider.

The experiment succeeds only if one vertical slice proves all of the following:

1. One Vibe composition can be represented by a Cloudflare workspace without exposing Workpiece/Gadget terminology as the product model.
2. A Builder task maps cleanly to Cloudflare's chat-over-Git proposal flow and produces an accepted Git revision.
3. The same Vibe Resource can be presented through at least two Tools without copying its domain data.
4. Placement can change - for example, a capability-owning component can move to a separate workpiece - without changing the Vibe semantic contract.
5. The same typed Vibe capability is callable by UI code and the Builder through a Gatekeeper-backed adapter.
6. Accepted behavior runs with no LLM or active Builder.
7. A broken accepted/source revision does not replace the Vibe last-good running revision.
8. Source and package state round-trip through ordinary Git/files without credentials, grants, private runtime data, or Builder history.
9. A simple local/in-memory adapter can implement the same Vibe capability and Resource/Tool contracts, proving Cloudflare types have not leaked upward.
10. The required Cloudflare OS patch set is small, named, and maintainable.

Until that evidence exists, the provider-neutral architecture in the [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md) remains the baseline.

## Decision Sought

Approve a Cloudflare OS substrate experiment for Vibe's hosted first edition.

Do not yet approve:

- Cloudflare OS as Vibe's permanent or exclusive runtime
- A production launch on the current upstream
- A deep or indefinite fork
- Cloudflare-specific APIs in app source
- Exposing Cloudflare's OT or workpiece storage model as Vibe's portable source ABI
- Abandoning a local or native runner

The decision after the experiment is one of:

- **Go:** Cloudflare OS becomes the hosted v0 substrate behind Vibe contracts.
- **Conditional go:** use selected components or a constrained fork while replacing the failing layer.
- **No-go:** retain the architectural lessons and continue with an independent implementation.

## Why This Is Worth Testing

Cloudflare OS already implements much of the hard platform work that Vibe would otherwise need to invent before validating its product:

- Agent-driven creation and modification of small applications
- Isolated client and server execution
- A typed API exposed to both UI code and the agent
- Narrow, credential-hiding resource brokers
- Multi-workpiece workspaces containing several executable or source-bearing units
- Real Git commits for accepted Gadget source, with chat-scoped proposal editing over a commit base
- Git-backed Worktrees for external repositories and reusable source
- Draft changes that can be previewed before reaching shared mainline code
- Step-transactional agent persistence that ties durable edits to their explanatory transcript
- Live collaboration and copy-by-Blueprint sharing
- Per-collaborator authority
- A deployment customization pattern around a pinned core

Those mechanisms align unusually well with Vibe. The main uncertainty is not whether the concepts fit. It is whether Cloudflare OS can be placed behind Vibe's boundaries without fighting its source model, user experience, or portability goals.

## Scope

This proposal specifies:

- The product and technical boundary between Vibe and Cloudflare OS
- The hosted runtime architecture
- The app, actor, Builder, capability, configuration, source, and package contracts needed at that boundary
- A concrete vertical-slice experiment
- Evidence required for a go/no-go decision
- The likely roadmap if the experiment succeeds

It does not specify:

- The complete Vibe protocol or manifest schema
- A production security review
- Pricing, billing, or multi-tenant operations
- Every Cloudflare service used in production
- The complete local or native runner
- A compatibility commitment to arbitrary Cloudflare OS Gadgets
- A migration of the existing foundation spec before the experiment


## Terminology

| Vibe term | Cloudflare OS mechanism in the hosted experiment | Important difference |
| --- | --- | --- |
| Composition / installation | One Cloudflare workspace plus Vibe metadata | Vibe models user intent and semantic composition; the workspace is provider placement |
| Resource | Vibe-owned domain data reached through runtime/storage/capability contracts | A Resource is not inherently a Gadget or Worktree |
| Tool | Vibe view/editor/transformer over Resources | Several Tools may co-locate in one Gadget |
| Workpiece placement | Gadget or Worktree with a `WorkpieceId` | Provider concept below the Vibe semantic layer |
| Executable placement | Gadget workpiece | Not every Vibe Tool or Actor gets its own Gadget |
| Editable Git placement | Worktree workpiece | Useful for modules/external repos; it need not be executable |
| Actor | Vibe module/API/lifecycle unit | May live inside a Gadget or cross a workpiece boundary when justified |
| Binding / capability | Declared Vibe requirement plus user-specific grant | Cloudflare provider typically implements the grant through a Gatekeeper |
| Connector | Vibe-facing capability adapter backed by a Gatekeeper or native provider | Secrets stay outside app/tool code |
| Builder | Vibe malleability agent and task service | A Cloudflare chat can implement one Vibe TaskDraft |
| TaskDraft | One proposed Vibe authoring transaction | Maps naturally to Cloudflare chat proposal state over Git |
| Accepted revision | Git commit accepted into provider mainline | Vibe still separately tracks last-good/running state |
| Package | Portable Vibe source, semantic manifest, requirements, recipes, and dependency lock | Excludes credentials, user data, and chat history |
| Template | Vibe template or Cloudflare Blueprint | Instantiation creates an independent copy |
| Module | Ongoing versioned reusable dependency | Cloudflare Blueprint copy semantics do not replace this |

### Placement principle

Cloudflare workpieces are deployment boundaries, not Vibe product semantics.

```text
Vibe semantic world

Resource <----> Tool <----> Tool
   \             \        /
    \          Composition
     \             |
        Capability / Recipe / Module

---------------- provider boundary ----------------

Cloudflare workspace

  Gadget      Gadget      Worktree
      \        |          /
       Gatekeeper capabilities
```

Co-locate by default. Split into another workpiece only when authority, state ownership, independent lifecycle, failure isolation, scaling, or reuse warrants a stronger boundary.

## Options Considered

### Option A: An app-centric Cloudflare OS distribution

Fork or extend the Cloudflare OS starter, retain the upstream core at a pinned revision, and replace the top-level product experience with Vibe's app library, run surface, and Builder.

**Advantages**

- Reuses the most complete set of upstream mechanisms.
- Produces the fastest credible end-to-end prototype.
- Exercises the real agent, Gadget, Gatekeeper, sharing, and storage paths together.
- Keeps deployment-owned branding, identity, routes, connectors, and AI policy outside the core when upstream seams permit it.

**Costs**

- The Vibe product must reshape a Workshop-first user experience.
- Some ordinary product changes may require patches to upstream core.
- The first edition is hosted and Cloudflare-dependent.
- The live source authority is initially Cloudflare OS, not a local Git working tree.

### Option B: Cloudflare OS only as a remote Builder backend

Keep the Vibe runtime independent and ask a remote Cloudflare OS deployment to edit portable Vibe projects through the Builder protocol.

**Advantages**

- Preserves a clean runtime from the beginning.
- Makes the Builder naturally shareable across apps.
- Reduces long-term dependency on Gadget execution.

**Costs**

- Gives up much of the runtime, binding, storage, preview, and collaboration machinery that makes Cloudflare OS attractive.
- Requires an independent Vibe runtime before the product can be tested.
- Creates a source synchronization problem immediately.
- Is not the shortest path to the first vertical slice.

### Option C: Reuse selected components only

Adopt ideas or libraries such as Cap'n Web, the Pi agent integration, or workerd without adopting the Workshop and Gadget model.

**Advantages**

- Maximum control over Vibe's product and portability model.
- Less upstream product coupling.
- Components can be evaluated independently.

**Costs**

- Vibe must build most platform coordination itself.
- Recreates many boundaries that Cloudflare OS has already integrated.
- Delays learning about the product experience.

### Recommendation

Use Option A for the experiment and, if it passes, for the hosted v0. Design the Vibe contracts so Option B or C can replace it later.

This recommendation is deliberately asymmetric:

- Reuse Cloudflare OS aggressively below the adapter boundary.
- Prevent Cloudflare OS assumptions from becoming the public app model.
- Treat every core patch as evidence about whether the boundary is viable.

## Target Architecture

```text
                             Vibe product layer

  +------------------+   +------------------+   +----------------------+
  | Vibe app UI      |   | Vibe Builder UI  |   | Files / editor / CLI |
  | and actor code   |   | or connect panel |   |                      |
  +--------+---------+   +--------+---------+   +----------+-----------+
           |                      |                        |
           v                      v                        v
  +------------------+   +------------------+   +----------------------+
  | @vibe/runtime    |   | @vibe/builder    |   | Vibe source/package  |
  | contracts        |   | protocol         |   | contracts            |
  +--------+---------+   +--------+---------+   +----------+-----------+
           |                      |                        |
  ======================== adapter boundary ============================
           |                      |                        |
           v                      v                        v
  +------------------+   +------------------+   +----------------------+
  | Cloudflare       |   | Workshop / agent |   | Gadget source /      |
  | runtime adapter  |   | adapter          |   | Blueprint adapter    |
  +--------+---------+   +--------+---------+   +----------+-----------+
           |                      |                        |
           +----------------------+------------------------+
                                  |
                                  v
                 +------------------------------------+
                 | Cloudflare OS hosted substrate      |
                 | Workshop, Gadgets, Gatekeepers,     |
                 | storage, collaboration, deployment |
                 +------------------------------------+

  A future local adapter implements the same Vibe contracts with a local
  runtime, filesystem/Git, IndexedDB or SQLite, and optional native services.
```

### Dependency rule

Code above the adapter boundary may depend on Vibe contracts. It must not import from Cloudflare OS, Workshop internals, Cloudflare Workers bindings, or Gatekeeper implementation packages.

Code below the boundary may translate Vibe contracts into pinned Cloudflare OS APIs.

This rule applies to generated app code as well as hand-written platform code. It is the most important technical condition in the proposal.

## Ownership by Layer

| Concern | Vibe owns | Cloudflare OS supplies in hosted v0 |
| --- | --- | --- |
| Product navigation | App library, open-app view, run mode, Builder connection | Underlying Workshop routes and session primitives |
| App API | Stable runtime interfaces and error model | RPC transport and Gadget client/server plumbing |
| Agent interaction | Builder protocol, change lifecycle, acceptance language | Agent loop, tools, model routing, draft machinery |
| Modular code | Actor contract, registry, health, enable/disable, declared dependencies | Isolated Gadget execution and RPC namespaces |
| Configuration | Schema, layers, editor, validation, provenance | Durable storage and synchronization |
| External authority | Binding requirement and grant model | Gatekeepers, approvals, credentials, observations |
| Source | Portable paths, snapshots, import/export, revision identifiers | Live Gadget source and collaborative editing |
| History | Accepted changes, last-good pointer, export lineage | Upstream draft/mainline history where adequate |
| Packaging | `.vibeapp` contract and installation semantics | Blueprint snapshot and `.gadget` machinery where compatible |
| Sharing | Vibe verbs and permission UX | Live collaboration and Blueprint copies |
| Operations | Vibe deployment profile and support policy | Cloudflare deployment substrate |

## Hosted App Contract

A hosted Vibe app is one Cloudflare OS Gadget instance with a Vibe manifest and a small runtime shim.

The manifest is readable source, not generated deployment metadata. A provisional shape is:

```json
{
  "schemaVersion": 1,
  "id": "example.hello-world",
  "name": "Hello World",
  "entrypoints": {
    "client": "src/client/App.tsx",
    "server": "src/server/index.ts"
  },
  "runtime": {
    "apiVersion": "vibe.runtime/v1",
    "features": ["config", "storage", "actors"]
  },
  "actors": [
    {
      "id": "greeting",
      "module": "src/actors/greeting.ts",
      "enabledByDefault": true
    }
  ],
  "bindings": [],
  "config": {
    "schema": "config/schema.json",
    "defaults": "config/defaults.json"
  }
}
```

The experiment may revise field names. It must preserve these semantics:

- The required runtime API version is explicit.
- Entry points and actors refer to readable source paths.
- Binding requirements are declarations, not embedded credentials or grants.
- Configuration defaults are packageable.
- User configuration and app data are not packageable by default.
- Unknown required features fail clearly instead of degrading silently.

## Runtime Contract

The first API should be intentionally small:

```ts
export interface VibeRuntime {
  readonly app: {
    id: string;
    instanceId: string;
    revision: string;
    mode: "preview" | "run";
  };

  readonly config: VibeConfig;
  readonly storage: VibeStorage;
  readonly actors: VibeActorRegistry;
  readonly bindings: VibeBindingRegistry;
  readonly events: VibeEventBus;
}

export interface VibeConfig {
  get<T>(key: string): Promise<T | undefined>;
  set<T>(key: string, value: T): Promise<void>;
  describe(): Promise<readonly VibeConfigField[]>;
}

export interface VibeBindingRegistry {
  get<T>(name: string): Promise<T>;
  status(name: string): Promise<VibeBindingStatus>;
}
```

The code is illustrative, not a frozen TypeScript definition. The experiment should validate:

- Whether the same interface is ergonomic in a Gadget client, Gadget server, and agent tool.
- Which methods can be synchronous in a local adapter but should remain asynchronous for portability.
- How errors are normalized without hiding actionable provider details.
- How runtime features are negotiated.
- How preview state is kept separate from accepted state.

### Cloudflare adapter

The Cloudflare runtime adapter translates:

- `storage` calls to the Gadget's durable storage
- `bindings` calls to Gatekeeper-backed capabilities introduced to the Gadget
- `actors` calls to typed Gadget server RPC namespaces
- `events` to the supported live synchronization or RPC path
- runtime identity and revision information to Vibe's portable form

App source sees none of those implementation details.


## Actors and Tools on Cloudflare OS

Vibe's semantic and implementation units do not map one-to-one onto Cloudflare workpieces.

Default mapping:

- Several lightweight Vibe Tools and Actors may live together in one Gadget.
- A Resource keeps Vibe-level identity even if its storage or access path changes.
- A Tool is a view/editor/transformer over Resources, not an execution process.
- A capability-owning Actor may cross into another Gadget, Worker, or Gatekeeper when authority or failure isolation justifies it.
- A Git-backed module or external repository may appear as a Worktree during authoring without becoming part of the running composition.
- Placement is recorded as implementation metadata so the Builder can change it without changing the public semantic model.

This preserves the existing Vibe actor rule: in-process by default, stronger isolation on evidence. Cloudflare's new workpiece model makes that rule easier to implement because Vibe no longer has to pretend the entire composition is one Gadget.

A minimal semantic relationship is:

```text
Resource
  -> viewed/edited by Tool
       -> implemented by Module
            -> placed in Gadget when executable

Capability
  -> granted through provider adapter
       -> Gatekeeper in Cloudflare provider

External/module source
  -> mounted as Worktree while being edited
```

The Builder should reason first about Resource, Tool, Composition, Recipe, Module, and Capability. Workpiece placement is a downstream implementation decision unless the user explicitly asks about it.

## Builder Boundary

The Builder is logically external even when its chat UI appears beside the app.

### Product behavior

- **Run mode:** shows the app. No model call occurs merely because the app opens.
- **Builder available:** shows a chat affordance. Opening it may connect to the deployment's Builder service.
- **Builder disconnected:** the app continues to run and configuration remains usable.
- **Builder unavailable:** the chat affordance can explain how to connect a compatible Builder later.
- **Build mode:** shows the app, preview status, Builder conversation, change review, and optional source/settings panels.

The user's LLM credential or model authorization belongs to the Builder service and user session. It is never compiled into, packaged with, or granted to the app.

### Hosted physical layout

Cloudflare OS may initially deploy the Workshop kernel and Builder facilities alongside the Gadget runtime. That physical co-location does not satisfy the boundary on its own. Vibe must access them through an explicit Builder client:

```ts
export interface VibeBuilderClient {
  attach(target: VibeBuildTarget): Promise<VibeBuildSession>;
  prompt(sessionId: string, message: string): Promise<VibeBuildTurn>;
  preview(changeId: string): Promise<VibePreview>;
  accept(changeId: string): Promise<VibeAcceptedRevision>;
  reject(changeId: string): Promise<void>;
  revert(revision: string): Promise<VibeAcceptedRevision>;
  exportSource(revision: string): Promise<VibeSourceSnapshot>;
}
```

The exact transport may be Cap'n Web in hosted v0. The public behavior is Vibe's.

### Run-without-Builder acceptance

The experiment must test more than hiding the chat panel:

1. Create and accept the Hello World revision.
2. terminate the build session;
3. remove or disable the user's model authorization;
4. open the run route in a fresh session;
5. verify the app renders and its ordinary interactions work;
6. verify no model request is made.

If Cloudflare OS cannot execute a Gadget without initializing agent infrastructure, record that as substrate coupling. It is a no-go if the coupling cannot be hidden behind a small, supportable adapter.

## Immediate Live Updates

Separating the Builder does not remove immediate feedback.

The interaction is:

```text
user prompt
    |
    v
Builder creates draft source
    |
    v
Cloudflare OS evaluates draft in preview instance
    |
    v
app surface reconnects or hot-reloads to preview revision
    |
    v
user accepts, rejects, or asks for another change
    |
    v
accepted revision becomes mainline and last-good candidate
```

The app and Builder do not need to be one module or one distributable. They need:

- A shared app instance identifier
- A build session
- A preview revision identifier
- A live preview transport
- A clear acceptance operation

Hosted v0 can use Cloudflare OS's existing draft and preview mechanics. A later local Builder can implement the same flow through a local dev server, filesystem watcher, and hot-module reload.


## Source and Revision Model

The largest August mismatch has mostly disappeared upstream.

Current Cloudflare OS stores accepted Gadget source as real Git objects and commits. A chat holds proposed edits over a known commit base; updating a stale chat performs a three-way merge into the proposal before acceptance. Live editing uses CodeMirror operational transforms, but OT state is not the durable source identity.

Vibe should therefore use one durable rule across providers:

```text
Git-compatible commit
  = portable source identity

provider-local OT / CRDT / filesystem overlay / HMR
  = live authoring and preview mechanism
```

### Hosted Cloudflare mapping

A Vibe `TaskDraft` should map to the provider's chat-scoped proposal rather than introducing a second worktree/overlay system:

```text
accepted Git head
  -> Vibe TaskDraft / Cloudflare chat
      -> source + config + composition proposals
      -> live preview
      -> update from head if stale
      -> Vibe validation
      -> accept
  -> new Git head
```

Vibe attaches additional semantics to the transaction: task purpose, recipe/config deltas, dependency changes, capability deltas, validation state, and last-good promotion. Those do not require a competing source authority.

### Step transactionality

A Builder step and the provenance that explains it should become durable atomically. The Cloudflare provider should reuse upstream step-transactional persistence for source effects where possible; Vibe should extend the same conceptual transaction to configuration, module, recipe, composition, and capability-request changes.

A crash before the authoring barrier should not leave unexplained partial customization.

### Portable source snapshot

A `VibeSourceSnapshot` contains normal relative paths, readable source, the Vibe semantic manifest, configuration schemas/defaults, dependency declarations and lock information, source revision identity, and known provenance.

It excludes credentials, grants, private app data, Builder chat, local overrides unless explicitly selected, and protected resource contents.

The Cloudflare adapter should export from the accepted Git tree. A future local provider may use an ordinary filesystem-backed Git repository. Moving between providers remains an explicit install/import/export operation; Vibe does not require silent bidirectional synchronization.

### Worktrees

Cloudflare Worktrees are a strong authoring primitive for source that is not an executable Gadget: external repositories, reusable modules, or code being prepared for upstream contribution. They are provider placement, not the Vibe package format.

### Determinism test

For a fixed accepted revision:

1. export twice;
2. compare normalized file paths and contents;
3. import into a fresh composition;
4. run acceptance checks;
5. export again;
6. compare the semantic snapshot and Git tree.

Generated timestamps, provider-local workpiece IDs, and deployment metadata must not make portable source drift.

## Last-Good and Recovery

Cloudflare OS's draft/mainline distinction is necessary but not sufficient for Vibe. Mainline means accepted source; last-good means an accepted artifact that passed Vibe's validation policy.

The hosted adapter must maintain:

- `previewRevision`: mutable candidate shown during a Builder turn
- `acceptedRevision`: source the user accepted
- `lastGoodRevision`: most recent accepted revision that passed required checks
- `runningRevision`: revision currently serving the run surface

The normal transition is:

```text
draft -> preview -> user accepts -> validate -> last-good -> run
             |                         |
             +-- reject/discard        +-- failure keeps prior last-good
```

Validation for the Hello World slice should include:

- Source parses or compiles.
- The app entry point loads.
- The Vibe manifest validates.
- Required actors register.
- Required bindings are either available or produce a designed setup state.
- A basic render probe succeeds.

If the new accepted revision fails, the user keeps the source and diagnostics, but the ordinary run route stays on the prior last-good revision. The user can repair, explicitly override where policy permits, or revert.

## Configuration as a First-Class Product Surface

Vibe should add a configuration system above Gadget storage instead of asking the agent to edit constants for every change.

An actor can publish a schema with:

- Key, type, default, label, description, and category
- Allowed values or validation constraints
- Whether a restart or actor reload is required
- Whether a value is packageable, per-installation, per-user, or secret
- Whether a value is advanced or experimental
- Migration behavior across schema versions

The shell renders a settings UI from that schema. The Builder can inspect and change the same values through the runtime API.

### Configuration layers

Highest precedence first:

1. Session or preview override
2. Per-user setting
3. Per-installation setting
4. Package or template default
5. Actor hard default

Each effective value should expose provenance so the user and agent can answer:

- What is the current value?
- Which layer set it?
- What would happen if this override were removed?
- Does changing it affect only me or everyone using this app?

### Config-first Builder policy

When fulfilling a request, the Builder should try:

1. an existing high-level setting;
2. an advanced setting;
3. a binding or connector option;
4. an actor replacement or extension;
5. a source change.

This is a preference, not a prohibition. The Builder should explain when a lower-level change is needed and may propose a new stable setting after resolving a recurring source-level problem.

### Configuration acceptance

The substrate experiment must add one setting, such as `greeting.text`, and prove:

- It appears in a generated settings view.
- Changing it updates the app without an LLM call.
- Its validation errors are understandable.
- Export includes its schema and default.
- Export excludes the installation's current override by default.

## Bindings, Gatekeepers, and Authority

Cloudflare OS Gatekeepers are a strong hosted implementation of Vibe bindings.

Vibe should expose two distinct records:

```ts
export interface VibeBindingRequirement {
  name: string;
  kind: string;
  apiVersion: string;
  capabilities: readonly string[];
  optional: boolean;
  reason: string;
}

export interface VibeBindingGrant {
  requirementName: string;
  provider: string;
  principal: string;
  resourceScope: readonly string[];
  grantedCapabilities: readonly string[];
  expiresAt?: string;
}
```

A package carries requirements. An installation or collaborator supplies grants.

The Cloudflare adapter should:

- Create or select an appropriate Gatekeeper for a requirement.
- Introduce only the narrowed capability to the Gadget.
- Keep provider credentials out of source and Gadget storage.
- Preserve Gatekeeper audit and approval behavior.
- Return a portable status and error model to the app.
- Record protected-resource observations needed for later sharing decisions.

The same Vibe binding API must be callable from UI code and as an agent tool. The agent should not receive a parallel privileged API.

### Collaborator authority

Live collaborators should act with their own:

- Identity
- Model authorization
- Connector accounts
- Binding grants
- approval decisions

Sharing an app must not silently lend the owner's authority. This principle applies even if the first Vibe prototype supports only one user.

## Sharing and Packaging

The product must use explicit verbs because Cloudflare OS supports materially different operations:

- **Invite to use:** live app, restricted use role
- **Invite to build:** live app, source-changing role
- **Share a copy:** independent instance from a package or Blueprint
- **Export source:** readable files for ordinary tools

### Hosted package mapping

The first `.vibeapp` package may be implemented as:

- A Vibe manifest and source snapshot
- Binding requirements
- Packageable configuration schemas and defaults
- Metadata and optional preview assets
- A Cloudflare OS Blueprint or `.gadget` representation as an implementation payload

The Vibe package contract is authoritative. A Cloudflare-specific payload is optional and replaceable.

### Package installation

Installing a package:

1. validates format and runtime API compatibility;
2. creates an independent app instance;
3. displays required and optional bindings;
4. asks the installer to supply their own grants;
5. applies defaults without copying the author's private settings;
6. runs validation;
7. records the installed source revision and package provenance.

No credential, live connection, private app data, build chat, or hidden owner authority crosses the boundary.

### Blueprint limitation and Vibe dependency layer

Cloudflare OS explicitly treats an instantiated Gadget as independent from its Blueprint and currently provides no automatic update path to existing instances. A Blueprint update changes the published snapshot for future installations, not the code already copied into related Gadgets.

Vibe should preserve that template behavior while adding a package layer above the Cloudflare representation:

- `.vibeapp` records authored app source, versioned module dependencies, an exact lock, and provenance.
- The Cloudflare adapter resolves the graph and may flatten it into a self-contained Gadget snapshot.
- The flattened Gadget is a derived runtime artifact, not the sole source authority.
- Module code may be cached and reused, but its state, configuration, bindings, and grants remain per app.
- Updates are explicit source transactions with compatibility checks, migration, preview, promotion, and rollback.

This layer is required if improvements to a renderer, actor, or editor component are to reach existing apps without manually porting copied source. See [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md).

Cloudflare OS already supplies part of the mechanism:

- The Gadget server loader passes every `.js` file in the Gadget to Worker Loader as a module map, with `server.js` as the entry module.
- The Gadget client currently receives only `client.js`; upstream marks client bundling as future work.
- The `UiBundle` contract anticipates content-addressed implementations and support-library version metadata, but does not implement a dependency resolver or lock.

The substrate experiment should therefore test a Vibe composition root over versioned code modules, actor modules, and presets. Resolve the graph at build time, preserve multiple server modules where supported, and bundle the client graph into the current self-contained `client.js` shape. The resulting Gadget remains runnable without a registry or Builder.

## Framework and Build Strategy

The Vibe foundation considers SvelteKit self-contained apps, in-browser build tools, and a local filesystem-oriented project. Cloudflare OS currently has its own Gadget client/server environment and React/Vite-based Workshop.

The experiment should not try to settle every framework question.

### Hosted v0 rule

Use the source and client/server shapes that the pinned Cloudflare OS Gadget runtime supports, wrapped in Vibe contracts. Keep authored source readable and unminified.

### Required experiment

Determine whether a Vibe app can use:

- Plain TypeScript and a minimal DOM or UI layer
- A supported component framework without Workshop-specific imports
- External package dependencies under a declared policy
- A deterministic source export

Arbitrary SvelteKit compatibility is not a condition for the Cloudflare OS experiment. SvelteKit self-contained apps remain a strong candidate for the independent local runner. If the hosted substrate can support Svelte cleanly without a deep build-system fork, that is useful evidence, not a prerequisite.

Similarly, `just-bash`, a TypeScript Git implementation, browser-contained compilers, and IndexedDB are local-runner candidates. They should not be inserted into the hosted experiment unless a required acceptance test cannot be completed without them.

## Deployment Strategy

Start from the [Cloudflare OS starter](https://github.com/cloudflare/cloudflare-os-starter/tree/9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee), which demonstrates a deployment-owned shell around a pinned upstream core.

The Vibe deployment owns:

- Brand and app-centric navigation
- Identity and onboarding
- Model-provider policy and user authorization
- Routes and public run surfaces
- Connector catalog and Vibe binding adapters
- Storage and operational configuration
- Vibe runtime and Builder adapters
- Default empty-app Blueprint
- Package import and export
- Telemetry, support, and incident policy

Upstream core remains pinned and updateable where possible.

### Patch budget

Every change to Cloudflare OS core must be logged with:

- Purpose
- Files and upstream subsystem touched
- Why an extension or deployment seam was insufficient
- Whether it should be proposed upstream
- Expected merge conflict risk
- Test that would detect breakage

The experiment is a no-go if ordinary Vibe product work repeatedly requires invasive edits across Workshop frontend, Workshop backend, and shared API internals. A small number of narrow patches to expose missing extension points is acceptable.

### Licensing

Both reviewed repositories are published under Apache-2.0 at the pinned revisions. Before distributing a derivative, confirm notice, attribution, trademark, dependency, and deployment obligations with appropriate legal review. This document is not legal advice.

## Proposed Repository Shape

If the experiment passes, organize Vibe-owned work so upstream coupling is visible:

```text
/
  apps/
    vibe-web/                     # app library, run view, Builder view
  packages/
    runtime-contracts/            # public app-facing interfaces
    builder-contracts/            # Builder client and change lifecycle
    actor-sdk/                    # actor descriptors and registry
    config-sdk/                   # schemas, layers, validation
    package-format/               # .vibeapp import/export
    cloudflare-runtime-adapter/   # Vibe runtime -> Gadget/Gatekeeper
    cloudflare-builder-adapter/   # Vibe Builder -> Workshop/agent
    cloudflare-package-adapter/   # Vibe package -> Blueprint
  blueprints/
    empty-vibe-app/
  deployments/
    cloudflare-os/
      upstream/                   # pinned core or submodule
      patches/
      src/                        # deployment-owned customization
      PATCHES.md
  specs/
```

Exact placement should follow upstream's extension conventions discovered during the experiment. The architectural requirement is that Vibe contracts and Cloudflare adapters remain visibly separate.


## Substrate Experiment

The revised experiment tests the remaining Vibe-specific boundaries rather than re-solving Git history or multi-Gadget placement.

### Stage 0: Pin and inventory current upstream

Actions:

- Pin Cloudflare OS at `687aab049cf084030a42093c10c4dde3d9c33fb4`.
- Inventory public APIs and any Vibe-required patches.
- Record the current Workpiece, Gadget, Worktree, Git, Gatekeeper, Blueprint, and chat-proposal seams.
- Create `PATCHES.md` before the first core change.

Evidence:

- Reproducible current checkout.
- Explicit adapter-versus-patch map.
- No assumption depends only on the obsolete August source model.

### Stage 1: Resource plus two Tools

Build the smallest semantic vertical slice around Project Notebook tasks.

```text
Task Resource
  -> List Tool
  -> second Tool, e.g. Kanban
```

Both Tools operate on the same Task identity/state. They may initially co-locate in one Gadget.

Evidence:

- Switching/adding a Tool does not copy Task data.
- Builder and UI can identify the Resource and Tool separately from the Gadget implementing them.
- Opening the accepted composition invokes no model.

### Stage 2: Malleability ladder

Starting from the List Tool, exercise:

```text
session-only presentation tweak
  -> keep as typed config/recipe
  -> add/swap a compatible Tool
  -> create a tiny Tool/module only when needed
  -> source change as final escalation
```

Evidence:

- The Builder chooses the smallest local reversible mechanism that satisfies the request.
- Kept changes are inspectable without replaying the authoring chat.
- Direct/manual configuration uses the same underlying contracts as Builder changes.

### Stage 3: Native TaskDraft over Cloudflare Git/OT

Ask the Builder for a source-level change.

Evidence:

- Vibe TaskDraft maps to Cloudflare chat proposal state.
- Preview runs before acceptance.
- Accepted source has a stable Git commit identity.
- Stale proposal update/merge is handled by upstream machinery rather than a second Vibe merge layer.
- Step provenance and durable source effects remain accountable together.

### Stage 4: Capability-backed placement split

Add one narrow external capability, then move the authority-owning implementation across a stronger provider boundary if warranted.

Evidence:

- Vibe semantic APIs do not change when placement changes.
- The Cloudflare adapter uses Gatekeepers; app/tool code receives no provider credential.
- UI and Builder use the same typed capability.
- A trivial local/in-memory adapter implements the same Vibe contract.

### Stage 5: Last-good and package round trip

Introduce a broken source change after one known-good accepted revision, then export/import the composition.

Evidence:

- Accepted Git head and Vibe last-good/running revision may differ safely.
- Ordinary Run mode remains on last-good after failed validation.
- Export contains readable source, semantic manifest, recipes/config schemas, dependency lock/provenance, and requirements.
- Export excludes credentials, grants, private runtime state, protected observations, and Builder history.
- Import into a fresh provider instance yields equivalent Resource/Tool behavior.

### Stage 6: Coupling report

Classify every dependency as Vibe contract, documented Cloudflare API, internal API, or core patch.

The experiment ends with **go**, **conditional go**, or **no-go** based on semantic leakage and patch burden, not on whether Cloudflare OS can store source in Git - that question is already answered upstream.

## Acceptance Matrix

| Capability | Test | Pass evidence | Decision impact |
| --- | --- | --- | --- |
| Persistent app | Accept Hello World, reopen without model | Render and request trace | Required |
| Immediate preview | Change source from chat | Preview updates within the active build session | Required |
| Runtime seam | Inspect generated app imports | Only Vibe public APIs | Required |
| Shared API | Invoke actor from UI and agent | Same schema and authorization path | Required |
| Module composition | Resolve one code module, actor module, and preset | Exact lock, separate actor state, self-contained Gadget build | Required |
| Configuration | Change greeting with Builder disconnected | Persistent update, no model request | Required |
| Last-good | Introduce invalid revision | Run route stays healthy | Required |
| Source portability | Export, edit, import, export | Readable deterministic snapshot | Required |
| Package privacy | Inspect package and fresh install | No data, credentials, grants, or chat | Required |
| Narrow authority | Grant and revoke demo binding | Designed setup and revoked states | Required |
| Maintainability | Review adapter and patch inventory | Bounded, named coupling | Required |
| Live sharing | Collaborator uses own authority | Separate principal and grants | May follow hosted v0 |
| Local runner | Run same package locally | Equivalent core behavior | Roadmap, not spike gate |
| SvelteKit | Run self-contained SvelteKit app | Supported build and runtime | Research, not spike gate |

## Go, Conditional-Go, and No-Go Rules

### Go

Choose Cloudflare OS for hosted v0 when all required acceptance rows pass and:

- The app runs without a model or active build session.
- Cloudflare-specific code is contained in adapters and deployment code.
- Vibe can maintain last-good independently of an agent draft.
- Source and package round trips are deterministic enough for ordinary development.
- The upstream core patch set is narrow and testable.
- The security model preserves per-user, narrow capabilities.

### Conditional go

Choose a constrained variant when product behavior passes but one replaceable layer fails. Examples:

- Retain Gadget execution but build a separate Vibe package exporter.
- Retain Gatekeepers but replace the Workshop UI.
- Retain the Workshop agent but run apps in an independent Vibe runtime.
- Vendor a small upstream module whose public seam is missing, while proposing the seam upstream.

The condition must name the replacement, owner, and exit test. "We will clean up the fork later" is not a condition.

### No-go

Do not build hosted v0 on Cloudflare OS if any of these remain true after the bounded experiment:

- Ordinary app execution cannot be separated from model or Builder initialization.
- Generated app code must depend on unstable Workshop internals.
- Immediate preview requires full production deployments.
- Readable source cannot be exported and reconstructed reliably.
- A package cannot exclude credentials, grants, private data, and chat.
- Last-good cannot protect the run surface from a failed edit.
- Routine Vibe UX changes demand pervasive core patches.
- The runtime cannot enforce Vibe's binding and actor boundaries.
- Required production behavior depends on unsupported or private infrastructure.

## Plan After a Go Decision

### Phase 1: Hosted Vibe prototype

- Productize the empty-app and Hello World paths.
- Add app library, run, build, settings, source, and recovery views.
- Stabilize runtime, actor, configuration, and Builder contract v0.
- Support a small set of Vibe-owned bindings.
- Maintain explicit upstream pins and patches.

Exit: a user can build and keep a useful local-data app through chat and settings, then export its readable source.

### Phase 2: Portable package and external Builder

- Stabilize `.vibeapp`.
- Publish the Builder attach and preview protocol.
- Let one shared Builder open multiple apps.
- Support source editing and explicit import/export workflows.
- Add installation profiles and permission review.

Exit: an app package is not bound to one Builder instance, and a compatible external Builder can modify it.

### Phase 3: Independent local runner

- Implement the Vibe runtime contract outside Cloudflare OS.
- Select the local source, build, Git, storage, and web/native shell stack.
- Run exported hosted apps where their declared features are supported.
- Add compatibility tests shared by hosted and local adapters.

Exit: the Hello World package and at least one actor/config/binding sample run on both substrates without app-source forks.

### Phase 4: Native shell and broader capabilities

- Wrap the local runner in an appropriate native shell.
- Add filesystem, background task, notification, and richer database capabilities through bindings.
- Preserve install-time permission review and per-user grants.

This phase is intentionally downstream. Cloudflare OS can accelerate product validation without deciding the permanent native architecture.

## Impact on the Foundation Spec

If the experiment passes, propose a focused revision to the [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md):

- Name Cloudflare OS as the hosted v0 substrate, not the universal architecture.
- Add a substrate adapter boundary to the system architecture.
- Separate live source authority by execution mode.
- Make Git-compatible commit identity first-class across providers while keeping live-edit synchronization provider-local.
- Add binding requirements, grants, per-collaborator authority, and observation provenance.
- Add Resources, Tools, and Compositions above Actors and map them to workpiece placement only when needed.
- Define the Builder as logically external even when hosted alongside the runtime.
- Move SvelteKit self-contained apps, `just-bash`, browser Git, and IndexedDB implementation choices to the local-runner track.
- Add Cloudflare OS draft preview as one implementation of the existing live-update contract.
- Add the source and package round-trip tests from this proposal.

Do not weaken these foundation commitments:

- Apps run without invoking an LLM.
- Chat remains available through a compatible Builder.
- Users can edit source with ordinary tools.
- The Builder and runtime communicate through explicit contracts.
- Configuration is usable without code or a model.
- App modules have declared APIs and authority.
- Broken edits do not replace last-good behavior.
- Credentials and private data are not distributed with apps.

If the experiment fails, no foundation change is required. The lessons document still informs the independent architecture.

## Risks and Mitigations

### Upstream maturity

**Risk:** Cloudflare OS is early access and its APIs may change quickly.

**Mitigation:** Pin revisions, isolate adapters, keep compatibility tests, and avoid claiming upstream stability.

### Product-shape mismatch

**Risk:** Reshaping a general Workshop into an app-first product creates a large fork.

**Mitigation:** Test navigation and lifecycle changes before building features; enforce the patch budget and no-go rule.

### Cloud dependency

**Risk:** Hosted v0 is not local-first and may inherit service limits or operational costs.

**Mitigation:** Describe it honestly as the hosted edition; preserve portable contracts and fund the local adapter only after product validation.

### Dual source authority

**Risk:** A provider's transient OT/proposal state and Vibe's portable Git revision identity are both treated as authoritative.

**Mitigation:** Git-compatible commits define portable source identity; provider-local edit state remains provisional until an explicit acceptance/promotion boundary.

### False portability

**Risk:** App source uses Vibe names but still relies on Cloudflare-only behavior.

**Mitigation:** Feature negotiation, adapter contract tests, a second minimal adapter or test double early, and package declarations for nonportable features.

### Authority confusion

**Risk:** Agents or collaborators receive broader access than the app contract implies.

**Mitigation:** Requirement/grant split, Gatekeeper mediation, per-principal grants, shared UI/agent APIs, revocation tests, and observation provenance.

### Configuration sprawl

**Risk:** Exposing every internal value creates an unusable or unstable settings surface.

**Mitigation:** Typed schemas, categories, safe defaults, advanced sections, versioned migrations, and ownership at the actor boundary.

### Framework lock-in

**Risk:** Gadget build assumptions prevent source from moving to a later runner.

**Mitigation:** Keep platform APIs behind `@vibe/*` packages, preserve readable source, declare build requirements, and validate round trips.

### Platform project eclipses product learning

**Risk:** The team spends its effort generalizing infrastructure before testing whether users want the app.

**Mitigation:** Limit the experiment to the vertical slice and stop at its decision report.

## Open Questions

These questions should be answered by code and evidence during the experiment:

1. Can the app library and run view be implemented through starter-owned routes, or do they require core Workshop changes?
2. Can accepted Gadgets run without initializing any agent or model path?
3. What is the smallest stable API that lets Vibe create, preview, accept, reject, and export a Gadget revision?
4. Does upstream expose enough revision identity to maintain Vibe's last-good pointer safely?
5. Can the Gadget client import a Vibe SDK package without leaking Workshop internals?
6. Can actor APIs be registered dynamically and disabled without rebuilding the whole app?
7. Which configuration scopes map cleanly to Gadget storage and collaborator identity?
8. Can a Blueprint round trip preserve all readable source while excluding data, history, and connections?
9. What exact protected-resource provenance can the package and sharing adapters query?
10. Can an app be exported into a framework-neutral snapshot, or must the package declare a Cloudflare-hosted build profile?
11. How much of Vibe's UI can live outside the upstream submodule?
12. Which missing extension points are realistic upstream contributions?

Questions deliberately deferred:

- The final native shell
- The full local build toolchain
- Whether Git or a TypeScript Git implementation is embedded in the browser
- Marketplace economics
- General third-party actor distribution
- Full offline collaboration

## Observable Success

The experiment is complete when a reviewer can:

1. create an empty Vibe app;
2. use chat to add Hello World;
3. see the draft immediately;
4. accept it and reopen the app without a model;
5. alter the greeting through settings with the Builder disconnected;
6. call one actor through both UI and agent;
7. grant and revoke one narrow capability;
8. survive one broken edit through last-good;
9. export, manually edit, and import readable source;
10. inspect a package and confirm it contains no private authority or data; and
11. read a complete adapter and patch inventory.

The decision report must state go, conditional go, or no-go and link each conclusion to that evidence.

## Upstream Sources

All source-specific statements in this proposal derive from the pinned revisions and official project material:

- [Cloudflare OS README at the reviewed revision](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/README.md)
- [Cloudflare OS contributor architecture notes](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/AGENTS.md)
- [Blueprint documentation](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/blueprints.md)
- [Sharing documentation](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/sharing.md)
- [Observer and protected-resource documentation](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/observers.md)
- [Cloudflare OS starter at the reviewed revision](https://github.com/cloudflare/cloudflare-os-starter/tree/9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee)
- [Cloudflare's introduction to Cloudflare OS](https://blog.cloudflare.com/cloudflare-os/)

The companion [lessons document](./2026-08-06-cloudflare-os-lessons.md) separates verified upstream behavior from Vibe recommendations in more detail.
