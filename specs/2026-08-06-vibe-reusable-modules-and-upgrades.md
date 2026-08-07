# Vibe Reusable Modules and Upgrades

**Date:** 2026-08-06  
**Status:** Proposed architecture - package resolver experiment pending  
**Related:** [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md), [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md), [Building Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md)

## One Sentence

Vibe separates copied app source from versioned reusable modules, so an improvement can be offered safely to related apps without silently rewriting their code, state, configuration, or grants.

## First-Screen Contract

Cloudflare OS Blueprints are good distribution snapshots but weak upgrade channels. Publishing a Blueprint captures committed Gadget source, binding requirements, and metadata. Instantiating it copies that source into a new independent Gadget. The Cloudflare OS Blueprint documentation explicitly says that existing instances do not receive automatic updates from their Blueprint.

That independence is desirable for ownership and safety, but it means a useful change made in one Gadget does not naturally reach its Blueprint, its source Gadget, or sibling instances. A substantial renderer, importer, editor command, accessibility improvement, or security fix may need to be ported repeatedly.

Vibe should preserve independent apps while adding a separate dependency relationship:

```text
Blueprint or template
  -> creates an independent app

Versioned module
  -> remains an explicit dependency
    -> can advertise a compatible update
      -> updates through preview, validation, migration, and rollback
```

The first slice is complete when two apps use the same pinned module, one module release adds a visible feature, both apps can preview and adopt it independently, and declining or rolling back the update leaves each app working.

## Reading Guide

The active design is:

1. Keep templates and Blueprints copy-based.
2. Put behavior worth maintaining across apps in versioned modules.
3. Pin exact module versions in an app lock.
4. Keep app state, configuration, bindings, and grants instance-local.
5. Apply upgrades as source transactions with validation and rollback.
6. Retain upstream provenance for source that must be vendored and edited.

The registry, marketplace, and public publishing policy are later product layers. The package identity, provenance, and upgrade transaction are foundation requirements.

## Research Finding: Cloudflare OS Blueprints Are Snapshots

At the reviewed Cloudflare OS revision:

- A Blueprint captures a clean snapshot of committed Gadget source.
- Blueprint code is stored by Blueprint ID and version.
- Updating a published Blueprint creates a new source snapshot.
- Instantiation copies that snapshot into a new independent Gadget.
- The new Gadget has its own storage, chat history, and bindings.
- Existing instances have no automatic update path from the Blueprint.
- The documentation notes that Yjs could support such a mechanism later.

Yjs can transport or merge text changes, but text synchronization alone does not answer:

- Which upstream version an app currently uses
- Whether a change is compatible
- Which files are locally owned
- Whether app data needs migration
- Whether configuration keys changed
- Whether new authority is requested
- How to test and roll back the update
- Whether the user intended a copy or an ongoing dependency

Vibe therefore needs a package and provenance model above any Yjs or Git merge mechanism.

## Separate Four Kinds of Reuse

### Template

A template creates an independent starting point.

Use it when:

- The recipient is expected to own and reshape the whole app.
- Future upstream changes are not inherently applicable.
- A self-contained copy is more valuable than an upgrade relationship.

Cloudflare OS Blueprints map naturally to this mode.

### Versioned module

A module remains a declared dependency with an immutable version and API.

Use it for:

- Editors and renderers
- Importers and exporters
- Authentication adapters
- Search, indexing, and synchronization
- Reusable UI components
- Vibe actors
- Security-sensitive shared behavior

The app may configure or wrap a module without copying its implementation into app-owned source.

### Vendored source

Vendored source is copied into the app because the user needs to edit it directly or because the runtime cannot resolve the module natively.

Vendoring must retain:

- Package identity
- Exact upstream version or content digest
- The unmodified base snapshot
- The local changes relative to that base
- License and attribution metadata

This permits a later three-way merge. A plain copy with no base identity is an intentional fork, not an upgradeable dependency.

### Configuration or recipe

A configuration value changes behavior without changing implementation. A recipe is a portable, reviewable configuration patch or high-level transformation.

Use these before source changes when the module already exposes the right control point. Examples include:

- Enabling SVG elements
- Adding strikethrough to an allowed Markdown extension set
- Selecting a parser mode
- Changing a toolbar layout
- Rebinding a capability

A recipe may declare a compatible module range. It must not claim to be a general upgrade mechanism when it rewrites arbitrary source.

## Target Architecture

