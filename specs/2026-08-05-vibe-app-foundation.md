# Vibe App Foundation

**Date:** 2026-08-05  
**Status:** Draft - implementation has not started  
**Owner:** TBD

## One Sentence

Create source-bearing applications whose integrated chat client connects to an optional shared Builder, allowing configuration, modular actor changes, and versioned source edits to appear live while the last-good app continues to run without an LLM or Builder.

## First-Screen Contract

This specification defines the product and implementation foundation for an application that users can change by chatting with it.

The repository is currently empty. There is no existing application, package layout, build pipeline, or documentation convention.

The target is a stable app shell containing an integrated Builder chat client and a sandboxed app canvas. The chat client may connect to a shared, separately installed Builder. The Builder owns model credentials, agent execution, source editing, builds, and version control. The distributed app owns its normal runtime, readable source, configuration schemas, current artifact, and user data.

The first vertical slice is complete when this sequence works:

1. Start the empty skeleton app.
2. Enter "Add Hello World" in its chat.
3. Let a deterministic fake agent modify the source and build a candidate.
4. See "Hello World" in the app canvas without restarting the outer shell.
5. Quit the app.
6. Disable the Builder and provide no model API key.
7. Reopen the app and see "Hello World" from the stored last-good artifact.
8. Submit a deliberately broken change and verify that the working app remains visible.

The first slice proves the source-to-runtime transaction. It does not attempt to prove that an LLM can reliably write arbitrary applications.

## Reading Guide

The active implementation path is:

- Product contracts and scope
- System architecture and ownership
- Builder protocol
- Authoring and promotion transactions
- Phase 0 and Phase 1 of the implementation plan
- Foundation success criteria

The actor, configuration, capability, packaging, and reference-app sections define constraints that early work must preserve. Their full implementations arrive in later phases.

The roadmap records plausible extensions. It is not part of the initial acceptance bar.

## Motivation

### Current State

As of this document's date, `/Volumes/p/vibe` contains no implementation or prior specification. The design therefore has no compatibility burden.

The product discussion established a high-level behavior:

- An empty app begins as a window or page with a chat and an app area.
- A user can ask the app to add a feature.
- The Builder may clarify the request, modifies the project, and updates the running app.
- The resulting behavior persists as ordinary code or configuration.
- Running the app later does not require an LLM.

The related Epicenter plugin architecture study at `/Volumes/p/RIFT/rift-transcription/reference/epicenter-plugin-architecture-feasibility.md` explored several useful boundaries: capability declarations, permission grants, message-oriented modules, supervised failure isolation, provenance, and a functional core around an imperative shell. This specification adopts those architectural principles without adopting Epicenter's exact plugin stack.

### Problems To Solve

1. Embedding a complete Builder in every app duplicates large dependencies, credentials, caches, model integrations, and update logic.
2. Removing chat from the app harms the most direct UX: request a change where the result will appear.
3. Direct source changes are powerful but expensive for small compatibility fixes that should be configuration.
4. An agent editing the active app in place can leave it broken halfway through a task.
5. Source rollback and user-data rollback have different meanings and must not be coupled.
6. Large, tightly coupled generated modules are difficult for both people and agents to understand, test, reuse, or disable.
7. Browser code, build tools, installed dependencies, model credentials, and native capabilities have different trust levels.
8. A distributable app must remain useful to a recipient who does not have the original Builder, API key, or user data.

### Desired State

The app feels self-modifying while remaining architecturally separated:

```text
user prompt
  -> integrated Builder Client
    -> optional shared Builder
      -> configuration or isolated source edit
        -> validated candidate
          -> atomic promotion
            -> visible app behavior
```

The app shell never needs to contain the full Builder. It contains a small, stable client that speaks a versioned protocol. A Builder may run in the same process during the walking skeleton, in a separate local process later, or remotely in a future version. These placements do not change the logical contract.

## Product Contracts

These contracts are requirements, not implementation suggestions.

### Runtime Independence

- An accepted app behavior must run without an LLM request.
- The app must load its last-good artifact without initializing the Builder.
- Model credentials must never be compiled into an app artifact.
- Losing the Builder may remove modification capabilities, but it must not remove ordinary app behavior.

### Integrated UX, Separate Builder

- Chat remains visually integrated into the app shell.
- The integrated component is a Builder Client, not the full Builder.
- One Builder can serve multiple apps.
- The Builder can be installed, updated, disabled, or recovered independently.
- If no Builder is available, chat preserves the user's prompt and explains how to connect or install one.

### Source Ownership

- Every app has normal, readable, unminified source files.
- The project can be opened in an ordinary editor.
- Manual edits are supported and visible to the Builder.
- A standard Git repository owns source history.
- Generated runtime artifacts are outputs, not the canonical source.

### Configuration Before Code

- The Builder checks configuration and actor substitution before proposing a broad source patch.
- Legitimate variation by user, device, provider, or environment should have a typed configuration point.
- Configuration must remain understandable, validated, reversible, and attributable to a layer.
- Configuration cannot bypass permissions or become an arbitrary command-execution surface.

### Reversible Change

- Agent edits happen in an isolated candidate workspace.
- The active app is not overwritten before the candidate builds and passes a health check.
- Promotion identifies both the source revision and artifact hash.
- A failed task leaves the last-good app active.
- Source rollback does not silently delete or replace user data.

### Modular Design

- State owners and failure boundaries communicate through explicit messages.
- Actors declare their protocol, capabilities, inputs, outputs, lifecycle, and health.
- Actors can be disabled or moved to a stronger isolation boundary without changing their logical interface.
- Not every function, component, or feature becomes an actor.

## Scope

This specification covers:

- The stable app shell, integrated Builder Client, and app canvas
- A shared and optional Builder
- A fixed SvelteKit application stack for the first version
- Readable source plus a self-contained last-good runtime artifact
- Git-backed source history and manual editing
- Candidate builds, health checks, atomic promotion, and rollback
- Transport-neutral Builder messages
- Agent, filesystem, command, Git, and build abstractions
- Actor boundaries and supervision
- Typed, layered configuration and config-first agent behavior
- Initial local persistence with IndexedDB
- Authoring and runtime permission planes
- A Project Notebook reference app
- A phased implementation and observable acceptance criteria

## Non-Goals

The first implementation does not include:

- A public marketplace or registry
- Builder self-modification
- Arbitrary native capabilities
- Multi-user collaboration or CRDT synchronization
- A cloud-hosted Builder
- Cross-app data access
- Multiple frontend frameworks
- Mobile-native UI
- Complete signing and publisher reputation infrastructure
- Remote Git hosting integration
- Automatic execution of untrusted package lifecycle scripts
- A formal distributed actor framework
- Event sourcing for all application data
- A mandatory Wasm plugin runtime

## Design Decisions

The status column distinguishes product decisions from provisional technology choices.

