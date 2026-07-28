# Temporal Server services at `v1.31.2`

All claims below are read directly from `github.com/temporalio/temporal` at tag `v1.31.2`
(commit `19a774302c613da9adc4436ab14278ccdca8e0a5`), either from `docs/architecture/` at that
tag (first-party, correctly-versioned) or from the Go source itself. No claim is drawn from
model memory.

## The service list, straight from source

`common/primitives/role.go` at this tag defines the roles:

```go
const (
	AllServices             ServiceName = "all"
	FrontendService         ServiceName = "frontend"
	InternalFrontendService ServiceName = "internal-frontend"
	HistoryService          ServiceName = "history"
	MatchingService         ServiceName = "matching"
	WorkerService           ServiceName = "worker"
	ServerService           ServiceName = "server"
	UnitTestService         ServiceName = "unittest"
)
```

`temporal/fx.go`'s `TopLevelModule` wires exactly five service providers: `HistoryServiceProvider`,
`MatchingServiceProvider`, `FrontendServiceProvider`, `InternalFrontendServiceProvider`,
`WorkerServiceProvider`. `all` / `server` aren't separate runtimes — they're the "run every
service role in one process" mode used by `temporal server start-dev` and single-binary deploys;
`unittest` is test scaffolding only.

**So the answer to "is Frontend/History/Matching/Worker/internal-frontend/UI complete and
current" is: five, not six.** The UI is not a Temporal Server service at all — see below.

## The five services

### 1. Frontend (`service/frontend`)

- **Responsibility:** the cluster's single external gRPC entrypoint. Implements `WorkflowService`
  (the public API: Start/Signal/Query/Update/Cancel/Reset workflows, poll for tasks, etc.),
  `OperatorService`, and the admin service. Also serves HTTP (REST/OpenAPI + Nexus callback
  handling) — see `http_api_server.go`, `nexus_http_handler.go`, `openapi_http_handler.go`,
  `admin_handler.go`, `operator_handler.go`, `workflow_handler.go` in `service/frontend/`.
- **Statefulness:** stateless (routes / validates / rate-limits, then forwards). Any instance can
  serve any request.
- **Scaling:** horizontal — add instances behind a load balancer; no partitioning.
- **Talks to:** History (per-shard RPCs to drive workflow state), Matching (poll/task RPCs it
  proxies from Workers), Visibility store, persistence for namespace metadata.
- **Default port (docker.yaml):** gRPC `7233`, HTTP `7243`, ringpop/membership `6933`.

### 2. Internal-frontend (`service/frontend`, same code, different identity)

This is the one departure from "five separately-implemented services" and it's a real, documented
finding, not an assumption: **internal-frontend is not separate code.** `temporal/fx.go`'s
`InternalFrontendServiceProvider` calls the exact same `genericFrontendServiceProvider` /
`frontend.Module` as the public `FrontendServiceProvider`, just with `primitives.InternalFrontendService`
as the service name. The only behavioural difference is wired at that call site:

```go
fx.Decorate(func() authorization.ClaimMapper {
    switch serviceName {
    case primitives.FrontendService:
        return params.ClaimMapper                    // customer-configured (e.g. JWT/OAuth)
    case primitives.InternalFrontendService:
        return authorization.NewInternalClaimMapper()  // always-authorized, no external auth
    }
})
```

- **Responsibility:** the same `WorkflowService` API as Frontend, but reachable only from inside
  the cluster's own network, and every caller is auto-authorized (`NewInternalClaimMapper`)
  instead of going through the customer's configured authorizer/claim mapper.
- **Why it exists:** so the cluster's own internal system workers (see Worker Service below) never
  have to hold or refresh customer-facing auth credentials just to call their own `WorkflowService`
  — relevant wherever an authorizer/claim mapper is actually configured (e.g. Temporal Cloud).
