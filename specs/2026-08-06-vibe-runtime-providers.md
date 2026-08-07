# Vibe Runtime Providers and Placement

**Date:** 2026-08-06  
**Status:** Proposed architecture - provider implementation pending  
**Refines:** Runtime-placement portions of [Building Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md)  
**Related:** [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md), [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md)

## One Sentence

Vibe treats Cloudflare OS execution as a versioned runtime service, so Cloudflare-hosted, self-hosted, Vibe-managed local, user-managed local, and bundled local runtimes can sit behind one provider contract, while ordinary apps share a runtime only within the same user profile, runtime release, and trust domain.

## First-Screen Contract

This document defines where Vibe apps run and how the shell connects to that execution environment.

The current Vibe foundation describes a local-first-capable app shell, an optional Builder, modular actors, readable source, configuration, bindings, and last-good recovery. The first Cloudflare OS substrate proposal assumes a hosted first edition while preserving a later local runner. Subsequent research established two additional facts:

- Cloudflare OS can run locally on `workerd`, not only on Cloudflare's hosted Workers service.
- An Electrobun shell can supervise a long-running local server, although product-grade packaging and isolation of a bundled `workerd` still require proof.

The target shape separates the Vibe product from runtime placement:

```text
Vibe shell and Builder
  -> Vibe runtime-provider protocol
    -> one selected runtime profile
       |
       +-> Cloudflare-hosted Cloudflare OS
       +-> remote self-hosted Cloudflare OS
       +-> Vibe-managed local workerd
       +-> user-managed local workerd
       +-> bundled local workerd
```

This architecture is complete when:

1. The same portable Hello World app can be installed on one remote provider and one local provider without app-source changes.
2. Opening the accepted app invokes no LLM.
3. Provider location and offline behavior are visible to the user.
4. Two ordinary apps in one profile share a local runtime, while an untrusted app can be assigned a separate runtime.
5. Local runtime installation requires no manual developer tooling in managed mode.
6. An incompatible or untrusted runtime is rejected before app code runs.
7. Moving an app between providers is explicit and does not silently copy credentials, grants, private data, or Builder history.
8. A remote app can request an approved device capability without receiving general access to the local shell.

The provider abstraction is a design decision. The default provider for Vibe's first public edition remains an experiment decision.

## Reading Guide

- Product and architecture contracts: Product Contract through Runtime Grouping
- Local runtime distribution: Managed Local Runtime through Runtime Updates
- Remote and hybrid behavior: Remote Providers through Device Capability Bridge
- Execution path: Implementation Plan and Success Criteria
- Background evidence: Research Findings and References

## Motivation

Treating "Cloudflare OS on Vibe" as a single deployment choice creates unnecessary coupling:

- A desktop shell appears to require bundling and signing `workerd` before the product can be tested.
- A hosted deployment appears to rule out local ownership and offline execution.
- Each Vibe app appears to need its own runtime, duplicating the Builder and kernel.
- Switching from Cloudflare-hosted to self-hosted or local execution appears to require an app rewrite.

Those conclusions follow only if the shell, runtime, app, and provider are treated as one unit.

The more useful model is:

```text
shell       = window, lifecycle, local secrets, native capability broker
Builder     = chat, planning, source changes, preview, acceptance
provider    = a configured place where Vibe runtime services are available
runtime     = Cloudflare OS kernel plus its execution and storage primitives
app         = a Gadget instance plus Vibe contracts and metadata
package     = portable app source and declared requirements
```

The shell can remain local while execution is local or remote. The Builder can remain logically external while appearing inside the shell. An app package can remain portable while its live state belongs to one selected provider.

## Scope

This specification defines:

- Runtime provider kinds and user-facing profiles
- The provider handshake and feature negotiation
- Local runtime discovery, installation, lifecycle, and updates
- The default sharing boundary for multiple apps
- Trust-domain isolation and separate-runtime rules
- Remote and hybrid execution
- The local device capability bridge
- Provider-visible source, data, and credential ownership
- Explicit app migration between providers
- Failure states and observable acceptance criteria

It does not:

- Select Electrobun, Tauri, Electron, or another final shell
- Freeze the complete transport or RPC schema
- Implement production Cloudflare hosting
- Implement a hardened sandbox for arbitrary hostile code
- Guarantee transparent offline execution for remote apps
- Design multi-region Durable Object replication
- Define billing, quotas, or marketplace policy
- Make arbitrary Cloudflare OS Gadgets portable to every provider
- Change the Vibe foundation in this revision

## Terminology

| Term | Meaning |
| --- | --- |
| Runtime provider | A configured service that implements the Vibe runtime-provider protocol |
| Runtime profile | User-visible configuration selecting a provider, identity, and policy |
| Runtime instance | One running local process or one remote service allocation |
| Runtime group | Apps intentionally sharing one runtime instance and failure boundary |
| Runtime release | A tested combination of `workerd`, Cloudflare OS, adapters, configuration, and protocol version |
| Placement | Whether an app, actor, binding, or state store runs locally or remotely |
| Trust domain | Apps allowed to share a host-level failure and exploit boundary |
| Device broker | Trusted shell component that mediates local OS capabilities for local or remote apps |
| Managed local | Vibe downloads, verifies, installs, and operates the runtime |
| System local | The user or administrator supplies an installed runtime |
| Bundled local | A signed runtime ships inside the Vibe application bundle |
| Remote self-hosted | A user or organization operates Cloudflare OS on its own server |
| Cloudflare-hosted | Cloudflare OS is deployed to Cloudflare Workers services |

