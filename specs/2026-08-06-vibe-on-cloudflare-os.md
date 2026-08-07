# Building Vibe on Cloudflare OS

**Date:** 2026-08-06  
**Status:** Proposed substrate experiment - architecture decision pending  
**Cloudflare OS revision targeted:** `aedcda8b3066ff666f57ae28ecef7341d6c2dee7`  
**Cloudflare OS starter revision targeted:** `9c18a2e8b0c3741e5f4813546bbf24be5bbb98ee`  
**Related:** [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md), [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md), [Vibe Runtime Providers and Placement](./2026-08-06-vibe-runtime-providers.md), [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md)

**Runtime placement note:** The companion runtime-provider specification refines this document's hosted-first assumption. Cloudflare-hosted, remote self-hosted, managed local, system local, and bundled local execution are provider profiles behind the same Vibe contracts. Where this document recommends hosted v0, treat that as a substrate candidate to test rather than a settled placement decision.

## One Sentence

Build the first hosted Vibe edition as an app-centric Cloudflare OS distribution behind Vibe-owned runtime, Builder, source, and package contracts, while preserving a credible path to a local runner.

## First-Screen Contract

Vibe can plausibly be built on Cloudflare OS, but it should not become "Cloudflare OS with different branding."

The proposed split is:

- Cloudflare OS supplies the first hosted execution substrate: the Workshop kernel, agent loop, isolated Gadget clients and servers, Gatekeepers, storage, collaboration, and Blueprint machinery.
- Vibe supplies the product model: an app-centric shell, an optional external Builder, modular actors, configuration-first customization, readable source export, last-good recovery, and a portable app contract.
- A narrow adapter layer prevents app code from importing Cloudflare OS APIs directly.

This is not yet the selected architecture. Cloudflare OS is early access, its natural product unit is a Gadget inside a Workshop, and its live source model differs from Vibe's file-and-Git direction. The first implementation must therefore be a bounded substrate experiment, not an open-ended fork.

The experiment succeeds only if one vertical slice proves all of the following:

1. A blank Vibe app opens as an ordinary app, with Builder chat available but not required to run it.
2. A user asks the Builder to add "Hello World," sees the change immediately, and accepts it.
3. The app can be closed and reopened with "Hello World" present without invoking an LLM.
4. App code uses Vibe-owned APIs rather than Cloudflare-specific imports.
5. The same typed capability is callable by the app UI and the Builder agent.
6. A user can change an exposed setting without an LLM or source edit.
7. A broken change does not replace the last-good app.
8. Readable source can be exported, edited with ordinary tools, and imported without silently losing meaning.
9. A shared package contains code and declared requirements, but no credentials or private runtime data.
10. The required Cloudflare OS patch set is small, named, and maintainable.

Until that evidence exists, the independent architecture in the [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md) remains the baseline.

## Decision Sought

Approve a Cloudflare OS substrate experiment for Vibe's hosted first edition.

Do not yet approve:

- Cloudflare OS as Vibe's permanent or exclusive runtime
- A production launch on the current upstream
- A deep or indefinite fork
- Cloudflare-specific APIs in app source
- Replacing readable files and Git with Yjs across all Vibe editions
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
- Draft changes that can be previewed before reaching shared mainline code
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
| App | One Gadget instance plus Vibe metadata | Vibe treats the app as the primary product, not as a panel inside a general Workshop |
| App UI | Gadget client | It imports only the Vibe runtime contract |
| App backend | Gadget server | It is reached through the Vibe runtime adapter |
| Actor | A Vibe module and API namespace inside an app | An actor is not necessarily a separate Gadget or Worker |
| Binding | A declared requirement plus a user-specific capability grant | A requirement never grants authority by itself |
| Connector | A Vibe-facing adapter backed by a Gatekeeper or native provider | Secrets stay outside app code |
| Builder | Vibe chat, planning, editing, preview, and acceptance service | It may be embedded in the hosted deployment but is logically external to the app |
| Mainline | The accepted Gadget source revision | Vibe also records a last-good revision and an export revision |
| Package | A portable Vibe source snapshot and requirement manifest | It must exclude credentials, user data, and chat history |
| Template | A Vibe package or a Cloudflare OS Blueprint with Vibe metadata | Installing creates an independent instance |

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