| Decision | Status | Class | Choice and rationale |
| --- | --- | --- | --- |
| Chat placement | Resolved | Design coherence | Keep chat in the app shell as a thin Builder Client. This preserves direct manipulation without duplicating the Builder. |
| Builder ownership | Resolved | Design coherence | Use one optional shared Builder for many apps. It owns credentials, agent execution, tools, and builds. |
| Runtime dependency | Resolved | Design coherence | Accepted app behavior must run without an LLM or Builder. |
| Source format | Resolved | Design coherence | Ship readable conventional source separately from generated artifacts. |
| Initial framework | Resolved for v0 | Taste under constraints | Use one fixed SvelteKit stack to reduce the build and repair search space. |
| Runtime artifact | Provisional | Evidence | Prefer one self-contained `app.html` artifact. A Phase 0 spike must confirm the supported SvelteKit build path and record a fallback. |
| Source history | Resolved | User requirement | Use standard Git objects and refs so history is inspectable by ordinary tools. |
| Promotion model | Resolved | Design coherence | Build candidates in isolation and atomically promote a pointer to the source revision and artifact. |
| Agent command facade | Provisional | Evidence | Evaluate `just-bash` behind `CommandRunner` and `ProjectFilesystem` interfaces. Do not treat it as the security boundary. |
| Embedded Git library | Provisional | Evidence | Evaluate `isomorphic-git` over the same project filesystem while preserving a standard `.git` directory. |
| Native TypeScript compilation | Deferred | Deferred | Revisit `scriptc` for a small bootstrap or native helper after the Svelte pipeline works. |
| Desktop shell | Deferred | Taste under constraints | Keep the UI browser-compatible. Compare Electron and Tauri before native packaging. |
| Builder transport | Deferred to spike | Evidence | Start with a transport-neutral protocol. Evaluate authenticated loopback HTTP/WebSocket for the web prototype and native IPC for a desktop shell. |
| Configuration syntax | Deferred to spike | Taste under constraints | Make the schema normative. Prefer YAML only if comment- and order-preserving round trips are reliable. |
| Actor placement | Resolved default | Design coherence | Run actors in-process by default, then move them to Workers, frames, or processes when permissions, reliability, or load justify it. |
| App data | Resolved for v0 | Design coherence | Use stable-origin local storage, with IndexedDB as the primary structured store. Keep it separate from Git. |
| Permissions | Resolved | Design coherence | Separate authoring permissions from runtime permissions. Manifest requests are not local grants. |
| User customization | Resolved | Design coherence | Prefer configuration, then actor composition, then source changes. A source change creates a local fork. |

## System Architecture

### Ownership Boundary

```text
Distributed app
+----------------------------------------------------------------+
| Stable shell                                                   |
|                                                                |
|  Builder Client ---------------- authenticated protocol ------+|----+
|                                                               ||    |
|  App canvas <---------------- candidate and active artifact ---+|    |
|                                                                |    |
|  Project identity, current pointer, config UI, permission UI    |    |
+----------------------------------------------------------------+    |
                                                                      |
Readable source + Git                                                 |
Content-addressed artifacts                                           |
Local configuration overrides                                         |
App data in an isolated storage partition                              |
                                                                      |
                                           Optional shared Builder <---+
                                           +--------------------------+
                                           | LLM credential store     |
                                           | Agent backends           |
                                           | Project editor           |
                                           | Build and test tools     |
                                           | Task and audit log       |
                                           | Multiple open projects   |
                                           +--------------------------+
```

The distributed app may bundle its thin shell or run inside a shared platform shell. Both forms use the same Builder protocol. The full Builder is never required in the distributable app package.

### Components

#### Stable Shell

The shell owns:

- Builder discovery and connection state
- The integrated chat surface
- Project identity
- The active and candidate canvas frames
- The current artifact pointer
- Runtime capability prompts and grants
- Configuration UI generated from actor schemas
- The runtime message bridge
- Recovery when a candidate or Builder fails

The shell does not own:

- Model credentials
- Prompt execution
- Source reasoning
- Dependency installation
- The canonical project history

#### Builder

The Builder owns:

- Provider adapters and BYOK credentials
- Agent execution and clarification
- Project filesystem access
- Source diffs
- Build, check, and test commands
- Git checkpoints
- Candidate creation and delivery
- Authoring permissions
- Task progress, diagnostics, and audit events

The Builder can expose a full editor UI, but the app needs only the protocol.

#### App Runtime

The app runtime is generated code executing in an isolated canvas. It owns:

- Application UI
- Domain behavior
- Actor instances that belong to the app
- Access to its assigned data namespace
- Requests for granted platform capabilities

It cannot directly access Builder credentials, Builder files, another app's data, or unrestricted native APIs.

#### Agent and Build Worker

The agent/build worker receives a candidate project view, approved tools, and explicit permissions. It does not receive the active runtime directory as a writable target. Crashing or timing out terminates the task without replacing the active artifact.

### Initial Repository Shape

The platform repository should begin with a small number of boundaries:

```text
vibe/
  apps/
    shell/                 # SvelteKit shell, chat client, and canvas host
    builder/               # Local Builder service and optional editor UI
  packages/
    contracts/             # Builder and actor message schemas
    app-sdk/               # Runtime bridge, storage, actors, and config client
    app-template/          # Empty source-bearing SvelteKit project
  examples/
    project-notebook/      # Reference app, added incrementally
  specs/
    2026-08-05-vibe-app-foundation.md
```

Do not create a package for every actor immediately. Extract another package only when it needs an independently enforced dependency boundary or release lifecycle.

## User Journeys

### Add "Hello World"

1. The user opens a skeleton app.
2. The shell loads the current blank artifact.
3. The user enters "Add Hello World."
4. The Builder Client creates a task.
5. The Builder may ask a clarification. The deterministic Phase 1 backend does not.
6. The Builder snapshots the project and gives an isolated copy to `FakeAgentBackend`.
7. The fake backend writes a known Svelte page containing "Hello World."
8. The Builder builds and health-checks the candidate.
9. The Builder creates a Git checkpoint and content-addressed artifact.
10. The shell preloads the candidate, then promotes it.
11. "Hello World" appears without reloading the shell or chat.
12. On the next launch, the shell reads the promoted pointer and loads the same artifact without contacting a model.

### Clarification

1. A task enters `waiting_for_user`.
2. The Builder emits a structured clarification containing a stable question ID.
3. The Builder Client renders the question in the existing conversation.
4. The user answers or cancels.
5. The answer is associated with the task and question ID.
6. The Builder resumes the same candidate workspace.

Clarification must not be simulated as an unstructured task failure.

### Builder Not Installed

1. The user can still run the app and open chat.
2. The shell stores the prompt in a local pending queue.
3. The client displays that editing requires a Builder and offers connection or installation.
4. After a Builder authenticates, the user confirms submission of the saved prompt.
5. The Builder opens the project and creates the task.

