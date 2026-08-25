# Buildbarn Architecture Decision Record #13: Persistent Workers

Author: Gergely Fábián<br/>
Date: 2026-08-25

# Context

Bazel's local executor has a feature called
[persistent workers](https://bazel.build/remote/persistent): when a
rule advertises `execution_requirements["supports-workers"] = 1`,
Bazel does not fork-exec a fresh process per action. Instead it
keeps a long-lived child process around, dispatches each action
to it as a framed `WorkRequest`, and reads back a framed
`WorkResponse`. The wire format is defined by
`src/main/protobuf/worker_protocol.proto` in the Bazel tree.

The latency win is large for compilers that pay a heavy
per-invocation startup cost — JVM warm-up, parser caches, type
indexing, class verification. `JavaBuilder`, `Scalac`,
`KotlinCompile`, the `rules_swift` worker, and any rule built on
`@bazel_tools//tools/build_defs/repo:worker.bzl` benefit. For
many real codebases the worker startup cost dominates the wall
time of a clean build's compile actions, and the per-incremental-
edit cost is dominated by it as well.

On the remote-execution side Buildbarn currently has no
equivalent. Every `RunRequest` from `bb_worker` to `bb_runner`
fork-execs a fresh process; the JVM warm-up is paid in full per
action. There is no mechanism by which a worker invocation
on action N would observe state from action N-1.

We propose to add persistent-worker support to
`bb-remote-execution`. The proposal is clean-room: the only
sources to be consulted during implementation are Bazel's
open-source repository (the protocol definition and
`WorkerParser.java`), the
[REAPI v2 specification](https://github.com/bazelbuild/remote-apis),
the existing Buildbarn codebase, and the Go standard library.
No other RBE implementation is to be referenced.

## Two activation paths

Bazel itself emits two distinct signals that a remote action
should be executed by a persistent worker. Both must be handled:

- **`supports-workers=1` (client-populated).** The client
  populates `Command.platform.properties` with `supports-workers=1`,
  optionally with `requires-worker-protocol={proto,json}` and
  `worker-key-mnemonic=<name>`. This is the path a custom
  rule takes when it sets `exec_properties` on
  `ctx.actions.run`. Bazel itself does **not** mirror
  `execution_requirements["supports-workers"]` into
  `Command.platform`, so a rule that only sets the execution
  requirement does not reach this path.

- **`persistentWorkerKey` (Bazel-populated).** With
  `--experimental_remote_mark_tool_inputs=true` on the Bazel
  side, Bazel automatically populates
  `Command.platform.properties.persistentWorkerKey=<hex>` for
  every spawn that declares `execution_requirements
  ["supports-workers"]=1` and has a non-empty tool input set —
  true for `rules_java`, `rules_scala`, `rules_kotlin` compile
  actions today. The value is Bazel's own content hash over
  the worker's tool files, argv, and env. No per-rule
  plumbing required; one Bazel flag is enough.

A real cluster will see both. The design must accommodate both
as first-class.

# Scope of persistency

This is the most important architectural call in the design, so
we state it explicitly.

We propose that persistency live **between actions, within a
single `bb_runner` process lifetime, keyed by a (pool-key,
working-directory) pair** — and no further. Concretely:

- A worker process is spawned the first time an action with a
  given (pool key, working directory) arrives at a `bb_runner`.
  The process stays alive across successive actions with the
  same (key, working directory). It dies when `bb_runner` evicts
  it (idle timeout, LRU eviction, max-requests recycling,
  memory-pressure eviction) or when `bb_runner` itself shuts
  down. Multiple workers per key may exist concurrently — one
  per working directory in use — so same-key parallelism is
  unbounded inside the runner subsystem (overload protection
  is a scheduler-level concern; see Alternatives Considered).
- On-disk build directories `persistent/pw-<hash>-<id>/` are
  materialized on `bb_worker` from a per-key subdir-id pool that
  hands a freed id back before allocating a fresh one. This
  preserves warm reuse on the runner pool (a same-key action
  arriving after a release lands on the same `<id>` and reuses
  the idle worker bound to that working directory). The trees
  persist across actions for the duration of the runner process
  and are not preserved across `bb_runner` restarts.

We rule out two larger scopes:

**Across `bb_runner` restarts ("invocation" persistency)** is
declined for v1. The worker process is structurally a child of
`bb_runner` and cannot survive the parent. Only the on-disk
`pw-<hash>-<id>/` trees could be reattached, and the integrity
story is uncomfortable: a stale tree left by a crashed previous
run may contain half-materialized inputs, dangling symlinks, or
files written but never `fsync`ed.

**Across `bb_worker`/`bb_runner` instances ("instance"
persistency)** is declined outright. It would require a
distributed lock manager (etcd / Redis / equivalent) to
serialize per-key access across hosts, a shared filesystem or
replicated build directories to give every host a view of the
`pw-<hash>-<id>/` trees, and a cross-instance pool-key
registry on the scheduler. The cost is large, and a meaningful
slice of the locality benefit is independently available via
the scheduler's existing invocation-stickiness mechanism, which
biases successive actions from one Bazel invocation toward the
same worker host. The remaining delta is not worth the
operational surface area.

The action-only scope is what the proposed implementation
delivers.

# High-level architecture

We propose to add persistent-worker support along the three
existing components, with the responsibilities split as
follows:

```
+-------------------+        +-------------------+        +-------------------+
|   bb_scheduler    |        |    bb_worker      |        |    bb_runner      |
|                   |        |                   |        |                   |
| action-key strip  |        | per-key subdir-id |        | worker process    |
| of per-action     | -----> | pool over on-disk | -----> | pool keyed by     |
| hint properties   |        | persistent/       |        | (key, workdir) +  |
|                   |        | pw-<hash>-<id>/   |        | protocol codec +  |
|                   |        | trees             |        | spawner           |
+-------------------+        +-------------------+        +-------------------+
```

The split follows the existing component boundaries:

- **`bb_runner`** owns the worker process. The worker is a
  host-process concern (stdin/stdout pipes, OS process
  lifetime, rlimit, prlimit, PR_SET_PDEATHSIG, OS-thread
  pinning). Putting the protocol codec and pool here keeps
  `bb_worker` unaware of long-lived processes; `bb_worker`
  continues to see one `RunRequest` per action, no matter
  whether the runner served it via fork-exec or via a
  persistent worker.

- **`bb_worker`** owns the on-disk persistent build directories
  and the per-key subdir-id pool that hands each concurrent
  same-key action a disjoint `pw-<hash>-<id>/` tree. It does
  not see the worker process or talk the worker protocol, and
  it does not impose its own per-key concurrency cap (see
  Alternatives Considered for why).

- **`bb_scheduler`** owns the routing decision. Because
  `persistentWorkerKey` is a per-action content hash, it
  cannot appear in a worker pool's advertised platform; we
  need a knob to strip it (and any other per-action hint
  property) from the routing key.

The rest of the document follows these three components in
order.

## One-action sequence

End-to-end, a single persistent-worker action flows as
follows:

1. The client puts `supports-workers=1` or
   `persistentWorkerKey=<hex>` in `Command.platform`.
   `Command.arguments` carries the worker startup argv with
   `@flagfile` entries interleaved.
2. `bb_scheduler` strips configured per-action hint
   properties (always including `persistentWorkerKey`)
   before forming the routing key, then matches the
   stripped key against worker pool advertisements.
3. A `bb_worker` thread acquires a persistent build
   directory under `persistent/pw-<hash>-<id>/` from the
   per-key subdir pool — a previously-released `<id>` when
   one is available (warm reuse), a fresh one otherwise
   (parallel spawn) — and overlays the new action's input
   root onto it.
4. `bb_worker` sends a `RunRequest` to `bb_runner` carrying
   the usual argv, env, working directory and output paths
   plus a `PersistentWorker` addendum: the pool key, the
   flag-file paths, and the mnemonic and action digest that
   let the worker's lifecycle log lines name the action
   that triggered them.
5. `bb_runner` acquires a worker for the (key, working
   directory) pair, writes a framed `WorkRequest` whose
   `arguments` are the concatenated flag-file contents in
   flag-file order, and reads back a framed `WorkResponse`.
6. `bb_runner` returns the worker's exit code and captured
   output as a `RunResponse`. The worker process stays
   alive; its handle goes back to the pool.
7. `bb_worker` releases the subdir id and returns the
   action result. The `pw-<hash>-<id>/` tree stays on disk,
   and the next action with the same key picks that id back
   up before a fresh one is allocated.

# Detecting a persistent-worker action

## Pool key

We propose two key namespaces:

- **`pwk:<hex>`** — the value of `persistentWorkerKey`
  verbatim, namespaced. Bazel's own content hash over tool
  files, argv, and env. A toolchain bump produces a new
  hash and routes to a fresh pool entry automatically.
  This is the recommended path.
- **`sw:<hex>`** — a hash over the worker's startup argv
  prefix, environment, and mnemonic. Used when activation
  came via `supports-workers=1` without a
  `persistentWorkerKey`.

The `sw:` namespace has a known sharp edge: it does not
cover tool input *contents*, so two actions whose argv
prefix and env agree but whose `bin/java` differs in bytes
share a pool entry, and the first one in wins. The
mitigation is to enable
`--experimental_remote_mark_tool_inputs` and get the
`pwk:` namespace instead.

We propose to support both, because the activation paths
are not interchangeable. A custom Starlark rule that sets
`supports-workers=1` through `exec_properties` bypasses
Bazel's tool-input marking and never gets a
`persistentWorkerKey`, flag or no flag; supporting `sw:`
keeps that path working. Deployments routing exclusively
through Bazel's native activation can ignore it.

Two actions sharing both key and working directory are
guaranteed the same worker process. Two sharing only the
key may run in parallel on distinct processes. Two with
different keys never share.

## Flag-file partitioning

The worker protocol carries the per-action arguments as
`WorkRequest.arguments`, not as the worker process's argv.
Bazel's convention is that `Command.arguments` holds the
worker's startup argv interleaved with `@flagfile`
entries; the worker is started with the startup argv, and
the contents of the flag-files become the per-action
`WorkRequest.arguments`.

Bazel's parser has a strict mode (exactly one flag-file,
at the end of argv) and a legacy mode (flag-files stripped
from anywhere in argv, contents concatenated). We propose
legacy mode, because `rules_java`'s Javac action emits
**two** flag-files per action — one for compile options,
one for source-file inputs — which strict mode rejects.
Bazel itself defaults to legacy for the same reason.

Escape rules match Bazel exactly: `@<path>` is a flag-file
and is stripped, `@@<rest>` is a literal-`@` escape and is
kept in argv, and a bare `@` is kept. An argv with no
flag-file is not a persistent-worker action and falls back
to fork-exec; an argv that is *only* flag-files is
rejected, since there is nothing to launch.

## Argv position of `--persistent_worker`

Bazel appends `--persistent_worker` as the **last**
element of the launched worker's argv, not at position 1,
and we propose to match that exactly.

The position matters in practice. Bazel-generated JVM
launcher wrappers parse leading `--jvm_flag=…` arguments
with a prefix loop that stops at the first non-matching
argument. With our flag at position 1 the loop terminates
immediately, the JVM flags fall through as program
arguments instead of JVM options, and the launcher exits
before reaching `main()` — silently, since the wrapper
dies before JVM startup would have produced any output.
Appending lets every prefix-parsing launcher consume its
own arguments first, exec the JVM with the right options,
and let the JVM see `--persistent_worker` and enter the
worker loop.

# `bb_runner`: worker pool, codec, spawner

## Pool

We propose a pool whose deduplication unit is the
**(key, working directory)** pair, not the key alone. Two
same-key acquires on distinct working directories receive
distinct worker processes concurrently; two same-(key, wd)
acquires share a worker (warm reuse). The
working-directory dimension is supplied by `bb_worker`'s
per-key subdir pool, which hands a freed `<id>` back
before allocating a fresh one — so a same-key action
arriving after a release lands on a working directory
whose runner-side worker is still cached.

`Acquire` reuses an idle worker for the pair, spawns a
fresh one, or joins an in-flight spawn for the same pair
so that N concurrent cold acquires produce one JVM rather
than N. That piggyback is the pool's only blocking
surface. When the pool is at its cap and no idle worker
can be evicted to make room, `Acquire` fails fast with
`ResourceExhausted` and the client retries through REAPI
backoff, rather than queueing inside the runner where the
scheduler cannot see it. `Release` returns the worker to
the pool, or kills it when the caller reports the round
trip as unhealthy.

We propose four recycling knobs, all optional: a pool-wide
cap on live workers, an idle timeout, a per-worker request
budget that bounds the heap footprint of a long-running
JVM, and a pool-wide RSS budget. Eviction always targets
idle workers — LRU order when the pool is full, largest
RSS first when the memory budget is exceeded — and never
pre-empts a worker mid-action.

Readiness is the pool's other externally visible property.
We propose that `CheckReadiness` fail once the pool has
been closed, so a runner in graceful shutdown drains
before its gRPC server stops accepting work. It reports
reachability and nothing stronger: worker binaries are
action-supplied, so `bb_runner` has no binary to
smoke-test a spawn against and a permanently-broken
spawner would still pass. The honest health signal is the
per-spawn outcome on `Acquire`.

The pool also pre-empts the dead-worker case at acquire
time with a liveness probe, so a worker the kernel
OOM-killed is torn down before its handle reaches the
caller instead of surfacing as a confusing write error on
the next action. The probe has to tell a recycled pid from
a live one — comparing process start time against a token
captured at spawn — since a bare "does this pid exist"
check would hand back a handle to an unrelated process.

## Codec

The protocol codec implements Bazel's two encodings:

- **proto** (default): each frame is a varint length
  followed by a serialised `WorkRequest` / `WorkResponse`.
- **JSON**: each frame is a single newline-terminated
  JSON object.

The mode is resolved per action on `bb_worker` from the
platform, with an operator-supplied fallback for the case
Bazel's own activation path creates: under
`--experimental_remote_mark_tool_inputs` Bazel relays
`persistentWorkerKey` without the sibling
`requires-worker-protocol`, so a JSON worker reached that
way carries no protocol signal on the wire at all and the
operator has to supply one out of band. Values outside the
allow-list are rejected at detection rather than silently
defaulting, so a typo fails the action immediately instead
of producing a confusing first-request framing error.

We propose the codec be **poison-on-error**: once any
framing error has occurred (size cap, malformed varint,
EOF mid-frame) it returns that error on every subsequent
operation and the worker is killed. There is no recovery
path — protocol corruption is indistinguishable from a
single byte of stray output on `stdout` by a buggy worker.
This matches Bazel's own behavior; the constraint is
structural.

On top of framing we propose a per-round-trip
`request_id`, echoed by the worker and checked on the
response. A mismatch means the response in hand belongs to
a previous action, so the worker is retired rather than
stale output being attributed to the current one. Workers
that ignore the field reply with the proto default, which
is tolerated.

## Spawner

The spawner turns a spec into a live worker process, and
is where the host-level concerns land: pipes wired into
the codec, `PR_SET_PDEATHSIG` so a `bb_runner` crash takes
its workers with it, per-worker rlimit caps, and a process
group per launcher so a kill reaches whatever the worker
forked rather than leaving it reparented to init. Several
of these are per-thread kernel attributes, so the spawn
sequence has to be pinned to a single OS thread — a detail
invisible in the design and mandatory in the
implementation.

We propose to wrap the stderr pipe in a bounded ring
buffer, so that whatever the worker printed in the seconds
before exit is in hand when the pool logs the kill. See
Observability.

## Runner decorator

We propose a decorator over `bb_runner`'s existing
`LocalRunner`. It falls through to the base runner unless
the `RunRequest` carries a `PersistentWorker` addendum —
detection proper happens on `bb_worker` — then acquires a
worker for the (key, working directory) pair, writes the
`WorkRequest`, reads the `WorkResponse`, and releases.

Both halves of the round trip select on `ctx.Done()`, so a
cancelled request pre-empts a wedged worker: the worker is
released as unhealthy, which closes the codec's pipe and
unblocks the goroutines. A non-zero
`WorkResponse.exit_code` is **not** a worker fault — it is
an ordinary action failure, and the worker goes back into
the pool healthy, exactly as in Bazel's local executor.

# `bb_worker`: on-disk layout and subdir pool

## Split-root layout

We propose to subdivide the on-disk build root into
two subdirectories on `bb_worker` startup:

```
build root/
  scratch/       # non-persistent per-action build dirs
    <id>/
    ...
  persistent/    # per-key persistent-worker trees
    pw-<hash>-<id>/
    ...
```

We propose to scope the cleaner that `bb_worker` triggers
on idle↔busy transitions to `scratch/` only, making it
**structurally incapable** of walking into `persistent/`
— no skip-list, no naming convention to couple against.
This is load-bearing: without the isolation, a
non-persistent action's `Release` walks the entire build
root via `RemoveAllChildren` and destroys live persistent
worker state mid-action. The cascade manifests as the JVM
seeing its input root vanish and exiting, then cascading
EOFs on the `bb_runner` codec.

The split applies to both the Native and Virtual backends,
and the configured build root still points at the parent
of both subdirs — no operator-visible configuration change
beyond enabling persistent workers.

## Per-key subdir pool

Concurrent same-key actions need disjoint trees, so
`bb_worker` hands each one a `pw-<hash>-<id>/` out of a
per-key id pool: a freed id when one is available, which
preserves warm reuse against the runner pool, otherwise
the next fresh id. There is no per-key cap and no blocking
behavior — see Alternatives Considered for why an in-tree
cap was rejected.

The pool must be a singleton at `bb_worker` process scope
rather than per worker thread. With a counter per thread,
concurrent acquires on the same fresh key would each
allocate id 0 and multiple goroutines would proceed into
the same directory — a TOCTOU cascade whose symptom is a
zoo of EEXIST-shaped errors that are hard to read, since
every thread appears to be holding a fresh id.

## Atomic input materialization

A persistent build directory survives across requests by
design, so every input-materialization layer must tolerate
the destination already existing *and* replace it
atomically. `Remove + Create` is not safe: there is a
window in which an in-flight `exec()` sees `ENOENT` on a
path it expected to find. We propose the pattern
`Link/Symlink/OpenAppend to a temporary name + Rename` for
every materialization surface, since `rename(2)` keeps the
destination continuously pointing at either the old entry
or the new one, never at nothing.

`Mknod` is the one special case: the executor uses it to
create `/dev/null` and friends inside each input root, and
on a surviving root the second action's `Mknod` hits
EEXIST. We propose the executor treat that as "already
correct" rather than failing the action.

The atomic-replace pattern leaves a temporary dentry
behind if the process dies in the microsecond between the
two syscalls. `scratch/` self-heals, because the cleaner
wipes the subtree between actions; the persistent trees do
not, so we propose `bb_worker` sweep them once at startup,
skipping entries newer than the sweep's own start so it
cannot race a materialization already in flight.

## Input root overlay semantics

When `bb_worker` re-acquires a persistent tree for a new
action it overlays the new action's input root onto the
existing one; unchanged files become no-ops via hardlink.
**The overlay does not remove inputs that exist on disk
but are not part of the new action's input root.** Worker
rules expect previous outputs to be visible in their
working directory, consistent with Bazel's local executor.

`tmp/` and `server_logs/` are wiped at the start of every
persistent worker action so they cannot leak across
requests — server logs in particular would otherwise be
re-uploaded with each subsequent action's response. The
input root proper is left alone, since its persistence is
the entire point of the feature.

# `bb_scheduler`: action key extraction

`bb_scheduler` matches an action's platform against
worker-pool advertised platforms using strict equality.
Worker pools advertise the stable, capability-defining
properties they support (OS, architecture, container
image, etc.); a property that varies per-action cannot be
advertised and would route every persistent-worker action
into an empty queue.

We propose to extend `PlatformKeyExtractorConfiguration`
with a configurable `ignore_property_names` list. Property
names in the list are stripped from the `Command.platform`
copy used to form the routing key, but **not** from the
`Command.platform` that reaches `bb_worker`. The
persistent-worker pool identity logic continues to see
them.

The well-known `persistentWorkerKey` is **always**
stripped, regardless of configuration. Its very purpose is
to be a per-action content hash, so there is no scenario
in which it could be part of a routing capability.

```jsonnet
actionRouter: {
  simple: {
    platformKeyExtractor: {
      action: {},
      ignorePropertyNames: ['client-hint', 'request-id'],
    },
    // ...
  },
},
```

If a deployment uses the `exec_properties`-driven
`supports-workers=1` activation path (rather than
`--experimental_remote_mark_tool_inputs`), the worker
pool's advertised `platform.properties` must include
`supports-workers=1` for the routing key to match. The
`persistentWorkerKey` strip does not help here because
activation is not via that property.

# Failure semantics

We propose the following distinctions, because they map
to operationally different responses:

- **Action-level failure** (non-zero
  `WorkResponse.exit_code`, e.g. a compile error). The
  worker is **not** killed. The next action on the same
  (key, working directory) reuses the same process. This
  matches Bazel's local executor.
- **Protocol error** (unparseable framing, mid-request
  EOF, size-cap rejection, `request_id` mismatch). The
  worker is killed and removed from the pool; the next
  action with the same (key, working directory) spawns a
  fresh one.
- **Context cancellation** while a request is in flight.
  The worker is killed — its codec pipe is closed, which
  unblocks the goroutines — so a wedged worker cannot pin
  a runner goroutine indefinitely.
- **Capacity exhaustion.** `Acquire` fails fast with
  `ResourceExhausted` rather than queueing, and the client
  retries through Bazel/REAPI backoff.
- **Tear-down.** Kills close stdin and wait briefly for
  the process to exit before falling back to SIGKILL. Both
  paths converge on the same cleanup, and which one ran is
  recorded on the kill log for post-mortem.

# Configuration surface

Three new configuration surfaces.

**`bb_runner`** gets a top-level `persistent_workers`
block holding the pool knobs: the instance cap, the idle
timeout, the request budget, the memory budget and its
sampling cadence, per-worker rlimits, and the optional
diagnostics directory. Every field is optional, numeric
fields disable at zero, and an absent block leaves the
runner behaving exactly as it does today — fork-exec every
action.

**`bb_worker`** gets a per-runner
`persistent_workers_enabled` bool, since runners live
under `build_directories[*].runners`; the split-root
layout is materialized only when it is set. A second,
process-wide block carries the codec fallback described
under "Codec", and an absent block there means "no
operator opinion", so a `proto`-only deployment configures
nothing at all. There is deliberately no per-key
concurrency knob — see Alternatives Considered for why an
in-tree cap was rejected.

**`bb_scheduler`** gets the `ignore_property_names`
repeated string on `PlatformKeyExtractorConfiguration`,
described above.

# Observability

We propose a Prometheus surface under
`buildbarn_runner_persistent_worker_*` — spawns, spawn
failures, kills by reason, live workers by state, parked
acquires, spawns in flight, pool-wide RSS — carrying a
`mnemonic` label so dashboards slice by rule kind without
dropping into per-key (sha256) labels.

The kill counter's `reason` label is the part worth
designing deliberately, because it mixes two event classes
that want different alerts. Policy-driven recycling — idle
timeout, LRU eviction, memory pressure, request budget,
pool close — is expected at a steady non-zero rate. Faults
and pre-emptions are not, and should page.

Metrics alone are not enough. Four worker death modes are
indistinguishable from outside the process — classpath
missing, launcher mishandled its arguments, cgroup OOM
kill, silent `System.exit(0)` — and telling them apart is
most of the operational cost of running JVM workers. We
propose a structured record on every kill, carrying the
reason, whether tear-down was graceful, the mnemonic and
action digest of the last action the worker served, the
exit status, the signal and peak RSS where the platform
supplies them, the tail of the worker's stderr, and any
JVM crash artefacts left in its cwd. We further propose an
optional diagnostics directory that tees each worker's
stderr to a file for live tailing and preserves those
crash artefacts before the build-directory cleaner can
race them.

Because the pool logs from inside critical sections, the
logger is to be non-blocking: records go to a buffered
channel drained by one goroutine, and overflow is counted
and reported rather than back-pressuring the pool.

# Alternatives considered

**Host the worker protocol on `bb_worker` rather than
`bb_runner`.** Rejected. `bb_worker` is REAPI-clean today
and unaware of host process concerns. Putting the codec,
the spawner, and the rlimit/PDEATHSIG/thread-pinning
machinery there would muddle the layering, and the per-
action `RunRequest`/`RunResponse` boundary between the
two components is the natural seam for "either fork-exec
or dispatch to a persistent worker — the caller
doesn't care."

**Per-process worker cache instead of per-key.**
Rejected. A flat cache keyed by argv-prefix only would
cause toolchain bumps to silently re-use stale workers.
Content-hash keys (Bazel's `persistentWorkerKey`) solve
this; we adopt them directly.

**In-place persistence on the same build directory used
by non-persistent actions.** Rejected. The cleaner that
runs on idle↔busy transitions walks
`RemoveAllChildren`, which destroys persistent state
mid-action. A skip-list inside the cleaner is fragile
(any future code path that walks the root needs to
know). The split-root layout is structural: nothing
that walks `scratch/` can reach `persistent/`.

**Per-instance subdir-id counter.** Rejected. With N
worker-thread instances each carrying an independent
counter, concurrent same-key acquires across instances
would each allocate id 0 → multiple goroutines into the
same `pw-<hash>-0/` directory → TOCTOU on the shared
tree. The counter must be a singleton at `bb_worker`
process scope.

**One-worker-per-key (no working-directory dimension).**
Rejected. Same-key actions arriving concurrently would
serialize on a single worker process. The
working-directory dimension lets `bb_worker`'s subdir
pool hand out disjoint trees per concurrent same-key
acquire, the runner pool grows in lockstep
(parallel-spawn), and warm reuse is preserved on the
released-id path. Cost is at most one extra worker per
parallel slot per key — bounded by external admission
control (see next entry).

**Per-key concurrency cap inside the persistent-worker
subsystem.** Rejected. A bounded per-key cap (on either
`bb_runner` or `bb_worker`) would prevent local per-key
overload — but it has no cluster-wide visibility.
Requests over the cap would park inside one worker pod
even when other pods in the cluster were idle for that
key, pessimizing cluster-level work distribution. Per-key
overload protection is a scheduler-level concern; stock
Buildbarn already provides the levers (worker
`Concurrency` to bound per-pod thread count; multiple
worker pod groups advertising distinct platform
properties so the scheduler can route heavy mnemonics
into a dedicated pool). Deployments needing finer
admission control can extend `bb_scheduler` with
additional protection features, but that is out of scope
for this ADR.

**`Remove + Create` for re-acquire idempotency.**
Rejected. Has a window where in-flight `exec()` /
`fork()` sees ENOENT on a path it expected to find.
Atomic Link/Symlink/OpenAppend + Rename is the only
race-free pattern.

**Stamp + reattach across `bb_runner` restarts (Tier 2
persistency).** Deferred. See "Scope of persistency"
above. Worth revisiting if usage data shows the
first-after-restart penalty matters.

**Cross-instance shared persistent dirs (Tier 3
persistency).** Declined. Operational cost dwarfs the
benefit; the scheduler's existing invocation-stickiness
already covers most of the locality win.

# Out of scope

These are explicit decisions, not gaps:

- **Multiplex workers** (`supports-multiplex-workers`).
  Single in-flight request per worker process.
  Multiplexing would require concurrent state in the
  codec and a worker-side thread pool to match. The path was not
  pursued for now because the rule-side support that
  would exercise it is incomplete — `rules_scala` does
  not support multiplexing at all, and `rules_java`'s
  multiplex path does not work out of the box on remote
  execution. Workarounds may exist, but the cost/benefit
  did not justify the work given the lack of a directly
  exercising consumer.
- **`WorkRequest.cancel`.** Not needed for single-
  request mode; ctx cancellation kills the worker
  instead.
- **`WorkRequest.inputs` population.** Bazel rules in
  widespread use don't rely on it; the input
  materialization is on disk via the input root.
- **`WorkRequest.sandbox_dir`.** Mutually exclusive
  with persistent workers in Bazel's own executor;
  same here.
- **Per-action `rusage` reporting for worker
  actions.** Workers span many actions; attributing
  CPU/RSS to a specific action is structurally
  ambiguous.
- **Synthetic worker dispatch in `CheckReadiness`.**
  Worker binaries are action-supplied; `bb_runner`
  has no binary in hand to smoke-test against. A
  misconfig that only surfaces under real actions
  remains a first-action failure.
- **cgroup-based resource caps.** rlimits via
  `prlimit(2)` are applied per worker process.
  cgroups would give per-pool aggregate caps, better
  OOM-killer ergonomics, and a kill that a descendant
  cannot escape by leaving the process group, but they
  require additional permissions / setup; deferred.
- **Persistent workers + `chroot_into_input_root`.**
  Mutually exclusive; we propose that `bb_runner`
  refuse to start when both are set. The chroot's
  lifecycle assumes per-action root creation/
  teardown, which is incompatible with the
  persistent root.

# Future work

- Tier 2 persistency (across `bb_runner` restarts) if
  metering shows the first-after-restart penalty is
  material. Would add a layout-version stamp, an
  integrity sweep, and a clean-shutdown bit.
- cgroup-based resource caps to complement
  `prlimit(2)`.
- Synthetic dispatch in `CheckReadiness` for
  deployments that pre-stage worker binaries
  out-of-band.
- Multiplex workers if/when a real workload depends
  on them.
