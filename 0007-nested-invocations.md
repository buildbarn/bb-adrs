# Buildbarn Architecture Decision Record #7: Nested invocations

Author: Ed Schouten<br/>
Date: 2021-12-27

# Context

On 2020-11-16, bb\_scheduler was extended to take the Bazel invocation
ID into account ([commit](https://github.com/buildbarn/bb-remote-execution/commit/1a8407a0e62bf559abcd4006e0ecc9d3de9838c7)).
With this change applied, bb\_scheduler no longer places all incoming
operations in a single FIFO-like queue (disregarding priorities).
Instead, every invocation of Bazel gets its own queue. All of these
queues combined are placed in another queue that determines the order of
execution. The goal of this change was to improve fairness, by making
the scheduler give every invocation an equal number of workers.

Under the hood, [`InMemoryBuildQueue` calls into `ActionRouter`](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/scheduler/in_memory_build_queue.go#L378), which then [calls into `invocation.KeyExtractor`](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/scheduler/routing/simple_action_router.go#L40)
to obtain an [`invocation.Key`](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/scheduler/invocation/key.go).
The latter acts as [a map key](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/scheduler/in_memory_build_queue.go#L1201)
for all invocations known to the scheduler.
[The default implementation](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/scheduler/invocation/request_metadata_key_extractor.go)
is one that extracts [the `tool_invocation_id` field from the REv2 `RequestMetadata` message](https://github.com/bazelbuild/remote-apis/blob/636121a32fa7b9114311374e4786597d8e7a69f3/build/bazel/remote/execution/v2/remote_execution.proto#L1836).

What is pretty elegant about this design is that `InMemoryBuildQueue`
doesn't really care about what kind of logic is used to compute
invocation keys. One may, for example, provide a custom
`invocation.KeyExtractor` that causes operations to be grouped by
username, or by [the `correlated_invocations_id` field](https://github.com/bazelbuild/remote-apis/blob/636121a32fa7b9114311374e4786597d8e7a69f3/build/bazel/remote/execution/v2/remote_execution.proto#L1840).

On 2021-07-16, bb\_scheduler was extended once more to use Bazel
invocation IDs to provide worker locality ([commit](https://github.com/buildbarn/bb-remote-execution/commit/902aab278baa45c92d48cad4992c445cfc588e32)).
This change causes the scheduler to keep track of invocation ID of the
last operation to run on a worker. When scheduling successive tasks, the
scheduler will try to reuse a worker with a matching invocation ID. This
increases the probability of input root cache hits, as it is not
uncommon for clients to run many actions having similar input root
contents.

At the same time, this change introduced [the notion of 'stickiness'](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/proto/configuration/bb_scheduler/bb_scheduler.proto#L177-L196),
where a cluster operator can effectively quantify the overhead of
switching between actions belonging to different invocations. Such
overhead could include the startup time of virtual machines, download
times of container images specified in REv2 platform properties. In the
case embedded hardware testing, it could include the time it takes to
flash an image to EEPROM/NAND and boot from it. By using a custom
`invocation.KeyExtractor` that extracts these properties from incoming
execution requests, the scheduler can reorder operations in such a way
that the number of restarts, reboots, flash erasure cycles, etc. is
reduced.

One major limitation of how this feature is implemented right now, is
that invocations are a flat namespace. The scheduler only provides
fairness by keying execution requests by a single property. For example,
there is no way to configure the scheduler to provide the following:

- Fairness by username, followed by fairness by Bazel invocation ID for builds
  with an equal username.
- Fairness by team/project/department, followed by fairness by username.
- Fairness both by correlated invocations ID and tool invocation ID.
- Fairness/stickiness by worker container image, followed by fairness by
  Bazel invocation ID.

In this ADR we propose to remove this limitation, with the goal of
improving fairness of multi-tenant Buildbarn setups.

# Proposed changes

This ADR mainly proposes that operations no longer have a single
`invocation.Key`, but a list of them. The flat map of invocations
managed by `InMemoryBuildQueue` will be changed to a
[trie](https://en.wikipedia.org/wiki/Trie), where every level of the
trie corresponds to a single `invocation.Key`. More concretely, if the
scheduler is configured to not apply any fairness whatsoever, all
operations will simply be queued within the root invocation. If the
`ActionRouter` is configured to always return two `invocation.Key`s, the
trie will have height two, having operations queued only at the leaves.

When changing this data structure, we'll also need to rework all of the
algorithms that are applied against it. For example, we currently use a
[binary heap](https://en.wikipedia.org/wiki/Binary_heap) to store all
invocations that have one or more queued operations, which is used for
selecting the next operation to run. This will be replaced with a heap
of heaps that mimics the layout of the trie. These algorithms will thus
all become a bit more complex than before.

Fortunately, there are also some places where the algorithms become
simpler. For the worker locality, we currently have to deal with the
case where the last task that ran on a worker was associated with
multiple invocations (due to in-flight deduplication). Or it may not be
associated with any invocation if the worker was created recently. In
the new model we can simply pick the
[lowest common ancestor](https://en.wikipedia.org/wiki/Lowest_common_ancestor)
of all invocations in case of in-flight deduplication, and pick the root
invocation for newly created workers. This ensures that every worker is
always associated with exactly one invocation.

For `InMemoryBuildQueue` to obtain multiple `invocation.Key`s for a
given operation, we will need to alter `ActionRouter` to return a list
instead of a single instance. The underlying `invocation.KeyExtractor`
interface can remain as is, though we can remove
[`invocation.EmptyKeyExtractor`](https://github.com/buildbarn/bb-remote-execution/blob/6d5d7a67f5552f9b961a7e1d81c3cd77f2086004/pkg/scheduler/invocation/empty_key_extractor.go).
The same behaviour can now be achieved by not using any key extractors
at all.

Finally we have to extend the worker invocation stickiness logic to work
with this nested model. As every level of the invocation tree now has a
different cost when switching to a different branch, we now need to let
the configuration accept a list of durations, each indicating the cost
at that level.

All of the changes proposed above have been implemented in
[this commit](https://github.com/buildbarn/bb-remote-execution/commit/744c9a65215179f19f20523acd522937a03aad56).

# Future work

With `invocation.EmptyKeyExtractor` removed, only
`invocation.RequestMetadataKeyExtractor` remains. This type is capable
of creating an `invocation.Key` based on the `tool_invocation_id`. We
should add more types, such as one capable of extracting the
`correlated_invocations_id` or authentication related properties (e.g.,
username).

Even though the scheduling policy of `InMemoryBuildQueue` is fair, there
is no support for [preemption](https://en.wikipedia.org/wiki/Preemption_(computing)).
At no point will the scheduler actively move operations from the
`EXECUTING` back to the `QUEUED` stage if it prevents other invocations
from making progress.
