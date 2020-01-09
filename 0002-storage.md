# Buildbarn Architecture Decision Record #2: Towards fast and reliable storage

Author: Ed Schouten<br/>
Date: 2020-01-09

# Context

Initial versions of Buildbarn as published on GitHub made use of a
combination of Redis and S3 to hold the contents of the Action Cache
(AC) and Content Addressable Storage (CAS). Due to their performance
characteristics, Redis was used to store small objects, while S3 was
used to store large ones (i.e., objects in excess of 2 MiB). This
partitioning was facilitated by the SizeDistinguishingBlobAccess storage
class.

Current versions of Buildbarn still support Redis and S3, but the
example deployments use bb\_storage with the "circular" storage backend
instead. Though there were valid reasons for this change at the time,
these reasons were not adequately communicated to the community, which
is why users are still tempted to use Redis and S3 up until this day.

The goal of this document is as follows:

- To describe the requirements that Bazel and Buildbarn have on storage.
- To present a comparative overview of existing storage backends.
- To propose changes to Buildbarn's storage architecture that allow us
  to construct a storage stack that satisfies the needs for most large
  users of Buildbarn.

# Storage requirements of Bazel and Buildbarn

In the common case, Bazel performs the following operations against
Buildbarn when executing a build action:

1. ActionCache.GetActionResult() is called to check whether a build
   action has already been executed previously. This call extracts an
   ActionResult message from the AC. If such a message is found, Bazel
   continues with step 5.
2. Bazel constructs a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree)
   of Action, Command and Directory messages and associated input files.
   It then calls ContentAddressableStorage.FindMissingBlobs() to
   determine which parts of the Merkle tree are not present in the CAS.
3. Any missing nodes of the Merkle tree are uploaded into the CAS using
   ByteStream.Write().
4. Execution of the build action is triggered through
   Execution.Execute(). Upon successful completion, this function
   returns an ActionResult message.
5. Bazel downloads all of the output files referenced by the
   ActionResult message from the CAS to local disk using
   ByteStream.Read().

By letting Bazel download all output files from the CAS to local disk,
there is a guarantee that Bazel is capable of making forward progress.
If any of the objects referenced by the build action were to disappear
during these steps, the RPCs will either fail with code `NOT_FOUND` or
`FAILED_PRECONDITION`. This instructs Bazel to reupload inputs from
local disk and retry. Bazel therefore never has to perform any
backtracking on its build graph, an assumption that is part of its
design.

This implementation places relatively weak requirements on Buildbarn in
terms of consistency and persistence. Given a sufficient amount of
storage space, Buildbarn will generally make sure that input files don't
disappear between ContentAddressableStorage.FindMissingBlobs() and the
completion of Execution.Execute(). It will also generally make sure that
output files remain present at least long enough for Bazel to download
them. Violating these weak requirements only affects performance; not
reliability.

