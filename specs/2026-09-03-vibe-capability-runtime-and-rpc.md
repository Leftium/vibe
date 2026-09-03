# Vibe Capability Runtime and RPC Adapters

**Date:** 2026-09-03  
**Status:** Proposed architecture - protocol experiment pending  
**Refines:** Runtime and actor portions of [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md)  
**Related:** [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md), [Vibe Runtime Providers and Placement](./2026-08-06-vibe-runtime-providers.md), [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md)

## One Sentence

Vibe apps compute and render inside an isolated app runtime, but every operation that crosses a trust, persistence, identity, device, or external-service boundary goes through a typed Vibe capability whose call-like API is implemented as validated runtime messages.

## First-Screen Contract

The existing foundation requires sandboxed app execution, typed actor interfaces, explicit capabilities, isolated state, and provider-neutral placement. The Cloudflare OS research recommends one typed API shared by the UI and agent, backed by capability-style RPC. It does not yet define how RPC, runtime messages, commands, and events fit together or whether a framework transport becomes part of the app contract.

The target shape is:

```text
untrusted app UI and domain logic
  -> typed Vibe capability API
    -> request, command, or subscription message
      -> runtime identity, validation, policy, and routing
        -> provider or actor
          -> result or event
```

The public contract is Vibe-owned and transport-neutral. SvelteKit remote functions may implement the first hosted web adapter, but apps do not depend directly on SvelteKit's generated endpoints or server execution model.

This architecture is proven when:

1. One reference app uses the same Vibe API without source changes against an in-memory local adapter and a SvelteKit remote-functions adapter.
2. App code cannot perform an authoritative effect without crossing the runtime boundary.
3. The runtime derives app identity and grants from trusted context, validates every request, and denies undeclared authority.
4. The UI and an agent can invoke the same scoped actor capability.
5. Queries, durable commands, and events have distinct observable behavior.
6. Replacing the SvelteKit adapter does not change the app-facing API.

## Scope

This specification defines:

- The ownership boundary between app code, the Vibe runtime, and providers
- The relationship between RPC and messages
- When to use queries, bounded mutations, durable commands, and events
- The minimum shape of a typed capability API
- Security and failure requirements at the runtime boundary
- How SvelteKit remote functions may serve as the first hosted adapter
- A vertical-slice experiment that can refine the protocol

It does not define:

- The complete capability catalog
- A final wire encoding or schema library
- A complete durable workflow engine
- A declarative UI protocol or Vibe-owned renderer
- A production security review
- Compatibility with arbitrary SvelteKit remote functions
- SvelteKit as a permanent runtime dependency

## Terminology

| Term | Meaning |
| --- | --- |
| Capability | A typed, scoped handle through which an app or agent may request operations from one resource or actor |
| Grant | Trusted runtime state authorizing a capability for a particular app, user, resource, and operation set |
| Request | A message expecting one correlated result; usually exposed as a Promise-returning method |
| RPC | The app-facing request/response programming model built over runtime messages |
| Query | A read operation that should not cause an externally visible mutation |
| Bounded mutation | A short mutation whose success or failure is known within one request lifetime |
| Durable command | Requested work that may outlive the caller, survive restart, require approval, or report progress |
| Event | A fact already observed by the runtime or an actor; it is not a request for work |
| Provider | A local or remote implementation of Vibe runtime capabilities |
| Authoritative effect | An operation that crosses a trust, persistence, identity, device, process, or external-service boundary |

Messages are the boundary representation. RPC, commands, and events are interaction semantics carried by messages.

## Design Decisions