The app must not silently upload the prompt to an unrelated service.

### Configuration-First Fix

1. The user asks for a behavior change.
2. The Builder queries the actor graph, configuration catalog, effective values, and provenance.
3. If an existing safe setting can express the request, the Builder proposes a config patch.
4. `ConfigActor` validates, dry-runs, applies, health-checks, and persists the change.
5. The relevant actor reloads at the lightest required apply mode.
6. No source fork is created.

If configuration cannot express the behavior, the Builder moves to actor substitution or a source patch and explains why.

### Manual Source Edit

1. The user opens the project in the shared Builder editor or another editor.
2. The Builder observes the file change.
3. The shell marks source and active artifact as different.
4. The user can request a build or enable safe automatic builds.
5. The candidate follows the same checks and promotion transaction as an agent change.

Chat and manual editing must share one project model. They must not maintain competing copies of source.

### Share An App

The export contains:

- Readable source
- Lockfile and build configuration
- `vibe.json`
- Selected standard Git history
- The last-good runtime artifact
- Required schemas and static assets

The export excludes by default:

- LLM and service credentials
- Local capability grants
- User data
- Device-specific overrides
- Builder task transcripts
- Build caches

## Project And Package Format

### App Project

An app project is a normal SvelteKit project with additional platform metadata:

```text
my-app/
  src/
    routes/
    features/
  static/
  package.json
  package-lock.json
  svelte.config.js
  vite.config.ts
  vibe.json
  vibe.config.yaml          # Provisional human-editable project defaults
  .git/
  .vibe/
    current.json
    transactions/
    builds/
      <artifact-hash>/
        app.html
        runtime-manifest.json
    tasks/
```

The exact lockfile follows the selected workspace package manager. The project must have one committed lockfile.

### Canonical Data

- `src/`, project configuration, manifests, and lockfiles are canonical source.
- `.git/` is canonical source history.
- `.vibe/builds/` is a reproducible artifact cache.
- `.vibe/current.json` is the atomic pointer to the active source revision and artifact.
- `.vibe/tasks/` is Builder metadata and may be pruned.
- App domain data is stored outside the source repository.
- Credentials and local grants are stored outside the project.

### Active Pointer

`current.json` has at least:

```json
{
  "schemaVersion": 1,
  "sourceCommit": "0123456789abcdef",
  "artifactHash": "sha256:abcdef",
  "promotedAt": "2026-08-05T00:00:00Z",
  "runtimeManifestVersion": 1
}
```

The Builder writes a complete temporary pointer and atomically replaces `current.json` only after the source revision and artifact are durable. Startup recovery inspects any transaction journal left by a crash and either completes or abandons it without guessing.

### App Manifest

`vibe.json` declares identity and requests, not local grants:

```json
{
  "schemaVersion": 1,
  "id": "example.project-notebook",
  "name": "Project Notebook",
  "framework": "sveltekit",
  "actors": [
    {
      "id": "notebook",
      "protocolVersion": 1,
      "entry": "src/features/notebook/actor.ts"
    }
  ],
  "runtimeCapabilities": [],
  "authoringCapabilities": [],
  "configuration": {
    "projectDefaults": "vibe.config.yaml"
  }
}
```

The manifest schema is versioned. Unknown required fields make the app incompatible rather than being ignored.

### Portable Package

A future `.vibeapp` file is a portable archive, not a minified-only deployment. Its first format should include:

- Manifest
- Source and assets
- Lockfile
- A compact standard Git bundle or selected refs
- Last-good artifact and runtime manifest
- Checksums

It must not include private data or credentials by default. Signing, publisher identity, base-app update channels, and marketplace metadata are later format extensions.

## Builder Protocol

### Protocol Requirements

The protocol must be:

- Versioned
- Transport-neutral
- Authenticated when crossing a process boundary
- Structured and schema-validated
- Streamable for task events
- Cancelable
- Reconnectable
- Explicit about project and task identity
- Forward-compatible for optional fields
- Strict about unknown required behavior

### Message Envelope

The shared contracts package should define a small envelope:

```ts
type BuilderMessage<T = unknown> = {
  protocol: "vibe.builder";
  protocolVersion: 1;
  type: string;
  messageId: string;
  requestId?: string;
  projectId?: string;
  taskId?: string;
  sentAt: string;
  payload: T;
};
```

Payload schemas are discriminated by `type`. Messages that fail validation are rejected before reaching a project or task handler.

### Minimum Operations

Connection and discovery:

- `builder.hello`
- `builder.authenticate`
- `builder.capabilities`
- `builder.disconnect`

Project operations:

- `project.open`
- `project.close`
- `project.status`
- `project.diff`
- `project.build`

Task operations:

- `task.create`
- `task.accepted`
- `task.progress`
- `task.clarification.requested`
- `task.clarification.answer`
- `task.candidate.ready`
- `task.failed`
- `task.cancel`
- `task.completed`

Promotion operations:

- `candidate.describe`
- `candidate.promote`
- `candidate.reject`
- `candidate.promoted`

The event names describe semantics, not a required RPC library.

### Task State Machine

```text
queued
  -> inspecting
    -> waiting_for_user -> inspecting
    -> editing
      -> validating
        -> candidate_ready
          -> promoting
            -> completed

Any nonterminal state
  -> canceling -> canceled
  -> failed
```

Each transition is persisted by the Builder. Reconnection returns the last known state plus events after a supplied cursor.

### Discovery And Authentication

Phase 0 must compare at least:

- Authenticated loopback HTTP plus WebSocket
- Browser extension or custom protocol mediation
- Native IPC in a packaged shell

For a browser prototype, the current recommendation is a loopback endpoint using a short-lived, app-scoped capability token. The exact handshake is not resolved until the spike demonstrates:

- Another website cannot silently control the Builder.
- A token cannot open an unrelated project.
- Origin checks do not depend on a wildcard.
- Reconnection works after either side restarts.
- The app can offer an install or connection flow when discovery fails.

## Authoring Transaction

### Interfaces

Technology candidates sit behind host-owned interfaces:

```ts
interface AgentBackend {
  run(task: AgentTask, context: AgentContext): AsyncIterable<AgentEvent>;
}

interface ProjectFilesystem {
  createCandidate(baseRevision: string): Promise<CandidateWorkspace>;
  readFile(path: string): Promise<Uint8Array>;
  writeFile(path: string, contents: Uint8Array): Promise<void>;
  diff(candidateId: string): Promise<ProjectDiff>;
}

interface CommandRunner {
  run(command: ApprovedCommand): Promise<CommandResult>;
}

interface VersionControl {
  snapshot(label: string): Promise<SourceRevision>;
  commit(candidateId: string, message: string): Promise<SourceRevision>;
  diff(from: string, to: string): Promise<ProjectDiff>;
}

interface BuildRunner {
  build(candidateId: string): Promise<BuildArtifact>;
  check(candidateId: string): Promise<CheckResult>;
}
```

