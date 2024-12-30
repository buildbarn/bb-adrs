# Buildbarn Architecture Decision Record #11: Rendezvous Hashing

Author: Benjamin Ingberg<br/>
Date: 2025-04-08

# Context

Resharding a Buildbarn cluster, that is changing the number, order or weight of
shards is a very disruptive process today. It essentially shuffles around all
the blobs to new shards forcing you to either drop all old state. Or, have a
period with a read fallback configuration where you can fetch the old state.

Buildbarn supports using drained backends for sharding and even adding an unused
placeholder backend at the end of it's sharding array to mitigate situation. It
is however a rare operation to deal with which means that, without a significant
amount of automation, there is a significant risk it's performed incorrectly.

This ADR describes how we can make that more accessible by changing the
underlying algorithm to one that is stable between resharding.

# Issues During Resharding

You might not have the ability to spin up a secondary set of storage shards to
perform a read fallback configuration over. This is a common situation in an
on-prem environment, where running two copies of your production environment may
not be feasible.

This is not necessarily a  blocker. You can reuse shards from the old topology
in your new topology. However, this has a risk of significantly reducing your
retention time since data must be stored according to the addressing schema of
both the new and the old topology simultaneously.

While it is possible to reduce the amount of address space that is resharded
with drained backends this requires advance planning.

# Improvements

In this ADR we attempt to improve the resharding experience by minimizing the
difference between different sharding topologies.

## Better Overlap Between Sharding Topologies

Currently, two different sharding topologies, even if they share nodes, will
have a small overlap between addressing schemas. This can be significantly
improved by using a different sharding algorithm.

