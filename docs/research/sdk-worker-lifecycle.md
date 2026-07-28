# Tracing the SDK worker: long-poll, task lifecycle, and replay

**Server source**: `temporalio/temporal` at pinned tag `v1.31.2` (commit `19a774302c613da9adc4436ab14278ccdca8e0a5`), read via `docs/architecture/*.md` and `gh api repos/temporalio/temporal/contents/...?ref=v1.31.2`.

**SDK source**: `temporalio/sdk-go` at tag `v1.46.0` (latest stable as of 2026-07-27, commit `8e2c89c7c9d8d41f633bf039063422dd10c1fec5`), package `internal/`. This is the only SDK read for this ticket — chosen because it's first-party, and its worker-loop internals are plain, synchronous Go (no separate "core" layer to trace through, unlike sdk-typescript/sdk-python which delegate polling and state machines to `sdk-core` in Rust). **Where SDKs differ**: sdk-go, sdk-java, and sdk-.NET each implement their own poller/cache/replay loop independently; sdk-typescript, sdk-python, sdk-ruby, and sdk-php all delegate this entire machinery to `sdk-core`, a shared Rust library — so tracing sdk-go's `internal/internal_task_pollers.go` + `internal_task_handlers.go` gets you sdk-go's own mechanics, not a mechanics-in-common view across all SDKs. The wire protocol (gRPC calls, event/command shape) is identical everywhere since it's server-defined; the *client-side* orchestration (poller loop, sticky cache, replay bookkeeping) is what's SDK- or core-specific.

## 1. The long-poll protocol — what a poller is doing while it waits

A worker process runs a pool of poller goroutines per task queue (one pool for workflow tasks, one for activity tasks). Each poller goroutine is a plain loop (`baseWorker.runPoller`, `internal/internal_worker_base.go:488`):

1. Reserve a local concurrency slot (bounded by `MaxConcurrentWorkflowTaskPollers` / activity equivalent).
2. Call `taskWorker.taskPoller.PollTask()` — for workflow tasks this is `workflowTaskPoller.poll()` (`internal_task_pollers.go:1181`), which issues **one gRPC call**, `PollWorkflowTaskQueue`, and blocks on it. Activity pollers do the same with `PollActivityTaskQueue` (`internal_task_pollers.go:1398`).
3. That gRPC call is opened with an explicit long-poll context: `newGRPCContext(context.Background(), grpcTimeout(pollTaskServiceTimeOut), grpcLongPoll(true))` (`internal_task_pollers.go:318`). A comment there pins the mechanics precisely: *"Server returns empty task after `dynamicconfig.MatchingLongPollExpirationInterval` (default is 60 seconds). `pollTaskServiceTimeOut` should be `MatchingLongPollExpirationInterval` + some delta for full round trip."*
4. The RPC returns one of: a real task (`TaskToken` populated) — dispatch it to the task-processing dispatcher and loop back to step 1 immediately; or an **empty response** (`response == nil || len(response.TaskToken) == 0`, `internal_task_pollers.go:1195` / `:1424`) — the server held the connection open for up to ~60s and nothing arrived, so the poller just calls `PollTask()` again.

Mechanically this **is** the truck-idles-empty metaphor, with one refinement worth keeping precise for the city: it isn't a truck that drives away and comes back — it's a truck that stays parked at the loading bay with its dock door held open (one blocking RPC) for up to a minute, and the instant that connection closes — loaded or not — another one is opened immediately. There's no drive-away/return cycle client-side; concurrency is expressed as *how many trucks are parked at once* (the slot pool), not how often one truck leaves and comes back. Server-side, the connection sits inside Matching's sync-match/long-poll machinery (`docs/architecture/matching-service.md`): Matching splits each task queue into partitions (default 4) that a poller's request is routed to, and if a task arrives with no matching poller, it forwards up toward the root partition where long-lived pollers are more likely to be waiting.

## 2. Workflow task vs activity task

Both are dispatched from Matching, but originate differently on the server (`docs/architecture/workflow-lifecycle.md`, `history-service.md`):

- **Workflow task**: created whenever the History Service determines the workflow needs to make progress — on `StartWorkflowExecution`, after an activity completes/fails, after a timer fires, on signal/update, etc. History Service appends `WorkflowTaskScheduled` to history, updates Mutable State, and enqueues a **Transfer Task** that the shard's queue processor turns into a `AddWorkflowTask` call to Matching.
- **Activity task**: created only as a result of a workflow task response containing a `ScheduleActivityTask` command. Same Transfer Task → Matching path.

