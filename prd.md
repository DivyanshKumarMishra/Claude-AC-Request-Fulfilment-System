# AC Service Request Fulfillment System — PRD

## Problem Statement

A service operator runs a fleet of field agents who fulfill on-demand service requests from clients (e.g., AC repair, on-site visits). Clients submit a request from a location and expect the system to dispatch the nearest available agent. Today there is no system: agent assignment is manual, prone to double-booking the same agent, prone to processing the same request twice when clients retry on flaky networks, and there's no consistent way to tell a client "we don't operate in your area" versus "no one is free right now — try again soon."

Operators need a backend that can:

- Take a service request with a location.
- Dispatch the nearest free agent within the operating area.
- Never assign one agent to two requests.
- Never create duplicate requests when a client retries.
- Cleanly communicate the three outcomes a client cares about: *assigned*, *busy — retry later*, *we don't serve here*.
- Track the agent through pickup → completion (or cancellation), and put them back in the pool.

## Solution

A single-instance, in-memory HTTP service that:

- Accepts service requests at `POST /requests` carrying a location and a client-supplied UUID v4 `idempotency_key`. Resolves the request to one of three outcomes synchronously and returns the result in the same response: **ASSIGNED** (200, with assignment), **NO_AGENTS_AVAILABLE** (200, no assignment, request persisted as `CREATED`), or **UNSERVICEABLE** (422, not persisted).
- Computes eligibility using **Haversine distance** in kilometers against a static set of **service zones** (each a center + radius_km, loaded once from config). A request is in-zone iff it lies within some zone's radius; agents are eligible iff they are within that same radius of the request. Overlapping zones resolve to the zone with the closest center.
- Picks the **nearest** eligible AVAILABLE agent, breaking ties by **least-recently-assigned** to spread load.
- Uses the `idempotency_key` as a uniqueness constraint on the `ServiceRequest` row. Replays return the current state of the existing request. A replay against a still-`CREATED` request **re-attempts assignment** — so clients can retry into a successful assignment once agents free up. The key is permanently bound to that request; it cannot be reused after the request reaches a terminal state.
- Manages agent lifecycle via `POST /agents` (register, starts `OFFLINE`), `PATCH /agents/:id/location`, `POST /agents/:id/go-online` (requires a fresh location — defeats stale coordinates on shift start), and `POST /agents/:id/go-offline`.
- Manages request lifecycle via `POST /requests/:id/complete` (agent-triggered) and `POST /requests/:id/cancel` (either party).
- Enforces a single state machine per entity. `BUSY → OFFLINE` is explicitly rejected with 409 — agents must complete or cancel first.
- Serializes all state-mutating operations under a single global mutex. Reads and location updates are lock-free. All assignment writes use a commit-at-end pattern: build new state in locals, apply atomically, abort cleanly on failure with no partial writes.

## User Stories

