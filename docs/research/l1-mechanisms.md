# L1 mechanism inventory

Pinned at `v1.31.2` ([Pin the Temporal Server version](https://github.com/0x63616c/temporalsimcity/issues/2)). All source citations are `https://github.com/temporalio/temporal/blob/v1.31.2/<path>` unless marked `(api repo)`, which cites `https://github.com/temporalio/api` — the public proto contract Temporal Server implements at this pin, read at its `master` ref since the api repo isn't itself version-pinned by the server tag. `docs/architecture/*.md` at the pinned tag was read first, per the map's standing preference.

**Altitude rule (L1) applied per item below:** in if the name is one an operator would say out loud *and* it has an observable behaviour or failure mode; out if it only exists as a Go type/internal implementation detail. Each entry ends with an explicit **In/Out** call.

---

## History shards & shard ownership

**Behaviour:** A cluster's Workflow Executions are partitioned into a fixed number of History Shards, set at cluster creation and never changed after. Each running History Service instance owns a subset of shards; ownership is coordinated by a `ShardController` using a Ringpop membership protocol. Owning a shard means synchronously handling RPCs for, and asynchronously running background task queues for, every workflow execution in it.

**Observable failure mode:** shard movement on host failure/rebalance — a shard whose owner dies gets reassigned, and in-flight requests against it fail with `ShardOwnershipLost` until the new owner takes over (`common/persistence/shardOwnershipLost`, referenced in `docs/architecture/retry.md`). This is a real, operator-visible event (shard reload) with a latency blip.

**Where it lives:** `service/history/shard/` (controller), `docs/architecture/history-service.md#history-shards`.

**In/Out: IN.** "Shard" is a word Temporal operators say constantly (`numHistoryShards`, shard rebalance dashboards); it has a concrete failure mode (ownership loss / shard reload).

---

## Internal task queues (History Service)

Not to be confused with user-facing Task Queues (Matching Service, below) — these are per-shard, internal to History. `docs/architecture/history-service.md#queue-processing`: "elsewhere in Temporal documentation, 'task queue' refers to the Task Queues of the Matching Service... the task queues we are discussing here are an internal implementation detail of the History Service."

- **Transfer Task Queue** — tasks available to execute immediately; executing one makes an RPC to Matching to enqueue a Workflow or Activity Task. This is the write side of the transactional-outbox pattern that keeps Matching consistent with History state. Source: `history-service.md#transfer-task-queue`, `service/history/transfer_queue_active_task_executor.go`.
- **Timer Task Queue** — tasks that become executable at a stored trigger time (e.g. a `workflow.sleep()` timer, or a schedule-to-start timeout). On expiry it appends `TimerFired` to history and creates a Transfer Task. Source: `history-service.md#timer-task-queue`.
- **Visibility Task Queue** — updates workflow metadata records in the visibility store (a separate database from the execution store). Source: `history-service.md#visibility-task-queue`.
- **Replication Task Queue** — mentioned but not detailed in `history-service.md` ("These queues include replication, visibility, and archival task queues, but here we focus on the Timer Task queue and the Transfer Task queue"); ships mutations to other clusters for multi-cluster replication.
- **Archival Task Queue** — drives history/visibility archival to blob storage on workflow close; implementation at `service/history/archival/` (`archiver.go`), executor at `service/history/archival_queue_task_executor.go`.
- **Outbound Task Queue** (new territory at this pin, powers Nexus/Callbacks) — a sharded "immediate" queue like Transfer, but its tasks target *external* destinations and are allowed long-running (~10s) HTTP calls. Grouped per (task type, source namespace, destination) via **multi-cursor** + a `GroupByScheduler`, each group passing through an in-memory buffer → concurrency limiter → rate limiter → circuit breaker. The circuit breaker trips after 5 consecutive `DestinationDownError`s, at which point tasks are rejected before even loading mutable state — an operator-visible degradation mode with its own metric (`circuit_breaker_executable_blocked`). Source: `docs/architecture/nexus.md#outbound-task-queue`.

**Observable failure mode (queues generally):** each queue processor periodically checkpoints an "ack level" — a compromise between reprocessing cost on shard reload and write load; a stuck/backlogged queue (e.g. Transfer backlog) is a real operational symptom operators watch via queue-depth metrics.

**In/Out:**
- **Transfer, Timer: IN.** Both have names used in real dashboards/logs (`transfer_task_queue`, timer task backlogs) and a direct observable behaviour (dispatch delay / timer firing).
- **Visibility, Replication, Archival task queues: IN**, but folded into their parent concepts below (Visibility store, Archival) rather than as standalone mechanisms — same underlying idea (a per-shard queue with a backlog), not worth tripling up in the city.
- **Outbound Task Queue / circuit breaker: IN, conditionally** — only if Nexus/Callbacks are in scope for the vertical slice (see Nexus call below). If Nexus is deferred past the slice, this stays fog.

---

## Workflow Execution History & Event Types

**Behaviour:** A linear, append-only sequence of History Events (branching only on `Reset` or conflict resolution) — the source of truth for a workflow. The full event sequence alone is sufficient to reconstruct all other execution state (mutable state, pending tasks). Event types are the publicly-exposed vocabulary of Temporal's event-sourcing model — e.g. `WorkflowExecutionStarted`, `WorkflowTaskScheduled/Started/Completed/Failed/Timeout`, `ActivityTaskScheduled/Started/Completed/Failed/TimedOut`, `TimerStarted/Fired`, `WorkflowExecutionContinuedAsNew`. Full enum: `temporal/api/enums/v1/event_type.proto` (api repo).

**Observable failure mode:** history growth without bound is *the* canonical Temporal failure mode operators are warned about — see History size/count limits below, which is really "what happens when this mechanism is abused."

**Where it lives:** `docs/architecture/history-service.md#workflow-execution-history`; persisted as `history_node`/`history_tree` tables in Cassandra.

**In/Out: IN.** "Event history" / "replay the history" is core Temporal vocabulary said out loud constantly; unbounded growth is its signature failure mode.

---

## Mutable State

**Behaviour:** Per-execution, in-memory-cached (and persisted) summary of current state — in-progress activities, timers, child workflows, etc. Could in principle be recomputed from History Events on every request, but that would be slow, so it's persisted and cached separately. On Cassandra it's stored in a single row (Cassandra has weak RDBMS support), mirroring the in-memory layout.

**Observable failure mode:** consistency is enforced via database transactions plus "the identity of the latest History Event reflected in Mutable State" — on a failed persist, the shard reloads mutable state from persistence rather than risk drift. A mutable-state/history mismatch (or its recovery reload) is the operator-visible edge here.

**Where it lives:** `docs/architecture/history-service.md#workflow-execution-mutable-state`; `service/history/workflow/mutable_state_impl.go`.

**In/Out: IN.** Directly named in error messages and logs (`mutable state`), and it's the thing that literally represents "where a workflow currently is" — essential to the metaphor.

---

## Workflow (context) cache & sticky queues

**Behaviour:** Two closely coupled mechanisms:
- **History-side workflow context cache**: an LRU-style, size- or count-bounded in-memory cache of loaded `WorkflowContext`/mutable-state objects on each History host, with a configurable eviction policy (`HistoryHostLevelCacheMaxSize` / `...MaxSizeBytes`, `HistoryCacheLimitSizeBased`, background eviction). Source: `service/history/workflow/cache/cache.go`.
- **Sticky task queue (worker-side cache)**: once a worker has a workflow's execution state cached in-process, the server can route subsequent Workflow Tasks for that run to a worker-private "sticky" task queue instead of the shared normal queue, avoiding a full history replay. The SDK worker side has its own bounded cache (`worker.stickyCacheSize` dynamic config, default `0` = SDK default; "shared between all workers in the process, cannot be changed after startup"). If the worker has evicted the workflow from its own cache (or a *different* worker polls the sticky queue), sticky affinity is lost and the server falls back to shipping the *full* history for a normal replay.

**Observable failure mode:** **sticky-queue eviction / cache miss** — a poller who lost the local cache entry (or a fresh worker who never had it) must fetch and replay the entire history instead of getting an incremental update; this is a real, named, latency-visible event (worth its own city detail — "the truck that forgot the load and has to go back for the whole manifest"). Also: `StickyScheduleToStartTimeout` — a Workflow Task sent to a sticky queue that nobody picks up within this timeout is redirected to the normal queue. Source: `service/history/workflow/task_generator.go` lines ~419-490 (`IsStickyTaskQueueSet`, `StickyScheduleToStartTimeout`).

**Where it lives:** `service/history/workflow/cache/`, `service/history/workflow/task_generator.go`, dynamic config `common/dynamicconfig/constants.go` (`WorkerStickyCacheSize`).

**In/Out: IN.** Both halves are named things operators tune and diagnose ("sticky queue timeout", "cache eviction"), with a sharp, teachable failure mode (full replay on a cache miss) that fits the depot/truck metaphor well — flag for the metaphor-mapping ticket.

---

## Task Queues & partitions (Matching Service)

**Behaviour:** The user-facing Task Queue concept — Matching Service instances hold Workflow/Activity Tasks that Worker processes long-poll for; the Frontend routes long-poll requests to the Matching instance that owns the requested queue. A single Task Queue serves many Workflow Executions. Each Task Queue is split into **partitions** (default 4) for throughput; partition ownership is reassignable, and partition metadata/backlog can load/unload from storage independently.

**Observable failure mode:** an idle/empty partition **forwards** its poller to its parent partition (and vice versa for a backlogged task with no local poller) so that a poller and a task can find each other across the partition tree; loading the root partition forces all sibling partitions to load too. Under-provisioned pollers relative to partitions is a real, diagnosable throughput problem.

**Where it lives:** `docs/architecture/matching-service.md`.

**In/Out: IN.** This *is* the depot metaphor's literal referent — task queue = depot, partitions = loading bays, forwarding = a bay redirecting a truck to the main dock. Core to the vertical slice.

---

## Task tokens

**Behaviour:** An opaque, server-issued token a worker must present back to complete/fail a task. `NewWorkflowTaskToken` carries `{namespaceID, workflowID, runID, scheduledEventID, startedEventId, startedTime, attempt, clock, version}`; `NewActivityTaskToken` carries a parallel activity-scoped set plus a `componentRef` (ties into CHASM's callback addressing, see below). The token is how the server matches a `RespondXTaskCompleted` call back to the exact in-flight task instance — not just the workflow, but *this* attempt of *this* task.

**Observable failure mode:** a token for an attempt that's no longer current (e.g. the task already timed out and a new attempt/transient task was created) is rejected — this is exactly the mechanism behind the speculative-workflow-task `StartTime`-in-token check (`docs/architecture/speculative-workflow-task.md#starttime-in-the-token`): "if it doesn't match the start time in mutable state, the Workflow Task can't be completed."

**Where it lives:** `common/tasktoken/token.go`.

**In/Out: OUT.** No operator says "task token" out loud as a debugging concept the way they say "task queue" or "shard" — it's plumbing that makes task completion safe, not a thing with its own dashboard or alert. Worth a one-line mention inside the Task Queue / Workflow Task explanation, not its own city fixture.

---

## Timeouts

**Behaviour:** `TimeoutType` enum (api repo, `temporal/api/enums/v1/workflow.proto`): `START_TO_CLOSE`, `SCHEDULE_TO_START`, `SCHEDULE_TO_CLOSE`, `HEARTBEAT`. Applied at both the Activity level (all four apply) and the Workflow Task level (schedule-to-start applies to sticky/speculative tasks; start-to-close bounds how long a worker gets to process a task once started). There are also whole-execution-scoped timeouts: **Workflow Execution Timeout** and **Workflow Run Timeout** (bound the whole execution / a single run respectively) — user-facing concepts, not separately detailed in the architecture docs at this pin.

**Observable failure mode:** each timeout type produces its own named timeout event/error an operator will see directly — e.g. `ActivityTaskTimedOut` with `TIMEOUT_TYPE_SCHEDULE_TO_START` means "no worker picked this up in time" (a capacity/poller problem), vs. `TIMEOUT_TYPE_HEARTBEAT` meaning "the activity stopped reporting progress" (a stuck/crashed activity). These are diagnostically distinct and operators reason about them differently.

**Where it lives:** `temporal/api/enums/v1/workflow.proto` (api repo); enforcement scattered through `service/history/timer_queue_active_task_executor.go` (Timer Task Queue, above) and, for sticky/speculative Workflow Tasks specifically, `docs/architecture/speculative-workflow-task.md`.

**In/Out: IN.** Named, operator-diagnosed, and each has a distinct failure signature — a natural fit for the city as "why did this truck/factory stall" scenarios.

---

## Retry policies

**Behaviour:** `RetryPolicy` (api repo, `temporal/api/common/v1/message.proto`): `initial_interval`, `backoff_coefficient` (exponential backoff multiplier, default cap ~100x initial interval via `maximum_interval`), `maximum_attempts` (0 = unlimited, bounded only by timeouts; 1 = disabled), `non_retryable_error_types` (exact-match error type list that stops retries early). Applied to Activities (and, per `docs/architecture/nexus.md`, to Nexus Operations and Callbacks via their own configurable policies) by server-side backoff machinery — `common/backoff` (`ThrottleRetry`/`ThrottleRetryContext`), governed by `backoff.IsRetryable` and `backoff.RetryPolicy`. A distinct retry policy exists for `ResourceExhausted` service errors.

**Observable failure mode:** exhausting `maximum_attempts` (or hitting a non-retryable error type, or a schedule-to-close timeout) is what turns a transient failure into a terminal `ActivityTaskFailed`/`WorkflowTaskFailed` — directly visible in history, and the whole reason retry policy tuning is an operator's first lever when something is flaky.

**Where it lives:** `common/backoff/`, `docs/architecture/retry.md`; proto: `temporal/api/common/v1/message.proto` (api repo).

**In/Out: IN.** "Retry policy", "max attempts", "non-retryable error" are said out loud constantly and have sharp, teachable failure modes (giving up vs. retrying forever within timeout).

---

## Persistence

**Behaviour:** The abstracted storage layer behind shards, mutable state, and history — Cassandra is the primary-supported backend (weakest RDBMS feature set, hence single-row mutable-state layout), with SQL implementations for MySQL/Postgres/SQLite also existing. History Shards map 1:1 to partitions of the `executions` table on Cassandra.

**Observable failure mode:** persistence errors are what force a shard to reload mutable state from storage (see Mutable State); persistence latency/availability directly gates every state transition, since transitions commit via a two-write transaction (append events, then transactionally update mutable state + tasks).

**Where it lives:** `schema/` directory (Cassandra CQL), `common/persistence/{cassandra,sql}/`; `docs/architecture/history-service.md#history-shards` and `#consistency-guarantees`.

**In/Out: OUT — as its own city fixture, IN as substrate.** "Persistence" isn't a thing an operator points at with an observable failure mode distinct from the shard/mutable-state/queue failures already covered — it's the ground everything else stands on. Represent it as the literal ground/soil layer under the city rather than a mechanism with its own behaviour, or fold it into the shard's "what happens when storage is slow" story.

---

## Visibility store

**Behaviour:** A separate database (can be a different backend from the primary execution/history store) holding queryable workflow metadata records — the thing `ListWorkflowExecutions`/the Temporal Web UI's search reads from. Kept eventually consistent with the source of truth via the per-shard Visibility Task Queue (above), which is why visibility can visibly lag mutable state.

**Observable failure mode:** visibility lag — a workflow's state visibly changes (e.g. it completes) before the visibility store reflects it, because the Visibility Task Queue processes asynchronously; a stuck visibility queue is a distinct, separately-alertable backlog from a stuck transfer/timer queue.

**Where it lives:** `docs/architecture/history-service.md#visibility-task-queue`; `common/persistence/visibility/`.

**In/Out: IN.** Named ("visibility"), independently operable/failable, and a good teaching moment about eventual consistency inside an otherwise strongly-consistent-feeling system.

---

## Archival

**Behaviour:** On workflow close, history and/or visibility records can be archived to a separate blob storage backend (`Targets []Target`, `HistoryURI`/`VisibilityURI` in `Request`), driven by the per-shard Archival Task Queue and the `service/history/archival.Archiver`.

**Observable failure mode:** archival failure means a closed workflow's history/visibility record isn't durably moved to cold storage before namespace retention deletes the live copy — a real data-loss-adjacent risk operators configure retention/archival policy specifically to avoid.

**Where it lives:** `service/history/archival/archiver.go`, `archival_queue_task_executor.go`.

**In/Out: IN, but low priority for the vertical slice.** Named and has a real failure mode, but it's a lifecycle-tail concern (what happens *after* a workflow closes and its retention period passes) rather than something visible in the gate→depot→river→records-hall slice. Good fit for a later "records hall" district detail, not blocking.

---

## Continue-as-new

**Behaviour:** A workflow can atomically finish its current run and immediately start a new run with a fresh, empty history, carrying forward only explicitly-passed input — the mechanism for keeping ostensibly "infinite" workflows (loops, long-running entities) from growing history without bound. Produces `WorkflowExecutionContinuedAsNew` (event type enum value 28, api repo) and, per CHASM's ExecutionKey model, a new `RunID` under the same `BusinessID`/WorkflowID.

**Observable failure mode:** *not* continuing-as-new when history is growing is what triggers `HistorySizeSuggestContinueAsNew` (see History size/count limits) — the two mechanisms are a matched pair: one is the escape hatch, the other is the warning that you need it.

**Where it lives:** `temporal/api/enums/v1/event_type.proto` (api repo, `EVENT_TYPE_WORKFLOW_EXECUTION_CONTINUED_AS_NEW`); CHASM `docs/architecture/chasm.md#executionkey-identity` for the RunID mechanics.

**In/Out: IN.** Directly named in SDKs and docs, has a clear before/after in the history record, and pairs naturally with the history-size-limit failure mode as a single teaching beat ("the workflow that reset its own truck manifest before it got too heavy to carry").

---

## History size / count limits

**Behaviour:** Namespace-scoped, per-execution caps enforced server-side: `HistorySizeLimitWarn` (10 MiB) / `HistorySizeLimitError` (50 MiB) on total history byte size; `HistoryCountLimitWarn` (10,240) / `HistoryCountLimitError` (51,200) on event count; `HistorySizeSuggestContinueAsNew` (4 MiB) — below the warn threshold, this is where the server starts telling the SDK "you should continue-as-new soon" via the Workflow Task Started event.

**Observable failure mode:** crossing `*LimitWarn` logs/metrics a warning; crossing `*LimitError` **fails the workflow outright** — a hard, operator-visible termination distinct from a normal completion or a task failure. This is arguably Temporal's most teachable "don't do this" failure mode.

**Where it lives:** `common/dynamicconfig/constants.go` lines 365-389 (`limit.historySize.*`, `limit.historyCount.*`).

**In/Out: IN.** Named limits with concrete numeric thresholds and a sharp, terminal failure mode (workflow forcibly failed) — a natural "why did my workflow die" scenario, and the direct payoff for teaching continue-as-new.

---

## Downstream in/out calls flagged by the version-pin ticket

The pin ticket ([Pin the Temporal Server version](https://github.com/0x63616c/temporalsimcity/issues/2)) asked this ticket to make an explicit in/out call on three things new-or-newly-unconditional at `v1.31.2`:

- **Serverless Workers / Worker Controller Instance (WCI).** Pre-release, `workercontroller.enabled` **disabled by default**, dispatches Task Queue work to AWS Lambda. It has a name an operator would say, but no default-on observable behaviour at this pin (nothing to fail, nothing running unless opted in). **OUT** — no default footprint, no failure mode to show without narrating an opt-in feature most users will never enable. Revisit only if a future pin makes it default-on.
- **Nexus (and its Outbound Task Queue / Callbacks machinery).** Unconditionally on since 1.31.0 (flag removed) — every cluster at this pin is running it whether or not a workflow uses it. It has real names (Nexus Operation, Endpoint, Callback), real state machines, and a real, distinctive failure mode (circuit breaker trip on a down destination, blocking even mutable-state loads). **IN, conditionally** — in as a mechanism worth documenting in the model doc regardless (it's part of what's actually running), but whether it gets its own *city district* in the vertical slice is a scope call for the metaphor-mapping ticket, not this one: the slice (gate → depot → river → records hall) doesn't obviously need cross-namespace calls to demonstrate core mechanics. Recommend: describe it in prose/model doc, defer its geometry.
- **CHASM.** A generalization of the state-machine/task/persistence machinery that Workflows, Schedules, and Nexus Operations are themselves now built on (per `docs/architecture/chasm.md` and `schedules.md`'s "all related implementation code located in `chasm/lib/scheduler`"). It has internal names (Component, Node, Transition, VersionedTransition, ComponentRef) but none of them are things an *operator* says out loud — they're the framework the server's own developers use to build Workflow, Scheduler, and Nexus Operation as three of its "Application State Machines." Under the altitude rule this reads as `historyEngineImpl`-shaped: an internal abstraction, not a user-facing mechanism. **OUT** as its own mechanism — but note for the domain-modeling session that Workflow/Schedule/Nexus-Operation sharing one underlying framework may be worth a single explanatory sentence in the model doc (why they behave so similarly), without exposing CHASM's own vocabulary in the city.

---

## Summary table

| Mechanism | In/Out |
|---|---|
| History shards & shard ownership | IN |
| Transfer Task Queue | IN |
| Timer Task Queue | IN |
| Visibility Task Queue | IN (folded into Visibility store) |
| Replication Task Queue | IN (folded into replication/multi-cluster, still fog) |
| Archival Task Queue | IN (folded into Archival) |
| Outbound Task Queue / circuit breaker | IN, conditional on Nexus scope call |
| Workflow Execution History & event types | IN |
| Mutable State | IN |
| Workflow (context) cache | IN |
| Sticky task queue | IN |
| Task Queues & partitions | IN |
| Task tokens | OUT |
| Timeouts (schedule-to-start/close, start-to-close, heartbeat) | IN |
| Retry policies | IN |
| Persistence | OUT as fixture / IN as substrate |
| Visibility store | IN |
| Archival | IN, low priority for slice |
| Continue-as-new | IN |
| History size/count limits | IN |
| Serverless Workers / WCI | OUT |
| Nexus | IN for model doc, geometry deferred |
| CHASM | OUT |