| Decision | Class | Choice | Rationale |
| --- | --- | --- | --- |
| App execution model | Design coherence | App code renders and computes normally inside an isolate; only authoritative effects must cross the runtime | This preserves Svelte ergonomics and local performance while mediating operations that carry authority or need supervision. |
| Public API | Design coherence | Apps and agents use typed capability interfaces | One semantic API improves discoverability, testing, agent use, and least-authority grants. |
| Boundary representation | Design coherence | All capability operations become validated runtime messages | Messages make routing, policy, tracing, placement, and provider substitution explicit. |
| Call semantics | Taste under constraints | Use Promise-returning RPC for queries and short bounded mutations | Local-call ergonomics are useful when the operation has a natural correlated result and a bounded lifetime. |
| Long-running work | Design coherence | Use durable commands plus status and result events | A caller should not need to remain connected while work waits, restarts, reports progress, or requests approval. |
| Notifications | Design coherence | Use events for facts and subscriptions, not commands disguised as broadcasts | Events can be observed by multiple consumers and do not imply one direct response. |
| UI representation | Taste under constraints | Do not require apps to emit a runtime-interpreted UI tree | A Vibe UI protocol would make Vibe responsible for a complete renderer and constrain components, animation, canvas, and new browser features. |
| Core semantics | Design coherence | Core messages describe platform mechanics; app-specific behavior stays in app code or versioned actor modules | A core operation such as `storage.put` is portable. A core operation such as `completeTodo` would make every domain change a runtime release. |
| Web transport | Evidence | Experiment with SvelteKit remote functions as the first hosted adapter | They already provide generated client/server calls, validation hooks, queries, commands, live queries, caching, and batching. |
| Framework coupling | Design coherence | SvelteKit remote functions are not the Vibe ABI | Remote functions are server-only, framework-generated, and currently experimental. Vibe also targets local, native, and Cloudflare OS placements. |
| Remote-function ownership | Design coherence | Only trusted runtime code defines remote functions | Generated app server code would otherwise inherit ambient server imports and authority. |
| Security model | Evidence | Isolation, capability grants, validation, partitioning, and quotas enforce security; message or RPC syntax does not | Either an ambient message bus or a broad RPC object can expose excessive authority. |

## Ownership

```text
App isolate owns:
  Svelte components and rendering
  domain logic
  ephemeral UI state
  requested capability operations
  handling results and events

Vibe runtime owns:
  trusted app and user identity
  capability grants and revocation
  input and output validation
  provider and actor routing
  app-scoped storage namespaces
  deadlines, cancellation, quotas, and supervision
  audit metadata and protocol negotiation

Provider or actor owns:
  implementation of the granted operation
  provider-specific failure translation
  durable state explicitly assigned to that actor
  external credentials hidden from app code
```

The runtime must not accumulate application-domain methods. Domain behavior belongs in app code or a versioned actor capability with its own protocol and state owner.

## App-Facing API

The API should look like ordinary typed TypeScript while preserving asynchronous semantics across every adapter:

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
  readonly commands: VibeCommandRegistry;
}

export interface VibeStorage {
  get<T>(key: string): Promise<T | undefined>;
  put<T>(key: string, value: T): Promise<void>;
  delete(key: string): Promise<void>;
}

export interface VibeCommandHandle<Result> {
  readonly id: string;
  status(): Promise<VibeCommandStatus>;
  events(): AsyncIterable<VibeCommandEvent>;
  result(): Promise<Result>;
  cancel(): Promise<VibeCancelResult>;
}
```

The initial contract should remain small. New methods require a concrete portable use case and defined permission, failure, and placement semantics.

### Query and bounded mutation

Queries and bounded mutations use request/response semantics:

```ts
async function saveDocument(runtime: VibeRuntime, document: Document) {
  await runtime.storage.put("document", document);
}
```

The method call is a facade. The adapter converts it into a request message, and the runtime returns a correlated result.

### Durable command

Work that may outlive a request returns a durable handle:

```ts
async function startImport(runtime: VibeRuntime, source: VibeFileHandle) {
  const task = await runtime.commands.start<ImportResult>("photos.import", {
    source
  });

  return task.id;
}
```

The app may reconnect to the command by ID. Disconnecting the UI does not implicitly cancel the work.

### Event

Events describe facts and may have multiple consumers:

```ts
async function observeImport(task: VibeCommandHandle<ImportResult>) {
  for await (const event of task.events()) {
    renderProgress(event);
  }
}
```

An event stream is not automatically an event log. The capability contract must say whether events are ephemeral, resumable, or durably retained.

## Runtime Messages

The exact wire encoding is deferred, but every adapter must preserve a common logical envelope:

```ts
type CapabilityRequest = {
  protocol: "vibe.runtime";
  protocolVersion: number;
  messageId: string;
  requestId?: string;
  capabilityId: string;
  operation: string;
  deadline?: string;
  payload: unknown;
};
```

Trusted transport context supplies the app, instance, user, runtime profile, and grants. The client payload must not be authoritative for those values.

The runtime performs these steps in order:

```text
receive message
  -> authenticate transport and derive caller identity
    -> resolve capability handle
      -> validate operation and payload
        -> check grant, policy, quota, and runtime mode
          -> invoke provider or actor
            -> validate and normalize result
              -> record audit metadata
                -> return result or publish event
```

Request and result messages require correlation. Mutations that may be retried require an idempotency rule. Providers must not silently invent stronger delivery guarantees than the logical contract declares.

## Capabilities and Authority

An app package declares a requirement. Installation or configuration creates a grant. A requirement never grants authority by itself.

```text
package requirement:
  calendar.read

user grant:
  account = personal@example.com
  calendars = [family]
  operations = [events.list]

