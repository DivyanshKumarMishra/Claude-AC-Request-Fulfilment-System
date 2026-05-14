# AC Service Request Fulfillment System — Spec

## 1. Overview

This system assigns service requests to available agents based on proximity, with correctness guarantees under concurrent requests and duplicate submissions.

The system focuses on:
- Accurate agent assignment
- Concurrency safety
- Idempotent request handling
- Clear service-area boundaries

---

## 2. Goals

- Assign the nearest available agent to a request
- Ensure no agent is assigned to multiple requests simultaneously
- Handle concurrent assignment safely
- Prevent duplicate request processing using idempotency
- Distinguish "no agents free right now" from "we don't operate here"
- Support agent lifecycle (registration, location updates, on/off duty)

---

## 3. Non-Goals

- Payments
- Authentication / authorization
- UI or frontend
- Real-time tracking of agents
- Distributed / multi-region deployment
- Persistence across restarts
- Queueing of unfulfilled requests

---

## 4. Key Concepts

### 4.1 Service Request
A request created by a client requiring assignment to an agent.

### 4.2 Agent
A service provider with a location and availability status.

### 4.3 Assignment
An immutable record mapping a request to an agent at a point in time.

### 4.4 Idempotency
Ensuring the same logical request (identified by `idempotency_key`) is not processed multiple times, while still allowing retries to progress a pending request.

### 4.5 Service Zone
A configured operating area (center + radius). Defines where the system operates and how far an agent may be dispatched.

---

## 5. Data Model

### 5.1 ServiceRequest
- `id` (UUID, server-generated)
- `location`: `{ lat, lng }`
- `status`: `CREATED | ASSIGNED | COMPLETED | CANCELLED`
- `idempotency_key` (UUID v4, client-supplied, unique)
- `created_at`
- `assigned_at` (nullable)
- `completed_at` (nullable)
- `cancelled_at` (nullable)

### 5.2 Agent
- `id` (UUID, server-generated)
- `location`: `{ lat, lng }`
- `status`: `OFFLINE | AVAILABLE | BUSY`
- `last_assigned_at` (nullable; used for tie-breaking)

### 5.3 Assignment
- `request_id` (primary key — one assignment per request, ever)
- `agent_id`
- `assigned_at`

Assignment rows are **immutable** once written. Completion and cancellation status live on `ServiceRequest`.

### 5.4 Service Zone (config, not runtime state)
- `name`
- `center`: `{ lat, lng }`
- `radius_km`

Loaded at boot from `config/service_zones.json`. Not mutable at runtime.

---

## 6. Functional Requirements

### 6.1 Create Request — `POST /requests`

**Input**
- `location`: `{ lat, lng }`
- `idempotency_key`: UUID v4

**Behavior**
1. Validate input (see §11).
2. If `idempotency_key` matches an existing request: skip to step 5 with that request.
3. Determine the **service zone** for the location:
   - If no zone contains the location → return `422 UNSERVICEABLE`. **Do not persist the request.**
4. Create a new `ServiceRequest` with `status = CREATED`.
5. If request is `CREATED`, attempt assignment (see §6.2). If already `ASSIGNED`/`COMPLETED`/`CANCELLED`, skip assignment.
6. Return the current state of the request plus its assignment (if any).

Replay of the same `idempotency_key`:
- Returns the existing request's current state.
- If the request is still `CREATED`, **re-attempts assignment** (so a retry can succeed once agents become available).
- Never creates a duplicate row.

### 6.2 Assign Agent (internal operation)

Executed atomically under the global mutex.

1. Determine the request's zone (computed in 6.1 or re-resolved).
2. Find all agents where:
   - `status = AVAILABLE`
   - `haversine(agent.location, request.location) ≤ zone.radius_km`
3. If no eligible agents → return `NO_AGENTS_AVAILABLE` (request stays `CREATED`, no assignment).
4. Pick the **nearest** agent by Haversine distance.
5. Tie-break: agent with the oldest `last_assigned_at` (nulls — never assigned — win first).
6. Commit atomically:
   - `agent.status = BUSY`
   - `agent.last_assigned_at = now`
   - Create `Assignment(request_id, agent_id, now)`
   - `request.status = ASSIGNED`
   - `request.assigned_at = now`