## Product Contract

### Placement is visible

Every open app shows its current runtime profile in an inspectable location:

```text
Running on This Device
Running on Vibe Cloud
Running on homeserver.example
```

The shell must not imply that a remote app is local or offline-capable.

### Apps do not select authority silently

An app package declares required features and bindings. It does not choose a provider, upload itself, or acquire a grant without the user's installation decision.

### Runtime use does not imply Builder use

Opening an accepted app connects to its runtime provider but does not invoke an LLM. A Builder session is separate and may use a hosted model, a local model, or no model.

### Provider changes are explicit

Moving an app creates or selects a new installation. It does not silently fail over, copy grants, or reinterpret provider-specific data.

### User secrets stay out of app packages

Model keys, provider credentials, runtime tokens, binding grants, and device-broker credentials belong to the shell, Builder, or provider profile. They are never compiled into the app or exported with a package.

### Provider-specific features are declared

An app that needs a local filesystem, remote background execution, live collaboration, or another non-universal feature declares it. The shell reports incompatibility before installation.

## Provider Catalog

### Cloudflare-hosted

Cloudflare OS is deployed to a Cloudflare account and uses hosted Workers, Durable Objects, Dynamic Workers, storage, and related services.

Best for:

- Zero local runtime setup
- Multi-device access
- Live collaboration
- Hosted background work
- Cloudflare's production defense in depth

Costs:

- Requires network access
- Places execution and live state on a remote provider
- Requires an account, identity, and operating policy
- Cannot directly access local-native resources

### Remote self-hosted

Cloudflare OS runs on a server controlled by the user or organization, using standalone `workerd` or another supported deployment.

Best for:

- Home servers
- Private organizational deployments
- Custom retention and network policy
- Long-running remote apps without Cloudflare hosting

Costs:

- The operator owns updates, backups, TLS, identity, monitoring, and host isolation
- Standalone Cloudflare OS deployment tooling is still maturing
- Self-hosted `workerd` does not inherit all of Cloudflare's defense in depth

### Vibe-managed local

The shell downloads an exact runtime release, verifies it, installs it into a versioned application-data directory, and supervises it.

Best for:

- Local ownership
- App execution without a network connection
- No manual developer setup
- Rollback between known runtime releases
- Shell-independent runtime installation

Costs:

- First installation normally requires a download
- Downloaded executable policy differs by operating system and distribution channel
- Vibe owns supply-chain security, migrations, and process supervision

### User-managed local

The user or administrator supplies an absolute path or registered service endpoint for a compatible runtime.

Best for:

- Development
- Homebrew, Nix, or managed enterprise installations
- Custom `workerd` builds
- Reusing a runtime operated outside the shell

Costs:

- Version and feature drift
- More setup and support complexity
- A malicious or unexpected executable path is a serious security risk

This mode is explicit and advanced. Vibe must not silently select the first `workerd` found on `PATH`.

### Bundled local

The shell ships a runtime release inside its signed application bundle.

Best for:

- Offline first launch
- Atomic shell/runtime distribution
- Predictable support

Costs:

- Nested executable signing and notarization
- Larger shell downloads
- One runtime build per platform and architecture
- Shell updates required for runtime updates unless an additional updater exists

Bundled local remains an option, not a prerequisite for local execution.

## Provider Comparison

| Property | Cloudflare-hosted | Self-hosted remote | Managed local | System local | Bundled local |
| --- | --- | --- | --- | --- | --- |
| Manual runtime setup | No | Yes | No | Yes | No |
| Offline app execution | No | No | Yes | Yes | Yes |
| Multi-device access | Native | Configurable | No by default | No by default | No by default |
| Live collaboration | Native | Configurable | Local-network or relay needed | Local-network or relay needed | Local-network or relay needed |
| Local native capabilities | Device bridge | Device bridge | Direct broker | Direct broker | Direct broker |
| Vibe operates runtime | Shared service | No | Yes | No | Yes |
| Nested signing problem | No | No | Download-policy issue | External operator | Yes |
| Runtime update owner | Deployment owner | Server operator | Vibe runtime manager | User or administrator | Shell updater |
| Strongest use | Hosted default | Private server | Local default | Developer/enterprise | Fully packaged local |

## User-Facing Runtime Profiles

The product should present goals, not infrastructure jargon:

| Profile | Provider kind | User-facing description |
| --- | --- | --- |
| Vibe Cloud | Cloudflare-hosted | Runs online and is available across devices |
| This Device | Managed local or bundled local | Runs and stores its live state on this device |
| Custom Server | Remote self-hosted | Connects to a server you or your organization operate |
| Developer Runtime | User-managed local | Connects to a compatible runtime you installed |