runtime capability:
  a scoped handle implementing only the granted interface
```

The app receives neither the underlying credential nor a general account-wide API. The same scoped capability may be exposed to an agent when the user's authoring policy allows it.

A capability ID is an unforgeable or runtime-validated reference, not a guessable service name that bypasses authorization. Revocation invalidates subsequent operations and produces a normalized permission failure.

## Isolation and Stability

Message mediation improves security and stability only when the runtime treats the app as hostile input.

Required controls include:

- Process, worker, or iframe isolation appropriate to the placement
- No ambient credentials, arbitrary network access, or trusted server imports
- Schema validation at every trust boundary
- Capability checks tied to trusted caller identity
- Per-app storage partitions
- CPU, memory, message-rate, payload-size, and storage quotas
- Deadlines and cancellation where the operation supports them
- Backpressure for streams and event subscriptions
- Result validation and normalized failures
- Audit records for consequential external effects
- Separate preview and accepted state or explicit simulation where required

Valid messages can still exhaust resources. The runtime must rate-limit and supervise the sender rather than treating schema validity as proof of safe behavior.

## SvelteKit Remote-Functions Adapter

SvelteKit remote functions are available since SvelteKit 2.27 and are currently experimental. A `.remote.ts` export becomes a client-side fetch wrapper backed by a generated HTTP endpoint. The framework currently provides `query`, `query.live`, `form`, `command`, and `prerender`, along with argument validation, query deduplication, batching, and mutation-driven refresh.

These features make remote functions a strong hosted-web transport experiment, not a stable Vibe contract.

### Placement in the architecture

```text
app code
  -> packages/app-sdk VibeRuntime facade
    -> SvelteKit remote-functions adapter
      -> trusted Vibe runtime service
        -> provider or actor
```

The adapter may map:

| Vibe interaction | SvelteKit mechanism |
| --- | --- |
| Query | `query` |
| Bounded mutation | `command` |
| Live subscription | `query.live` |
| Progressive form submission | `form`, when the app SDK exposes a form-specific facade |
| Durable command | Start with `command`; observe through `query` or `query.live`; durability belongs to Vibe |

`prerender` is a build optimization, not a runtime capability primitive.

### Trust boundary

Only trusted Vibe runtime packages may define `.remote.ts` functions. Generated app code must not:

- Define remote functions
- Import `$app/server`
- Import private environment modules
- Import trusted server implementation modules
- Run in the runtime server's authority domain without a separate isolate

Candidate validation must enforce these rules structurally. Prompt instructions and code review are not security boundaries.

Remote-function arguments are externally callable input and require a Standard Schema or equivalent validator. The adapter derives authorization from trusted request state such as an authenticated session and server-owned app mapping. It must not authorize access from client-controlled route, parameter, URL, app ID, or capability metadata.

### Gateway shape

The first experiment should compare two shapes:

1. Dedicated remote functions for stable core capabilities such as storage and configuration. This preserves SvelteKit's operation-specific query caching and types.
2. A generic validated gateway for dynamic actor and binding capabilities. This avoids generating a remote endpoint for every installed module.

Both shapes sit behind the same app SDK. The experiment chooses their boundary based on bundle size, schema generation, trace quality, cache behavior, and dynamic module support. It does not expose either shape as the portable app API.

## Provider Adapters

The same Vibe API may use different transports and placements:

| Placement | Likely adapter |
| --- | --- |
| Hosted SvelteKit | Remote functions over generated HTTP endpoints |
| Local browser | Direct in-process adapter for non-authoritative state or `MessagePort` to a worker |
| Native shell | IPC to a supervised local runtime and device broker |
| Cloudflare OS | Cap'n Web capability adapter to a Gadget server or Gatekeeper |
| Tests | Deterministic in-memory capabilities and recorded effects |

A local adapter may avoid serialization as an optimization, but it must preserve the same permission, failure, cancellation, and asynchronous semantics as remote placement.

## Failure Model

The public contract uses a small normalized failure taxonomy while preserving provider diagnostics in trusted traces:

```ts
type VibeRuntimeErrorCode =
  | "invalid_request"
  | "unauthenticated"
  | "permission_denied"
  | "capability_unavailable"
  | "conflict"
  | "deadline_exceeded"
  | "cancelled"
  | "resource_exhausted"
  | "provider_failure"
  | "protocol_mismatch";
