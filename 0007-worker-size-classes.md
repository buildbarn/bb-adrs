# Buildbarn Architecture Decision Record #7: Worker size classes

Author: Ed Schouten<br/>
Date: 2021-05-14

# Context

To reduce flakiness, it makes sense to run bb\_runner inside an
environment that both enforces and guarantees strict resource limits.
For example, when running Buildbarn on top of Kubernetes, one may
declare [memory](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
and [CPU resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/).
By using [the static CPU manager policy](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/),
system calls like `sched_getaffinity(2)` will also return a CPU core
count that matches the configured CPU limits, ensuring that actions
create thread pools that are properly sized. By enabling these options,
it's generally safe to run a large number of bb\_runner processes on a
single system without risking that overconsumption of resources of one
build action leads to slowdowns or failures of other actions that happen
to run concurrently.

A disadvantage of setting up such limits is that they may lead to
reduced utilization rates. When looking at the resource utilization of
actions in aggregate, the distribution generally tends to be
[Pareto-like](https://en.wikipedia.org/wiki/Pareto_distribution). Most
of the actions are single-threaded and use less than 100 MB of RAM,
while a couple of outliers can benefit from multi-threading and/or use
multiple gigabytes of memory. Provisioning all workers for the
worst-case resource utilization is not desirable.

One way to address this is to provide workers that are identical in
terms of execution environment, but have different CPU counts and memory
sizes. Workspaces may be configured to run actions on smaller workers by
default, having explicit annotations on targets to run actions on larger
workers as needed. The routing of actions takes place by giving the
different sets of workers distinct REv2 platform properties. There are,
however, a couple of downsides to this approach:

- Even though Bazel's mechanisms for defining platform properties are
  improving in terms of granularity (e.g., through the recently added
  [execution groups](https://docs.bazel.build/versions/master/exec-groups.html)),
  there are still many places where platform properties cannot be
  controlled on a per-action basis. For example, for sharded tests it is
  not possible to control platform properties on a per-shard basis.
- Users who are unaware of cost implications may use the annotations too
  liberally, causing builds to become needlessly expensive.
- As a codebase evolves, annotations to increase resource allocations
  may become unnecessary. Regular processed need to be defined to
  periodically check whether annotations are still necessary, and
  remove/readjust them where needed.
- Changes to how clusters are set up (both in terms of hardware and
  software) may require that annotations in the codebase are adjusted.
  This means that any changes made by cluster operators may cause builds
  to break for its users. This makes it harder to operate Buildbarn as a
  managed service, especially when engagement between users and cluster
  operators is intended to be minimal.
- It is virtually impossible for third-party libraries to ship with
  BUILD files that are annotated accurately, as there is no
  standardization in terms of platform properties.

# Proposal

The goal of this ADR is to propose changes to Buildbarn, making it
capable of automatically choosing an optimally sized worker based on
statistics gathered from previous executions of similar actions.

## Size classes

First and foremost, we need to make Buildbarn aware of the concept of
sizes of workers. Whereas the scheduler currently groups workers by an
instance name and platform properties, the new scheduler will need to
take a size class into account. This allows all workers, regardless of
their size, to accept requests for the same platform properties.

We define size classes as positive integer values that denote how large
a worker is relative to others of the same platform. It is recommended
that all resource types (CPU count, memory size, disk quota) are scaled
linearly, though this is not a strong requirement. Below is an example
of what the specifications of workers using size classes may look like.

| Size class | vCPU count | Memory size | Disk quota |
| ---------- | ---------- | ----------- | ---------- |
| 1          | 1          | 4 GiB       | 10 GiB     |
| 2          | 2          | 8 GiB       | 20 GiB     |
| 4          | 4          | 16 GiB      | 40 GiB     |
| 8          | 8          | 32 GiB      | 80 GiB     |

The size class of a worker can be configured in bb\_worker's
configuration file, similar to how platform properties are configured.

## Size class selection and learning

When the scheduler receives actions from clients and enqueues them, it
will need to analyze the action to select a size class on which to run
the action. It is not unlikely that the prediction is incorrect. Such a
misprediction may cause execution to fail or slow down. We should make
sure that the user-noticeable impact of this remains minimal:

- If a failure or timeout occurred on a worker that isn't part of the
  largest size class, the action must be retried on the largest size
  class. This rules out the possibility that the failure was caused by a
  lack of resources.
- If an action runs unusually slow on a small size class, it should not
  cause the build as a whole to stall. For example, if we know from
  historical measurements that an action always completes within a
  minute on an 8-core worker, running it for half an hour on a single
  core worker is pointless. The system should detect such stalls,
  killing the action and rescheduling the action on a larger worker.

To facilitate this, we'll add an Analyzer interface that
InMemoryBuildQueue will call into to fingerprint an action. The
resulting Selector will pick a size class on which the action should run
initially, based on a list of size classes available at that time. It
also returns an execution timeout that the worker processing the action
should respect, which may be lower than the timeout requested by the
client. The Learner object can be used to capture the outcome of the
execution on the size class, but also to request re-execution of the
action on the largest size class in case of failures.

```go
type Analyzer interface {
	// Analyze an REv2 Action. This operation should be called
	// without holding any locks, as it may to block.
	Analyze(ctx context.Context, digestFunction digest.Function, action *remoteexecution.Action) (Selector, error)
}

type Selector interface {
	// Given a list of size classes that are currently present,
	// select a size class on which the action needs to be
	// scheduled.
	Select(sizeClasses []uint32) (int, time.Duration, Learner)
	// Clients have abandoned the action, meaning that no size class
	// selection decision needs to be made.
	Abandoned()
}

type Learner interface {
	// The action completed successfully.
	Succeeded(duration time.Duration)
	// The action completed with a failure. If this method returns a
	// nil Learner, the execution failure is definitive and should
	// be propagated to the client. If this method returns a new
	// Learner, execution must be retried on the largest size class.
	Failed(timedOut bool) (time.Duration, Learner)
	// Clients have abandoned the action, meaning that execution of
	// the action was terminated.
	Abandoned()
}
```

Right now bb\_worker is responsible for applying a default execution
timeout and enforcing a maximum value. Because this logic may interact
poorly with the timeout returned by Selector.Select(), we're going to
make the interfaces above completely responsible for picking the
execution timeout. Workers are simply going to trust the value that the
scheduler provides.

## Initial Size Class Cache (ISCC)

The statistics that are gathered by instances of Learner are valuable
information, as losing them means that successive builds are slower and
more expensive. We should provide facilities for preserving this data
across scheduler restarts. One easy way to achieve this is to reuse
blobstore, the layer that is already responsible for storing AC, CAS
and ICAS contents (see [ADR#4](0004-icas.md)). Using blobstore, we can
create a new key-value store where the key corresponds to a digest and
the value corresponds to a Protobuf message. We'll call this data store
the Initial Size Class Cache, or ISCC for short.

For the key, we want to use a digest that only captures the shape of the
action, ignoring aspects that tend to differ when (minor) code changes
occur. An ideal candidate would have been the digest of the REv2 Command
message, but with
[platform properties being promoted from Command To Action](https://github.com/bazelbuild/remote-apis/pull/167)
it fails to capture all the information we need. This is why we'll
instead recompute a digest of the Action message having all irrelevant
fields stripped.

For the value, we'll declare a simple Protobuf message named
PreviousExecutionStats that mostly just contains a series of recent
execution outcomes grouped by size class.

```proto
message PreviousExecution {
  oneof outcome {
    google.protobuf.Empty failed = 1;
    google.protobuf.Duration timed_out = 2;
    google.protobuf.Duration succeeded = 3;
  }
}

message PerSizeClassStats {
  repeated PreviousExecution previous_executions = 1;
  ...
}

message PreviousExecutionStats {
  map<uint32, PerSizeClassStats> size_classes = 1;
  ...
}
```

bb\_browser will be extended to display this data as well, making it
easier to see what's going on underneath the hood.

## Size class selection algorithm

The intent is not just to run actions on larger size classes in case
they fail on smaller size classes. We also want to use those workers if
we know build times improve. For this we first need to quantify what
counts as an improvement. How much faster does an action need to run on
a larger size class for it to be worth the costs? Alternative phrasing:
how much slowdown are we willing to tolerate, knowing that execution on
a smaller size class is cheaper? A 5% reduction in execution time isn't
worth doubling resources for. On the other hand, a 50% reduction (i.e.
true linear scaling) is often not attainable. The desired cutoff is
somewhere in between.

To make this quantifiable, we can use execution times observed on the
largest size class to determine what acceptable execution times on
smaller workers are. In our initial implementation we will use the
following expression:

T<sub>acceptable</sub> = T<sub>largest</sub> ⋅ (S<sub>largest</sub> / S<sub>smaller</sub>)<sup>x</sup>

In this expression, T denote execution times, while S denote size class
values. Exponent x is user-configurable, indicating how strongly
execution times need to improve on larger size classes. Alternative
phrasing: how strongly execution times may increase on smaller size
classes. Below is a table of acceptable execution times for an action
that is known to complete in 1 minute on a worker with size class 8,
using different values of x.

| Size class   | x = 0.3 | x = 0.5 | x = 0.7 |
| ------------ | ------- | ------- | ------- |
| 1            | 112.0s  | 169.7s  | 257.2s  |
| 2            | 90.9s   | 120s    | 158.3s  |
| 4            | 73.9s   | 84.9s   | 97.5s   |
| 8 (baseline) | 60s     | 60s     | 60s     |

Once we know the acceptable execution time of an action on a given size
class, we can compare it against all outcomes previously observed on
that size class to compute a probability of success. Because outcomes
may also include failures and timeouts, it makes most sense to treat all
of the outcomes as [Bernouilli trials](https://en.wikipedia.org/wiki/Bernoulli_trial).
We can use the resulting probability directly to randomly choose whether
the action should run on a smaller size class.

When scheduling actions on a smaller size class, we want to use an
execution timeout that is proportional to the acceptable execution time.
Using the acceptable execution time literally introduces a high
probability of failure if actions on smaller size classes run very close
to what is considered acceptable. This is why we should allow specifying
a small multiplier (e.g., 1.5) that can be used to increase the timeout
on smaller size classes, thereby adding some tolerance.

For actions that only take a fraction of a time to run, the feedback
driven analysis that we perform may be far too granular. At some point
other factors such as client latency dominate the overall user
experience. For example, if we determine that an action generally
completes within 25ms on the largest size class, we don't want to
schedule actions on a smaller size class using a 75ms timeout. It makes
the probability of timeouts far too high. We'll prevent this from
happening by allowing a minimum acceptable execution time to be
configured.

In the absence of execution times on the largest size class, we can also
use the minimum acceptable execution time to attempt to run actions on
smaller size classes. This prevents the need for forcing all actions to
be run on the largest size class initially to obtain a baseline
execution time, making builds of a brand new codebase less expensive.

In case actions fail completely (i.e., on the largest size class), the
failure should be propagated back to the client. When this happens, it
is fairly likely that other failures for actions of the same shape will
occur later on. For example, if a user is repeatedly compiling a single
source file to fix compiler errors, we will see many failing invocations
of similar actions. To prevent this from adding unnecessary delays, we
can add some caching, temporarily 'boosting' such actions to only run on
the largest size class.

The algorithm described above is going to be implemented in Buildbarn as
part of type FeedbackDrivenAnalyzer. It will use a
PreviousExecutionStatsPool to read entries from the Initial Size Class
Cache and update them.

# Future work

For this initial implementation, we never run actions in parallel.
Instead of terminating actions on a smaller size class once a reduced
timeout value has reached, it might make more sense to spawn a second
execution on the largest size class in parallel. This ensures that
execution times remain optimal for actions that don't have any
parallelism.

This implementation doesn't take any scheduler queue sizes into account.
Even if an action has an elevated probability of failing on a small size
class, it may still be worth attempting it if workers for larger size
classes have long queues.

The current implementation of FeedbackDrivenAnalyzer only uses outcomes
of a given size class to make decisions related to that size class. For
example, if an action is known to fail a lot on size class 4, it won't
use this information when deciding to run the action on size class 2.

It is questionable whether the reduced Action digest used by the Initial
Size Class Cache (ISCC) is accurate enough for fingerprinting actions
with similar execution profiles. For example, changing a C/C++ compiler
flag may not cause the digest for a test target to change. It may make
sense to take [RequestMetadata.configuration_id](https://github.com/bazelbuild/remote-apis/blob/ce7036ef54173c2d5a507f0bb3d9a80ec51c33c8/build/bazel/remote/execution/v2/remote_execution.proto#L1794)
into account. Unfortunately, it's only stored in gRPC metadata, meaning
that any tools that don't preserve this information (e.g., a tool for
replaying build actions) would no longer get cache hits.