A poller receiving a workflow task gets a `PollWorkflowTaskQueueResponse` carrying (part or all of) the event history plus a task token; receiving that task server-side triggers `RecordWorkflowTaskStarted`, which appends `WorkflowTaskStarted` to history and starts a workflow-task-timeout timer. An activity task poller gets an activity input payload and a token; receipt triggers `RecordActivityTaskStarted` and an activity-timeout timer.

## 3. `RespondWorkflowTaskCompleted` and the commands it returns

After the worker advances the workflow (deterministically executing workflow code against the delivered events), it calls `RespondWorkflowTaskCompleted` with a list of **Commands** (`enum CommandType` in `temporalio/api`) — e.g. `ScheduleActivityTask`, `StartTimer`, `CompleteWorkflowExecution`, `StartChildWorkflowExecution`. The History Service handler (`workflow_task_handler_callbacks.go`) accumulates each command into Mutable State updates, appends the corresponding history events (`WorkflowTaskCompleted` plus one event per command, e.g. `ActivityTaskScheduled`), and creates whatever Transfer/Timer Tasks the commands imply. This closes the loop: `poll → advance → respond-with-commands → new events → new transfer task → next poll`, traced event-by-event in `workflow-lifecycle.md` for the `callActivity` example (steps 1–7 there).

Separately from events/commands there's a **message protocol** (`docs/architecture/message-protocol.md`), added for Workflow Update: `protocolpb.Message` values travel attached to a workflow task in both directions. It exists because Update needed a channel that could vanish without a trace if rejected (history is immutable, so a written event can't be un-written, and older SDKs assume 1 command → exactly 1 event, which Update rejection breaks). Queries are always processed last, after events and messages.

## 4. Sticky queues and what a sticky miss costs

**Purpose**: avoid re-shipping (and the worker re-processing) the full event history on every workflow task. Client-side, the SDK keeps a **workflow execution cache** keyed by run ID (`internal_worker_cache.go`) holding live `workflowExecutionContextImpl` state — this is the same cache referenced by "the workflow cache" in the ticket. Default size 10,000 executions per process (`defaultStickyCacheSize`, `SetStickyWorkflowCacheSize` overrides it), shared across all workers in one process via a refcounted `sharedWorkerCache`.

When a workflow task completes and the cache holds that execution, `workflowTaskPoller.getNextPollRequest()` (`internal_task_pollers.go:1116`) sets `StickyAttributes` on the *next* poll — the poller then polls a **sticky task queue**: a synthetic per-worker queue named from a UUID (`getWorkerTaskQueue(wtp.stickyUUID)`), with `Kind: TASK_QUEUE_KIND_STICKY` and `NormalName` pointing back at the real queue. Server-side this routes subsequent workflow tasks for that run back to *the same worker process*, which can advance the workflow from its cached in-memory state using just the *new* events since the cache checkpoint, instead of the whole history.

Poller mode (`Sticky` / `NonSticky` / `Mixed`, `internal_task_pollers.go:1107-1141`) governs which queue is polled; in `Mixed` mode (the common case) the poller prefers sticky whenever the sticky queue has a backlog, otherwise picks whichever of sticky/regular currently has fewer outstanding poll requests.

**Cost of a sticky miss** — three distinct failure/degrade paths surfaced in source:

- **Cache eviction under memory/size pressure**: the LRU (`cache.New(cacheSize-1, ...)`) evicts the least-recently-used execution, calling `wc.onEviction()` → `clearState()` (`internal_task_handlers.go:657-676`). The *next* task for that run then has no cached state; the worker must fetch and replay history from scratch (full replay, §5).
- **`StickyScheduleToStartTimeout`**: the sticky poll carries `ScheduleToStartTimeout: StickyScheduleToStartTimeout` (`internal_task_pollers.go:735`). If no sticky-affinitized worker polls in time (worker died, restarted, or is just busy), the server times out the sticky attempt and falls back to scheduling a *normal* (non-sticky) workflow task instead — any worker can then pick it up, but again with no client-side cache hit, so full history + full replay.
- **Speculative-task-on-evicted-cache edge case** (`speculative-workflow-task.md`): if a *speculative* workflow task (used for Workflow Update) lands on a sticky queue whose cache entry was just evicted, the worker's fallback `GetWorkflowExecutionHistory` call returns history *without* the speculative events (they were never persisted), producing a `premature end of stream` error. The worker recovers by failing that workflow task and clearing stickiness — visible in history as one extra failed-task attempt, "but it doesn't happen often."