If any step fails, no partial state is written (commit-at-end).

### 6.3 Outcomes for `POST /requests`

| Outcome | HTTP | Body | Persisted? |
|---|---|---|---|
| ASSIGNED | 200 | `{ request, assignment }` | yes |
| NO_AGENTS_AVAILABLE | 200 | `{ request, assignment: null, reason: "no_agents_available" }` | yes (status `CREATED`) |
| UNSERVICEABLE | 422 | `{ error: "unserviceable_location" }` | **no** |

### 6.4 Complete Request — `POST /requests/:id/complete`

Triggered by the agent on completing the job.

- Allowed only when `request.status = ASSIGNED`.
- Effect:
  - `request.status = COMPLETED`, `completed_at = now`
  - `agent.status = AVAILABLE`
- Invalid state → `409 Conflict`.

### 6.5 Cancel Request — `POST /requests/:id/cancel`

May be triggered by either party. Optional `cancelled_by` field in body (`"client" | "agent"`) for record-keeping; not validated.

- Allowed when `request.status ∈ {CREATED, ASSIGNED}`.
- Effect:
  - `request.status = CANCELLED`, `cancelled_at = now`
  - If `ASSIGNED`: `agent.status = AVAILABLE` (the agent does not get `last_assigned_at` reset).
- Invalid state (`COMPLETED` or `CANCELLED`) → `409 Conflict`.

### 6.6 Get Request — `GET /requests/:id`

Returns `{ request, assignment }`. 404 if not found.

---

## 7. Agent Endpoints

### 7.1 Register — `POST /agents`

**Input**
- `location`: `{ lat, lng }`

**Behavior**
- Server generates `id` (UUID v4).
- Initial `status = OFFLINE`.
- Returns the new agent.
- Not idempotent — re-registering creates a new agent.

### 7.2 Update Location — `PATCH /agents/:id/location`

**Input**
- `location`: `{ lat, lng }`

**Behavior**
- Allowed when agent's status is `AVAILABLE` or `BUSY`.
- Rejected with `409` when `OFFLINE` (use `go-online` to refresh location instead).
- Does **not** acquire the global mutex — last write wins.

### 7.3 Go Online — `POST /agents/:id/go-online`

**Input**
- `location`: `{ lat, lng }` — required; refreshes the agent's stored location.

**Behavior**
- Allowed when `status = OFFLINE`.
- Effect: `status = AVAILABLE`, `location = <new>`.
- `status = AVAILABLE` already → `200 no-op` (still refreshes location).
- `status = BUSY` → `409 Conflict`.

### 7.4 Go Offline — `POST /agents/:id/go-offline`

**Behavior**
- Allowed when `status = AVAILABLE`.
- `status = OFFLINE` already → `200 no-op`.
- `status = BUSY` → `409 Conflict` (agent must `complete` or `cancel` the current request first).

### 7.5 Get Agent — `GET /agents/:id`

Returns the agent. 404 if not found.

---

## 8. Distance Calculation

Use the **Haversine** formula over `(lat, lng)` treated as points on a sphere.

```
R = 6371 km
φ1, φ2 = lat1, lat2 (radians)
Δφ     = φ2 - φ1
Δλ     = (lng2 - lng1) (radians)

a = sin²(Δφ/2) + cos(φ1)·cos(φ2)·sin²(Δλ/2)
d = 2 · R · atan2(√a, √(1−a))    # in km
```

---

## 9. Service Zones

- Loaded from `config/service_zones.json` at boot.
- Schema: `[{ name: string, center: {lat, lng}, radius_km: number }, ...]`.
- A request location is **in zone Z** iff `haversine(location, Z.center) ≤ Z.radius_km`.
- A request is **unserviceable** iff it is in no zone.
- If a location falls in multiple zones (overlap), use the zone whose center is **closest** to the request.
- The zone's `radius_km` is used for **both**:
  - The boundary check above (is the request in our service area?), and
  - The maximum agent-to-request distance for dispatch eligibility.

Agents may register from any coordinates — there is no constraint that an agent be inside a zone. Eligibility depends only on the agent-to-request distance.

---

## 10. Concurrency Model

- The system runs as a single process with all state in memory.
- A single **global mutex** guards every state-changing operation:
  - Create request + assign
  - Complete
  - Cancel
  - Go-online, go-offline