- **Statefulness / scaling:** identical to Frontend — stateless, horizontally scaled.
- **Off by default:** in the stock `config/docker.yaml` at this tag, the entire `internal-frontend:`
  block is gated behind `{{- if env "USE_INTERNAL_FRONTEND" }}` — it does not run unless opted in.
  Under the map's altitude rule (name an operator would say + observable behaviour/failure mode)
  this is borderline: it has the name, and a real failure mode (auth bypass boundary), but it's
  invisible in the common self-hosted case. Flagging for **Lock the altitude**, not deciding here.
- **Default port when enabled:** gRPC `7236`, HTTP `7246`, ringpop/membership `6936`.

### 3. History (`service/history`)

- **Responsibility:** owns Workflow Execution state. Handles two RPC categories: (1) requests
  from the User Application relayed via Frontend (Start/Cancel/Query/Update/Signal/Reset — see
  `service/history/api/`), and (2) requests from Workers reporting task completion
  (`RespondWorkflowTaskCompleted`, `RespondActivityTaskCompleted`, etc.). On each, it appends
  Workflow History Events, updates **Mutable State** (the cached/persisted summary of in-flight
  activities, timers, child workflows), and enqueues internal tasks.
- **Statefulness: the one genuinely stateful, partitioned service.** Workflow Executions are
  partitioned into a fixed number of **History Shards**, decided at cluster creation and never
  changed afterward. Each History Service *instance* owns a subset of shards; ownership is
  coordinated by a `ShardController` using the Ringpop membership protocol. "Owning" a shard means
  synchronously handling RPCs for its executions and asynchronously running that shard's internal
  task queues (Transfer, Timer, Visibility, Replication, Archival — see
  `docs/architecture/history-service.md`).
- **Internal task queues (not the user-facing Task Queue concept):**
  - **Transfer Task Queue** — "this needs to happen now"; execution results in an RPC to Matching
    to enqueue a Workflow or Activity Task (the Transactional Outbox pattern: History writes the
    Transfer Task in the same DB transaction as the state change, so Matching enqueueing becomes
    eventually-consistent-but-guaranteed).
  - **Timer Task Queue** — "this needs to happen at time T" (workflow sleeps, activity
    schedule-to-start timeouts, retry timers, etc.).
  - Visibility, replication, archival queues exist per shard too but aren't detailed in the
    architecture doc beyond naming.
- **Scaling:** by increasing the number of History Service processes, which lets Ringpop
  redistribute the (fixed-count) shards across more owners — not by changing shard count.
- **Talks to:** persistence (shard-partitioned `executions` table — Cassandra/MySQL/Postgres/SQLite),
  Matching (via Transfer Task execution), other History shards during replication, Frontend (as
  the RPC origin for user-driven calls).
- **Default port:** gRPC `7234`, ringpop/membership `6934`. No HTTP port — never called directly
  by anything outside the cluster.

### 4. Matching (`service/matching`)