In short: a sticky hit means "small diff, cheap"; a sticky miss means "full history fetch + full replay from event 1," which is strictly more server and worker work, not a protocol failure — the mechanism degrades gracefully by design.

## 5. Replay — precisely what re-executes, and why

This is decided per-event, not per-task. In `historyEventIterator` (`internal_task_handlers.go:282`):

```go
return event.GetEventId() <= eh.workflowTask.task.GetPreviousStartedEventId() || isCommandEvent(event.GetEventType())
```

and, at the point workflow-task processing walks the event list (`internal_task_handlers.go:1118`):

```go
isReplay := len(reorderedEvents) > 0 && reorderedHistory.IsReplayEvent(reorderedEvents[len(reorderedEvents)-1])
```

So: **every history event whose event ID is `<=` the workflow task's `PreviousStartedEventId`** (the `WorkflowTaskStarted` event ID of the *last workflow task the worker previously completed* for this run) **is a replay event**; everything after that ID is "live." `PreviousStartedEventId` is exactly the client-side cache checkpoint — with a sticky hit, the server only ships events after that ID in the first place, so there's little/nothing to replay; with a cache miss, the server ships the whole history and the worker replays everything up to (and not including) the new `WorkflowTaskStarted`.

**What "replay" means mechanically**: the SDK re-runs the workflow function from the top, feeding it the same sequence of events it saw before, so that all the same deterministic decision points (which branch, which command, in which order) are reached again — but instead of issuing new commands to the server, replayed command-generation is diffed against what the *history itself* already recorded happened. That diff is `matchReplayWithHistory` (`internal_task_handlers.go:~1579-1621`): if a replayed command doesn't correspond to the next history event (missing command, extra command, or wrong type), the SDK raises a `nondeterministic workflow` error tagged `[TMPRL1100]` and fails the workflow task rather than silently diverging.