The app detail view shows:

- Active profile
- Online or offline state
- Runtime and protocol versions
- Storage location at a human level
- Required device presence
- Last successful connection
- Export and move actions

## Architecture

### Provider-neutral shell

```text
Vibe shell
  |
  +-> app library and windows
  +-> Builder client
  +-> source/package manager
  +-> local secrets
  +-> device capability broker
  |
  `-> RuntimeProviderClient
        |
        +-> local supervisor adapter
        |     `-> workerd
        |
        `-> remote transport adapter
              `-> Cloudflare-hosted or self-hosted Cloudflare OS
```

The shell depends on Vibe provider contracts. It does not contain Cloudflare-specific app behavior.

### App path

```text
Vibe app source
  -> @vibe/runtime
    -> Cloudflare OS adapter
      -> Gadget client/server and Gatekeepers
        -> local or remote execution primitive
```

App source does not change when provider placement changes. Feature availability may change, and the manifest must declare requirements.

## Provider Contract

The initial interface should be small enough to implement with a local process and a remote endpoint:

```ts
export type RuntimeProviderKind =
  | "cloudflare-hosted"
  | "remote-self-hosted"
  | "local-managed"
  | "local-system"
  | "local-bundled";

export interface RuntimeProviderProfile {
  id: string;
  name: string;
  kind: RuntimeProviderKind;
  endpoint?: string;
  executable?: string;
  requestedRelease?: string;
}

export interface RuntimeProviderClient {
  connect(profile: RuntimeProviderProfile): Promise<RuntimeSession>;
  inspect(profile: RuntimeProviderProfile): Promise<RuntimeProviderStatus>;
  installPackage(
    session: RuntimeSession,
    app: VibeAppPackage
  ): Promise<VibeAppInstallation>;
  openApp(
    session: RuntimeSession,
    installationId: string
  ): Promise<VibeAppEndpoint>;
  exportApp(
    session: RuntimeSession,
    installationId: string
  ): Promise<VibeAppExport>;
}
```

The transport is not frozen. A hosted implementation may use HTTPS, WebSocket, and Cap'n Web. A local implementation may use loopback HTTP, a Unix-domain socket plus proxy, or another authenticated local transport.

### Handshake

Before app code runs, the shell and provider negotiate:

```ts
export interface RuntimeHandshake {
  providerId: string;
  providerKind: RuntimeProviderKind;
  protocolVersion: string;
  runtimeRelease: string;
  cloudflareOsRevision: string;
  workerdVersion?: string;
  supportedFeatures: readonly string[];
  authentication: readonly string[];
  placement: "local" | "remote";
  limits?: RuntimeLimits;
}
```

The handshake must be signed or authenticated by the selected provider relationship. A response from an arbitrary process on the expected port is not trusted.

### Feature negotiation

Portable baseline features may include:

- Client rendering
- Server actor APIs
- App-scoped durable storage
- Configuration
- Accepted and last-good revisions
- Package import and export

Optional features may include:

- Live collaboration
- Hosted background execution
- Local device capabilities
- Scheduled jobs
- Browser automation
- Large-object storage
- Provider-specific AI services

Missing required features block installation. Missing optional features produce a designed degraded state.

## Runtime Grouping

### Default rule

Ordinary apps share one local runtime only when all of these fields match:

```ts
export interface RuntimeGroupKey {
  operatingSystemUser: string;
  vibeProfileId: string;
  providerProfileId: string;
  runtimeRelease: string;
  trustClass: "personal" | "reviewed" | "untrusted";
}
```

In product terms:

```text
Personal profile, reviewed personal apps
  -> one local workerd
       +-> app A
       +-> app B
       `-> app C

Untrusted package
  -> separate runtime group

Work profile
  -> separate runtime group
```

This matches Cloudflare OS's intended use of one kernel serving multiple isolated Gadgets without making all apps on the machine one security domain.

### Why not one runtime per ordinary app

One runtime per app duplicates:

- Workshop kernel
- Builder integration
- Gatekeeper services
- Ports and health checks
- Logs
- Runtime update work
- Memory and startup cost

Use a separate runtime when the isolation benefit is deliberate.

### Separate-runtime triggers

Create a different runtime group when:

- The app is untrusted or comes from a marketplace policy requiring stronger isolation.
- The app requires an incompatible runtime release.
- The app needs experimental compatibility flags.
- The app has unusual resource requirements.
- The user requests a dedicated runtime.
- The app belongs to a different organizational identity or retention policy.
- A security policy requires a separate OS sandbox, container, or VM.

### No cross-user system daemon

Do not place unrelated operating-system users into one machine-wide local runtime. Each OS user gets separate runtime state, identity, credentials, and process ownership.

A remote multi-user service may share infrastructure, but its tenant isolation is an operator security responsibility beyond this local grouping rule.

### Multiple windows

Multiple Vibe windows and app shortcuts may attach to one runtime group:

```text
Vibe window A --+
Vibe window B --+-> authenticated local runtime session -> one workerd
App shortcut C -+
```

The local supervisor owns a single-instance lock and publishes an authenticated endpoint to later windows. A window must not infer trust from a port number alone.

## Local Runtime Lifecycle

The local supervisor owns:

- Runtime discovery and installation
- Profile and data directories
- Runtime-group locks
- Port or socket allocation
- Configuration generation
- Process creation
- Readiness and health checks
- Log collection
- Crash detection
- Graceful and forced shutdown
- Version negotiation
- Update and rollback

The local supervisor does not own:

- App business logic
- Builder decisions
- Provider grants
- Gadget-native capabilities
- User source as an opaque internal format

### Startup flow

```text
shell requests runtime group
  -> runtime manager resolves installed release
    -> validates manifest, digest, and executable
      -> acquires group lock
        -> creates authenticated local session
          -> starts workerd with explicit config and data paths
            -> waits for authenticated health response
              -> returns app endpoint