- **Responsibility:** owns Task Queues — the thing SDK Workers long-poll. Frontend routes
  worker poll requests to the Matching instance responsible for the requested queue; Matching
  hands back Workflow/Activity Tasks (or blocks the long-poll if there's nothing to give). A
  single Task Queue can carry tasks for many Workflow Executions.
- **Statefulness:** partially — holds in-memory task backlogs and outstanding-poller state per
  partition it owns; the durable backlog can also spill to persistence.
- **Scaling:** Task Queues are split into **partitions** (default 4, configurable) for throughput;
  partition ownership is reassignable and partitions load/unload independently. When a partition is
  idle (no tasks or no pollers) it **forwards** the poller or the task up to its **parent
  partition**, hoping to find a match there; for small partition counts the root is every child's
  direct parent, for larger counts the parents form a tree of depth > 2 converging on the root.
  Loading the root partition forces every other partition of that queue to load too, guaranteeing a
  forwarding path exists between a backlogged child and a long-waiting poller.
  (`docs/architecture/matching-service.md`, which notes "additional documentation of Matching
  Service internals is not yet available" — the partition/forwarding mechanic above is the whole
  of what's officially documented; deeper internals would need source-reading beyond this ticket's
  scope.)
- **Talks to:** History (Transfer Task execution enqueues into Matching), SDK Workers (long-poll
  RPCs, proxied through Frontend), persistence (backlog spillover).
- **Default port:** gRPC `7235`, ringpop/membership `6935`. No HTTP port.

### 5. Worker Service — **internal**, not to be confused with SDK Workers (`service/worker`)

This is the load-bearing distinction the ticket calls out, and it's unambiguous in source: the
package doc-comment on `service/worker/service.go` says outright —

> `Service` represents the temporal-worker service. This service hosts all background processing
> needed for temporal cluster: Replicator: Handles applying replication tasks generated by remote
> clusters. Archiver: Handles archival of workflow histories.

- **What it is:** a Temporal Server component that is itself a Temporal *client* — it runs system
  workflows (using an SDK client internally, `sdkClientFactory`/`sdk.ClientFactory`) that implement
  cluster housekeeping. It is part of the cluster; operators deploy it as one of the five service
  roles.
- **What it runs**, all visible in `service/worker/service.go` `Start()`/subpackage list:
  - `scanner` — background scanners: Task Queue scanner, Build-Id scavenger, History scanner,
    Executions scanner (each individually toggleable via dynamic config).
  - `replicator` — applies replication tasks from remote clusters, started only when
    `clusterMetadata.IsGlobalNamespaceEnabled()` (i.e. multi-cluster replication is on).
  - `parentclosepolicy` — enforces parent-close-policy on child workflows, started only if
    `EnableParentClosePolicyWorker` is on.
  - `batcher` — backs the batch-operation APIs (bulk signal/cancel/terminate).
  - `deletenamespace`, `dlq`, `migration`, `addsearchattributes`, `workerdeployment` — other
    system-workflow-backed subsystems in the same package tree.
  - `perNamespaceWorkerManager` — starts SDK-client-based workers *per namespace* for
    namespace-scoped system workflows (e.g. schedules).
- **Statefulness:** effectively stateless as a process (it's a workflow client + local
  scanner/replicator goroutines); the durable state lives in the workflows it drives, in History.
- **Scaling:** horizontal; `membershipMonitor.GetResolver(primitives.WorkerService)` shows it
  participates in Ringpop membership like the other services, for coordination of its background
  jobs across instances.
- **Talks to:** its own cluster's Frontend or internal-frontend (as a Temporal SDK client, to
  start/signal the system workflows it hosts), History (`resource.HistoryClient`), Matching
  (`resource.MatchingClient`), persistence directly for some scanner/replicator paths.
- **Default port:** gRPC `7239`, ringpop/membership `6939`.