```text
Vibe app
  |
  +-> app-owned source
  +-> app manifest
  +-> exact dependency lock
  +-> local configuration overlays
  +-> instance state and grants
  |
  +-> versioned actor/module packages
        |
        +-> readable source
        +-> typed API
        +-> config schema
        +-> capability requirements
        +-> data migration hooks
        +-> compatibility metadata

Vibe runtime
  -> resolves locked package digests
  -> reuses cached code and build artifacts
  -> creates separate state and authority per app
```

Sharing code does not mean sharing state or credentials. Two apps can execute the same module release while receiving separate actor instances, storage namespaces, configuration overlays, bindings, and grants.

## Composition Roots and Module Categories

### Composition root, not inheritance parent

An app should have an explicit composition root that selects modules, connects their APIs, supplies defaults, and owns app-specific glue. This fills the useful role of a "parent module" without creating implicit source inheritance.

```text
slides app composition root
  |
  +-> slides core
  +-> Markdown pipeline
  |     +-> strikethrough extension
  |
  +-> element renderer registry
  |     +-> SVG renderer
  |
  +-> theme
  +-> export actor
```

A composition root may consume other composition modules. An app may also have separate roots for distinct runtime surfaces, such as client UI, server behavior, and background work. The dependency graph, rather than a privileged universal parent, determines the relationship.

The composition root owns:

- Module selection and compatible version ranges
- Typed extension points
- Wiring between module APIs
- Default configuration
- App-specific layout and orchestration

It does not own module implementation, module-private state, credentials, or grants.

Prefer extension-point collections over subclassing:

```ts
export interface SlidesComposition {
  markdownExtensions: readonly MarkdownExtension[];
  elementRenderers: readonly ElementRenderer[];
  commands: readonly Command[];
  actors: readonly ActorBinding[];
}
```

This lets SVG, strikethrough, tables, diagrams, accessibility checks, and other features arrive as contributions to stable interfaces. A parent source file does not need to absorb every implementation.

### Code modules

Code modules run within an existing app or actor process. They are appropriate for parsers, renderers, utilities, UI components, and deterministic transformations.

They share the containing process's failure and authority boundary. They should not receive ambient bindings merely because they were imported.

### Actor modules

Actor modules own a typed API, lifecycle, and state boundary. They are appropriate for search, synchronization, import/export, background work, and capability-bearing integrations.

Apps may share the same actor package version, but each app receives a separate actor instance by default:

```text
shared actor implementation
  != shared actor state
  != shared configuration
  != shared bindings
  != shared grants
```

An intentionally shared actor service is a different product relationship and requires an explicit shared identity and authorization model.

### Presets and configuration recipes

Presets select modules and provide supported configuration without adding executable implementation of their own. They are appropriate for Markdown feature sets, toolbar layouts, themes, and compatibility profiles.

For example, strikethrough may be a configuration value in an existing Markdown module, a small code extension, or part of a named preset. The Builder should choose the smallest representation that matches the behavior.

## App Manifest and Lock

The authored manifest declares intent:

```yaml
schemaVersion: 1
app:
  id: com.example.deck
  sourceVersion: 7

composition:
  root: "@vibe/slides-app"

dependencies:
  "@vibe/slides-app":
    range: "^2.4.0"
  "@vibe/svg-elements":
    range: "^1.1.0"

recipes:
  - package: "@vibe/markdown-preset"
    range: "^3.0.0"
    config:
      strikethrough: true
```

The lock records the exact resolved graph:

```yaml
lockVersion: 1
packages:
  "@vibe/slides-app@2.4.3":
    digest: "sha256:..."
    source: "https://packages.example/@vibe/slides-app/2.4.3"
    apiVersion: 2
    permissionsDigest: "sha256:..."
  "@vibe/svg-elements@1.1.0":
    digest: "sha256:..."
    source: "https://packages.example/@vibe/svg-elements/1.1.0"
    apiVersion: 1
    permissionsDigest: "sha256:..."
```

Version ranges describe acceptable upgrades. The lock determines what runs. Mutable tags such as `latest` may help discovery but must never be runtime identity.

## Module Contract

A reusable module should publish:

- Stable package identity
- Immutable version and content digest
- Readable source
- Typed public API
- Supported Vibe runtime range
- Configuration schema and defaults
- Capability requests
- State schema version
- Upgrade and downgrade compatibility
- Migration hooks or migration instructions
- Test or validation entrypoints
- License and provenance

An actor module additionally declares:

- Actor protocol version
- Message or RPC surface
- Failure and restart policy
- Whether it supports hot replacement
- Which state belongs to the actor