These are conceptual contracts. Exact method names may change when the first consumers exist, but ownership must not collapse into one unrestricted agent object.

### Candidate Workspace

Every task operates against an immutable base revision and a writable overlay or worktree:

```text
active source revision
  -> isolated candidate filesystem
    -> agent or manual edits
      -> diff
        -> check and build
          -> health check
            -> source checkpoint and artifact
              -> atomic current pointer
```

The agent sees:

- A writable `/project` candidate
- A read-only SDK and contract reference
- A disposable `/tmp`
- Explicit custom commands
- Only approved network access

The agent does not write directly to `.vibe/current.json`, the active artifact directory, Builder credentials, or another project.

### Command Facade

Evaluate `just-bash` as the shell-like interface because a familiar command vocabulary may simplify agent integration. If selected:

- Run it in a Worker or utility process where possible.
- Mount only the candidate project, read-only SDK, and temporary directory.
- Make network unavailable by default.
- Replace broad host commands with explicit commands such as `vibe-build` and `vibe-check`.
- Expose a restricted Git command surface for status, diff, and log.
- Keep build execution and promotion under host control.
- Treat path validation, process isolation, and capability checks as separate security layers.

If the spike rejects `just-bash`, the interfaces above allow a minimal custom command runner or another virtual shell without changing Builder or agent semantics.

### Git Semantics

The app owns a standard Git repository.

- Manual and agent edits share one history.
- A task records its base tree before editing.
- Dirty manual changes are captured in a recoverable pre-task snapshot without silently discarding them.
- An accepted source change gets a visible checkpoint with its task ID.
- A failed candidate may retain a private diagnostic ref but does not advance the active pointer.
- Undo selects an earlier source revision and builds a new candidate; it does not rewrite unrelated user history by default.
- Source and artifact provenance are linked in `current.json` and the task log.

The first version may use app-owned branches and refs. Supporting arbitrary existing repositories with custom hooks, submodules, or complex branch policies is deferred.

### Promotion

Default policy:

- Reversible local changes that request no new capability may auto-promote after checks.
- New permissions, dependency changes, destructive data migrations, and Builder self-changes require explicit approval.
- A review mode may require approval for every source promotion.

Promotion is an idempotent transaction:

1. Freeze the candidate diff.
2. Run source checks and build.
3. Load the candidate with preview data and perform a health check.
4. Create a source revision.
5. Store the artifact under its content hash.
6. Durably write a transaction record containing both identifiers.
7. Atomically replace the active pointer.
8. Notify the shell.
9. Mark the task complete.

On restart, the Builder or shell reconciles incomplete transaction records. It never infers an active source revision from the newest build directory.

## Live Update And Last-Good Runtime

### Two Update Mechanisms

HMR and artifact promotion solve different problems:

- HMR may provide a fast inner loop while the Builder is connected.
- Atomic artifact promotion makes an accepted change persistent and recoverable.

HMR is optional. Correctness cannot depend on an in-memory development server.

### Canvas Promotion

The shell keeps the chat and outer window mounted:

```text
active frame: artifact A
candidate frame: artifact B, hidden

load B
  -> runtime handshake
    -> manifest compatibility check
      -> health check
        -> reveal B and retire A
```

If candidate B fails to load, times out, emits an incompatible manifest, or fails health, frame A remains active.

The swap is not a native app restart. Transient UI state inside the generated app may reset. Persistent state remains in the assigned app data namespace.

### Stable Origin

Each app requires a stable and isolated origin or equivalent desktop storage partition. Path-only isolation under one shared web origin is insufficient for mutually untrusted apps.

The origin provider may use:

- A dedicated loopback origin in the browser prototype
- A custom protocol and partition in Electron
- A webview storage partition in another native shell

The shell and app canvas must be cross-origin or equivalently isolated. The app uses a schema-validated `MessageChannel` bridge for shell capabilities.

### Candidate Data

A candidate must not mutate production data during a health check.

The app SDK therefore receives a storage mode:

- `active`: normal persistent namespace
- `preview`: cloned or disposable preview namespace
- `migration`: explicit migration transaction against a recoverable snapshot

Phase 1 can prohibit schema-changing candidates. A later `DataMigrationActor` provides backup, dry-run, apply, and rollback semantics before such candidates may promote.

### Changes That May Reload

- Component and style changes: HMR or canvas swap
- Ordinary source changes: canvas swap
- Live configuration: actor update only
- `restart_actor` configuration: one actor restart
- App data schema: migration plus canvas swap
- Runtime capabilities: approval plus canvas reload
- Shell or native bridge code: full shell restart

## Actor Architecture

### Module Versus Actor

A module is a unit of code organization and reuse.

An actor is a module that owns state, lifecycle, messages, an independently useful capability, or a failure boundary. Every actor is a module. Most helpers and UI components should remain ordinary modules.

Create an actor when at least one of these is true:

- It owns independently mutable state.
- It needs a distinct permission set.
- It must restart or fail independently.
- It has more than one implementation.
- It is reusable across apps.
- It crosses a Worker, frame, process, or native boundary.

Do not create an actor only to shorten a file.

### Actor Contract

```ts
type ActorManifest = {
  id: string;
  protocolVersion: number;
  capabilities: string[];
  accepts: string[];
  emits: string[];
};

interface Actor {
  start(context: ActorContext): Promise<void>;
  handle(message: ActorMessage): Promise<ActorResult | void>;
  health(): Promise<ActorHealth>;
  stop(reason: StopReason): Promise<void>;
}
```

Actor messages are versioned, serializable, and schema-validated:

```ts
type ActorMessage<T = unknown> = {
  protocol: "vibe.actor";
  protocolVersion: 1;
  id: string;
  from: string;
  to: string;
  kind: "command" | "event" | "query" | "result";
  type: string;
  correlationId?: string;
  payload: T;
};
```

Actors do not reach into another actor's internal state. They send intents, queries, results, and events.

### Platform Actor Catalog

| Actor | Owns | Likely initial placement |
| --- | --- | --- |
| `BuilderConnectionActor` | Discovery, authentication, reconnection, protocol cursor | Shell process |
| `AgentActor` | Agent task lifecycle and clarification | Builder process |
| `ProjectActor` | Project identity, source status, candidate workspaces | Builder process |
| `VersionControlActor` | Snapshots, diffs, refs, accepted revisions | Builder process |
| `BuildActor` | Checks, builds, artifact metadata | Builder worker |
| `PreviewActor` | Candidate load, health, frame swap | Shell process |
| `PermissionActor` | Requests, local grants, allow/ask/deny | Shell and Builder policy stores |
| `ConfigActor` | Schemas, layers, provenance, apply transaction | Shell process |
| `AppRuntimeActor` | Runtime handshake and app actor supervision | App canvas |
| `DataMigrationActor` | Data snapshots, migration dry-run and commit | Isolated worker or canvas |

