# Buildbarn Architecture Decision Record #11: Resharding Without Downtime

Author: Benjamin Ingberg<br/>
Date: 2024-12-30

# Context

Resharding a Buildbarn cluster without downtime is today a multi step process
that can be done as the following steps:

1. Deploy new storage shards.
2. Deploy a new topology with a read fallback configuration. Configure the old
   topology as the primary and the new topology as the secondary.
3. When the topology from step 2 has propagated, swap the primary and secondary.
4. When your new shards have performed sufficient amount of replication, deploy
   a topology without fallback configuration.
5. Once the topology from step 4 has propagated, tear down any unused shards

This process enables live resharding of a cluster. It works because each step is
backwards compatible with the previous step. That is, accessing the blobstore
with the topology from step N-1 will resolve correctly even if some components
are already using the topology from step N.

The exact timing and method needed to perform these steps depend on how you
orchestrate your buildbarn cluster and your retention aims. This process might
span an hour to several weeks.

Resharding a cluster is a rare operation, so having multiple steps to achieve it
is not inherently problematic. However, without a significant amount of
automation of the cluster's meta-state there are large risks for performing it
incorrectly.

# Issues During Resharding

## Non-Availability of a Secondary Set of Shards

You might not have the ability to spin up a secondary set of storage shards to
perform the switchover. This is a common situation in an on-prem environment,
where running two copies of your production environment may not be feasible.

This is not necessarily a  blocker. You can reuse shards from the old topology
in your new topology. However, this has a risk of significantly reducing your
retention time since data must be stored according to the addressing schema of
both the new and the old topology simultaneously.

While it is possible to reduce the amount of address space that is resharded with
drained backends this requires advance planning.

## Topology changes requires restarts

Currently, the only way to modify the topology visible to an individual
Buildbarn component is to restart that component. While mounting a Kubernetes
ConfigMap as a volume allows it to reload on changes, Buildbarn programs lack
logic for dynamically reloading their blob access configuration.

A properly configured cluster can still perform rolling updates. However, there
is a trade-off between the speed of rollout and the potential loss of ongoing
work. Since clients automatically retry when encountering errors, losing ongoing
work may be the preferred issue to address. Nonetheless, for clusters with very
expensive long-running actions, this could result in significant work loss.

The amount of restarted components can be reduced by routing traffic via
internal facing frontends which are topology aware.

# Improvements

## Better Overlap Between Sharding Topologies

Currently, two different sharding topologies, even if they share nodes, will
have a small overlap between addressing schemas. This can be significantly
improved by using a different sharding algorithm.