- Pure reads (`GET` endpoints) and `PATCH /agents/:id/location` are **not** under the mutex.
- All assignment writes use a **commit-at-end** pattern: gather the intended state changes in local variables, then apply them in one block. If any pre-commit step throws, no state is modified.

Under this model:
- Two concurrent `POST /requests` cannot both grab the same agent.
- An agent cannot change status concurrently with being selected.
- A location update racing with an assignment may cause assignment to use a slightly stale location — acceptable, no correctness violation.

---

## 11. Idempotency

- `idempotency_key` is required on `POST /requests`.
- Must be a valid **UUID v4**; otherwise `400`.
- Acts as a **uniqueness constraint** on `ServiceRequest`, not a response cache.
- **No TTL** — keys live for the lifetime of the process.
- **Permanent binding** — once a request exists for a key, that key cannot be reused for a new request, even after the original is `COMPLETED` or `CANCELLED`.

Replay behavior:

| Current state | Replay returns |
|---|---|
| `CREATED` | Re-attempts assignment, returns current state |
| `ASSIGNED` | Existing request + assignment (no work done) |
| `COMPLETED` | Existing request + assignment + completion (200) |
| `CANCELLED` | Existing request (200) |

Other endpoints are not idempotent and do not accept `idempotency_key`.

---

## 12. State Transitions

### ServiceRequest
```
        ┌──── assign ───────────► ASSIGNED ──── complete ───► COMPLETED
CREATED │                            │
        │                            └─── cancel ───┐
        └────── cancel ────────────────────────────►│
                                                    ▼
                                                CANCELLED
```

Terminal states (`COMPLETED`, `CANCELLED`) are immutable.

### Agent
```
                  go-online (+location)
        OFFLINE  ─────────────────────►  AVAILABLE
           ▲                                │  ▲
           │ go-offline                     │  │
           └────────────────────────────────┘  │
                                               │
                              assign  ◄───►  complete / cancel
                                               │
                                            BUSY
```

`BUSY → OFFLINE` is **rejected** (`409`). The agent must `complete` or `cancel` first.

---

## 13. Validation

| Field | Rule |
|---|---|
| `location.lat` | number, `[-90, 90]` |
| `location.lng` | number, `[-180, 180]` |
| `idempotency_key` | matches UUID v4 regex |
| `agent_id`, `request_id` (in URLs) | matches UUID v4 regex |

Any violation → `400 Bad Request` with `{ error: "validation_failed", field, reason }`.

---

## 14. Error Handling

| Case | HTTP |
|---|---|
| Malformed input | `400` |
| Resource not found | `404` |
| Invalid state transition (e.g., complete a `CREATED` request, BUSY agent going offline) | `409` |
| Idempotent no-op (e.g., `go-online` on already-`AVAILABLE` agent) | `200` |
| Unserviceable location | `422` |
| Internal failure during assignment | rollback in-memory state, `500` |

Error body shape: `{ error: <slug>, ... }`.

---

## 15. Tie-breaking & Load Distribution

When multiple agents are equidistant from a request:

- Primary order: ascending Haversine distance.
- Secondary order: ascending `last_assigned_at` (nulls first — agents who have never been assigned win).

This provides light load balancing without compromising the "nearest first" rule.

---

## 16. Assumptions

- System runs as a single instance.
- In-memory storage is sufficient; restart wipes all state.
- Number of agents is relatively small (linear scan over all agents per assignment is acceptable).
- Service zone configuration is static for the process lifetime.
- Clocks are monotonic and consistent (single process).

---

## 17. Resolved Open Questions

| Question | Resolution |
|---|---|
| Two agents equidistant? | Tie-break by least-recently-assigned (§15). |
| Should requests be queued if no agents available? | No queue. Client retries with the same `idempotency_key`; retry re-attempts assignment (§11). |
| What if assignment fails midway? | Commit-at-end pattern with in-memory rollback (§10). No recovery logic. |
| Should agent selection consider load balancing instead of only distance? | Distance remains primary; load balancing folded into tie-break only (§15). |
| How to handle agent becoming unavailable during assignment? | Impossible under the global mutex. `BUSY → OFFLINE` is explicitly rejected with `409` (§12). |