The catalog names logical responsibilities. Phase 1 may implement several as typed modules in one process, provided dependencies follow these boundaries.

### Placement And Supervision

An actor may run:

- In-process
- In a Web Worker
- In a sandboxed iframe
- In a Builder utility process
- In a native process

The actor registry tracks lifecycle and health. A supervisor may restart, disable, or replace an actor according to policy. A crash must produce a bounded failure event rather than an unhandled cross-system exception.

Actor replacement requires a compatible protocol. An actor can request capabilities, but only the shell or Builder permission owner grants them.

### Generated Feature Convention

```text
src/features/reminders/
  manifest.ts
  messages.ts
  actor.ts
  state.ts
  config.schema.json
  ui/
  tests/
```

The manifest and messages give an agent a narrow entry point. The Builder should use the actor graph to select relevant source instead of loading the entire project by default.

## Configuration Architecture

### No-Fork Escalation Ladder

Configuration is the least expensive form of self-modification:

```text
preference
  -> high-level preset
    -> detailed typed configuration
      -> compatible actor substitution
        -> install an extension actor
          -> patch app source
            -> change Builder or platform core
```

The Builder starts at the top and moves downward only when the higher layer cannot express the requested behavior safely.

### Schema

Each configurable actor publishes a typed and versioned schema. A setting includes:

- Stable key
- Type
- Default
- Description and rationale
- Examples
- Simple or advanced visibility
- Allowed scopes
- Secret classification
- Capability implications
- Validation constraints
- Apply mode
- Deprecation and migration metadata

Example:

```yaml
actor: reminders
protocolVersion: 1
settings:
  frequency:
    type: enum
    values: [off, daily, weekdays, weekly]
    default: daily
    description: How often the app checks for due tasks.
    visibility: simple
    scopes: [project, device, user, session]
    applyMode: live

  quietHours:
    type: object
    visibility: advanced
    applyMode: live
    properties:
      start:
        type: time
        default: "22:00"
      end:
        type: time
        default: "08:00"
```

The schema is normative. YAML is only a possible human-editable representation.

### Layers And Provenance

Effective configuration resolves in this order:

```text
platform defaults
  <- app project
    <- installed recipe or profile
      <- device
        <- user
          <- session
```

Secrets are stored separately and appear as opaque references.

Every effective value must explain:

- Its current value
- Which layer supplied it
- The overridden lower-layer values
- Whether it differs from the app default
- What will reload if changed
- Which capabilities it requires

### Apply Modes

- `live`: send a validated config update to the running actor.
- `restart_actor`: restart only the owning actor.
- `reload_canvas`: reload the generated app.
- `rebuild_app`: generate and promote a new artifact.
- `restart_shell`: restart the outer shell; reserved for platform-level settings.

Use the lightest correct mode.

### Transaction

```text
proposed patch
  -> schema validation
    -> capability check
      -> actor dry-run
        -> apply to candidate effective config
          -> health check
            -> persist or rollback
```

The actor owns the meaning of its settings. `ConfigActor` owns layers, validation, provenance, migration, and transactional application.

### Config-First Agent Behavior

Before a broad source search, the Builder supplies the agent with:

- Actor graph
- Actor manifests
- Configuration catalog
- Effective values and provenance
- Current capability grants
- Relevant recent failures

For every accepted source change, the Builder performs a configurability review:

1. Could this behavior reasonably vary by user, device, provider, or environment?
2. Can it be expressed safely as structured data?
3. Does exposing it create a stable public contract worth maintaining?
4. Is a preset better than a raw low-level field?
5. What is the safe default and apply mode?

The answer may correctly be "keep this internal." Configurability is a product API, not a request to expose every implementation detail.

### Recipes

A recipe is a shareable, no-code configuration patch with:

- Target actor and protocol range
- Preconditions
- Typed patch
- Explanation
- Capability changes
- Validation or health check
- No embedded secrets

Community recipes can later graduate into:

- A documented preset
- A new default
- Automatic environment detection
- An official actor improvement
- A platform fix

This creates a low-friction public experiment space without making every edge case a permanent source fork.

## Security And Trust

### Trust Boundaries

1. Recovery bootstrap: minimal code capable of loading last-good state or replacing the Builder.
2. Stable shell: trusted UI, project identity, permission UI, config UI, preview control.
3. Shared Builder: trusted with approved projects and credentials.
4. Agent/build worker: untrusted or partially trusted task execution.
5. App runtime: untrusted generated UI with only granted capabilities.
6. Imported project and dependencies: untrusted until inspected and approved.

Builder self-modification is forbidden until the recovery bootstrap is independently updateable and cannot be overwritten by the Builder it restores.

### Two Permission Planes

Authoring capabilities include:

- Read or write project paths
- Run approved commands
- Use network hosts
- Install dependencies
- Run package scripts
- Read external files
- Modify Builder or shell source

Runtime capabilities include:

- App data storage
- Network hosts
- User-selected files
- Clipboard
- Notifications
- Camera or microphone
- Background work
- Native APIs

A permission request has:

- Capability name
- Scope or pattern
- Requesting actor
- Reason
- Duration
- Default policy: `allow`, `ask`, or `deny`

The app manifest requests capabilities. The local user or administrator grants them. Exports never copy the grants.

### Runtime Isolation

The web implementation must use:

- A dedicated per-app origin or equivalent storage partition
- A sandboxed cross-origin frame
- CSP response headers or equivalent custom-protocol headers
- A schema-validated `MessageChannel`
- Exact target origins during handshake
- No wildcard capability messages
- No raw same-origin `srcdoc` execution

If the frame needs persistent IndexedDB, it may require both scripts and same-origin semantics inside its dedicated origin. It must still remain cross-origin from the shell.

### Dependencies

Dependency installation can execute code. The first version should use a fixed template dependency set. Adding or changing dependencies requires:

- A visible diff
- Explicit authoring permission
- Lockfile update
- Build isolation
- Package-script policy
- A record in the task audit log

The system must not infer safety from a package being popular.

### Credentials

- LLM keys live only in the Builder credential store.
- Runtime service credentials use opaque credential references and scoped Builder or shell brokers.
- Secrets are redacted from task logs and diffs where possible.
- Export scans reject known secret-bearing files by default.
- Generated source must not receive a raw LLM key.

### Imported Apps

An imported app moves through explicit states:

```text
downloaded
  -> inspectable
    -> source trusted for build
      -> sandboxed runtime allowed
        -> optional capabilities granted
```

Viewing source does not imply permission to install dependencies or run it.

## State, Data, And History