For this purpose we replace the implementation of
`ShardingBlobAccessConfiguration` with one that uses [Rendezvous
hashing](https://en.wikipedia.org/wiki/Rendezvous_hashing). Rendezvous hashing
is a lightweight and stateless technique for distributed hash tables. It has a
low overhead with minimal disruption during resharding.

Sharding with Rendezvous hashing gives us the following properties:
 * Removing a shard is _guaranteed_ to only require resharding for the blobs
   that resolved to the removed shard.
 * Adding a shard will reshard any blob to the new shard with a probability of
   `weight/total_weight`.

This effectively means adding or removing a shard triggers a predictable,
minimal amount of resharding, eliminating the need for drained backends.

```proto
message ShardingBlobAccessConfiguration {
  message Shard {
    // unchanged
    BlobAccessConfiguration backend = 1;
    // unchanged
    uint32 weight = 2;
  }
  // NEW:
  // Shards identified by a key within the context of this sharding
  // configuration. The key is a freeform string which describes the identity
  // of the shard in the context of the current sharding configuration.
  // Shards are chosen via Rendezvous hashing based on the digest, weight and
  // key of the configuration.
  //
  // When removing a shard from the map it is guaranteed that only blobs
  // which resolved to the removed shard will get a different shard. When
  // adding shards there is a weight/total_weight probability that any given
  // blob will be resolved to the new shards.
  map<string, Shard> shards = 2;
  // NEW:
  message Legacy {
    // Order of the shards for the legacy schema. Each key here refers to
    // a corresponding key in the 'shard_map' or null for drained backends.
    repeated string shard_order = 1;
    // Hash initialization seed used for legacy schema.
    uint64 hash_initialization = 2;
  }
  // NEW:
  // A temporary legacy mode which allows clients to use storage backends which
  // are sharded with the old sharding topology implementation. Consumers are
  // expected to migrate in a timely fashion and support for the legacy schema
  // will be removed by 2025-12-31.
  Legacy legacy = 3;
}
```

## Migration Instructions

If you are fine with dropping your data and repopulating your cache you can
start using the new sharding algorithm right away. For clarity we have an
example on how to perform a non-destructive migration.

### Configuration before Migration

Given the following configuration showing an unmirrored sharding topology of two
shards:

```jsonnet
{
  ...
  contentAddressableStorage: {
    sharding: {
      hashInitialization: 11946695773637837490,
      shards: [
        {
          backend: {
            grpc: {
              address: 'storage-0:8980',
            },
          },
          weight: 1,
        },
        {
          backend: {
            grpc: {
              address: 'storage-1:8980',
            },
          },
          weight: 1,
        },
      ],
    },
  },
  ...
}
```

### Equivalent Configuration in New Format

We first want to non-destructively represent the exact same topology with the
new configuration. The legacy parameter used here will make Buildbarn internally
use the old sharding algorithm, the key can be any arbitrary string value.

```jsonnet
{
  ...
  contentAddressableStorage: {
    sharding: {
      shards: {
        "0": {
          backend: { grpc: { address: 'storage-0:8980' } },
          weight: 1,
        },
        "1": {
          backend: { grpc: { address: 'storage-1:8980' } },
          weight: 1,
        },
      },
      legacy: {
        shardOrder: ["0", "1"],
        hashInitialization: 11946695773637837490,
      },
    },
  },
  ...
}
```

### Intermediate State with Fallback (Optional)

If you want to preserve cache content during migration you can add this
intermediate step that uses the new adressing schema as it's primary
configuration but falls back to the old adressing schema for missing blobs.

If you are willing to drop the cache this step can be skipped.

```jsonnet
{
  ...
  contentAddressableStorage: {
    readFallback: {
      primary: {
        sharding: {
          shards: {
            "0": {
              backend: { grpc: { address: 'storage-0:8980' } },
              weight: 1,
            },
            "1": {
              backend: { grpc: { address: 'storage-1:8980' } },
              weight: 1,
            },
          },
        },
      },
      secondary: {
        sharding: {
          shards: {
            "0": {
              backend: { grpc: { address: 'storage-0:8980' } },
              weight: 1,
            },
            "1": {
              backend: { grpc: { address: 'storage-1:8980' } },
              weight: 1,
            },
          },
          legacy: {
            shardOrder: ["0", "1"],
            hashInitialization: 11946695773637837490,
          },
        },
      },
      replicator: { ... }
    },
  },
  ...
}
```

### Final Configuration

When the cluster has used the read fallback configuration for an acceptable
amount of time you can drop the fallback and have only the new configuration.

```jsonnet
{
  ...
  contentAddressableStorage: {
    sharding: {
      shards: {
        "0": {
          backend: { grpc: { address: 'storage-0:8980' } },
          weight: 1,
        },
        "1": {
          backend: { grpc: { address: 'storage-1:8980' } },
          weight: 1,
        },
      },
    },
  },
  ...
}
```

## Implementation

Rendezvous hashing was chosen for it's simplicity of implementation,
mathematical elegance and not needing to be parameterized in configuration.

> Of all shards pick the one with the largest $-\frac{weight}{log(h)}$ where `h` is a
hash of the key/node pair mapped to (0, 1).

The implementation uses an optimized fixed point integer calculation. Checking a
server takes approximately 5ns on a Ryzen 7 7840U. Running integration
benchmarks on large `FindMissingBlobs` calls shows us that the performance is
within variance until we reach the 10k+ shards mark.

```
goos: linux
goarch: amd64
cpu: AMD Ryzen 7 7840U w/ Radeon  780M Graphics     
BenchmarkSharding10-16             17619            677008 ns/op
BenchmarkSharding100-16             6465           1636342 ns/op
BenchmarkSharding1000-16            1488           8842389 ns/op
BenchmarkSharding10000-16            783          18426834 ns/op
BenchmarkLegacy10-16               16959            704753 ns/op
BenchmarkLegacy100-16               6562           1574817 ns/op
BenchmarkLegacy1000-16              1558           8045423 ns/op
BenchmarkLegacy10000-16              958          12146523 ns/op
```

## Other Algorithms

There are some minor advantages and disadvantages to the other algorithms, in
particular with regards to how computationally heavy they are. However, as noted
above, the performance characteristics are not relevant for clusters below 10k
shards.

Other algorithms considered were: [Consistent
hashing](https://en.wikipedia.org/wiki/Consistent_hashing) and
[Maglev](https://storage.googleapis.com/gweb-research2023-media/pubtools/2904.pdf).

### Consistent Hashing

Consistent Hashing can be reduced into a special case of Rendezvous with the
advantage that it can precomputed into a sorted list. There we can binary search
for the target in `log(W)` time. This is in contrast to to Rendezvous hashing
which takes `N` time to search.

* `N` is the number of shards
* `W` is the sum of weights in its reduced form (i.e. smallest weight is 1 and
  all weights are integer)

There are some concrete disadvantages to Consistent Hashing. A major one is that
it is only unbiased on average, any given sharding configuration with Consistent
Hashing will have a bias towards one of the shards.

The performance advantages of Consistent Hashing are also unlikely to
materialize in practice. Straight linear search with Rendezvous is often faster
than binary searching for reasonable N. Rendezvous is also independent on the
weight of the individual shards which improves the relative performance of
Rendezvous for shards that are not evenly weighted.

### Maglev

Maglev in turn uses an iterative method that precomputes an evenly distributed
lookup table of any arbitrary size. After the precomputational step the target
shard is a simple lookup of the key modulo the size of the table.

The disadvantage of Maglev is that it aims for minimal resharding rather than
optimal resharding. When removing a shard from the set of all shard, the
algorithm tolerates a small amount of unecessary resharding. This is the order
of a few percentiles depending on the number of shards and size of the lookup
table.

Precomputing the lookup table is also a fairly expensive operation which impacts
startup time of components or requires managing the table state externally.