**vs. SDK Workers:** an SDK Worker is *user* code — a process the operator or developer runs
*outside* the cluster, built with an SDK (Go/Java/TypeScript/Python/.NET/PHP), that long-polls
Task Queues on Matching (via Frontend) to execute the user's own Workflow and Activity code.
Temporal never runs user code; the Worker Service is Temporal running *its own* code, inside the
cluster, to keep the cluster itself working. `docs/architecture/README.md` draws this line
explicitly: "User-hosted processes" (User Application + Worker, using an SDK as a gRPC client) vs.
"Temporal Cluster" (History Service shards + Matching Service — the doc's high-level list doesn't
even mention the Worker *Service* by that name, because from the perspective of "what does a user
application talk to," it's invisible; it only shows up once you read the source).

## The UI: not a Temporal Server service

There is no sixth service. `repos/temporalio/temporal` at `v1.31.2` has no `ui` directory and no
UI-serving code in `service/`. The Web UI (`temporalio/ui`, bundled into a distributable via
`temporalio/ui-server`) is a **separate project** that talks to the cluster as a gRPC/HTTP client
of Frontend, the same way `tctl`/`temporal` CLI or any SDK does. It ships bundled inside the
`temporalio/server` all-in-one Docker image and the `temporal server start-dev` binary for
convenience, but it is not one of the five `primitives.ServiceName` roles and does not join
Ringpop membership. **Correction to the ticket's premise:** treating it as a peer of
Frontend/History/Matching/Worker/internal-frontend would be wrong — it belongs in the city as an
external building (a viewer), not a cluster service.

## Membership and routing (Ringpop / hashing)

- All five cluster-internal service roles — frontend, internal-frontend, history, matching,
  worker — run a `membership.Monitor` (`common/membership/ringpop/monitor.go`), one Ringpop
  **SWIM** gossip ring per service role (`rings map[primitives.ServiceName]*serviceResolver` on
  the monitor — so it's five independent rings in one process space, not one ring for the whole
  cluster).
- Each service instance's `hostinfo.go`/`fx.go` under `common/membership` sets up gRPC name
  resolution (`grpc_resolver.go`) backed by the ring, so gRPC clients (e.g. Frontend calling
  History) resolve a logical service name to a live member via the ring rather than a fixed
  address list.
- Bootstrap is via a static host list (`discovery/statichosts`) or persisted cluster metadata;
  ongoing membership uses SWIM gossip with a ~20s healthy-heartbeat cutoff
  (`healthyHostLastHeartbeatCutoff`) and join/leave scheduling with a 15s clock-skew guard
  (`maxScheduledEventTimeSeconds`).
- For History specifically, ring membership is what `ShardController` watches to know which of
  the fixed shard set this instance currently owns (shards move to a new owner when the ring
  changes — a node join/leave triggers shard rebalancing, not a shard-count change).
- Every service also gets its own membership port in `docker.yaml`
  (frontend `6933`, internal-frontend `6936`, matching `6935`, history `6934`, worker `6939`) —
  those are the SWIM gossip ports, separate from the gRPC service ports.

## RPCs crossing service boundaries (summary)

| From → To | What crosses | Why |
|---|---|---|
| User App / SDK Worker → Frontend (or internal-frontend) | `WorkflowService` gRPC/HTTP (Start/Signal/Query/Update/Cancel, PollWorkflowTaskQueue, PollActivityTaskQueue, RespondWorkflowTaskCompleted, RespondActivityTaskCompleted, ...) | Public API surface |
| Frontend → History | per-shard `HistoryService` RPCs, routed by workflow-id → shard hash | drive workflow state transitions |
| Frontend → Matching | `MatchingService` RPCs, routed by task-queue → partition hash | relay Worker long-polls |
| History → Matching | `MatchingService` `AddWorkflowTask`/`AddActivityTask` (via Transfer Task execution) | enqueue tasks once History has durably recorded the need for them |
| Worker Service → Frontend/internal-frontend | `WorkflowService` (as an SDK client) | system workflows are ordinary workflows from the cluster's own point of view |
| Worker Service → History / Matching | direct `resource.HistoryClient` / `resource.MatchingClient` calls | scanners/replicator read state directly rather than only through workflows |
| History ↔ History (cross-cluster) | replication RPCs | multi-cluster replication, driven by the Worker Service's `replicator` when global namespaces are enabled |

## Confidence / gaps

- Matching's internal partition-forwarding algorithm is documented only at the level quoted above
  ("Additional Documentation of Matching Service internals is not yet available" is the doc's own
  disclaimer) — deeper detail would require reading `service/matching/` source directly, which
  this ticket didn't need to do to answer "what does each service do."
- CHASM, Nexus, and Serverless Workers/WCI are flagged (per ticket #2's resolution) as scope calls
  for a later ticket, not services in their own right — CHASM lives inside History/Frontend as a
  new execution substrate, Nexus is now-always-on routing logic inside Frontend, and WCI (disabled
  by default) is a new dispatch target *from* Matching, not a sixth process type.