| Concern | Owner | Included in app export by default |
| --- | --- | --- |
| Source files and project config | Git working tree | Yes |
| Source history | Git | Selected history |
| Active runtime | Content-addressed artifact store | Yes, last-good only |
| Prompt lifecycle and diagnostics | Builder task log | No |
| Effective local config | Shell ConfigActor store | No |
| Project config defaults | Git | Yes |
| App domain data | IndexedDB or later data adapter | No |
| Credentials | Builder credential store | No |
| Capability grants | Shell and Builder permission stores | No |

### Data Contract

- Source rollback never implicitly deletes app data.
- The app SDK gives storage a stable app ID and schema version.
- A code revision declares the data schema versions it can read.
- A migration is explicit, versioned, and tested against a recoverable snapshot.
- A failed migration blocks candidate promotion.
- A downgrade that cannot read the current data schema reports incompatibility instead of corrupting data.
- Backup and export are app capabilities, not Git operations.

LocalStorage may hold small preferences. IndexedDB is the initial structured data store. More capable local databases, filesystem access, sync, and cloud services remain roadmap adapters.

### Base App And Local Fork

Configuration-only customization does not create a source fork.

When source changes, provenance records:

- Base app identity and version
- Upstream source revision, if known
- User branch or local revision
- Builder task that introduced the divergence

A future base update can perform a three-way merge. The exact update UI and conflict policy are deferred, but early project metadata must not erase the distinction between upstream and local work.

Base provenance solves updates to a whole copied app, but reusable behavior should not require whole-app rebases. Vibe distinguishes templates, versioned modules, vendored source, and configuration recipes. Exact module locks allow an improvement to be offered across related apps while keeping adoption, state migration, configuration, bindings, grants, and rollback local to each app. The detailed contract is defined in [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md).

## Reference App: Project Notebook

The reference app is a local-first Project Notebook with projects, tasks, notes, attachments, reminders, import/export, and optional GitHub synchronization.

It is useful because its feature sequence crosses the important boundaries gradually:

- Ordinary UI and local data
- Data schema migration
- Files
- Search and derived state
- Notifications and background behavior
- Network and credentials
- Manual source editing
- Rollback without data loss
- Sharing without private data

### Actor Graph

```text
Svelte UI
  -> NotebookActor
       -> notebook data store
       -> search module
       -> ImportExportActor
       -> AttachmentActor -> file capability
       -> ReminderActor -> notification capability
       -> GitHubSyncActor -> network + credential broker
```

Projects, tasks, and notes begin inside one `NotebookActor` because they share a transactional domain model. Search begins as a pure module. Attachments, reminders, import/export, and GitHub sync justify actors through independent permissions, lifecycle, reuse, or failure behavior.

### Feature Sequence

1. Create projects, tasks, and notes in IndexedDB.
2. Add Kanban ordering and perform a schema migration.
3. Add attachments with file permission.
4. Add search, filters, import, and export.
5. Add reminders with schema-generated configuration.
6. Change reminder frequency through configuration, without a source edit.
7. Add GitHub import with network permission and a credential reference.
8. Make a manual source edit and build it through the same candidate path.
9. Undo the source change while keeping notebook data.
10. Export the app and verify that another user receives source and last-good behavior, but not data, credentials, grants, or transcripts.

### Configuration Scenario

The Project Notebook should demonstrate both a high-level control and an advanced escape hatch:

- High level: reminder frequency and quiet hours
- Advanced: notification delivery endpoint or provider-specific compatibility options

The advanced option must be typed and capability-checked. It must not accept an unrestricted shell command. The settings UI and Builder must both show the effective value and its source layer.

## Potential App Catalog

The reference app is not the only intended use. The following categories help test whether the architecture generalizes.

### Local-State Apps

- Habit tracker
- Reading log
- Personal CRM
- Recipe book
- Inventory tracker
- Study planner
- Daily journal
- Lightweight finance tracker

These mostly exercise UI, IndexedDB, import/export, and configuration.

### Rich Data And File Apps

- Photo organizer
- Markdown knowledge base
- Audio annotation tool
- Document comparison utility
- Local media catalog
- Dataset explorer

These add file permissions, background work, indexing, and larger storage.

### API-Connected Apps

- GitHub issue dashboard
- RSS reader
- Weather display
- Personal analytics dashboard
- Home automation panel
- Notification router

These add host allowlists, credentials, rate limits, offline behavior, and provider configuration.

### Native Or Background Apps

- Clipboard history
- Screenshot organizer
- Menu-bar timer
- Local transcription helper
- Folder watcher
- Notification scheduler

These require a native shell, runtime capabilities, background actors, and stronger isolation.

### Cloud And Collaborative Apps

- Shared task board
- Small team CRM
- Collaborative notes
- Form and workflow builder

These require identity, synchronization, conflict semantics, hosting, and server capabilities. They are roadmap validation cases, not first-version requirements.

## Implementation Plan

### Phase 0: Feasibility Spikes

These spikes are timeboxed and produce small checked-in findings or tests. They must not grow into alternate product implementations.

- [ ] Confirm a supported SvelteKit path that emits a self-contained runtime artifact suitable for frame loading.
- [ ] Confirm that readable source remains canonical and independent from the artifact.
- [ ] Compare authenticated loopback Builder transport with the browser's origin and installation constraints.
- [ ] Prototype a stable per-app origin or storage partition that preserves IndexedDB across artifact swaps.
- [ ] Mount one candidate project in `just-bash` and verify path containment, copy-on-write behavior, custom commands, cancellation, and default-denied network.
- [ ] Use `isomorphic-git` against the same candidate filesystem and verify that an external Git client recognizes the repository.
- [ ] Record a fallback for every rejected candidate.
- [ ] Decide the package manager and commit its lockfile before feature work.

Phase 0 exits when artifact, transport, origin, and project-filesystem choices have evidence. The product contracts remain fixed even if a candidate library is rejected.

### Phase 1: Deterministic Walking Skeleton

This is the first implementation milestone and must stay deterministic.

- [ ] Scaffold the TypeScript workspace with `apps/shell`, `apps/builder`, `packages/contracts`, `packages/app-sdk`, and `packages/app-template`.
- [ ] Define and validate the Builder envelope and the task events used by the Hello World flow.
- [ ] Create the empty SvelteKit app template with readable source and `vibe.json`.
- [ ] Implement a shell page with a persistent chat region and an isolated app canvas.
- [ ] Implement `BuilderConnectionActor` against an in-process adapter that uses the transport-neutral protocol.
- [ ] Implement `FakeAgentBackend`, mapping the exact Hello World request to a known source edit.
- [ ] Create an isolated candidate workspace for every task.
- [ ] Implement host-owned check and build commands.
- [ ] Store build outputs by content hash.
- [ ] Preload candidates and require a runtime handshake and health response.
- [ ] Implement the transaction journal and atomic `current.json` replacement.
- [ ] Keep the active frame mounted when a candidate build or health check fails.
- [ ] Reload from `current.json` with all agent and model adapters disabled.
- [ ] Add a deliberately broken fake-agent case.
- [ ] Add an end-to-end test for the full first-screen proof.