Bazel recently gained a feature called
['Builds without the Bytes'](https://github.com/bazelbuild/bazel/issues/6862).
By enabling this feature using the `--remote_download_minimal` command
line flag, Bazel will no longer attempt to download output files to
local disk. This feature causes a significant drop in build times and
network bandwidth consumed. This is especially noticeable for workloads
that yield large output files. Buildbarn should attempt to support those
workloads.

Even with 'Builds without the Bytes' enabled, Bazel assumes it is always
capable of making forward progress. This strengthens the storage
requirements on Buildbarn as follows:

1. Bazel no longer downloads output files referenced by
   ActionResult messages, but may use them as inputs for other build
   actions at any later point during the build. This means that
   ActionCache.GetActionResult() and Execution.Execute() may never return
   ActionResult messages that refer to objects that are either not
   present or not guaranteed to remain present during the remainder of
   the build.
2. Technically speaking, it only makes sense for Bazel to call
   ContentAddressableStorage.FindMissingBlobs() against objects that
   Bazel is capable of uploading. With 'Builds without the Bytes'
   enabled, this set no longer has to contain output files of dependent
   build actions. However, this specific aspect is not implemented by
   Bazel. It still passes in the full set. Even worse: returning
   these objects as absent tricks Bazel into uploading files that are
   not present locally, causing a failure in the Bazel client.

To tackle the first requirement, Buildbarn recently gained a
CompletenessCheckingBlobAccess decorator that causes bb\_storage to only
return ActionResult entries from the AC in case all output files are
present in the CAS. Presence is checked by calling
BlobAccess.FindMissing() against the CAS, which happens to be the same
function used to implement ContentAddressableStorage.FindMissingBlobs().

This places strong requirements on the behaviour of
BlobAccess.FindMissing(). To satisfy the first requirement,
BlobAccess.FindMissing() may not under-report the absence of objects. To
satisfy the second requirement, it may not over-report. In other words,
it has to be *exactly right*.

Furthermore, BlobAccess.FindMissing() now has an additional
responsibility. Instead of only reporting absence, it now has to touch
objects that are present, ensuring that they don't get evicted during
the remainder of the build. Because Buildbarn itself has no notion of
full build processes (just individual build actions), this generally
means that Buildbarn needs to hold on to the data as long as possible.
This implies that underlying storage needs to use an LRU or pseudo-LRU
eviction policy.

# Comparison of existing storage backends

Now that we've had an overview of our storage requirements, let's take a
look at the existing offering of Buildbarn storage backends. In addition
to consistency requirements, we take maintenance aspects into account.

## Redis

Advantages:

- Lightweight, bandwidth efficient protocol.
- An efficient server implementation exists.
- Major Cloud providers offer managed versions: Amazon ElastiCache,
  Google Cloud Memorystore, Azure Cache for Redis, etc.
- Pipelining permits efficiently implementing FindMissing() by sending a
  series of `EXISTS` requests.
- [Supports LRU caching.](https://redis.io/topics/lru-cache)

Disadvantages:

- The network protocol isn't multiplexed, meaning that clients need to
  open an excessive number of network sockets, sometimes up to dozens
  of sockets per worker, per storage backend. Transfers of large objects
  (files) may hold up requests for small objects (Action, Command and
  Directory messages).
- Redis is not designed to store large objects. The maximum permitted
  object size is 512 MiB. If Buildbarn is used to generate installation
  media (e.g., DVD images), it is desirable to generate artifacts that
  are multiple gigabytes in size.
- The client library that is currently being used by Buildbarn,
  [go-redis](https://github.com/go-redis/redis), does not use the
  [context](https://golang.org/pkg/context/) package, meaning that
  handling of cancelation, timeouts and retries is inconsistent with how
  the rest of Buildbarn works. This may cause unnecessary propagation of
  transient error conditions and exhaustion of connection pools. Of
  [all known client library implementations](https://redis.io/clients#go),
  only [redispipe](https://github.com/joomcode/redispipe) uses context.
- go-redis does not support streaming of objects, meaning that large
  objects need to at some point be stored in memory contiguously, which
  causes heavy fluctuations in memory usage. Only the unmaintained
  [shipwire/redis](https://github.com/shipwire/redis) library has an
  architecture that would support streaming.

Additional disadvantages of Redis setups that have replication enabled,
include:

- They violate consistency requirements. A `PUT` operation does not
  block until the object is replicated to read replicas, meaning that a
  successive `GET` may fail. Restarts of read replicas may cause objects
  to be reported as absent, even though they are present on the master.
- It has been observed that performance of replicated setups decreases
  heavily under high eviction rates.

## Amazon S3, Google Cloud Storage, Microsoft Azure Blob, etc.

Advantages:

- Virtually free of maintenance.
- Decent SLA.
- Relatively inexpensive.

Disadvantages:

- Amazon S3 is an order of magnitude slower than Amazon ElastiCache
  (Redis).
- Amazon S3 does not provide the right consistency model, as
  [it only guarantees eventual consistency](https://docs.aws.amazon.com/AmazonS3/latest/dev/Introduction.html#ConsistencyModel)
  for our access pattern.
  [Google Cloud Storage offers stronger consistency.](https://cloud.google.com/blog/products/gcp/how-google-cloud-storage-offers-strongly-consistent-object-listing-thanks-to-spanner)
- There are no facilities for performing bulk existence queries, meaning
  it is hard to efficiently implement FindMissing().
- Setting TTLs on objects will make a bucket behave like a FIFO cache,
  which is incompatible with 'Builds without the Bytes'.
  Turning this into an LRU cache requires triggering `COPY` requests
  upon access. Again, due to the lack of bulk queries, calls like
  FindMissing() will become even slower.

## bb\_storage with the "circular" storage backend

Advantages:

- By using gRPC, handling of cancelation, timeouts and retries is either
  straightforward or dealt with automatically.
- [gRPC has excellent support for OpenTracing](https://github.com/grpc-ecosystem/grpc-opentracing),
  meaning that it will be possible to trace requests all the way from
  the user to the storage backend.
- gRPC has excellent support for multiplexing requests, meaning the
  number of network sockets used is reduced dramatically.
- By using gRPC all across the board, there is rich and non-ambiguous
  error message propagation.
- bb\_storage exposes decent Prometheus metrics.
- The "circular" storage backend provides high storage density. All
  objects are concatenated into a single file without any form of
  padding.
- The "circular" storage backend has a hard limit on the amount of
  storage space it uses, meaning it is very unlikely storage nodes will
  run out of memory or disk space, even under high load.
- The performance of the "circular" storage backend doesn't noticeably
  degrade over time. The data file doesn't become fragmented. The hash
  table that is used to look up objects uses a scheme that is inspired
  by [cuckoo hashing](https://en.wikipedia.org/wiki/Cuckoo_hashing). It
  prefers displacing older entries over newer ones. This makes the hash
  table self-cleaning.

Disadvantages:

- The "circular" storage backend implements a FIFO eviction scheme,
  which is incompatible with 'Builds without the Bytes'.
- The "circular" storage backend is known to have some design bugs that
  may cause it to return corrupted data when reading blobs close to the
  write cursors.
- When running Buildbarn in the Cloud, a setup like this has more
  administrative overhead than Redis and S3. Scheduling bb\_storage on
  top of Kubernetes using stateful sets and persistent volume claims may
  require administrators to periodically deal with stuck pods and
  corrupted volumes. Though there are likely various means to automate
  these steps (e.g., by running recovery cron jobs), it is not free in
  terms of effort.

## bb\_storage with the "inMemory" storage backend

Advantages:

- The eviction policy can be set to FIFO, LRU or random eviction, though
  LRU should be preferred for centralized storage.
- Due to there being no persistent state, it is possible to run
  bb\_storage through a simple Kubernetes deployment, meaning long
  amounts of downtime are unlikely.
- Given a sufficient amount of memory, it deals well with objects that
  are gigabytes in size. Both the client and server are capable of
  streaming results, meaning memory usage is predictable.

In addition to the above, it shared the gRPC-related advantages of
bb\_storage with the "circular" storage backend.

Disadvantages:

- There is no persistent storage, meaning all data is lost upon crashes
  and power failure. This will cause running builds to fail.
- By placing all objects in separate memory allocations, it puts a lot
  of pressure on the garbage collector provided by the Go runtime. For
  worker-level caches that are relatively small, this is acceptable. For
  centralized caching, this becomes problematic. Because Go does not
  ship with a defragmenting garbage collector, heap fragmentation grows
  over time. It seems impossible to set storage limits anywhere close to
  the available amount of RAM.

# A new storage stack for Buildbarn

Based on the comparison above, replicated Redis, S3 and bb\_storage with
the "circular" storage backend should **not** be used for Buildbarn's
centralized storage. They do not provide the right consistency
guarantees to satisfy Bazel. That said, there may still be valid use
cases for these backends in the larger Buildbarn ecosystem. For example,
S3 may be a good fit for archiving historical build actions, so that
they may remain accessible through bb\_browser. Such use cases are out
of scope for this document.

This means that only non-replicated Redis and bb\_storage with the
"inMemory" storage backend remain viable from a consistency standpoint,
though both of these also have enough practical disadvantages that they
cannot be used (no support for large objects and excessive heap
fragmentation, respectively).

This calls for the creation of a new storage stack for Buildbarn. Let's
first discuss what such a stack could look like for simple, single-node
setups.

## Single-node storage

For single-node storage, let's start off with designing a new storage
backend named "local" that is initially aimed at replacing the
"inMemory" and later on the "circular" backends. Instead of coming up
with a brand-new design for such a storage backend, let's reuse the
overall architecture of "circular", but fix some of the design flaws it
had:

- **Preventing data corruption:** Data corruption in the "circular"
  backend stems from the fact that we overwrite old blobs. Those old
  blobs may still be in the process of being downloaded by a user. An
  easy way to prevent data corruption is thus to stop overwriting
  existing data. Instead, we can let it keep track of its data by
  storing it in a small set of blocks (e.g., 10 blocks that are 1/10th
  the size of total storage). Whenever all blocks become full, the
  oldest one is discarded and replaced by a brand new block. By
  implementing reference counting (or relying on the garbage collector),
  requests for old data may complete without experiencing any data
  corruption.
- **Pseudo-LRU:** Whereas "circular" uses FIFO eviction, we can let our
  new backend provide LRU-like eviction by copying blobs from older
  blocks to newer ones upon access. To both provide adequate performance
  and reduce redundancy, this should only be performed on blobs coming
  from blocks that are beyond a certain age. By only applying this
  scheme to the oldest 1/4 of data, only up to 1/3 of data may be stored
  twice.
- **Preventing 'tidal waves':** The "circular" backend would be prone to
  'tidal waves' of writes. When objects disappear from storage, it is
  highly probable that many related files disappear at the same time.
  With the pseudo-LRU design, the same thing applies to refreshing
  blobs: when BlobAccess.FindMissing() needs to refresh a single file
  that is part of an SDK, it will likely need to refresh the entire SDK.
  We can amortize the cost of this process by smearing writes across
  multiple blocks, so that they do not need to be refreshed at the same
  time.

Other Buildbarn components (e.g., bb\_worker) communicate to the
single-node storage server by using the following blobstore
configuration:

```jsonnet
{
  local config = { grpc: { address: 'bb-storage.example.com:12345' } },
  contentAddressableStorage: config,
  actionCache: config,
}
```

## Adding scalability

For larger setups it may be desired to store more data in cache than
fits in memory of a single system. For these setups it is suggested that
the already existing ShardingBlobAccess is used:

```jsonnet
{
  local config = {
    sharding: {
      hashInitialization: 3151213777095999397,  // A random 64-bit number.
      shards: [
        {
          backend: { grpc: { address: 'bb-storage-0.example.com:12345' } },
          weight: 1,
        },
        {
          backend: { grpc: { address: 'bb-storage-1.example.com:12345' } },
          weight: 1,
        },
        {
          backend: { grpc: { address: 'bb-storage-2.example.com:12345' } },
          weight: 1,
        },
      ],
    },
  },
  contentAddressableStorage: config,
  actionCache: config,
}
```

In the example above, the keyspace is divided across three storage
backends that will each receive approximately 33% of traffic.

## Adding fault tolerance

The setup described above is able to recover from failures rapidly, due
to all bb\_storage processes being stateful, but non-persistent. When an
instance becomes unavailable, a system like Kubernetes will be able to
quickly spin up a replacement, allowing new builds to take place once
again. Still, there are two problems:

- Builds running at the time of the failure all have to terminate, as
  results from previous build actions may have disappeared.
- Due to the random nature of object digests, the loss of a single shard
  likely means that most cached build actions end up being incomplete,
  due to them depending on output files and logs stored across different
  shards.

To mitigate this, we can introduce a new BlobAccess decorator named
MirroredBlobAccess that supports a basic replication strategy between
pairs of servers, similar to [RAID 1](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_1).
When combined with ShardingBlobAccess, it allows the creation of a
storage stack that roughly resembles [RAID 10](https://en.wikipedia.org/wiki/Nested_RAID_levels#RAID_10_(RAID_1+0)).

- BlobAccess.Get() operations may be tried against both backends. If it
  is detected that only one of the backends possesses a copy of the
  object, it is replicated on the spot.
- BlobAccess.Put() operations are performed against both backends.
- BlobAccess.FindMissing() operations are performed against both
  backends. Any inconsistencies in the results of both backends are
  resolved by replicating objects in both directions. The intersection of
  the results is then returned.

The BlobAccess configuration would look something like this:

```jsonnet
{
  local config = {
    sharding: {
      hashInitialization: 3151213777095999397,  // A random 64-bit number.
      shards: [
        {
          backend: {
            mirrored: {
              backendA: { grpc: { address: 'bb-storage-a0.example.com:12345' } },
              backendB: { grpc: { address: 'bb-storage-b0.example.com:12345' } },
            },
          },
          weight: 1,
        },
        {
          backend: {
            mirrored: {
              backendA: { grpc: { address: 'bb-storage-a1.example.com:12345' } },
              backendB: { grpc: { address: 'bb-storage-b1.example.com:12345' } },
            },
          },
          weight: 1,
        },
        {
          backend: {
            mirrored: {
              backendA: { grpc: { address: 'bb-storage-a2.example.com:12345' } },
              backendB: { grpc: { address: 'bb-storage-b2.example.com:12345' } },
            },
          },
          weight: 1,
        },
      ],
    },
  },
  contentAddressableStorage: config,
  actionCache: config,
}
```

With these semantics in place, it is completely safe to replace one half
of each pair of servers with an empty instance without causing builds to
fail. This also permits rolling upgrades without serious loss of data by
only upgrading half of the fleet during every release cycle. Between
cycles, one half of systems is capable of repopulating the other half.

This strategy may even be used to grow the storage pool without large
amounts of downtime. By changing the order of ShardingBlobAccess and
MirroredBlobAccess, it is possible to temporarily turn this setup into
something that resembles [RAID 01](https://en.wikipedia.org/wiki/Nested_RAID_levels#RAID_01_(RAID_0+1)),
allowing the pool to be resharded in two separate stages.

To implement this efficiently, a minor extension needs to be made to
Buildbarn's buffer layer. To implement MirroredBlobAccess.Put(), buffer
objects need to be cloned. The existing Buffer.Clone() realises this by
creating a full copy of the buffer, so that writing to each of the
backends can take place at its own pace. For large objects copying may
be expensive, which is why Buffer.Clone() should be replaced by one
flavour that copies and one that multiplexes the underlying stream.

# Alternatives considered

Instead of building our own storage solution, we considered switching to
distributed database systems, such as
[CockroachDB](https://www.cockroachlabs.com) and
[FoundationDB](https://www.foundationdb.org). Though they will solve all
consistency problems we're experiencing, they are by no means designed
for use cases that are as bandwidth intensive as ours. These systems are
designed to only process up to about a hundred megabytes of data per
second.  They are also not designed to serve as caches, meaning separate
garbage collector processes need to be run periodically.

Data stores that guarantee eventual consistency can be augmented to
provide the required consistency requirements by placing an index in
front of them that is fully consistent. Experiments where a Redis-based
index was placed in front of S3 proved successful, but were by no means
fault-tolerant.

# Future work

For Buildbarn setups that are not powered on permanently or where
RAM-backed storage is simply too expensive, there may still be a desire
to have disk-backed storage for the CAS and AC. We should extend the new
"local" storage backend to support on-disk storage, just like "circular"
does.

To keep the implementation simple and understandable, MirroredBlobAccess
will initially be written in such a way that it requires both backends
to be available. This is acceptable, because unavailable backends can
easily be replaced by new non-persistent nodes. For persistent setups,
this may not be desired. MirroredBlobAccess would need to be extended to
work properly in degraded environments.

Many copies of large output files with only minor changes to them cause
cache thrashing. This may be prevented by decomposing large output files
into smaller chunks, so that deduplication on the common parts may be
performed. Switching from SHA-256 to a recursive hashing scheme, such as
[VSO-Hash](https://github.com/microsoft/BuildXL/blob/master/Documentation/Specs/PagedHash.md),
makes it possible to implement this while retaining the Merkle tree
property.