```

### Shutdown policy

The initial policy should be:

- Keep the runtime alive while any Vibe window, Builder session, scheduled local actor, or declared background task needs it.
- After the last consumer disconnects, wait a short idle grace period.
- Flush and stop gracefully.
- Force termination after a bounded timeout.
- Preserve data independently of process lifetime.

An optional "keep Vibe running in the background" setting may change this policy later.

### Crash behavior

If the runtime crashes:

1. Mark all attached apps unavailable without losing their shell state.
2. Capture exit status and bounded logs.
3. Attempt a bounded restart when policy permits.
4. Reopen the prior last-good app revisions.
5. Stop retrying after repeated failure.
6. Offer diagnostics, runtime rollback, export, and safe mode.

One app crash should not be represented as a whole-runtime crash when Dynamic Worker isolation contains it.

## Managed Local Runtime

Managed local is the preferred way to deliver local execution without requiring the user to install developer tools.

### Runtime release

A runtime release is promoted as a tested unit:

```ts
export interface RuntimeReleaseManifest {
  schemaVersion: 1;
  releaseId: string;
  protocolVersion: string;
  cloudflareOsRevision: string;
  workerdVersion: string;
  platform: string;
  architecture: string;
  artifactUrl: string;
  artifactSize: number;
  sha256: string;
  signature: string;
  supportedFeatures: readonly string[];
}
```

The manifest must be authenticated. HTTPS alone does not make a mutable manifest an adequate release identity.

### Installation flow

```text
fetch signed release manifest
  -> verify manifest signature
    -> select exact platform artifact
      -> download to temporary version directory
        -> verify size, digest, and signature
          -> validate executable version
            -> atomically rename into runtimes/<releaseId>
              -> run health check
                -> mark release available
```

Never pipe a remote installer into a shell. Never run an artifact before verification. Never overwrite the active runtime in place.

### Directory shape

```text
Vibe user data/
  runtime-releases/
    2026-08-06.1/
      manifest.json
      workerd
      cloudflare-os/
      config/
  profiles/
    personal/
      runtime.json
      groups/
        reviewed/
          data/
          endpoint.json
          logs/
```

The shell uses platform-appropriate application-data locations rather than a hard-coded home-directory folder.

### First-run behavior

If no local runtime is installed:

- Explain the download size and publisher.
- Offer managed installation.
- Offer a configured remote provider.
- Offer advanced selection of a system runtime.
- Do not silently run a shell installer.

After installation, app execution must work without a network connection unless the app itself declares remote requirements.

### Operating-system policy

Managed download avoids nesting `workerd` inside the shell bundle, but it does not automatically solve:

- macOS downloaded-executable and sandbox policy
- Application-store restrictions
- Antivirus and endpoint-management policy
- Code-signing expectations for the helper itself

The runtime artifact should be signed where the platform supports it. The local-managed spike must test the actual intended distribution channel.

## User-Managed Local Runtime

The user supplies an absolute executable path or local service endpoint.

Vibe validates:

- The target exists and is owned or approved under current policy.
- The executable or service reports an expected identity.
- Protocol and feature versions are compatible.
- The runtime is not a shell script or unexpected indirection unless developer mode explicitly permits it.
- The data directory is not shared accidentally with another profile.
- The connection is authenticated.

Vibe does not:

- Search arbitrary `PATH` entries automatically.
- Fall back to a different runtime after validation failure.
- Upgrade the user's runtime.
- Assume a compatible version because the executable is named `workerd`.

Developer mode may support Cloudflare OS's documented local Wrangler setup, but that is not the consumer runtime contract.

## Bundled Local Runtime

Bundled local remains useful when:

- Offline first launch is required.
- The distribution channel allows the helper.
- Nested signing and notarization are reliable.
- Shell and runtime should update atomically.

The shell must still access it through the same local supervisor and provider contract. Bundling is a distribution choice, not a second runtime architecture.

If bundled installation fails on one platform, Vibe can use managed local or remote execution without changing app source.

## Runtime Updates

### Compatibility

Each runtime profile pins a runtime release. Apps declare a Vibe runtime API version and optional Cloudflare compatibility requirements.

The shell updates only to a release that:

- Speaks a compatible provider protocol
- Supports every installed app's required baseline features
- Has passed migration and rollback tests

### Atomicity

Never replace an active runtime directory in place:

```text
active release A
  -> install and validate release B
    -> stop group on A
      -> migrate or open copied state with B
        -> validate
          -> switch profile pointer to B
            -> retain A for rollback