Phase 1 exit:

- "Hello World" appears without an outer-shell restart.
- It survives a cold launch with no model call.
- A broken candidate does not replace it.
- Source and artifact provenance are inspectable.

### Phase 2: Shared Builder And Source Ownership

- [ ] Move the Builder behind the selected authenticated local transport without changing the client protocol.
- [ ] Let one Builder open and identify two app projects.
- [ ] Add one installed-agent adapter for rapid prototyping.
- [ ] Add one embedded API-driven agent adapter using the user's Builder-side key.
- [ ] Persist provider credentials in the Builder credential store.
- [ ] Implement clarification, progress streaming, cancellation, and reconnection cursors.
- [ ] Add standard Git snapshots, accepted task revisions, diffs, log, and undo.
- [ ] Detect manual source edits and route their builds through the candidate pipeline.
- [ ] Add the Builder-absent pending-prompt flow.
- [ ] Ensure the shell bundle has no dependency on agent or provider packages.

Phase 2 exit:

- One shared Builder modifies two apps.
- Every accepted source task has a visible diff and checkpoint.
- A manual edit and chat edit use the same project history.
- Removing or stopping the Builder does not stop either app.

### Phase 3: Actor And Configuration Foundation

- [ ] Define the actor manifest, message schemas, lifecycle, health, and registry.
- [ ] Implement platform actors as typed modules with the specified ownership.
- [ ] Add supervision and the ability to disable a failing nonessential actor.
- [ ] Generate an actor graph for Builder context selection.
- [ ] Implement `ConfigActor` with schemas, validation, layers, provenance, and transactional application.
- [ ] Render a basic settings UI from the same schemas used by the Builder.
- [ ] Add live and `restart_actor` apply modes.
- [ ] Teach the agent path to inspect configuration before source.
- [ ] Build the first Project Notebook slice.
- [ ] Add reminders and demonstrate a config-only feature change.

Phase 3 exit:

- A user changes reminder behavior without an LLM or rebuild.
- The UI explains the effective value and layer.
- Invalid configuration rolls back.
- A failing optional actor can be disabled while the notebook remains usable.

### Phase 4: Data Safety And Capabilities

- [ ] Give every app a stable isolated origin or desktop partition.
- [ ] Implement the schema-validated runtime `MessageChannel` bridge.
- [ ] Implement separate authoring and runtime capability manifests.
- [ ] Add allow, ask, and deny policy with local grants.
- [ ] Implement preview data namespaces.
- [ ] Implement migration snapshot, dry-run, apply, health, and rollback.
- [ ] Add one real runtime capability, preferably notifications.
- [ ] Add one credential-brokered network integration to the reference app.
- [ ] Add untrusted import states and explicit build approval.

Phase 4 exit:

- Unauthorized file, network, and cross-app data access is denied.
- A migration failure preserves the active artifact and prior data.
- Grants remain local when the app is exported.

### Phase 5: Distribution And Desktop Shell

- [ ] Define and version the `.vibeapp` archive.
- [ ] Export source, selected Git history, manifest, schemas, and last-good artifact.
- [ ] Exclude data, keys, grants, device overrides, and transcripts.
- [ ] Import and inspect an app without running it.
- [ ] Implement the Builder install, connection, and prompt-resume UX.
- [ ] Package the shell with the selected desktop technology.
- [ ] Validate app storage, runtime bridge, recovery, and Builder independence in the packaged app.

Phase 5 exit:

- A recipient can run the last-good app immediately.
- The recipient can inspect and manually edit source.
- The recipient can connect their own Builder and credentials later.
- No sender data or secret is present.

## Edge Cases And Required Behavior

| Condition | Required behavior |
| --- | --- |
| Builder is missing | Run the app, preserve new prompts, and offer connection or installation. |
| Builder disconnects mid-task | Keep the candidate isolated, show reconnectable task state, and retain the active app. |
| Agent times out | Stop its tools, mark the task failed, and preserve diagnostic diff if safe. |
| Clarification is unanswered | Keep the task resumable; do not guess after a timeout. |
| Candidate does not compile | Report diagnostics and keep last-good active. |
| Candidate never sends health | Time out and reject it. |
| Shell crashes during promotion | Recover from the transaction journal and atomic current pointer. |
| Working tree changes during a task | Stop before promotion and ask to rebase, merge, or restart from the new base. |
| Manual changes are dirty | Snapshot them before the task; never discard them. |
| Git metadata is corrupt | Run last-good if possible, block source promotion, and offer recovery/export. |
| Artifact is missing | Rebuild from the recorded source revision if a trusted Builder is available; otherwise report recovery requirements. |
| Config is invalid | Reject it before persistence and explain the exact schema error. |
| Config requires a new capability | Ask before applying; denial leaves the prior value active. |
| Actor crashes repeatedly | Disable or quarantine it according to supervision policy and expose recovery controls. |
| Data migration fails | Restore the snapshot and reject the candidate. |
| Old code cannot read new data | Refuse that rollback unless a compatible data path exists. |
| Storage quota is exhausted | Preserve current state, stop writes, and offer export or cleanup. |
| Imported app requests build | Require trust and authoring permission before installing or executing dependencies. |
| Protocol versions are incompatible | Keep runtime behavior available and explain which Builder or shell must update. |
| Source and artifact identifiers differ | Trust `current.json` for runtime, flag the inconsistency, and block new promotion until reconciled. |

## Open Questions

### 1. What physically hosts the first non-native app?

Options:

- A browser UI plus a small local runtime host
- One Node process containing shell host and optional Builder modules
- Electron from the first milestone

Recommendation: keep the UI browser-compatible and use the smallest local host that can serve isolated origins and artifacts. Do not select Electron solely to avoid solving the logical separation.

Decision trigger: the Phase 0 origin and Builder transport spikes.

### 2. What transport connects the Builder Client to the local Builder?

Options:

- Authenticated loopback HTTP/WebSocket
- Browser extension mediation
- Native IPC
- Custom URL protocol plus a persistent channel

Recommendation: loopback HTTP/WebSocket for the web prototype if the security spike succeeds, with a transport adapter for native IPC later.

Decision trigger: proof that unrelated sites cannot discover, authenticate to, or select projects in the Builder.

### 3. How is the self-contained SvelteKit artifact produced?

Options:

- Supported SvelteKit inline bundle output
- Static adapter plus a controlled inlining step
- A small multi-file artifact served as one immutable directory

Recommendation: prefer the supported inline output. Accept a content-addressed directory if a single file adds fragility or breaks runtime behavior.

Decision trigger: the Phase 0 build spike.

### 4. Is YAML the human configuration format?

Options:

- YAML
- JSON with comments
- TOML
- Generated UI plus internal canonical JSON

Recommendation: use a schema independent of syntax. Choose YAML only after verifying lossless comment and order preservation.