```

The app must be able to distinguish denial, temporary unavailability, conflict, and indeterminate completion. Provider-specific errors may be attached as safe diagnostics but must not leak credentials or private host details.

For a mutation whose response is lost, the runtime must not claim failure if completion is unknown. The operation contract must provide an idempotency key, a status lookup, or an explicit `indeterminate` outcome before automatic retries are allowed.

## Implementation Plan

### Phase 0: Contract spike

- [ ] Define a small `VibeRuntime` interface with app identity, configuration, storage, one sample actor, one binding, commands, and events.
- [ ] Define schemas for request, result, normalized error, command status, and event envelopes.
- [ ] Build a deterministic in-memory adapter that records requested effects.
- [ ] Build a SvelteKit remote-functions adapter owned by trusted runtime code.
- [ ] Run one unchanged reference app against both adapters.
- [ ] Compare dedicated core remote functions with a generic dynamic-capability gateway.
- [ ] Record latency, bundle, caching, validation, error, and trace behavior before freezing protocol version 1.

### Phase 1: Enforced app boundary

- [ ] Build the app canvas as a separate trust and build boundary.
- [ ] Reject forbidden server imports and app-defined remote functions.
- [ ] Bind every request to trusted app, instance, user, and grant context.
- [ ] Partition storage by app instance and separate preview state from accepted state.
- [ ] Enforce initial payload, message-rate, execution-time, and storage quotas.
- [ ] Expose the same sample actor interface to the UI and a test agent.

### Phase 2: Durable commands and events

- [ ] Add durable command IDs, status lookup, cancellation, and restart recovery.
- [ ] Define ephemeral versus resumable event subscription behavior.
- [ ] Add idempotency handling for one consequential mutation.
- [ ] Add effect proposal and approval before one externally visible action.
- [ ] Verify that the UI can disconnect and reconnect without losing command ownership.

### Phase 3: Second real transport

- [ ] Implement either a local worker/IPC adapter or a Cloudflare OS Cap'n Web adapter.
- [ ] Run the reference app without source changes.
- [ ] Resolve semantic differences in the Vibe contract rather than leaking provider-specific behavior into app code.

## Open Questions

1. **Where should dedicated remote functions end and the dynamic gateway begin?**
   - Options: dedicated functions for every capability, one generic gateway, or a hybrid.
   - Current recommendation: dedicated functions for the small stable core and a generic gateway for versioned actors and bindings.
   - Trigger: Phase 0 measurements and schema-generation experience.

2. **Which event streams must be resumable?**
   - Options: all events are ephemeral, all are logged, or durability is declared per capability.
   - Current recommendation: declare durability per capability; command status and final result must survive reconnect even when progress events do not.
   - Trigger: the import and approval vertical slices.

3. **How are capability handles represented across adapters?**
   - Options: opaque runtime IDs, signed references, transport-native object capabilities, or an adapter-specific representation behind the SDK.
   - Current recommendation: expose typed handles in the SDK and keep their serialized representation adapter-private until two transports exist.
   - Trigger: Phase 3 implementation.

4. **Which local operations may use a direct adapter?**
   - Options: route every call through an isolate or allow direct calls for non-authoritative state.
   - Current recommendation: allow direct calls only when they preserve the same ownership and permission boundary; use worker or process messages wherever untrusted code reaches authority.
   - Trigger: profiling the local reference app and defining its isolate boundary.

## Success Criteria

- [ ] App source imports only Vibe-owned runtime contracts, not transport-specific APIs.
- [ ] One reference app runs unchanged through two adapters.
- [ ] A query, bounded mutation, durable command, and event subscription each demonstrate their documented semantics.
- [ ] An undeclared capability call is denied before provider execution.
- [ ] Invalid input is rejected at the transport boundary.
- [ ] App code cannot import trusted server modules or define a privileged remote endpoint.
- [ ] Credentials remain outside app source, runtime messages, logs, and portable packages.
- [ ] A command survives UI disconnect and exposes a later status or result.
- [ ] Traces correlate caller, capability, operation, policy decision, provider, and result without leaking secrets.
- [ ] The SvelteKit adapter can be replaced without changing the app-facing API.

## References

- [Vibe App Foundation](./2026-08-05-vibe-app-foundation.md) - stable shell, app canvas, source ownership, actors, and permission planes
- [What Vibe Can Learn From Cloudflare OS](./2026-08-06-cloudflare-os-lessons.md) - Cap'n Web capability RPC, shared UI and agent API, isolation, and Gatekeepers
- [Vibe Runtime Providers and Placement](./2026-08-06-vibe-runtime-providers.md) - provider-neutral placement, trust domains, device broker, and feature negotiation
- [Vibe Reusable Modules and Upgrades](./2026-08-06-vibe-reusable-modules-and-upgrades.md) - actor protocols, capability requests, versioning, and state ownership
- [SvelteKit remote functions](https://svelte.dev/docs/kit/remote-functions) - current framework behavior and experimental status