## Actors on Cloudflare OS

Cloudflare OS and Vibe use different module granularity. A Gadget is a good app isolation boundary, but creating a Gadget or Dynamic Worker for every small actor would make modularity expensive and obscure.

The hosted mapping should therefore be:

- One Vibe app instance maps to one Gadget by default.
- One actor maps to one typed API namespace and lifecycle module inside that Gadget.
- An actor declares its configuration, events, storage namespace, and binding requirements.
- The actor registry can enable, disable, inspect, and health-check actors independently.
- An actor crosses into a separate Gatekeeper or Worker only when authority, failure isolation, scaling, or reuse justifies the boundary.

A minimal descriptor:

```ts
export interface VibeActorDescriptor {
  id: string;
  apiVersion: string;
  enabledByDefault: boolean;
  requires: readonly VibeBindingRequirement[];
  configSchema?: string;
  start(context: VibeActorContext): Promise<VibeActor>;
}
```

This preserves the plugin-architecture lesson: small modules communicate through standard APIs, can be shared independently, and can be disabled when they fail. It avoids forcing one infrastructure process per conceptual module.

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

This is the largest mismatch between Vibe and Cloudflare OS.

Cloudflare OS uses collaborative live documents and distinguishes agent draft changes from mainline. Vibe's foundation favors readable files, Git history, and durable last-good artifacts. Trying to make both systems concurrently authoritative would create ambiguous revisions and fragile synchronization.

### Rule: one source authority per execution mode

- In hosted v0, the Gadget mainline is the live source authority.
- In a future local runner, the filesystem and Git working tree are the live source authority.
- Moving between them is an explicit export or import operation.
- Silent bidirectional synchronization is out of scope.

### Portable source snapshot

A `VibeSourceSnapshot` contains:

- Normalized relative paths
- UTF-8, unminified source text
- The Vibe manifest
- Configuration schemas and packageable defaults
- Dependency declarations and a reproducible lock representation
- A source revision identifier
- The parent import or export revision when known
- Format and runtime API versions

It excludes:

- Credentials and tokens
- User binding grants
- App data
- User configuration unless explicitly selected
- Build chat
- Observed private resource identifiers
- Deployment secrets

### Hosted history

For the experiment:

- Cloudflare OS draft state powers conversational iteration.
- Accepting a change creates a Vibe accepted revision.
- Validation promotes that revision to last-good.
- Source export produces a deterministic snapshot for that accepted revision.
- Git may store exported snapshots, but it is not the live hosted authority.

If the hosted edition later exposes Git as a first-class workflow, it should use explicit import, export, branch, and merge operations. It should not pretend Yjs edits and a Git working tree are automatically the same transaction.

### Determinism test

For a fixed accepted revision:

1. export twice;
2. compare normalized file paths and contents;
3. import the export into a fresh app;
4. run its acceptance checks;
5. export again;
6. compare the semantic snapshot.

Generated timestamps, instance identifiers, and deployment metadata must not make source content drift.

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

The experiment is one vertical slice. It is not a general platform build.

### Stage 0: Pin and inventory

Actions:

- Pin the two reviewed upstream revisions.
- Record licenses and required notices.
- Map supported extension points in Workshop frontend, Workshop backend, shared API, Gadget runtime, Gatekeepers, and Blueprint import/export.
- Create `PATCHES.md` before the first core change.
- Record all assumptions that are supported only by upstream README text rather than a stable API.

Evidence:

- Reproducible upstream checkout
- Dependency and service inventory
- Initial extension-versus-patch map

Stop condition:

- The pinned project cannot run in its documented local environment or requires unavailable private infrastructure.

### Stage 1: Empty Vibe app

Actions:

- Create a Vibe deployment profile from the starter.
- Add an app library route and an open-app route.
- Create an empty-app Blueprint with a Vibe manifest.
- Render the app as the primary surface.
- Place Builder chat beside it in build mode.
- Provide a run route without the Builder panel.

Evidence:

- A new user can create and open an empty app.
- Refresh and reopen preserve the app instance.
- The run view has no model-dependent startup behavior.

Stop condition:

- Making the app primary requires a broad rewrite of the Workshop's core navigation and lifecycle.

### Stage 2: Hello World through chat

Actions:

