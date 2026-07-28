# Temporal, in our own words

This is TemporalSimCity's model of Temporal. It is not a summary of Temporal's documentation — it
is the construct the city is designed off. Every visual decision downstream cites this document.

**Pinned at `temporalio/temporal@v1.31.2`** (2026-07-08, commit `19a7743`) — see
[Pin the Temporal Server version](https://github.com/0x63616c/temporalsimcity/issues/2). Server
claims trace to that tag. Proto-level names (`RetryPolicy`, `TimeoutType`, event-type enums) live
in `temporalio/api`, which the pin does not span; those are read at that repo's `master` and are
marked **(api)** wherever they appear. SDK claims trace to `temporalio/sdk-go@v1.46.0`.

**Correctness bar:** no claim here that can't be traced to pinned source or first-party docs.
Where we are uncertain, [§9](#9-what-we-are-not-sure-about) says so rather than smoothing it over.

Built from three research documents, which hold the detail this one compresses:
[services](research/temporal-server-services.md) ·
[L1 mechanisms](research/l1-mechanisms.md) ·
[SDK worker lifecycle](research/sdk-worker-lifecycle.md).

---

## 1. The shape of the thing

**Temporal is a durable event log with a dispatcher bolted to it.**

A workflow's truth is an append-only sequence of History Events. Nothing else is authoritative —
every other piece of state is a cache or an index derived from that log and, in principle,
reconstructible from it. Progress happens when someone reads the log, decides what should happen
next, and appends the result.

That "someone" is never Temporal. **Temporal never runs your code.** The cluster stores the log,
decides when a decision is due, and hands work out to whoever asks for it. Your code lives in your
own processes, on your own machines, and comes to the cluster to collect work.

**Temporal pulls; it does not push.** There is no path by which the cluster reaches out and invokes
you. Workers open long-lived connections and wait. This single fact drives most of the topology,
and it's the reason a cluster with no workers looks perfectly healthy while getting nothing done.

Three consequences worth holding onto, because they explain most of what follows:

1. **Every state change is a durable write before it is an action.** The log is appended first;
   dispatch happens afterwards, driven off that write.
2. **Anything fast is a cache, and every cache can miss.** A miss is never an error — it's a
   fallback to re-deriving from the log, which is slower but always correct.
3. **Nothing is detected; things time out.** There is no liveness channel to your code. Failure is
   inferred from the absence of an expected message before a deadline.

---

## 2. The boundary

The most important line in the system separates **the cluster** from **your code**.

| | Cluster | Your code |
|---|---|---|
| Who runs it | Temporal (self-hosted or Cloud) | You |
| What it holds | Event log, mutable state, task queues | Workflow and Activity implementations |
| How they meet | gRPC, and always **worker-initiated** | |
| Trust | The cluster never executes user logic | The worker never mutates history directly |

Everything crossing that line is an RPC the worker started — including receiving work. The cluster
answers; it never calls.

Two things sit awkwardly against this line and are worth naming early, because both are easy to
mistake for something they aren't:

- **The Worker Service is not your worker.** It is one of the cluster's own five service roles,
  running the cluster's housekeeping (scanners, replication, batch operations) as ordinary
  workflows against the cluster's own frontend. It is Temporal running *Temporal's* code. An
  **SDK Worker** is your process, outside the cluster, running *your* code. Same word, opposite
  sides of the boundary. (Sources: `service/worker/service.go`.)
- **The Web UI is not part of the server.** `temporalio/ui` is a separate project and just another
  API client of the frontend — same standing as the CLI or any SDK. It ships inside the all-in-one
  Docker image for convenience, which is what makes it look built-in. It is not one of the five
  service roles and it does not join cluster membership.

---

## 3. The five services

`temporal/fx.go`'s `TopLevelModule` at the pin wires exactly five service roles. `all` and `server`
are not additional services — they are the run-everything-in-one-process mode used by
`temporal server start-dev`.

**Frontend** — the cluster's only external door. Stateless: it validates, rate-limits, routes, and
forwards. Any instance serves any request. It implements the public `WorkflowService` API and also
speaks HTTP for REST and Nexus callbacks. Scales by adding instances behind a load balancer.

**Internal-Frontend** — *the same code as Frontend*, running under a different service identity.
The only behavioural difference is that its `ClaimMapper` auto-authorizes every caller instead of
consulting the operator's configured authorizer, so the cluster's own system workers never need
customer-facing credentials to call their own API. **Off by default** — gated behind
`USE_INTERNAL_FRONTEND` in the stock config. Not a distinct subsystem; a second door with the lock
removed.

**History** — the one genuinely stateful service, and the heart of the system. It owns Workflow
Execution state: it appends History Events, updates Mutable State, and enqueues internal tasks.
Executions are partitioned across a **fixed** number of History Shards, set at cluster creation and
never changed. Each History instance owns a subset of shards. Scaling means more instances sharing
the same fixed shard count — never more shards.

**Matching** — owns the Task Queues that workers poll. Holds task backlogs and outstanding pollers,
splits each queue into **partitions** (default 4), and routes a poll to the partition that can
serve it. When a partition has a task and no poller, or a poller and no task, it **forwards** up
toward the root partition so the two can find each other.

**Worker Service** — cluster housekeeping, as described in §2: scanners, the replicator, the
batcher, parent-close-policy enforcement, per-namespace system workers. Effectively stateless; its
durable state lives in the workflows it drives.

All five join **independent per-role membership rings** (Ringpop/SWIM gossip, one ring per role,
~20s heartbeat cutoff). Membership is not decoration — it is how gRPC resolves a logical service
name to a live host, and for History specifically it is what tells a `ShardController` which shards
it currently owns. **A node joining or leaving triggers shard rebalancing.**

### How they talk

```
User app / SDK Worker ──gRPC──▶ Frontend ──┬──▶ History   (drive workflow state, routed by workflow-id → shard)
                                            └──▶ Matching  (relay worker long-polls, routed by task-queue → partition)

History ──▶ Matching     AddWorkflowTask / AddActivityTask, via Transfer Task execution
Worker Service ──▶ Frontend (as an SDK client)  ·  ──▶ History / Matching (direct, for scanners)
History ◀──▶ History     cross-cluster replication, when global namespaces are enabled
```

Note the direction that is **absent**: nothing goes from the cluster out to an SDK Worker. Every
arrow touching a worker was opened by that worker.

---

## 4. The mechanisms

Twenty mechanisms passed the altitude rule — a name an operator says out loud, *and* an observable
behaviour or failure mode. The full inventory with source citations is in
[`research/l1-mechanisms.md`](research/l1-mechanisms.md); this is the working set, grouped by what
they're for.

**Storing truth**

- **Workflow Execution History** — the append-only event log. Branches only on reset or conflict
  resolution. Sufficient on its own to reconstruct everything else.
- **Mutable State** — the derived summary of where an execution currently is: in-flight
  activities, timers, child workflows. Persisted and cached because recomputing it from the log on
  every request would be too slow. On Cassandra it's a single row.
- **History Shards** — the fixed partitioning of executions across History instances, with
  ownership assigned by membership.
- **Persistence** — the storage layer underneath all of it (Cassandra primary; MySQL, Postgres,
  SQLite supported). Deliberately **not** treated as a mechanism in its own right: it has no
  failure mode distinct from the shard, mutable-state and queue failures it surfaces through. It's
  the ground, not a building.

**Getting work moved**

- **Task Queues and partitions** (Matching) — the user-facing queues workers poll, split into
  partitions that forward toward a root.
- **Internal task queues** (History, per shard — a different concept sharing the name):
  - **Transfer** — "do this now." Executing a transfer task is what calls Matching to enqueue a
    workflow or activity task. This is a transactional outbox: History writes the transfer task in
    the same transaction as the state change, so dispatch is eventually-consistent but guaranteed.
  - **Timer** — "do this at time T." Sleeps, timeouts, retry backoffs.
  - **Visibility**, **Replication**, **Archival** — same per-shard queue shape, folded into their
    parent concepts below.
  - **Outbound** — Nexus and callbacks; external destinations, long calls, with a circuit breaker.
    Conditionally in — see §8.
- **Sticky task queues + the workflow cache** — *one mechanism with two halves*. A server-side
  cache of loaded workflow context, and a worker-side cache (default 10,000 runs per process) that
  lets the server route a run's next task back to the worker that already has it in memory. They
  share a single failure mode, which is why we refuse to count them separately.

**Bounding time and failure**

- **Timeouts** (api) — `SCHEDULE_TO_START`, `START_TO_CLOSE`, `SCHEDULE_TO_CLOSE`, `HEARTBEAT`,
  plus whole-execution and per-run timeouts. Each has a distinct diagnostic meaning; see §7.
- **Retry policies** (api) — initial interval, backoff coefficient, maximum interval, maximum
  attempts, non-retryable error types.

**Bounding growth**

- **History size and count limits** — namespace-scoped: warn at 10 MiB / 10,240 events, **fail the
  workflow outright** at 50 MiB / 51,200 events, and start suggesting continue-as-new at 4 MiB.
- **Continue-as-new** — atomically end the current run and start a fresh one with an empty history,
  carrying forward only what's explicitly passed. New `RunID`, same workflow id.

These two are a matched pair and should always be taught together: one is the failure, the other is
the escape hatch, and neither makes sense alone.

**Reading and retaining**

- **Visibility store** — a separate, queryable database of workflow metadata, kept eventually
  consistent via the per-shard visibility queue. This is what search and list operations read, and
  why they can lag.
- **Archival** — moving closed workflows' history and visibility records to blob storage before
  retention deletes the live copy.

**Deliberately excluded:** task tokens (real, but plumbing — no operator debugs by task token) and
CHASM (see §8).

---

## 5. Life of a workflow

The canonical trace, from `docs/architecture/workflow-lifecycle.md` at the pin. The workflow is
the simplest thing that isn't trivial: call one activity, return its result.

```javascript
function myWorkflow() {
  result = callActivity(myActivity);
  return result;
}
```

**1 — Start.** The user application calls `StartWorkflowExecution` on Frontend, which forwards to
History. History writes, in one transaction: history initialized to
`[WorkflowExecutionStarted, WorkflowTaskScheduled]`, mutable state, and a **Transfer Task**. Only
after that commit does the shard's queue processor pick up the transfer task and call
`AddWorkflowTask` on Matching. *The client's start call has already returned by then* — dispatch is
asynchronous to the acknowledgement.

**2 — A worker picks up the workflow task.** A worker was already blocked in
`PollWorkflowTaskQueue` (§6). Matching calls `RecordWorkflowTaskStarted` on History, which appends
`WorkflowTaskStarted`, updates mutable state, and starts a **workflow task timeout** timer. History
reads the events and returns them; Matching hands them to Frontend, which hands them to the worker.
The worker advances the workflow until it blocks on the activity call.

**3 — The worker asks for an activity.** It calls `RespondWorkflowTaskCompleted` carrying a
**command**: `ScheduleActivityTask`. History appends `WorkflowTaskCompleted` and
`ActivityTaskScheduled`, updates mutable state, and writes another transfer task — which the queue
processor turns into `AddActivityTask` on Matching.

This is the loop, and it's worth stating plainly: **the worker never mutates history. It returns
commands, and the server decides what events those become.** A command is a request; an event is a
fact.

**4 — A worker picks up the activity task.** `PollActivityTaskQueue` → Matching →
`RecordActivityTaskStarted` on History → `ActivityTaskStarted` appended, activity timeout timer
started. The worker executes your activity code. This may be a completely different process from
the one in step 2 — nothing requires them to be the same.

**5 — The activity finishes.** `RespondActivityTaskCompleted` carries the result. History appends
`ActivityTaskCompleted` (result and all) **and** `WorkflowTaskScheduled` — the activity's
completion is itself the reason the workflow needs to make progress again. Another transfer task,
another `AddWorkflowTask`.

**6 — A worker picks up the new workflow task.** Same shape as step 2. The workflow advances; this
time it reaches its end.

**7 — Completion.** `RespondWorkflowTaskCompleted` with a `CompleteWorkflowExecution` command.
History appends `WorkflowTaskCompleted` and `WorkflowExecutionCompleted`, then enqueues the
close-out tasks: visibility update, archival, retention.

The finished log — eleven events for one activity call:

```
 1  WorkflowExecutionStarted
 2  WorkflowTaskScheduled
 3  WorkflowTaskStarted
 4  WorkflowTaskCompleted
 5  ActivityTaskScheduled
 6  ActivityTaskStarted
 7  ActivityTaskCompleted
 8  WorkflowTaskScheduled
 9  WorkflowTaskStarted
10  WorkflowTaskCompleted
11  WorkflowExecutionCompleted
```

**If the activity fails instead:** `RespondActivityTaskFailed` → History appends
`ActivityTaskFailed` **and** a fresh `ActivityTaskScheduled` for attempt 2, plus an activity-retry
timer. Note what does *not* happen: the workflow is not woken. It never learns about the failure,
because from its perspective the activity is simply still running. Retry is entirely a
server-side concern until attempts are exhausted.

---

## 6. What a worker is actually doing

A worker runs a pool of poller goroutines per task queue. Each one:

1. takes a local concurrency slot,
2. issues **one** `PollWorkflowTaskQueue` (or `PollActivityTaskQueue`) gRPC call and blocks,
3. gets either a task or — after the server holds the connection open for
   `MatchingLongPollExpirationInterval`, **default 60 seconds** — an empty response,
4. immediately opens another.

**There is no drive-away-and-return cycle.** The connection is held open, not repeatedly
re-established, and it reopens instantly whether or not it carried anything. Worker concurrency is
therefore *how many polls are open at once* — the slot pool — never how frequently one poll
repeats. Anything the city animates must reflect this; a shuttling animation would teach the wrong
model.

### Replay

Replay is decided **per event**, not per task. An event is a replay event if:

```
event.EventId <= workflowTask.PreviousStartedEventId
```

`PreviousStartedEventId` is the `WorkflowTaskStarted` id of the last workflow task this worker
completed for this run — that is, the cache checkpoint. Everything at or below it has been seen
before; everything above it is new.

To replay, the SDK **re-runs your workflow function from the top**, feeding it the events it
already has. But the commands that re-run produces are not sent anywhere — they're diffed against
what history says actually happened (`matchReplayWithHistory`). A mismatch — missing command, extra
command, wrong type — raises `[TMPRL1100]` and fails the workflow task rather than silently
diverging.

**That error is the determinism requirement.** Not folklore, not a style rule: given the same
events, your workflow code must produce the same commands in the same order. Which is exactly why
wall-clock time, randomness, unordered map iteration and I/O are banned inside workflow code —
their results would differ on the re-run. They belong in activities, whose *results* are recorded
as events and therefore read back off history instead of being recomputed.

Replay is not a rare recovery path. It runs whenever a worker doesn't have hot state for a run:
first task after an eviction, a sticky miss, a worker restart, or the offline replayer tool.

### Sticky queues and what a miss costs

Once a worker has a run cached, the server routes that run's next workflow task to a **sticky
queue** — a private, per-worker queue — so it receives only the new events. When that breaks, the
server ships the full history and the worker replays from event 1.

Three ways it breaks, all mundane: **LRU eviction** under cache pressure; **`StickyScheduleToStart`
timeout**, where no sticky-affinitized worker polls in time and the task falls back to the normal
queue; and a documented edge case where a speculative workflow task lands on a queue whose entry
was just evicted.

A sticky miss is **not an error**. It is the system's designed fallback: slower, more work for
everyone, always correct.

### Heartbeats and cancellation

Activity heartbeats are throttled to roughly **80% of the activity's `HeartbeatTimeout`** — sent at
a cadence proportionate to how long the server will wait before declaring the activity dead, not on
every call your code makes.

And here is the part that surprises people: **cancellation is delivered in the heartbeat's
response.** There is no separate channel. An activity that never heartbeats can never learn it was
cancelled. From the activity's point of view, "you were cancelled" and "the server has moved on
without you" arrive identically — as a cancelled context.

---

## 7. What happens when each part fails

Failure is not detected in this system. **It is inferred from absence before a deadline.** The
table below is the failure vocabulary the city needs to be able to show.

| What breaks | How it surfaces | What recovers it |
|---|---|---|
| **History host dies** | Shards reassign via membership; in-flight requests to that shard fail with `ShardOwnershipLost` | New owner picks up the shard; brief latency blip, no data loss |
| **Mutable state write fails** | Shard reloads mutable state from persistence rather than risk drift | Re-derivation from the log |
| **No worker is polling** | `ActivityTaskTimedOut` with `TIMEOUT_TYPE_SCHEDULE_TO_START` | Nothing automatic — this is a capacity problem, and the cluster looks healthy throughout |
| **Worker dies mid-activity** | Heartbeat stops → `HeartbeatTimeout` fires (or `StartToCloseTimeout` if no heartbeat configured) → `ActivityTaskTimedOut` | Retry per `RetryPolicy`, on **any** available worker. **No crash detection exists** — only the timeout |
| **Zombie worker returns after timeout** | Its heartbeat gets `NotFound` — that attempt's token is dead | Client maps it to cancellation; the zombie stands down instead of racing the retry |
| **Activity keeps failing** | `ActivityTaskFailed` per attempt, retried with backoff | Terminal once `maximum_attempts` is hit, a non-retryable error type matches, or schedule-to-close expires |
| **Worker cache evicts a run** | Sticky miss → full history fetch → full replay | Automatic, correct, just slower |
| **Workflow code changed incompatibly** | `[TMPRL1100]` nondeterminism error; the workflow task fails | Nothing automatic. Needs versioning or a reset — this is a *code* problem, not an infrastructure one |
| **History grows unbounded** | Warnings at 10 MiB / 10,240 events; **workflow terminated** at 50 MiB / 51,200 | Continue-as-new, which the server starts suggesting at 4 MiB |
| **Visibility queue backs up** | Search and list results lag reality; a workflow completes before it looks completed | Drains on its own; alerted separately from transfer/timer backlogs |
| **Nexus destination is down** | Circuit breaker trips after 5 consecutive failures; tasks rejected before mutable state is even loaded | Breaker closes when the destination recovers |

The two rows that carry the most teaching weight are the two with **nothing** in the recovery
column: no workers polling, and nondeterministic code. Everything else the system heals by itself.
Those two need a human.

---

## 8. What's in, and what's out

Settled by [Inventory the L1 mechanisms](https://github.com/0x63616c/temporalsimcity/issues/4):

- **Serverless Workers / Worker Controller Instance — out.** New at this pin, pre-release, disabled
  by default. Has an operator-facing name but no default-on behaviour, so nothing to observe and
  nothing to fail. Revisit only if a future pin turns it on by default.
- **Nexus — in, for this document; geometry deferred.** Unconditionally on since 1.31.0, so every
  cluster at this pin runs it whether or not a workflow uses it, and it has a genuinely distinctive
  failure mode (the circuit breaker above). Whether it earns a place in the city — and with it the
  Outbound Task Queue — is [#9](https://github.com/0x63616c/temporalsimcity/issues/9)'s call.
- **CHASM — out.** It's a framework the server's own developers build on, not a thing operators
  name. But it earns one sentence here, because it explains something otherwise mysterious:
  **Workflows, Schedules and Nexus Operations are all built on the same underlying state-machine
  and persistence framework**, which is why they behave so similarly. We use none of its
  vocabulary.

Ruled out for the whole effort, per the map: connecting to a real cluster, emulating the server,
and everything past the vertical slice.

---

## 9. What we are not sure about

Flagged rather than smoothed:

- **`ActivityTaskStarted` is drawn dashed** in the pinned lifecycle doc's step-4 diagram, unlike
  every other event, and the doc never says why. We suspect it relates to when the event is
  actually persisted versus merely recorded in mutable state, but we have not confirmed this
  against source. **Do not build a visual distinction on this until it's checked.**
- **Matching's forwarding algorithm** is documented only at the level of "idle partitions forward
  toward the root." The architecture doc explicitly disclaims deeper coverage. Anything the city
  shows about *how* a task hops partitions needs a source read of `service/matching/` first.
- **Schedules** were left out of the mechanism inventory because the pinned doc marks the
  CHASM-based Scheduler as not yet generally available. But operators say "schedule" out loud
  constantly, which is precisely the altitude test. Unresolved; belongs to
  [Lock the altitude](https://github.com/0x63616c/temporalsimcity/issues/8).
- **Replication and multi-cluster** are named throughout the source but were not traced. If they
  ever get geometry, they need their own research pass first.
- **The SDK view is sdk-go's.** Go, Java and .NET each implement their own poller, cache and replay
  loop; TypeScript, Python, Ruby and PHP delegate all of it to `sdk-core` in Rust. The wire
  protocol is identical everywhere because the server defines it — but client-side orchestration
  details in §6 are sdk-go's, and shouldn't be presented as universal.