**What determinism requires**, read directly off this mechanism rather than folklore: workflow code must, given the same sequence of history events, always regenerate the same sequence of commands. Concretely that rules out anything whose *outcome* could differ between the original run and a later replay — real time, random numbers, iterating unordered maps, network/file I/O, unbounded goroutine races — inside workflow code; all of that has to happen in Activities (whose *inputs and results* are captured as history events, so replay reads that result back off history rather than re-executing the side effect) or via SDK-provided deterministic primitives (`workflow.Now`, `workflow.SideEffect`, `workflow.Go`). `IsReplaying()` (`internal_event_handlers.go:737`, backed by the same `isReplay` flag) exists specifically so workflow code can suppress replay-visible side effects it must run once (e.g., a `defer` that shouldn't fire twice) — the SDK sets `isReplay = true` even while catching up an execution that has since completed, when running under the standalone Replayer (`isInReplayer`, `internal_task_handlers.go:1074`), because that's a full re-derivation of the whole run, not a live continuation.

**City implication**: replay isn't a rare recovery path — it's not "load event, sync" repeated forever. Any time a worker doesn't already have hot cached state for a run (first task after eviction, sticky miss, worker restart, or the standalone Replayer tool run offline against a history export), what happens is a fast re-run of workflow code against already-known events, silently regenerating the same commands, until it reaches the first genuinely new event — at which point it starts producing new commands for real.

## 6. The workflow cache

Covered mechanically in §4: `internal_worker_cache.go` — a process-wide, refcounted, LRU `cache.Cache` (`go.temporal.io/sdk/internal/common/cache`) keyed by run ID, storing `*workflowExecutionContextImpl` (the live, already-replayed-up-to-now workflow coroutine state). It is what makes sticky queues worth having: the "diff since last task" experience *requires* this cache to still hold state, and eviction (LRU pressure, `PurgeStickyWorkflowCache`, or process restart) is precisely what forces the next task for that run back to full replay. Default capacity 10,000 concurrent-run entries per worker process, shared across every `Worker` instance in that process (not just one task queue).

## 7. Activity heartbeats and cancellation

`RecordHeartbeat(ctx, details...)` (`internal_activity.go:397`, exposed to user code as `activity.RecordHeartbeat`) is a **no-op for local activities** and otherwise calls `serviceInvoker.Heartbeat`, which is throttled (`temporalInvoker.Heartbeat`, `internal_task_handlers.go:2179`): the *first* call in a window goes straight through; subsequent calls within `heartbeatThrottleInterval` are buffered and only the latest is actually sent when the throttle timer fires. `heartbeatThrottleInterval` defaults to 80% of the activity's `HeartbeatTimeout` if one is set (`getHeartbeatThrottleInterval`, `internal_task_handlers.go:2583`), capped at `maxHeartbeatThrottleInterval` — so heartbeats are deliberately not sent on every call the activity makes, only at a cadence proportionate to how long the server will wait before declaring the activity dead.

Each actual heartbeat is a synchronous `RecordActivityTaskHeartbeat` RPC (`internalHeartBeat`, `internal_task_handlers.go:2233`). The **response**, not a separate poll, is how cancellation is delivered: the server returns a value the client maps to a `*CanceledError` when cancellation was requested (workflow-level `RequestCancelActivity`, or the activity's task/workflow no longer exists), and `internalHeartBeat`'s `switch` (`:2273-2298`) calls `i.cancelHandler(err)` — which cancels the Go `context.Context` passed into the activity function. There's no separate polling channel for cancellation: an activity that never heartbeats can never learn its cancellation was requested until it returns. The same switch also maps `NotFound` / `NamespaceNotActive` / `NamespaceNotFound` errors, and retryable transient errors that persist past the heartbeat timeout, into the same cancellation path — from the activity's point of view, "the server doesn't want this attempt anymore" and "you were explicitly cancelled" both surface identically as a cancelled context.

## 8. Worker crash mid-activity — how it's recovered

There is no crash *detection* in this protocol, only **absence of heartbeat** (or absence of `RespondActivityTaskCompleted`/`Failed`) past a timeout. Two server-side timers govern this, both visible in `workflow-lifecycle.md`'s timer-queue diagrams and `history-service.md`'s Timer Task Queue section:

- If the activity was started with a `HeartbeatTimeout` and the worker stops heartbeating (process died, thread wedged, network partition), the server's heartbeat timer fires once that interval elapses since the last recorded heartbeat.
- If no `HeartbeatTimeout` was set, only `StartToCloseTimeout` (and `ScheduleToCloseTimeout`) bound how long a stuck/crashed attempt is tolerated.

Either timeout firing appends `ActivityTaskTimedOut` to history and — if the activity's `RetryPolicy` allows another attempt — the standard "Activity fails and is retried" path from `workflow-lifecycle.md` runs: `ActivityTaskScheduled` (attempt N+1) is appended, a fresh Activity Task is enqueued in Matching, and *any* available worker (not necessarily the one that crashed) can pick it up. This is also where the `NotFound` heartbeat-response case in §7 becomes concrete: if the original worker somehow comes back and heartbeats *after* the server already timed it out and retried, the heartbeat RPC returns `NotFound` — the original attempt's task token is dead — and the client maps that to cancellation too, so the zombie attempt stops rather than racing the new one. Nothing here is worker-crash-specific engineering; it's the same timeout-and-retry machinery that handles a slow activity, a network partition, or a genuinely dead process, which is itself worth keeping in the city: there's no "worker heartbeat to the cluster" liveness channel in this path — only per-activity heartbeats, per-task timeouts, and the long-poll connection's own liveness (an SDK worker that's gone simply stops holding poll connections open and stops responding to tasks it already has).

## Summary for the metaphor

- **Long-poll = truck parked at an open dock door, not a truck doing round trips.** Concurrency = number of trucks parked at once; each "trip" is one blocking RPC held open up to ~60s by Matching, immediately reopened on empty return.
- **Workflow task vs activity task** = two different bay types at the depot; workflow tasks are dispatched by History's own state machine, activity tasks only exist because a workflow task asked for one.
- **Sticky queue** = a private bay reserved for the truck that already has this shipment's manifest in its cab (the cache) — cheaper for everyone as long as that truck stays around; lose the manifest (eviction, restart, timeout) and the next truck has to read the whole manifest from the depot's paper trail (full replay) before it can drive.
- **Replay** = reading the paper trail to reconstruct exactly where you left off, silently, without re-doing the deliveries — only once you reach the point where the trail runs out do you start doing new work for real.
- **Heartbeat** = the truck radioing in periodically while parked at a delivery; the depot's only way to say "never mind, stand down" is in the reply to that radio call — a truck that goes silent just gets timed out and the delivery reassigned, no distinct "truck is dead" signal exists.

## Sources

- `temporalio/temporal@v1.31.2`: `docs/architecture/workflow-lifecycle.md`, `docs/architecture/matching-service.md`, `docs/architecture/message-protocol.md`, `docs/architecture/speculative-workflow-task.md`, `docs/architecture/history-service.md`.
- `temporalio/sdk-go@v1.46.0`, package `internal/`: `internal_task_pollers.go`, `internal_worker_base.go`, `internal_worker_cache.go`, `internal_task_handlers.go`, `internal_event_handlers.go`, `internal_activity.go`.