Decision trigger: round-trip editing, diff readability, validation quality, and implementation complexity.

### 5. How are accepted source changes promoted?

Options:

- Auto-promote every valid candidate
- Require review for every source change
- Auto-promote reversible changes, ask for risk elevation

Recommendation: auto-promote reversible changes with no new permissions, dependencies, or data migration. Ask when risk increases and offer a global review mode.

Decision trigger: usability testing and the first real agent backend.

### 6. How are dirty manual edits represented?

Options:

- Require a clean working tree
- Automatically create a normal commit
- Create an internal Git snapshot/ref without advancing the user's branch

Recommendation: create a disclosed internal snapshot/ref, then base the task on that exact tree.

Decision trigger: the Git filesystem spike and recovery testing with an ordinary external Git client.

### 7. How much Git history travels in an app package?

Options:

- Full repository
- Shallow history
- Selected refs in a compact Git bundle
- Source snapshot only

Recommendation: selected refs in a standard compact Git bundle. Keep the format compatible with ordinary Git.

Decision trigger: real app package size and update/merge requirements.

### 8. What is the first real agent backend?

Options:

- Invoke an installed agent
- Embed one provider directly
- Run an agent in a browser container

Recommendation: use an installed-agent adapter to test the boundary quickly, then ship one embedded BYOK provider so end users need no separate agent installation.

Decision trigger: Phase 2 integration effort, distribution size, and provider API requirements.

### 9. Electron or Tauri for the first desktop package?

Options:

- Electron
- Tauri
- Another webview shell

Recommendation: decide after the web-compatible runtime and permission bridge exist. Favor the option that can reuse the JavaScript build toolchain while maintaining per-app isolation and a small immutable recovery path.

Decision trigger: packaging spike measuring update, process isolation, storage partition, build-tool, and recovery needs.

### 10. When may the Builder change itself?

Options:

- Never
- Only through signed external updates
- Through an explicit self-edit mode

Recommendation: defer self-edit mode. It requires an immutable or independently signed bootstrap that can restore the Builder and shell.

Decision trigger: recovery bootstrap, permission UX, signed update path, and reliable rollback all exist.

### 11. How are base app updates merged into a local fork?

Options:

- Standard Git three-way merge
- Reapply Builder tasks semantically
- Rebase configuration recipes and source commits separately

Recommendation: preserve Git provenance and config layers now. Choose the user-facing merge strategy after two real versioned apps exist.

Decision trigger: the first base update against an app with both config-only and source-level customization.

### 12. How isolated must actors be by default?

Options:

- Every actor in a Worker
- In-process unless risk justifies isolation
- Placement declared entirely by each app

Recommendation: in-process by default, with host policy able to strengthen placement. Untrusted actor packages may require a stronger default later.

Decision trigger: measured failure modes, performance, and third-party actor packaging.

## Success Criteria

### Foundation Acceptance

- [ ] A skeleton app launches and renders its current artifact without an LLM call.
- [ ] Entering "Add Hello World" creates a readable source diff.
- [ ] The candidate builds in an isolated workspace.
- [ ] "Hello World" appears without restarting the outer shell or losing the chat.
- [ ] A cold launch with the Builder and model adapters disabled still shows "Hello World."
- [ ] A broken candidate leaves the prior app visible and runnable.
- [ ] `current.json` names the exact source revision and artifact hash.
- [ ] Manual source files remain readable and buildable with ordinary tools.
- [ ] No LLM key appears in source, artifact, runtime storage, or an exported package.
- [ ] The co-located walking skeleton uses the same message schemas expected by an external Builder.

### Source Ownership Acceptance

- [ ] One shared Builder opens two apps without duplicating credentials or model adapters.
- [ ] Every accepted source task has a task ID, diff, source checkpoint, and artifact.
- [ ] A manual edit and agent edit share one Git history.
- [ ] Undo changes code without deleting app data.
- [ ] Builder absence does not prevent normal app use.
- [ ] A prompt entered while disconnected can be resumed after explicit reconnection.

### Actor And Configuration Acceptance

- [ ] The Builder can list actors, messages, configuration schemas, and capability requests.
- [ ] A valid configuration change applies without a source edit.
- [ ] The settings UI and Builder use the same schema.
- [ ] Effective configuration reports value provenance.
- [ ] An invalid config patch does not alter the running app.
- [ ] A failing optional actor can be disabled while unrelated behavior continues.
- [ ] The Builder searches relevant actor modules before broad project source.

### Data And Security Acceptance

- [ ] App data persists across artifact promotion and source rollback.
- [ ] Candidate health checks do not mutate active data.
- [ ] A failed migration restores the prior snapshot.
- [ ] App runtimes cannot access another app's data.
- [ ] Authoring and runtime grants are separate and local.
- [ ] Network, files, credentials, and native capabilities are unavailable without grants.
- [ ] Export contains source and last-good behavior but excludes data, keys, grants, device overrides, and task transcripts.

## Deferred Roadmap

The following work is compatible with the architecture but does not belong to the initial implementation:

- Builder self-edit mode with immutable recovery
- Configuration recipe sharing and marketplace
- Signed actor packages and complete apps
- More native capabilities
- `scriptc` as a native bootstrap or helper
- Browser-contained build backend such as WebContainers
- Remote Builders
- Hosted deployment
- Cloud databases and sync adapters
- Collaboration and CRDTs
- Cross-app capability brokerage
- Multiple frontend frameworks
- Mobile shells
- Automatic recipe promotion based on community fixes
- Base-app update and merge UX

## References

- `/Volumes/p/RIFT/rift-transcription/reference/epicenter-plugin-architecture-feasibility.md` - Related study of plugin boundaries, capabilities, actor isolation, and provenance.
- https://svelte.dev/blog/advent-of-svelte#Day-22:-self-contained-apps - Self-contained Svelte app direction to validate in Phase 0.
- https://github.com/vercel-labs/just-bash - Candidate virtual command and filesystem facade.
- https://github.com/vercel-labs/scriptc - Candidate future native TypeScript bootstrap/helper.
- https://isomorphic-git.org/ - Candidate JavaScript Git implementation using standard Git data.
- https://webcontainers.io/ - Possible later browser-contained build backend.
- VS Code extension host and Workspace Trust documentation - Model for separating UI, tools, permissions, and untrusted workspaces.
- Browser extension manifest and permission documentation - Model for declared requests and local grants.
- Electron and Tauri security documentation - Inputs to the desktop shell decision.

## Revisit Rules

This specification should be revised when:

- A Phase 0 spike rejects a provisional technology choice.
- An implemented contract needs a semantic change.
- A roadmap item moves into an active milestone.
- A second real app demonstrates that the Project Notebook biased the abstraction incorrectly.
- A security review finds that a stated trust boundary cannot enforce its responsibility.

Small implementation details do not require a spec revision unless they change ownership, observable behavior, compatibility, security, or the proof of completion.