- Give the Builder only the Vibe manifest and app-facing APIs in its authoring instructions.
- Prompt: "Add Hello World to this app."
- Stream or reload the draft into the preview surface.
- Accept the change.
- Close the build session and open a fresh run session.
- Disable model authorization and open the app again.

Evidence:

- The visible text is produced by accepted source.
- Immediate preview works without restarting the whole deployment.
- Reopen renders the accepted result.
- Network or service logs show no LLM call during ordinary run.
- Generated code contains no Cloudflare OS import.

Stop condition:

- Preview requires publishing a full deployment for every edit.
- Accepted Gadget execution depends on a live model call.

### Stage 3: Shared typed actor API

Actions:

- Add a small `greeting` actor.
- Package it as a versioned actor module.
- Add a composition root that consumes the actor through its typed API.
- Expose one typed method, such as `getGreeting()`.
- Call it from the app UI.
- Expose the same method to the Builder agent.
- Open two apps using the same actor package and verify separate actor state and grants.
- Disable the actor and verify a designed unavailable state.

Evidence:

- UI and agent use one API definition and authority path.
- Both apps resolve the same actor code identity without sharing state or authority.
- Disabling the actor does not corrupt the rest of the app.
- Errors cross the adapter in Vibe's normalized form.

Stop condition:

- The agent needs a privileged duplicate API or ambient access to call the actor.

### Stage 4: Config without code

Actions:

- Publish a schema for `greeting.text`.
- Generate the settings control.
- Change it to "Hello Vibe" with the Builder disconnected.
- Reopen the app.
- Export the source package.

Evidence:

- The app updates without an LLM call.
- The value persists for the intended scope.
- The package contains the schema and default, not the installation override.
- The Builder can later discover the same setting rather than editing source.

Stop condition:

- Configuration can be changed only by editing Gadget source.

### Stage 5: Capability binding

Actions:

- Add one narrow demonstration binding whose credential or resource authority lives outside the app.
- Declare the requirement in the manifest.
- Install or open the app without a grant and show a setup state.
- Grant the binding and call it from UI and agent.
- Inspect the package and app storage for credential leakage.

Evidence:

- Requirement and grant are distinct.
- App code never receives provider credentials.
- UI and agent use the same narrowed API.
- Revocation produces a designed state.

Stop condition:

- The Gadget or agent requires ambient account credentials.

### Stage 6: Failure and last-good

Actions:

- Accept a valid revision and mark it last-good.
- Ask the Builder for an intentionally invalid change.
- Show diagnostics in preview.
- Attempt acceptance.
- Open the run route.
- Repair or revert.

Evidence:

- The failed revision remains inspectable.
- The run route continues to serve the previous last-good revision.
- Recovery does not require manual database repair or deployment rollback.

Stop condition:

- A malformed draft can make the ordinary app unavailable or destroy the prior source.

### Stage 7: Source and package round trip

Actions:

- Export the accepted app twice.
- Compare normalized snapshots.
- Inspect the manifest, composition root, exact lock, and included module sources.
- Edit the greeting in an ordinary text editor.
- Import into a fresh app.
- Supply a fresh binding grant.
- Run validation and export again.

Evidence:

- Source is readable and unminified.
- Equivalent exports are deterministic.
- The exported package is self-contained while retaining module identity and provenance.
- The imported app behaves equivalently.
- Private data, settings, grants, and chat do not cross.
- The new app is independent of the original.

Stop condition:

- Meaningful source is trapped in opaque state or cannot be reconstructed deterministically.

### Stage 8: Coupling report

Actions:

- List every Vibe-owned package.
- List every upstream core patch.
- Classify each dependency as public API, documented extension, internal API, or patch.
- Estimate the effect of updating upstream by one representative revision.
- Record any feature that could not be expressed through Vibe contracts.

Evidence:

- A coupling map another engineer can audit.
- A clear recommendation: go, conditional go, or no-go.

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
- Clarify that Git is first-class in the local edition and explicit import/export history in hosted v0.
- Add binding requirements, grants, per-collaborator authority, and observation provenance.
- Map Vibe actors to API namespaces by default, not one process per actor.
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

**Risk:** Yjs/Gadget state and Git disagree.

**Mitigation:** One authority per mode, explicit import/export, immutable revision identifiers, no silent two-way sync.

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