Internal files are not an API. The Builder should avoid modifying package internals unless the user explicitly vendors or forks the module.

## Upgrade Discovery

An update source may be:

- A package registry
- A local workspace package
- A Builder-created candidate release
- A trusted peer or organization catalog
- A security advisory
- An upstream Git revision for vendored source

The runtime never executes a newly discovered version automatically. Discovery produces an update candidate containing:

- Current and proposed identities
- Source and generated-artifact diff
- API and schema compatibility report
- Capability and grant delta
- Configuration changes
- Required data migrations
- Affected apps
- Publisher and signature information

One candidate may affect many apps, but each app retains its own adoption state.

## Upgrade Transaction

```text
discover candidate
  -> verify identity and signature
    -> resolve complete dependency graph
      -> compare APIs, config, capabilities, and data schemas
        -> build in isolation
          -> migrate a recoverable state snapshot
            -> run module and app checks
              -> preview affected apps
                -> request approval
                  -> atomically promote lock and migrations
                    -> retain previous lock and state snapshot
```

Failure at any point leaves the current release active.

An upgrade that adds a capability request is not a routine code update. It requires a new grant decision. An update must not inherit broader authority merely because an earlier version was trusted.

## Local Changes and Three-Way Updates

If a user changes app-owned integration code, an ordinary dependency upgrade should not overwrite it.

If a user changes vendored module source, Vibe records:

```text
base package version B
  + local patch L
  + proposed upstream version U
    -> three-way merge candidate
```

The Builder may resolve conflicts, but it must show which changes came from upstream, the user, and the current task. The result becomes a new local revision and remains divergent until explicitly published or returned upstream.

For heavily modified packages, the Builder should offer:

- Keep the current fork
- Rebase onto the new upstream version
- Replace with upstream and discard the local patch
- Extract the local behavior into an extension module
- Publish the fork under a new package identity

## Cross-App Propagation

Vibe should support an inventory query:

```text
which apps use @vibe/slides?
which versions do they use?
which have local patches?
which would need new grants or migrations?
```

The Builder can then prepare a batch proposal. Batch does not mean all-or-nothing:

- Compatible untouched apps may share one validated build result.
- Each app gets its own preview and state migration.
- Locally modified apps get a merge proposal.
- Incompatible apps remain pinned.
- Users may adopt now, defer, or decline.

Organizations may define an auto-update policy for reviewed patch releases, but Vibe's base product should default to notification and explicit promotion. Security updates may use stronger policy, with a visible audit trail and rollback.

## Example: SVG and Strikethrough

Suppose one slide Gadget gains SVG elements and Markdown strikethrough.

If the feature is implemented directly in copied Gadget files:

```text
slide instance A changed
  -> Blueprint may be manually republished
    -> future instances receive the change
    -> existing parent and sibling instances remain unchanged
```

In the Vibe model:

1. Determine whether the behavior belongs in the slide renderer, a Markdown module, or configuration.
2. Add SVG support to a versioned renderer module.
3. Expose strikethrough as a typed Markdown option if it is policy rather than a new implementation.
4. Release the module or preset with compatibility metadata.
5. Find apps whose lock references the compatible package line.
6. Preview and promote the update independently for each app.

If a sibling uses a different renderer, the Builder may port the idea, but it should not pretend the same package update applies.

## Parent and Sibling Apps

Visual or conversational parentage does not imply source inheritance. A parent app and child Gadget may have different responsibilities, APIs, and trust boundaries.

The reusable "parent" is normally the composition root package. Sibling apps can use the same composition root, different versions of it, or different roots that share lower-level modules. Updating a leaf module can therefore reach every compatible sibling without requiring the whole root to change. Updating the root is appropriate when wiring, defaults, or extension contracts change.

Reusable behavior crosses those boundaries only through:

- A shared composition root
- A common versioned module
- A declared actor API
- A configuration recipe supported by both
- An explicit source-porting task

This avoids hidden inheritance where changing a parent unexpectedly changes every child. The relationship graph remains inspectable.

## Cloudflare OS Adapter

### Current substrate capability

At the reviewed Cloudflare OS revision, the Gadget server loader already collects every Gadget `.js` file into a Worker Loader module map and designates `server.js` as the main module. Server-side Gadget source can therefore be organized as multiple local modules.