1. As a client, I want to submit a service request with my location and receive an immediate response telling me whether an agent has been assigned, so that I know if help is on the way.
2. As a client, I want the system to pick the *nearest* available agent automatically, so that I get help faster.
3. As a client, I want to retry my request safely after a network failure using the same idempotency key, so that I don't accidentally create two requests.
4. As a client, I want a retry to actually progress my pending request (assign an agent that has now become free), so that retrying isn't just a useless echo of the previous result.
5. As a client, I want a clear, distinct response when my location is outside the operating area, so that I know not to keep retrying.
6. As a client, I want a different response when the area is serviceable but everyone is busy, so that I know retrying later is worthwhile.
7. As a client, I want to fetch the current state of my request by ID, so that I can poll for updates.
8. As a client, I want to cancel a request that hasn't been completed yet, so that I can back out of a job I no longer need.
9. As a client, I want my cancel to release the agent back to the pool, so that the agent can be dispatched to someone else immediately.
10. As an agent, I want to be onboarded into the system with a registered identity and an initial location, so that I can later go on duty.
11. As an agent, I want to register without immediately being assignable, so that I can finish onboarding before receiving jobs.
12. As an agent, I want to explicitly go online at the start of my shift, providing my current location, so that I'm not dispatched from stale coordinates left over from yesterday.
13. As an agent, I want to update my location periodically while available or on a job, so that future dispatches use up-to-date positions.
14. As an agent, I want my location update while on a job to succeed without disrupting my current assignment, so that I can move freely without state churn.
15. As an agent, I want to be blocked from updating my location while offline, so that off-shift movements don't pollute dispatch data.
16. As an agent, I want to mark a request as complete when the job is done, so that I am returned to the available pool.
17. As an agent, I want to cancel a request I cannot fulfill, so that the client knows and the system releases me.
18. As an agent, I want to go offline at the end of my shift, so that I stop receiving dispatches.
19. As an agent, I want the system to refuse going offline while I'm on an active job, so that I can't accidentally abandon a customer.
20. As an agent, I want repeat go-online or go-offline calls to be idempotent no-ops, so that retries don't fail or flap my state.
21. As an operator, I want to define service zones in a config file (city center + radius), so that the system knows where I operate.
22. As an operator, I want requests outside all zones to be rejected without persisting anything, so that out-of-area noise doesn't fill the data store.
23. As an operator, I want overlapping zones to deterministically resolve to the closest center, so that edge-of-zone cases have predictable behavior.
24. As an operator, I want exactly one agent per request and one active request per agent at any time, so that I never double-book or double-dispatch.
25. As an operator, I want concurrent request submissions to be handled safely without two requests claiming the same agent, so that high-throughput periods don't corrupt state.
26. As an operator, I want load distributed lightly across equidistant agents (least-recently-assigned wins ties), so that a small number of well-placed agents don't dominate.
27. As an operator, I want all assignment work to be atomic — either all writes commit or none do, so that a mid-operation failure can never leave an agent BUSY without an assignment, or a request ASSIGNED to no one.
28. As an operator, I want the system to reject malformed inputs with 400 (bad lat/lng, non-UUID idempotency key), so that the data store only contains valid records.
29. As an operator, I want invalid state transitions to be rejected with 409 (completing a non-assigned request, going offline while BUSY), so that the state machine cannot be subverted.
30. As an operator, I want service zone configuration to be loaded once at boot and static for the process lifetime, so that the dispatch behavior is predictable and auditable.
31. As an operator, I want to fetch an agent's current state by ID, so that I can debug operational issues.
32. As an operator, I want unserviceable rejections to be cheap (no DB write, no idempotency-key consumption), so that bad clients cannot fill memory with garbage requests.

## Implementation Decisions

### Module breakdown

The system is decomposed into 10 modules, optimized for deep, pure cores around a thin coordinating shell:

1. **GeoDistance** — pure Haversine. Interface: `distanceKm(p1: LatLng, p2: LatLng): number`. No state, no dependencies.
2. **ServiceZoneResolver** — given a request location and the loaded zone list, returns the containing zone (closest-center-wins on overlap) or `null`. Pure given inputs.
3. **AgentSelector** — given a request, candidate agents, and the request's zone, returns the chosen agent or `null`. Encapsulates: filter by `status = AVAILABLE`, filter by `distance ≤ zone.radius_km`, pick minimum distance, tie-break by `last_assigned_at` ascending (nulls first). Pure given inputs.
4. **Validation** — UUID v4 regex check, lat/lng range checks. Pure.
5. **Clock** — `now(): Date`. Injected wherever timestamps are needed, so tests can fix time deterministically.
6. **ConfigLoader** — reads `config/service_zones.json` at boot, validates schema, returns the immutable zone list. Crashes on malformed config (fail-fast).
7. **Store** — owns the in-memory state and the single global mutex. Exposes:
   - `withLock(fn)` — serializes state-mutating operations.
   - Per-entity get/put/find operations for requests, agents, assignments, idempotency-key → request_id index.
   - All entity mutations go through here. `PATCH /agents/:id/location` and pure reads bypass the lock.
8. **RequestService** — composes `Store`, `GeoDistance`, `ServiceZoneResolver`, `AgentSelector`, `Validation`, `Clock`. Implements `createOrReplay`, `complete`, `cancel`. Owns the commit-at-end discipline (build the intended state in locals, apply in one block under the lock).
9. **AgentService** — composes `Store`, `Validation`, `Clock`. Implements `register`, `updateLocation`, `goOnline`, `goOffline`.
10. **HttpApi** — Express/Fastify (TBD by implementer) routes. Translates service results to HTTP status codes and bodies. No business logic beyond shape-of-error → status mapping.

### Module depth rationale

`GeoDistance`, `ServiceZoneResolver`, `AgentSelector`, and `Validation` are intentionally **deep**: they encapsulate the math-and-rules core behind tiny, stable interfaces. Their inputs and outputs are values, not handles, so they are trivially unit-testable and can be evolved (e.g., swapping tie-break heuristics) without touching the orchestration layer. The orchestration in `RequestService` / `AgentService` is medium-depth because its job is coordination, not invention.

### Key interface contracts