```

If validation fails, the profile remains on A. State migrations that cannot roll back require an explicit backup and recovery plan.

### Independent and shell-coupled updates

Managed and system runtimes may update independently from the shell. The handshake handles compatibility.

Bundled runtimes normally update with the shell. The provider protocol remains the same in both cases.

## Remote Providers

### Remote connection

Remote providers require:

- TLS
- Provider identity verification
- User authentication
- Session expiration and revocation
- Explicit organization or tenant selection
- Protocol and feature negotiation
- Clear data-location disclosure

The shell may cache provider metadata and readable source snapshots. It must not claim a live remote app is available offline unless an offline execution profile actually exists.

### Cloudflare-hosted mode

In Cloudflare-hosted mode, Vibe deploys or connects to Cloudflare OS on Workers. The provider contract adapts Cloudflare identity, Durable Objects, Dynamic Workers, Gatekeepers, storage, collaboration, and Blueprint machinery to Vibe terms.

This is not a generic "hosted workerd" process. It is a hosted implementation of the same runtime behaviors.

### Self-hosted mode

A self-hosted provider may run:

- Standalone `workerd` on a server
- A container or VM containing the Vibe runtime release
- Another compatible implementation of the provider protocol

The provider reports implementation and feature information during the handshake. Vibe does not assume that every self-hosted environment supports the complete Cloudflare-hosted feature set.

### Remote app availability

When the provider is unavailable:

- Keep local shell navigation and cached metadata usable.
- Show the last known runtime and revision.
- Do not present stale data as current.
- Permit source/package export if a local accepted snapshot exists.
- Queue only operations explicitly designed for deferred delivery.
- Do not silently start a different provider with stale state.

## Hybrid Mode

Hybrid mode means shell, Builder, app execution, data, and capabilities may have different placements.

The architecture permits this matrix:

| Component | Local option | Remote option |
| --- | --- | --- |
| Shell | Desktop Vibe | Browser shell |
| Builder | Local process or local model | Hosted Builder or hosted model |
| App execution | Local `workerd` | Hosted or self-hosted runtime |
| Live app state | Local runtime storage | Remote provider storage |
| Source history | Local files and Git | Provider snapshots and remote history |
| External SaaS binding | Local or remote Gatekeeper | Remote Gatekeeper |
| Native device binding | Local device broker | Not available without device bridge |

The product should not expose every combination initially. Ship named, tested profiles:

- Vibe Cloud
- This Device
- Custom Server
- Developer Runtime

Additional placement controls can appear when an app or actor genuinely needs them.

## Device Capability Bridge

A remote app cannot directly call the local filesystem, clipboard, notifications, camera, or installed applications.

The trusted shell provides a device broker:

```text
remote Gadget
  -> narrow typed capability call
    -> authenticated provider session
      -> outbound device channel
        -> local device broker
          -> current grant and user policy
            -> operating-system operation
```

### Rules

- The shell initiates the outbound device connection. It does not expose a general inbound service.
- A remote provider cannot manufacture a device grant.
- Grants identify provider, user, app installation, capability, resource scope, and expiration.
- The user can inspect and revoke grants locally.
- The Gadget receives results, not ambient shell APIs or provider credentials.
- High-risk side effects use the same approval and deferred-effect model as other Vibe bindings.
- The device broker is unavailable when the shell is offline unless an explicitly installed background agent is running.

### Placement-aware requirements

An app manifest may declare:

```ts
export interface VibeCapabilityRequirement {
  name: string;
  kind: string;
  placement: "any" | "local-device" | "provider";
  devicePresence?: "always" | "while-open" | "on-demand";
  capabilities: readonly string[];
}
```

The exact schema remains open. The durable rule is that placement and required device presence are visible before installation.

## Source and State Ownership

### Live authority

One provider owns live execution state for an app installation:

```text
selected provider
  owns:
    live Gadget source/mainline
    runtime data
    accepted and last-good runtime revisions
    active binding introductions

shell
  owns:
    provider selection
    local source exports and Git history
    device grants
    user-facing migration workflow
```

This does not prevent local readable source. It prevents two providers from silently believing they are the active state authority.

### Portable and provider-specific state

Classify state as:

- **Portable source:** manifest, actors, configuration schemas, package defaults
- **Portable app data:** data with a declared export/import representation
- **Provider-specific state:** implementation data with no portable representation
- **Private authority:** credentials, grants, tokens, observed protected resources
- **History:** Builder chat and provider revision logs

Only portable source is always portable. The app declares or implements export for portable app data. The other categories do not move by default.

## Moving an App

Provider migration is an explicit install-and-switch flow:

```text
export accepted source package
  -> optionally export portable app data
    -> inspect target provider compatibility
      -> install independent target instance
        -> request new binding grants
          -> import selected portable data
            -> validate and establish last-good
              -> user switches active installation