The client path is less modular. `getUiBundle()` currently returns only the raw contents of `client.js`, and the source marks bundling as future work. The `UiBundle` contract anticipates a content-addressed implementation shared across Gadgets and optional support-library version metadata, but does not yet define or resolve a versioned package graph.

Blueprint snapshots copy the Gadget file collection. Neither multiple local files nor Yjs source roots preserve an upgrade relationship after instantiation.

A Cloudflare OS Blueprint remains useful as:

- An installation snapshot
- A hosted import/export payload
- A source and binding-requirement carrier
- A deployment-provided starting format

It is not the authoritative dependency graph.

For a Cloudflare-hosted Vibe app:

1. Vibe stores the manifest, lock, provenance, and package source graph as Vibe metadata.
2. The Cloudflare adapter resolves and builds that graph.
3. The adapter preserves server modules where the Worker Loader supports them and bundles client modules into a self-contained `client.js`.
4. If other Gadget constraints require flattened source, the adapter emits derived files without changing authored package ownership.
5. The derived snapshot records the source graph and build identity that produced it.
6. Builder edits apply to authored app or package source, not silently to flattened generated files.
7. Blueprint export may carry the resolved Gadget for compatibility, while `.vibeapp` export carries the authoritative graph.

This resembles a lockfile plus a bundled deployment artifact. The artifact is self-contained at runtime without erasing where its modules came from.

Initial Vibe versions should resolve modules at build time. Runtime loading from a mutable package registry would weaken reproducibility, offline behavior, and authority review. Content-addressed runtime caching may reuse already-resolved packages and artifacts without changing the exact app lock.

## Portable Export

A `.vibeapp` export should remain usable if a registry disappears. It may include:

- App-owned readable source
- Manifest and exact lock
- Required package source snapshots
- Package licenses and provenance
- Configuration schemas and packageable defaults
- Migration code
- Optional prebuilt artifacts

It excludes credentials, grants, private state, and Builder chat unless the user selects a separate data export.

Import verifies every included package digest. A package snapshot in the export satisfies the lock without turning the package into app-owned source.

## Security and Supply Chain

Reusable updates increase blast radius. Required controls include:

- Immutable package versions
- Content digests
- Publisher signatures or equivalent authenticated provenance
- Dependency graph review
- Capability-delta review
- Build isolation
- Package-script policy
- Revocation and advisory metadata
- Last-good lock and state recovery
- Separate trust for source, build artifact, and publisher

One compromised shared module must not gain every app's credentials. App-specific capability injection and grants remain the authority boundary.

## Agent Policy

When asked to add or fix behavior, the Builder should search in this order:

```text
existing configuration
  -> compatible recipe
    -> dependency update
      -> extension module
        -> app-owned integration code
          -> vendored package fork
```

This is a preference, not a ban on source editing. The Builder should explain when it chooses a local fork because that choice changes future update cost.

When an improvement appears reusable, the Builder may propose extracting it into a module. It must not publish code or update other apps without authorization.

## Design Decisions

| Decision | Class | Choice | Rationale |
| --- | --- | --- | --- |
| Blueprint semantics | Evidence | Independent installation snapshot | Matches Cloudflare OS and preserves recipient ownership |
| Reusable behavior | Design coherence | Versioned module dependency | Keeps an upgrade relationship without shared mutable app source |
| App assembly | Design coherence | Explicit composition root and typed extension points | Supports parent-like reuse without hidden source inheritance |
| Module execution shapes | Design coherence | Code module, actor module, or preset | Matches behavior to the smallest useful isolation and lifecycle boundary |
| Runtime identity | Security | Exact lock and content digest | Ranges and mutable tags are insufficient execution identities |
| Module state | Design coherence | Per-app by default | Shared code must not imply shared private data |
| Module authority | Security | Per-app capability injection and grants | Package trust does not confer ambient credentials |
| Routine updates | Taste under constraints | Notify, preview, validate, then promote | Avoids silent breakage while keeping updates practical |
| Local package edits | Design coherence | Vendored source plus base provenance | Enables three-way updates and honest fork status |
| Cloudflare representation | Design coherence | Derived flattened Gadget plus authoritative Vibe graph | Preserves Cloudflare compatibility without losing dependencies |
| Initial resolution time | Product contract | Resolve and bundle at build time | Keeps accepted apps deterministic, self-contained, and offline-capable |
| Offline export | Product contract | Include locked readable package sources | A registry outage must not brick an accepted app |
| Shared runtime cache | Optimization | Reuse identical package artifacts | Saves space without changing per-app ownership |

## Implementation Plan