For this purpose we replace the implementation of
`ShardingBlobAccessConfiguration` with one that uses [Rendezvous
hashing](https://en.wikipedia.org/wiki/Rendezvous_hashing). Rendezvous hashing
is a lightweight and stateless technique for distributed hash tables. It has a low
overhead with minimal disruption during resharding.

Sharding with Rendezvous hashing gives us the following properties:
 * Removing a shard is _guaranteed_ to only require resharding for the blobs
   that resolved to the removed shard.
 * Adding a shard will reshard any blob to the new shard with a probability of
   `weight/total_weight`.

This effectively means adding or removing a shard triggers a predictable,
minimal amount of resharding, eliminating the need for drained backends.

```
message ShardingBlobAccessConfiguration {
    message Shard {
        // unchanged
        BlobAccessConfiguration backend = 1;
        // unchanged
        uint32 weight = 2;
    }
    // unchanged
    uint64 hash_initialization = 1;

    // Was 'shards' an array of shards to use, has been replaced with
    // 'shard_map'
    reserved 2;

    // NEW:
    // Shards identified by a key within the context of this sharding
    // configuration. The key is a freeform string which describes the identity
    // of the shard in the context of the current sharding configuration.
    // Shards are chosen via Rendezvous hashing based on the digest, weight,
    // key and hash_initialization of the configuration.
    //
    // When removing a shard from the map it is guaranteed that only blobs
    // which resolved to the removed shard will get a different shard. When
    // adding shards there is a weight/total_weight probability that any given
    // blob will be resolved to the new shards.
    map<string, Shard> shard_map = 3;
}
```

### Other Algorithms

Other algorithms considered were: [Consistent
hashing](https://en.wikipedia.org/wiki/Consistent_hashing) and
[Maglev](https://storage.googleapis.com/gweb-research2023-media/pubtools/2904.pdf).

Rendezvous hashing was chosen for it's simpleness of implementation,
mathematical elegance and not needing to be parameterized in configuration.

_Of all shards pick the one with the largest `-weight / log(h)` where h is a
hash of the key/node pair mapped to `]0, 1[`._

There are some minor advantages and disadvantages to the other algorithms, in
particular with regards to how computationally heavy they are. However, in a
typical buildbarn cluster with dozens of shards, the amount of computation
required to pick a shard is unlikely to be relevant.

##### Consistent Hashing

Consistent hashing can be reduced into a special case of Rendezvous with the
advantage that it can precomputed into a sorted list. There we can binary search
for the target in `log(W)` time. This is in contrast to to Rendezvous hashing
which takes `N` time to search.

* `N` is the number of shards
* `W` is the sum of weights in its reduced form (i.e. smallest weight is 1 and
  all weights are integer)

In practice buildbarn clusters only have a small number of shards and doing a
straight linear search with Rendezvous is faster for any reasonable cluster (say
less than 100 shards). Rendezvous is also independent on the weight of the
individual shards which improves the relative performance of Rendezvous for
shards that are not evenly weighted.

#### Maglev

Maglev in turn uses an iterative method that precomputes an evenly distributed
lookup table of any arbitrary size. After the precomputational step the target
shard is a simple lookup of the key modulo the size of the table.

The disadvantage of Maglev is that it aims for minimal resharding rather than
optimal resharding. When removing a shard from the set of all shard, the
algorithm tolerates a small amount of unecessary resharding. This is the order
of a few percentiles depending on the number of shards and size of the lookup
table.

## Handling Topology Changes Dynamically

To reduce the amount of restarts required buildbarn needs a construct that
describes how to reload the topology.

### RemotelyDefinedBlobAccessConfiguration

```
message RemotelyDefinedBlobAccessConfiguration {
    // Fetch the blob access configuration from an external service
    buildbarn.configuration.grpc.ClientConfiguration endpoint = 1;

    // Duration to reuse the previous configuration when not able to reach
    // RemoteBlobAccessConfiguration.GetBlobAccessConfiguration
    // before the component should consider itself partitioned from the cluster
    // and return UNAVAILABLE for any access requests.
    //
    // Recommended value: 10s
    google.protobuf.Duration remote_configuration_timeout = 2;
}
```

This configuration calls service that implements the following:

```
service RemoteBlobAccessConfiguration {
    // A request to subscribe to the blob access configurations.
    // The service should immediately return the current blob access
    // configuration. After the initial blob access configuration the stream
    // will push out any changes to the blob access configuration.
    rpc GetBlobAccessConfiguration(GetBlobAccessConfigurationRequest) returns (stream BlobAccessConfiguration);
}

message GetBlobAccessConfigurationRequest {
    enum StorageBackend {
        CAS = 0;
        AC = 1;
        ICAS = 2;
        ISCC = 3;
        FSAC = 4;
    }
    // Which storage backend that the service should describe the topology for.
    StorageBackend storage_backend = 1;
}
```

A simple implementation of this service could be a sidecar container that
dynamically reads a configmap.

A more complex implementation might:
 * Read prometheus metrics and roll out updates to the correct mirror.
 * Increase the number of shards when the metrics indicate that the retention
   time has fallen below a desired value.
 * Stagger read fallback configurations which are removed automatically after
   sufficient amount of time has passed.

### JsonnetDefinedBlobAccessConfiguration

We refer to a secondary configuration file that will be dynamically loaded at
runtime. This configuration may be modified in which case Buildbarn will reload
it.

```
message JsonnetDefinedBlobAccessConfiguration {
    // Path to the jsonnet file which describes the blob access configuration
    // to use. This will be monitored for changes.
    string path = 1;

    // External variables to use when parsing the jsonnet file. These are used
    // in addition to the process environment variables which are always used.
    map<string, string> external_variables = 2;

    // Buildbarn will use the inotify api to watch for any changes to the
    // underlying jsonnet file any libsonnet file it includes and any optional
    // lock file specified. Should the inotify api not be able to notice file
    // modifications Buildbarn can be configured to periodically poll for
    // changes to the configuration files.
    //
    // This typically happens because the underlying file system does not
    // support the inotify api and/or because the file is hidden by bind
    // mounts.
    //
    // Setting this value to 0 will disable the periodic refresh interval.
    google.protobuf.Duration refresh_interval = 3;

    // An optional path to a lockfile which buildbarn will check the presense
    // for before before parsing the jsonnet file. Buildbarn will only parse
    // the jsonnet file if the lockfile does not exist.
    //
    // The lockfile is there to prevent unintentional partial updates to the
    // BlobAccessConfiguration. Such as when having a jsonnet file which
    // imports a libsonnet file where each file gets updated separately.
    string lock_file_path = 4;
}
```
