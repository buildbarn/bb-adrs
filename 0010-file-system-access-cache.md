# Buildbarn Architecture Decision Record #10: The File System Access Cache

Author: Ed Schouten<br/>
Date: 2023-01-20

# Context

Buildbarn's workers can be configured to expose input roots of actions
through a virtual file system, using either FUSE or NFSv4. One major
advantage of this feature is that it causes workers to only download
objects from the Content Addressable Storage (CAS) if they are accessed
by actions as part of their execution. For example, a compilation action
that ships its own SDK will only cause the worker to download the
compiler binary and standard libraries/headers that are used by the code
to be compiled, as opposed to downloading the entire SDK.

Using the virtual file system generally to make actions run faster. We
have observed that in aggregate, it allows actions to run at >95% of
their original speed, while eliminating the downloading phase that
normally leads up to execution. Also, the reduction in bandwidth against
storage nodes allows clusters to scale further.

Unfortunately, there are also certain workloads for which the virtual
file system performs poorly. As most traditional software reads files
from disk using synchronous APIs, actions that observe poor cache hit
rates can easily become blocked when the worker needs to make many round
trips to the CAS. Increased latency (distance) between storage and
workers only amplifies these delays.

A more traditional worker that loads all data up front would not be
affected by this issue, for the reason that it can use parallelism to
load many objects from the CAS. This means that one could avoid this
issue by creating a cluster consisting of both workers with, and without
a virtual file system. Tags could be added to build targets declared in
the user's repository to indicate which kind of worker should be used
for a given action, based on observed performance characteristics.
However, such a solution would provide a poor user experience.

# Introducing the File System Access Cache (FSAC)

What if there was a way for workers to load data up front, but somehow
restrict this process to data that the worker **thinks** is going to be
used? This would be optimal, both in terms of throughput and bandwidth
usage. Assuming the number of inaccuracies is low, this shouldn't cause
any harm:

- **False negatives:** In case a file or directory is not loaded up
  front, but does get accessed by the action, it can be loaded on demand
  using the logic that is already provided by the virtual file system
  and storage layer.

- **False positives:** In case a file or directory is loaded up front,
  but does not get used, it merely leads to some wasted network traffic
  and worker-level cache usage.

Even though it is possible to use heuristics for certain kinds of
actions to derive which paths they will access as part of their
execution, this cannot be done in the general case. Because of that, we
will depend on file system access profiles that we obtained from
historical executions of similar actions. These profiles will be stored
in a new data store, named the File System Access Cache (FSAC).

When we added the Initial Size Class Cache (ISCC) for the automatic
worker size selection, we observed that when combined,
[the Command digest](https://github.com/bazelbuild/remote-apis/blob/f6089187c6580bc27cee25b01ff994a74ae8658d/build/bazel/remote/execution/v2/remote_execution.proto#L458-L461)
and [platform properties](https://github.com/bazelbuild/remote-apis/blob/f6089187c6580bc27cee25b01ff994a74ae8658d/build/bazel/remote/execution/v2/remote_execution.proto#L514-L522)
are very suitable keys for grouping similar actions together. The FSAC
will thus use the same 'reduced Action digests' as its keys as the ISCC.

As the set of paths accessed can for some actions be large, we're going
to apply two techniques to ensure the profiles stored in the FSAC remain
small:

1. Instead of storing all paths in literal string form, we're going to
   store them as a Bloom filter. Bloom filters are probabilistic data
   structures, which have a bounded and configurable false positive
   rate. Using Bloom filters, we only need 14 bits of space per path to
   achieve a 0.1% false positive rate.
1. For actions that acess a very large number of paths, we're going to
   limit the maximum size of the Bloom filter. Even though this will
   increase the false positive rate, this is likely acceptable. It is
   uncommon to have actions that access many different paths, **and**
   leave a large portion of the input root unaccessed.

# Tying the FSAC into bb\_worker

To let bb\_worker make use of the FSAC, we're going to add a
PrefetchingBuildExecutor that does two things in parallel:

1. Immediately call into an underlying BuildExecutor to launch the
   action as usual.
1. Load the file system access profile from the FSAC, and fetch files in
   the input root that are matched by the Bloom filter contained in the
   profile.

The worker and the action will thus 'race' with each other to see who
can fetch the input files first. If the action wins, we terminate the
prefetching.

Furthermore, we will extend the virtual file system layer to include the
hooks that are necessary to measure which parts of the input root are
being read. We can wrap the existing InitialContentsFetcher and Leaf
types to detect reads against directories and files, respectively. As
these types are strongly coupled to the virtual file system layer, we
will add new {Unread,Read}DirectoryMonitor interfaces that
PrefetchingBuildExecutor can implement to listen for file system
activity in a loosely coupled fashion.

Once execution has completed and PrefetchingBuildExecutor has captured
all of the paths that are being accessed, it can write a new file system
access profile into the FSAC. This only needs to be done if the profile
differs from the one that was fetched from the FSAC at the start of the
build.

# Tying the FSAC into bb\_browser

The file system access profiles in the FSAC may also be of use for
people designing their own build rules. Namely, the profiles may be used
to determine whether the input root contains data that can be omitted.
Because of that, we will extend bb\_browser to overlay this information
on top of the input root that's displayed on the action's page.

# Future work

Another interesting use case for the profiles stored in the FSAC is that
they could be used to perform weaker lookups against the Action Cache.
This would allow us to skip execution of actions in cases where we know
that changes have only been made against files that are not going to be
accessed anyway.