### Phase 0: Package identity spike

- [ ] Define manifest, lock, package, provenance, and capability-delta schemas.
- [ ] Define a composition-root contract and typed extension-point collections.
- [ ] Build two trivial apps against one local code module.
- [ ] Add one actor module with separate state for each app.
- [ ] Add one configuration preset that selects a supported feature.
- [ ] Resolve one exact digest from a local package directory.
- [ ] Keep state and configuration separate for each app.
- [ ] Export and import both apps without network access.

Exit:

- The same module source is not duplicated into app-owned source, and both apps run from an exact lock.

### Phase 1: Safe module update

- [ ] Publish a second local module version.
- [ ] Discover affected apps.
- [ ] Produce source, API, config, and capability diffs.
- [ ] Build and preview each candidate.
- [ ] Promote one app while leaving the other pinned.
- [ ] Roll the promoted app back.

Exit:

- One release can propagate as an offer without becoming a forced global mutation.

### Phase 2: State migration and local patches

- [ ] Add a versioned actor state migration.
- [ ] Validate it against a recoverable snapshot.
- [ ] Record a local vendored patch.
- [ ] Produce a three-way merge against a new upstream version.
- [ ] Preserve provenance through promotion and rollback.

Exit:

- A substantial update preserves app data and distinguishes upstream work from local divergence.

### Phase 3: Cloudflare adapter

- [ ] Map an authoritative Vibe graph to a runnable Gadget snapshot.
- [ ] Preserve multiple server modules through the current Worker Loader module map.
- [ ] Bundle multiple client modules into the current single `client.js` delivery shape.
- [ ] Record the graph and build identity beside the Gadget revision.
- [ ] Export a compatibility Blueprint and a full `.vibeapp`.
- [ ] Rebuild after a module update without editing derived files.
- [ ] Test the SVG and strikethrough example across two apps.

Exit:

- Cloudflare-hosted execution remains self-contained while Vibe retains upgradeable module identity.

## Open Questions

### What is the first package transport?

Options:

- Local filesystem packages
- Git repositories and pinned commits
- A Vibe registry
- OCI artifacts

Recommendation: start with local packages and content digests. Add Git as a source before designing a public registry.

### How much semantic versioning should Vibe trust?

Recommendation: use version ranges for discovery, but derive compatibility from declared API, state, configuration, and capability metadata plus actual validation. A version number is a claim, not proof.

### Can packages depend on packages?

Recommendation: yes, with a complete exact lock, cycle rejection, bounded graph size, and one visible explanation of version conflicts. Avoid hidden runtime resolution.

### When may updates be automatic?

Recommendation: defer automatic promotion until signatures, rollback, migrations, capability deltas, and organization policy are proven. Caching and update discovery may be automatic from the start.

### Should a package expose source editing by default?

Recommendation: source should be readable by default. Editing switches the dependency to a workspace override or vendored fork so the ownership change is explicit.

## Success Criteria

- [ ] Two apps can use one exact module release without sharing state, config, bindings, or grants.
- [ ] Two apps can share a composition root while independently pinning and upgrading compatible leaf modules.
- [ ] Code modules, actor modules, and presets have distinct observable ownership and lifecycle behavior.
- [ ] A module update can be discovered once and adopted independently by both apps.
- [ ] An app may remain pinned without losing ordinary operation.
- [ ] A capability increase requires a new grant decision.
- [ ] A failed build or migration leaves the last-good lock and state active.
- [ ] A local module modification retains base provenance and supports a three-way update.
- [ ] A `.vibeapp` export runs offline with its locked readable package sources.
- [ ] A Cloudflare Blueprint remains a valid snapshot even though it is not the dependency authority.
- [ ] The Builder can identify every app affected by a module release.
- [ ] Derived flattened Gadget source is distinguishable from authored source.

## References

- [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md)
- [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md)
- [Building Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md)
- [Cloudflare OS Blueprint design at the reviewed revision](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/docs/blueprints.md)
- [Cloudflare OS Blueprint archive implementation](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-backend/src/blueprint-archive.ts)
- [Cloudflare OS Blueprint snapshot and initialization implementation](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-backend/src/overseer.ts)
- [Cloudflare OS Gadget server module construction](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-backend/src/overseer.ts#L2261-L2307)
- [Cloudflare OS current client bundle path](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-backend/src/overseer.ts#L9017-L9040)
- [Cloudflare OS `UiBundle` direction](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/packages/workshop-shared/src/api.ts#L1139-L1160)