- **`AgentSelector.pick(request, agents, zone): Agent | null`** — returns null if no AVAILABLE agent is within `zone.radius_km`. Caller distinguishes this from UNSERVICEABLE by the presence/absence of a zone (resolved upstream).
- **`Store.withLock(fn)`** — guarantees mutual exclusion across all state-mutating operations system-wide. The callback receives a transaction-like handle that records intended writes; the lock holder commits them at the end. If the callback throws, nothing is written.
- **`RequestService.createOrReplay(input)`** returns a discriminated result: `{ outcome: "ASSIGNED", request, assignment }` | `{ outcome: "NO_AGENTS_AVAILABLE", request }` | `{ outcome: "UNSERVICEABLE" }`. The HTTP layer maps these to status codes and response bodies. UNSERVICEABLE intentionally returns no entity — no row is created.
- **Idempotency lookup happens after input validation but before zone resolution**: a malformed key never indexes into the store, and an unserviceable location never consumes a key.

### Schema notes

- `ServiceRequest.idempotency_key` is a unique index. The store maintains a secondary `Map<idempotency_key, request_id>` for O(1) lookup on replay.
- `Agent.last_assigned_at` is updated on `assign` only; not on complete or cancel. Tie-break wants "longest since dispatched," not "longest idle."
- `Assignment` has `request_id` as PK — one assignment per request, forever. Re-dispatching the same request is not supported in this PRD.

### Concurrency invariants enforced

- Two concurrent `POST /requests` cannot both grab the same agent: both block on the same mutex; the second one re-reads agent state and sees the now-`BUSY` agent filtered out.
- `complete`/`cancel`/`go-offline` cannot race with assignment selection: all under the mutex.
- Location updates intentionally do **not** acquire the mutex — they are last-write-wins on a single field. An assignment computation may use a slightly stale location; this is acceptable and does not affect correctness invariants (no double-assignment, no orphan state).

### Error → HTTP mapping (owned by HttpApi)

| Result | HTTP |
|---|---|
| Validation error from any service | 400 |
| Resource not found | 404 |
| Invalid state transition | 409 |
| Idempotent no-op | 200 |
| UNSERVICEABLE outcome | 422 |
| Unhandled exception (after rollback) | 500 |

### Open implementation choices (deferred to implementer)

- Language and HTTP framework: not constrained by this PRD. The module shapes above are framework-agnostic.
- Logging format: not in scope.
- Process management / health endpoints: not in scope.

## Testing Decisions

### What makes a good test in this codebase

- Tests assert **external observable behavior** of a module, not its internal call sequence. They feed inputs through the module's public interface and assert on returned values, on state changes visible through the module's own read API, or on outcomes of subsequent operations.
- Tests must be **deterministic**. Anywhere `now()` matters (tie-break by `last_assigned_at`, ordering of replays across state transitions), the `Clock` module is stubbed.
- Tests must not depend on internal field names, private helpers, or call-count assertions on collaborators. Refactoring an internal helper should never break a test.
- Each test names the **business invariant** it protects in its description (e.g., "one agent cannot be assigned to two concurrent requests"), not the function it exercises.

### Modules with heavy unit coverage

- **GeoDistance** — known-good Haversine values for several real city pairs; symmetry (`d(a,b) == d(b,a)`); zero for identical points; never negative.
- **ServiceZoneResolver** — point inside one zone; point outside all zones; point at exact radius boundary; point inside overlapping zones picks closer center; empty zone list.
- **AgentSelector** — no candidates; one candidate within range; one candidate just outside range; multiple candidates picks nearest; ties broken by oldest `last_assigned_at`; ties with all-null `last_assigned_at` are deterministic (insertion order or sort-stable behavior); BUSY/OFFLINE agents filtered out.
- **Validation** — boundary lat/lng values, non-numeric inputs, missing fields, non-UUID and non-v4 UUID strings.

### Modules with integration coverage through services

Exercised through `RequestService` and `AgentService` (using a real `Store` and stubbed `Clock`), without going through HTTP:

- Happy path: create → ASSIGNED → complete frees the agent.
- Idempotent replay: same key returns same request across all four current states (CREATED, ASSIGNED, COMPLETED, CANCELLED), with the CREATED replay re-attempting assignment and succeeding once an agent becomes free.
- UNSERVICEABLE: out-of-zone location returns the outcome without persisting any request and without consuming the idempotency key (a subsequent valid request with the same key proves this).
- NO_AGENTS_AVAILABLE: all agents BUSY or OFFLINE → request persisted as CREATED, retry succeeds after one is freed by `complete`.
- Concurrent assignment: two `createOrReplay` calls launched in parallel with one eligible agent — exactly one wins ASSIGNED, the other gets NO_AGENTS_AVAILABLE.
- Tie-break determinism: two agents at identical distance, different `last_assigned_at` → older one wins.
- All 409 invalid-transition paths: complete-a-CREATED, complete-a-COMPLETED, cancel-a-COMPLETED, cancel-a-CANCELLED, go-offline-while-BUSY, location-update-while-OFFLINE.
- Cancellation paths: cancel CREATED (no agent to release), cancel ASSIGNED (agent returns to AVAILABLE).
- Commit-at-end: inject a failure during the assignment commit block and assert no partial writes leaked (agent not BUSY, no Assignment row, request still CREATED).

### Light HTTP-layer coverage

Routes are tested only for:
- Correct status code for each service outcome.
- Path-parameter validation (bad UUID in URL → 400).
- Request-body shape errors → 400.

No business-logic re-testing at this layer.

### Skipped

- `Clock` — trivial interface.
- `ConfigLoader` — boot-time only; covered by integration tests that load a fixture config.
- `Store` — covered transitively by service-level tests.

### Prior art

This is a greenfield project; there is no prior art in this repo. The test style above mirrors common patterns for service-oriented systems with pure cores: heavy unit tests on pure modules + integration tests through service boundaries + thin route tests.

## Out of Scope

- Authentication and authorization. All endpoints are open; there is no notion of client identity beyond the `idempotency_key`.
- Payments, invoicing, pricing.
- Frontend, mobile app, agent app, dispatcher dashboard.
- Real-time tracking of agents (no streaming positions, no WebSocket).
- Push notifications to client or agent.
- Persistence across process restarts. All state lives in memory and is lost on shutdown.
- Multi-instance / distributed deployment. The global-mutex concurrency model only works in a single process.
- Queueing of unfulfilled requests. There is no background worker that retries assignments; the client must retry.
- Reassignment after agent unavailability mid-job. Once a request is ASSIGNED, the only terminal transitions are `complete` and `cancel`.
- Hot reload of service zone configuration.
- Rate limiting, abuse protection.
- Telemetry, metrics, structured logging conventions.
- Service zone shapes beyond circles (no polygons, no admin-boundary geofences).
- Surge / dynamic pricing or surge-based dispatch radius.
- Multi-tenancy or per-operator partitioning of agents.
- Dispatch by agent skill, vehicle type, or product category (the spec models agents as fungible).

## Further Notes

- **Why a global mutex, not per-agent locks?** The "small number of agents" assumption in the spec, combined with single-process in-memory deployment, makes the global mutex the simplest correct choice. Per-agent locking would force a careful protocol around tie-breaking and re-reads of agent state, in exchange for throughput gains that don't matter at this scale. The mutex is encapsulated inside `Store.withLock` so swapping it for finer-grained locking later is a localized change.
- **Why is `idempotency_key` UUID v4 and not free-form?** Without auth there is no client namespace; UUIDs effectively make collisions impossible without inventing a per-client identity. Free-form strings would require us to decide what happens when two different clients reuse `"abc123"`.
- **Why does retry re-attempt assignment on CREATED?** This is the practical answer to "what should retries do?" Strict response caching would freeze a `null` assignment forever, leaving the client no way to make progress short of using a new key for a fundamentally identical request. Re-attempting on retry preserves uniqueness (one request row), preserves the client's ability to use one key per logical request, and produces the obviously-right behavior when capacity opens up.
- **Why is the zone radius a single value used for both "in zone" and "dispatch range"?** Two separate values would introduce a class of confusing cases (request in-zone but no agent inside dispatch range despite being in the same zone). Unifying them makes "in our zone" mean exactly "an agent in our zone can serve you," which matches operator intent.
- **Why `BUSY → OFFLINE` is rejected rather than auto-cancelling?** Auto-cancel would silently transition a paying client's request to CANCELLED without their action, requiring notification machinery that is out of scope. Forcing the agent to `complete` or `cancel` first preserves clean two-party accountability.
- **Why no `last_assigned_at` reset on cancel?** The field exists to prevent a small set of well-placed agents from monopolizing a busy area. Resetting on cancel would reward an agent for cancelling — exactly the wrong incentive. It tracks "longest since dispatched," not "longest idle."
- **Single-process is a hard constraint of this design, not an incidental detail.** Several invariants (the global mutex, in-memory idempotency index, atomicity-by-language-runtime) depend on it. A multi-process or multi-region version is a different system, not an evolution of this one.
