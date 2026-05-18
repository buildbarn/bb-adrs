# Buildbarn Architecture Decision Record #13: Builds without the (large) blobs

Author: Benjamin Ingberg</br>
Date: 2026-05-18

# Context

With the introductino of Split and Splice blobs to the REv2 (Remote
Execution API v2) specification there is an opening to implement the
communication such that large blobs are no longer needed. Without large
blobs there is no need to implement bytestream read and write, no need
to have your storage architecture support arbitrary large files and no
need to support streaming of data in the communication layer.

# Approach

This proposal presents a buildbarn implementation where all "large"
(above a certain size) blobs are internally split into chunks. As a
consequence, the size of all blobs in storage will have a known upper
bound.

If clients communicate using the split and splice blob requests the
communication will be efficient and behave as expected, while older
clients can communicate with the system as if the large blobs were still
there and have the request translated in transit.

To ensure chunk sizes are well distributed with a tight upper and lower
bounds we intend to use
[RepMaxCDC](https://github.com/buildbarn/go-cdc#repmaxcdc) as the
algorithm.

# Definitions
 * blob -  A data object of arbitrary size
 * chunk - A part of a blob
 * chunk list - A list of the constituent chunks that make up a blob
 * $X$ - A stochastic variable representing the size of an individual
   chunk
 * $x_{min}$ - The minimum size of a chunked blob
 * $x_{max}$ - The maximum size of a chunked blob, for RepMaxCDC this is
   $2\cdot x_{min}$

# Benefits

 * Storage layer deduplication of chunks
 * Large blobs can have their lifetime refresh spread out over multiple
   pages
 * Compression at rest can be safely done inside a bounded amount of
   space
 * No more streaming functions, all grpc methods are stateless and
   unary.

# Identification of chunked blobs

Given a REv2 digest we need to identify which chunks that blob consists
of. An REv2 digest consists of two pieces, a hash value of the content
and the size of the content. Given our use of RepMaxCDC, we can identify
if a blob must be chunked by only its size. Any blob smaller than
$x_{max}$ would be a blob of a single chunk with the same digest as the
blob. Any blob larger than $x_{max}$ would require a call to the
SplitBlobs api to identify its underlying chunks.

# Maximum Blob Size

A limitation of the split blobs api is that it returns a single grpc
message. This means that at a protocol level the maximum number of
chunks which a blob can be represented as is as many as can fit into a
single grpc message. Depending on which parameters we pick for our
chunking algorithm this gives us an upper bound for blobs that we can
express using this api.

This size is between $\frac{g}{d}\cdot x_{min}$ and $\frac{g}{d}\cdot
x_{max}$ where $g$ is the maximum grpc message size and $d$ is the size
in bytes of the digest. While a degenerate blob which chunks into only
chunks of size $x_{min}$ is possible the average blob size $E[X]$ when
using RepMaxCDC can be calculated with Rényi's parking constant as
$1/0.7475... \cdot x_{min} = 1.3376... \cdot x_{min}$.

Using constants optimized for performance, such as $x_{min}=256KiB$,
$g=2MiB$ and $d=72B$ (the wire size of a repeated digest based on sha256
less than $512KiB$) we get a maximum blob size of approximately $7GiB$.
Less optimized constants can be used to push this to $100GiB$ but going
further requires changes to the REv2 api.

# Compression

The REv2 protocol requires a server to support IDENTITY compression and
may optimally support ZSTD compression. In our implementation we will
always be compressing blobs at the storage layer with zstd and
decompress it for clients which demand uncompressed data and compress it
for clients which upload uncompressed data. A neat property of using
zstd compression is that two zstd compressed blobs can be trivially
concatenated and the resulting blob is a zstd compressed blob.

# Where is implementation required

By letting our internal usage of the REv2 api be a valid subset of the
api there will be no requirement for us to perform the translation
between blobs and chunks other than at the entrypoint of the cluster.
Internally workers, frontends, the scheduler and bb-portal can directly
communicate using chunked blobs.

## Translation layer

As the cluster receives an api request for any blob that requires
chunking, the chunking information must be stored about the blob.
Additionally, since we are not storing the original blob anywhere, this
chunking data must be accessible when performing the translation from
chunks to blob. We therefore require a key value store implementing the
semantics of getChunkList(digest) and putChunkList(digest). By using the
already existing split and splice blobs api calls as our get and put
implementation any upstream REv2 implementation that supports our
chunking method could be used.

### google.bytestream.Bytestream.Write

This function only has to exist on the frontend server implementation
for compatibility with unchunked clients. This means that frontend
processes must allocate a buffer large enough to fit the look ahead of
the blob and chunk data as it arrives. The Write call must synchronously
store the chunk list by calling SpliceBlob(digest, chunklist) before
finishing the call.

### google.bytestream.Bytestream.Read

Similarly this function also only has to exist in the frontend server
implementation for compatibility with unchunked clients. It must read
the chunk list by calling SplitBlob(digest) and dynamically stitch the
chunks back together.

### build.bazel.remote.execution.v2.ContentAddressableStorage.FindMissingBlobs

Find missing blobs must grab the chunk list of any blob larger than
$x_{max}$ and return those blobs as missing if either the chunk list is
unknown or any of the chunks in the chunk list is missing.

### build.bazel.remote.execution.v2.ContentAddressableStorage.BatchUpdateBlobs

For any blob in BatchUpdateBlobs larger than $x_{max}$ we must
dynamically chunk and synchronously store the chunk list of the blob.

### build.bazel.remote.execution.v2.ContentAddressableStorage.BatchReadBlobs

For any blob BatchReadBlobs larger than $x_{max}$ we must stitch the
blob back together using the chunk list of the blob.

## Lifetime renewal

Care must be taken with regard to lifetime renewal, when receiving a get
or find missing blobs request for an unchunked blob this must be
translated into something that renews the lifetime of the blobs
constituent chunks.

## Client Support

This implementation model will give immediate improvement towards the
overall cluster efficiency without needing any support added to clients.
Clients which speak RepMaxCDC over the Split and Splice blob api will
have additional over the wire improvements. The first client to
implement this will can bb-clientd, which can then be used with older
versions of Bazel to translate the communication upstream.

# Implementation Steps

## Step 1: Follow the protocol

Implement split and splice blob api support in bb-storage in the manner
described by the protocol without any automatic translation. This
implementation is not recommended to be used as it puts dual storage
requirements on the storage layer (both the chunks and the spliced blob
must be available in storage).

## Step 2: Automatic translation

Add logic to bb-storage which wraps all api calls to the storage layer
and expresses them with the primitives of split and splice blob.

## Step 3: Remove support for large blobs

Remove storage and transfer layer support for large blobs outside of the
automatic translation layer. Store all blobs as compressed chunks.

# Rolling out the implementation

The easiest way for a cluster administrator to roll out the
implementation would be to drop the entire cache and repopulate it from
scratch. However, some cluster operators may wish to roll this out in a
non disruptive manner. This section describes how to perform that.

The issue is that any specific blob could be addressed by either the old
schema or the new schema. By using a version where both schemas can be
represented in the storage layer (such as the after step 2) we can then
perform the remapping of the blobs in the frontend processes.

This can be solved by using readFallbackConfigurations and multiple
frontend deployments in an intermediary step.

Set up two frontend deployments, one with the old schema and one with
the new schema, both using the same underlying storage configuration.

```jsonnet
# Old frontend version
{
  shards: {
    "storage-0": ...
  }
}
```

```jsonnet
# New frontend version
{
  shards: {
    "storage-0": ...
  }
}
```

Then modify the **older** frontend version to use the new schema but
fallback to the old schema.

```jsonnet
# Old frontend version
{
    readFallbackConfiguration: {
        primary: newSchemaFrontends,
        # unchanged from before
        fallback: {
            "storage-0": ...
        },
        replicator: ...
    }
}
```

Once this has been running for a sufficient amount of time, delete the
old schema server and use the new auto translating schema as your
primary entrypoint.