```

The source installation may succeed even when some data or capabilities cannot move. The shell reports those differences before switching.

The original installation remains available until the user explicitly removes it.

### No transparent failover

Do not silently fail from remote to local or local to remote. Transparent failover would require synchronized source, Durable Object state, configuration, grants, and effect history. Without that, it risks running stale code or repeating side effects.

Read-only cached UI and explicitly designed offline modes may be added independently.

## Security Boundaries

### Local endpoint

A local runtime must:

- Bind only to an intended loopback interface or private socket.
- Use a high-entropy per-session credential.
- Validate `Origin` and session audience.
- Reject unauthenticated health, control, and app-management calls as appropriate.
- Avoid permissive CORS.
- Prevent arbitrary local webpages from attaching.
- Rotate session credentials across runtime starts.
- Avoid writing bearer credentials into broadly readable files or command lines.

Port locality is not authentication.

### Remote endpoint

A remote provider must:

- Use TLS.
- Authenticate provider and user.
- Scope sessions to provider, tenant, profile, and device.
- Support revocation.
- Separate control-plane and app capabilities.
- Make operator access and data location understandable.

### Runtime trust

Before starting a local runtime, Vibe verifies:

- Release manifest authenticity
- Artifact digest and signature
- Expected runtime identity and version
- Configuration paths
- Profile and trust-domain ownership

User-managed runtimes are marked as such. A runtime can inspect the app state it hosts, so provider trust is a product-level decision.

### Gadget isolation

Sharing one `workerd` relies on its logical Worker and capability boundaries. Because standalone `workerd` is not by itself a complete hostile-code sandbox, untrusted apps may require a separate OS-sandboxed process, container, VM, or Cloudflare-hosted execution.

The shell's native RPC bridge must never be injected into Gadget frames. Gadgets reach native resources only through Vibe capabilities.

## Failure Modes

| Failure | Required behavior |
| --- | --- |
| Provider is offline | Show explicit offline state; keep cached shell metadata; do not switch providers silently |
| Runtime protocol mismatch | Block connection and offer a compatible runtime or shell update |
| Missing optional feature | Open a designed degraded state |
| Missing required feature | Block installation before app code runs |
| Managed download interrupted | Leave active release unchanged; resume or restart safely |
| Runtime digest mismatch | Delete or quarantine artifact and report verification failure |
| Runtime crashes | Preserve shell state, collect bounded logs, attempt bounded recovery |
| One Gadget fails | Contain failure where runtime isolation permits; keep other apps available |
| Device broker disconnects | Mark local-device bindings unavailable without revoking unrelated grants |
| Migration partially fails | Keep source and target installations distinct; do not switch active pointer |
| Runtime update fails validation | Roll back to the prior runtime release |
| Local port is claimed | Retry through the supervisor; never attach based only on the expected port |

## Research Findings

### Cloudflare OS is deployable in more than one place

At the reviewed revision, Cloudflare OS documents:

- A complete local development run through Wrangler and `workerd`
- Local data stored under `.wrangler`
- Cloudflare-hosted deployment
- The ability in principle to run entirely on standalone `workerd`
- Incomplete production self-hosting documentation and tooling

Implication: provider-neutral placement is consistent with upstream intent, but standalone product packaging remains Vibe work.

### Audio TTS proves Electrobun service supervision

At revision `42a998b2e046b2bb1b761b54107b7de6cf91d7d7`, Audio TTS:

- Uses an Electrobun Bun main process
- Finds a loopback port
- Spawns a Python FastAPI server
- Streams logs
- Polls a health endpoint
- Proxies UI requests through typed RPC
- Shuts down the child process
- Packages backend source with the application

It bootstraps `uv`, Python, dependencies, and model data after installation rather than bundling the Python executable. Its release workflow targets macOS ARM64. Its local server uses permissive CORS and no Vibe-grade session authentication.

Implication: Electrobun has demonstrated the supervisor pattern. It has not yet proved Vibe's signed helper, cross-platform runtime release, local authentication, or hardened isolation requirements.

### Electrobun Doom proves native copying, not release signing

At revision `4ce99f1240eb545eae76ad55956e6cc9f5900e3d`, Electrobun Doom:

- Copies a native `libdoom.dylib` through Electrobun's `build.copy`
- Finds the copied library at runtime
- Loads it through Bun FFI
- Builds the library with a simple Clang Makefile

The checked-in library is a thin macOS ARM64 Mach-O with an ad-hoc linker signature. The project does not enable Electrobun code signing or notarization, and its Electrobun dependency points to a neighboring development checkout.

Current Electrobun signing code signs known frameworks and recursively signs executables and libraries under `Contents/MacOS`. An arbitrary native file copied into app resources is not demonstrated as receiving equivalent Developer ID signing.

Implication: a bundled `workerd` is mechanically credible, but Vibe should package it as an explicit helper executable, sign it before the outer app bundle, then notarize and test the complete release. Plain `build.copy` is not sufficient proof. This strengthens the case for a dedicated bundled-executable build facility and preserves Vibe-managed external installation as a simpler independent lifecycle.

### External runtime installation is plausible but security-sensitive

Audio TTS demonstrates a user-friendly external-runtime bootstrap. Deno, Bun, package managers, and system administrators also provide ways to distribute native helpers independently from a shell.

Implication: Vibe-managed local installation is a valid first-class mode, but it must use verified artifacts and atomic version directories rather than an unverified shell installer.

## Design Decisions

| Decision | Class | Choice | Rationale |
| --- | --- | --- | --- |
| Runtime placement boundary | Design coherence | Provider protocol | Keeps app and shell contracts independent of hosting |
| Supported provider families | Evidence and coherence | Hosted, self-hosted, managed local, system local, bundled local | All are plausible implementations of the Cloudflare OS runtime model |
| Default local sharing | Taste under constraints | One runtime per profile, release, provider, and trust class | Reuses the kernel without making all apps one trust domain |
| Ordinary per-app runtime | Design coherence | Not default | Duplicates kernel and services without a routine benefit |
| Untrusted app isolation | Security | Separate runtime group or stronger remote/OS sandbox | A logical Worker boundary is not complete host defense |
| System-wide daemon | Security | Do not share across OS users | Identity, state, and credentials require user ownership |
| Local runtime discovery | Security | Explicit managed install or selected absolute path | Avoids path substitution and version drift |
| Runtime download | Security | Signed manifest, verified artifact, atomic install | Makes external installation supportable and auditable |
| Provider migration | Design coherence | Explicit export/install/rebind/switch | Live state and authority cannot fail over safely by assumption |
| Remote native access | Design coherence | Outbound device capability bridge | Preserves narrow authority without exposing the shell |
| First public default | Deferred | Decide after provider spikes | Product speed, local ownership, and distribution policy need evidence |

## Implementation Plan

### Phase 0: Provider contract and two test adapters

- [ ] Define provider kinds, profile identifiers, handshake, feature negotiation, and normalized errors.
- [ ] Define runtime release and runtime group identifiers.
- [ ] Add an in-memory or test provider for contract tests.
- [ ] Implement one remote adapter against a Cloudflare OS deployment.
- [ ] Implement one developer-only local adapter against the documented local Cloudflare OS environment.
- [ ] Install and run the same Hello World package through both adapters.
- [ ] Verify app source contains no provider-specific imports.
- [ ] Record every upstream API or patch required by either adapter.

Exit:

- One shell-side client can list, install, open, and export the same app through local and remote providers.

### Phase 1: Runtime grouping and local supervision

- [ ] Define `RuntimeGroupKey` and single-instance locking.
- [ ] Start one local runtime for two ordinary apps.
- [ ] Attach two shell windows to it.
- [ ] Place an untrusted test app in a separate runtime group.
- [ ] Add authenticated health, logs, shutdown, crash detection, and bounded restart.
- [ ] Verify one Gadget failure does not automatically stop unrelated apps.
- [ ] Verify one runtime failure has an honest group-level blast radius.

Exit:

- Runtime sharing and separation match the documented policy.

### Phase 2: Managed local runtime

- [ ] Define and sign a runtime release manifest.
- [ ] Build or obtain exact platform `workerd` artifacts.
- [ ] Install into versioned directories with digest verification.
- [ ] Generate Cloudflare OS configuration and data paths.
- [ ] Start, validate, update, roll back, and remove an inactive release.
- [ ] Test the intended macOS, Windows, and Linux distribution paths.
- [ ] Confirm no Node, pnpm, Wrangler, or manual runtime installation is required.

Exit:

- A fresh supported machine can install local execution from Vibe and reopen Hello World offline.

### Phase 3: Remote and hybrid product profiles

- [ ] Add Vibe Cloud and Custom Server profiles.
- [ ] Implement provider identity and user authentication.
- [ ] Display placement, state ownership, and offline expectations.
- [ ] Define reconnect, session expiration, and provider-offline behavior.
- [ ] Verify an app opens without a Builder or model authorization.

Exit:

- A user can choose local, hosted, or custom-server execution without changing app source.

### Phase 4: Device capability bridge

- [ ] Establish an outbound authenticated device session.
- [ ] Define device-scoped binding requirements and grants.
- [ ] Implement one low-risk capability such as notifications.
- [ ] Add approval, revocation, audit, and device-offline behavior.
- [ ] Verify the remote Gadget cannot access undeclared native operations.

Exit:

- A remote app can use one approved device capability through the same Vibe binding model as a local app.

### Phase 5: Explicit provider migration

- [ ] Export accepted source and one portable data set.
- [ ] Inspect target-provider compatibility.
- [ ] Install an independent target instance.
- [ ] Rebind capabilities with new grants.
- [ ] Validate target last-good.
- [ ] Switch active installation only after user confirmation.
- [ ] Preserve the source installation for rollback.

Exit:

- An app can move between two providers without copying credentials or silently dropping unsupported state.

## Open Questions

### What should be the first public default?

Options:

- Cloudflare-hosted for zero setup
- Managed local for local ownership
- A setup choice between the two

Current recommendation:

Implement remote and developer-local adapters first, then decide from packaging, latency, reliability, and onboarding evidence. Do not make the decision only from architectural preference.

### Should the local runtime survive after every window closes?

Options:

- Stop after a grace period
- Remain as a login or background service
- Let background actors opt in

Current recommendation:

Stop after a grace period unless a declared background task requires it. Revisit when real apps need scheduling or notifications.

### How many trust classes should Vibe expose?

Options:

- Personal versus untrusted
- Personal, reviewed, and untrusted
- A policy-defined enterprise set

Current recommendation:

Start with reviewed personal and isolated untrusted groups internally. Avoid presenting a complex trust taxonomy until marketplace installation exists.

### Can managed `workerd` run under every target distribution policy?

Options:

- Downloaded verified helper
- Bundled signed helper
- User-installed helper
- Remote-only on restricted channels

Decision trigger:

Run signed, notarized, sandboxed, antivirus, and application-store experiments on each intended channel.

### What data is portable across providers?

Options:

- Source only
- Source plus app-defined export
- Generic storage snapshots

Current recommendation:

Guarantee source portability. Add app-defined portable data. Do not promise generic Durable Object migration until it has an explicit stable representation.

### How does a device bridge preserve end-to-end authority?

Open issues:

- Session establishment
- Remote provider impersonation
- Local grant storage
- Device naming
- Multiple devices
- Deferred effects
- Offline queues

Decision trigger:

Design and threat-model the first notification binding before generalizing the protocol.

## Success Criteria

- [ ] One portable app package runs through local and remote provider adapters.
- [ ] Accepted app execution makes no model request.
- [ ] Provider placement and offline behavior are visible.
- [ ] A managed local install requires no developer toolchain.
- [ ] The local endpoint rejects unauthenticated and wrong-origin clients.
- [ ] Two ordinary apps share one local runtime group.
- [ ] An untrusted app can run in a separate runtime group.
- [ ] Two Vibe windows attach safely to one runtime.
- [ ] A runtime crash produces bounded recovery and useful diagnostics.
- [ ] Runtime update failure leaves the prior release usable.
- [ ] A remote app receives no local-native authority without a device grant.
- [ ] Moving an app does not copy credentials, grants, private state, or Builder history.
- [ ] Provider incompatibility is detected before app code runs.
- [ ] Source remains readable and exportable from every supported provider.

## Impact on Existing Specs

### Vibe App Foundation

A later focused revision should:

- Add runtime providers and profiles to the system architecture.
- Distinguish app portability from live-state portability.
- Add placement-aware binding requirements.
- Add runtime groups and trust domains.
- Clarify offline behavior by provider.

The foundation's existing product contracts remain unchanged until that revision is reviewed.

### Building Vibe on Cloudflare OS

This document refines its runtime-placement recommendation:

- Hosted Cloudflare OS is one provider, not the sole v0 architecture.
- Local `workerd` uses the same Vibe contracts.
- The substrate spike should exercise at least one local and one remote provider.
- The final first-edition default remains open pending evidence.

The Cloudflare adapter boundary, Gadget model, Gatekeepers, source concerns, and go/no-go criteria remain applicable.

### Cloudflare OS Lessons

No change is required. Capability binding, provenance, draft/mainline, sharing, and Blueprint lessons apply regardless of provider placement.

## References

- [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md)
- [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md)
- [Building Vibe on Cloudflare OS](./2026-08-06-vibe-on-cloudflare-os.md)
- [Cloudflare OS README at the reviewed revision](https://github.com/cloudflare/cloudflare-os/blob/aedcda8b3066ff666f57ae28ecef7341d6c2dee7/README.md)
- [Kenton Varda local `workerd` demonstration](https://youtu.be/RmS5s6Wbin4?t=1007)
- [`workerd` repository and security guidance](https://github.com/cloudflare/workerd)
- [Audio TTS main process at the reviewed revision](https://github.com/blackboardsh/audio-tts/blob/42a998b2e046b2bb1b761b54107b7de6cf91d7d7/src/bun/index.ts)
- [Audio TTS Electrobun configuration](https://github.com/blackboardsh/audio-tts/blob/42a998b2e046b2bb1b761b54107b7de6cf91d7d7/electrobun.config.ts)
- [Electrobun Doom configuration at the reviewed revision](https://github.com/blackboardsh/electrobun-doom/blob/4ce99f1240eb545eae76ad55956e6cc9f5900e3d/electrobun.config.ts)
- [Electrobun Doom native Makefile](https://github.com/blackboardsh/electrobun-doom/blob/4ce99f1240eb545eae76ad55956e6cc9f5900e3d/native/Makefile)
- [Electrobun code-signing implementation at the reviewed revision](https://github.com/blackboardsh/electrobun/blob/36fc170f7267f43906fbe78c9860ba4b4d2e1298/package/src/cli/index.ts)
- [Electrobun architecture](https://github.com/blackboardsh/electrobun/blob/36fc170f7267f43906fbe78c9860ba4b4d2e1298/docs/src/content/docs/electrobun/guides/architecture/overview.mdx)
