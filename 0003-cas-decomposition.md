# Buildbarn Architecture Decision Record #3: Decomposition of large CAS objects

Author: Ed Schouten<br/>
Date: 2020-04-16

# Context

[The Remote Execution protocol](https://github.com/bazelbuild/remote-apis/blob/master/build/bazel/remote/execution/v2/remote_execution.proto)
that is implemented by Buildbarn defines two data stores: the Action
Cache (AC) and the Content Addressable Storage (CAS). The CAS is where
almost all data is stored in terms of size (99%+), as it is used to
store input files and output files of build actions, but also various
kinds of serialized Protobuf messages that define the structure of build
actions.

As the name implies, the CAS is [content addressed](https://en.wikipedia.org/wiki/Content-addressable_storage),
meaning that all objects are identified by their content, in this case a
cryptographic checksum (typically [SHA-256](https://en.wikipedia.org/wiki/SHA-2)).
This permits Buildbarn to automatically deduplicate identical objects
and discard objects that are corrupted.

Whereas most objects stored in the AC are all similar in size
(kilobytes), sizes of objects stored in the CAS may vary a lot. Small
[Directory](https://github.com/bazelbuild/remote-apis/blob/b5123b1bb2853393c7b9aa43236db924d7e32d61/build/bazel/remote/execution/v2/remote_execution.proto#L673-L685)
objects are hundreds of bytes in size, while container layers created
with [rules\_docker](https://github.com/bazelbuild/rules_docker) may
consume a gigabyte of space. This is observed to be problematic for
Buildbarn setups that use sharded storage. Each shard will receive a
comparable number of requests, but the amount of network traffic and CPU
load to which those requests translate, fluctuates.

Buildbarn has code in place to checksum validate data coming from
untrusted sources. For example, bb\_worker validates all data received
from bb\_storage. bb\_storage validates all data obtained from clients
and data read from disk. This is done for security reasons, but also to
ensure that bugs in storage code don't lead to execution of malformed
build actions. Unfortunately, there are an increasing number of use
cases where partial reads need to be performed (lazy-loading file
systems for bb\_worker, Bazel's ability to resume aborted transfers,
etc.). Such partial reads currently still cause entire objects to be
read, simply to satisfy the checksum validation process. This is
wasteful.

To be able to address these issues, this ADR proposes the addition to
some new features to Buildbarn's codebase to be able to eliminate the
existence of such large CAS objects altogether.

# Methods for decomposing large objects

In a nutshell, the idea is to partition files whose size exceed a
certain threshold into smaller blocks. Each of the blocks will have a
fixed size, except the last block in a file, which may be smaller. The
question then becomes what naming scheme is used to store them in the
CAS.

The simplest naming scheme would be to give all blocks a suffix that
indicates the offset in the file. For example, a file that is decomposed
into three blocks could use digests containing hashes `<HASH>-0`,
`<HASH>-1` and `<HASH>-2`, where `<HASH>` refers to the original file
hash. Calls to `BlobAccess.Get()`, `Put()` and `FindMissing()` could be
implemented to expand to a series of digests containing the block number
suffix.

Unfortunately, this approach is too simplistic. Because `<HASH>` refers
to the checksum of the file as a whole, there is no way the consistency
of individual blocks can be validated. If blocks are partitioned across
shards, there is no way an individual shard can verify the integrity of
its own data.

A solution would be to store each of the blocks using their native
digest, meaning a hash of each of them is computed. This on its own,
however, would only allow storing data. There would be no way to look up
blocks afterwards, as only the combined hash is known. To be able to
retrieve the data, we'd to store a separate manifest that contains a
list of the digests of all blocks in the file.

Even though such an approach would allow us to validate the integrity of
individual blocks of data, conventional hashing algorithms wouldn't
allow us to verify manifests themselves. There is no way SHA-256 hashes
of individual blocks may be combined into a SHA-256 hash of the file as
a whole. SHA-256's [Merkle-Damgård construction](https://en.wikipedia.org/wiki/Merkle–Damgård_construction)
does not allow it.

# VSO-Hash: Paged File Hash

As part of their [BuildXL infrastructure](https://github.com/microsoft/BuildXL),
Microsoft published a hashing algorithm called ["VSO-Hash"](https://github.com/microsoft/BuildXL/blob/master/Documentation/Specs/PagedHash.md)
or "Paged File Hash". VSO-Hash works by first computing SHA-256 hashes
for every 64 KiB page of data. Then, 32 page hashes (for 2 MiB of data)
are taken together and hashed using SHA-256 to form a block hash.
Finally, all block hashes are combined by [folding](https://en.wikipedia.org/wiki/Fold_(higher-order_function))
them.

By letting clients use VSO-Hash as a digest function, Buildbarn could be
altered to decompose blobs either in 64 KiB or 2 MiB blocks, or both
(i.e., decomposing into 64 KiB chunks, but having two levels of
manifests). This would turn large files into [Merkle trees](https://en.wikipedia.org/wiki/Merkle_tree)
of depth two or three. Pages, blocks and their manifests could all be
stored in the CAS and validated trivially.

Some work has already been done in the Bazel ecosystem to
facilitate this. For example, the Remote Execution protocol already
contains an enumeration value for VSO-Hash, which was
[added by Microsoft in 2019](https://github.com/bazelbuild/remote-apis/pull/84).
There are, however, various reasons not to use VSO-Hash:

- Microsoft seemingly only added support for VSO-Hash as a migration
  aid. There are efforts to migrate BuildXL from VSO-Hash to plain
  SHA-256. The author has mentioned that support for VSO-Hash will likely
  be retracted from the protocol once this migration is complete.
- Bazel itself doesn't implement it.
- It is somewhat inefficient. Because the full SHA-256 algorithm is used
  at every level, the Merkle-Damgård construction is used excessively.
  The algorithm could have been simpler by defining its scheme directly
  on top of the SHA-256 compression function.
- Algorithms for validating files as a whole, (manifests of) pages and
  (manifests of) blocks are all different. Each of these needs a
  separate implementation, as well as a distinct digest format, so that
  storage nodes know which validation algorithm to use upon access.
- It effectively leaks the decomposition strategy into the client.
  VSO-Hash only allows decomposition at the 64 KiB and 2 MiB level.
  Decomposition at any other power of two would require the use of an
  entirely different hashing algorithm.

# BLAKE3

At around the time this document was written, a new hashing algorithm
was published, called [BLAKE3](https://github.com/BLAKE3-team/BLAKE3/).
As the name implies, BLAKE3 is a successor of BLAKE, an algorithm that
took part in the [SHA-3 competition](https://en.wikipedia.org/wiki/NIST_hash_function_competition).
In the end BLAKE lost to Keccak, not because it was in any way insecure,
but because it was too similar to SHA-2. The goal of the SHA-3
competition was to have a backup in case fundamental security issues in
SHA-2 are found.

An interesting improvement of BLAKE3 over its predecessor BLAKE2 is
that it no longer uses the Merkle-Damgård construction. Instead,
chaining is only used for chunks up to 1 KiB in size. When hashing
objects that are larger than 1 KiB, a binary Merkle tree of 1 KiB chunks
is constructed. The final hash value is derived from the Merkle tree's
root node. If clients were to use BLAKE3, Buildbarn would thus be able
to decompose files into blocks that are any power of two, at least 1 KiB
in size.

BLAKE3 uses an Extendable-Output Function (XOF) to post-process the
state of the hasher into an output sequence of arbitrary length. Because
this process invokes the BLAKE3 compression function, it is not
reversible. This means that manifests cannot contain hashes of its
blocks, as that would not allow independent integrity checking of
the manifest. Instead, manifest entries should hold the state of the
hasher (i.e., the raw Merkle tree node). That way it's possible both to
validate and convert to a block digest.

For chunk nodes (that represent 1 KiB of data or less), the hasher state
can be stored in 97 bytes (256 bit initialization vector, 512 bit message,
7 bit size, 1 bit chunk-start flag). For parent nodes (that
represent more than 1 KiB of data), the hasher state is only 64 bytes
(512 bit message), as other parameters of the compression function are
implied. Because of that, decomposition of files into 1 KiB blocks
should be discouraged. Buildbarn should only support decomposition into
blocks that are at least 2 KiB in size. That way manifests only contain
64 byte entries, except for the last entry, which may be 64 or 97 bytes
in size.

# BLAKE3ZCC: BLAKE3 with a Zero Chunk Counter

The BLAKE3 compression function takes a `t` argument that is for two
different purposes. First of all, it is used as a counter for every 512
bits of hash output generated by the XOF. Secondly, it contains the
Chunk Counter when compressing input data. Effectively, the Chunk
Counter causes every 1 KiB chunk of data to be hashed in a different way
depending on the offset at which it is stored within the file.

For our specific use case, it is not desirable that hashing of data is
offset dependent. It would require that decomposed blocks contain
additional metadata that specify at which offset the data was stored in
the original file. Otherwise, there would be no way to validate the
integrity of the block independently. It also rules out the possibility
of deduplicating large sections of repetitive data (e.g., deduplicating
a large file that contains only null bytes to just a single chunk).

According to section 7.5 of the BLAKE3 specification, the Chunk Counter
is not strictly necessary for security, but discourages optimizations
that would introduce timing attacks. Though timing attacks are a serious
problem, we can argue that in the case of the Remote Execution protocol
such timing attacks already exist. For example, identical files stored
in a single Directory hierarchy will only be uploaded to/downloaded from
the CAS once.

For this reason, this ADR proposes adding support for a custom hashing
algorithm to Buildbarn, which we will call BLAKE3ZCC. BLAKE3ZCC is
identical to regular BLAKE3, except that it uses a Zero Chunk Counter.
BLAKE3ZCC generates exactly the same hash output as BLAKE3 for files
that are 1 KiB or less in size, as those files fit in a single chunk.

# Changes to the Remote Execution protocol

All of the existing hashing algorithms supported by the Remote Execution
protocol have a different hash size. Buildbarn makes use of this
assumption, meaning that if it encounters a [Digest](https://github.com/bazelbuild/remote-apis/blob/b5123b1bb2853393c7b9aa43236db924d7e32d61/build/bazel/remote/execution/v2/remote_execution.proto#L779-L786)
message whose `hash` value is 64 characters in size, it assumes it
refers to an object that is SHA-256 hashed. Adding support for BLAKE3ZCC
would break this assumption. BLAKE3 hashes may have any size, which
makes matching by length impossible. The default hash length of 256 bits
would also collide with SHA-256.

To solve this, we could extend Digest to make it explicit which hashing
algorithm is used. The existing `string hash = 1` field could be
replaced with the following:

```protobuf
message Digest {
  oneof hash {
    // Used by all of the existing hashing algorithms.
    string other = 1;
    // Used to address BLAKE3ZCC hashed files, or individual blocks in
    // case the file has been decomposed.
    bytes blake3zcc = 3;
    // Used to address manifests of BLAKE3ZCC hashed files, containing
    // Merkle tree nodes of BLAKE3ZCC hashed blocks. Only to be used by
    // Buildbarn, as Bazel will only request files as a whole.
    bytes blake3zcc_manifest = 4;
  }
  ...
}
```

By using type `bytes` for these new fields instead of storing a base16
encoded hash in a `string`, we cut down the size of Digest objects by
almost 50%. This causes a significant space reduction for Action,
Directory and Tree objects.

Unfortunately, `oneof` cannot be used here, because Protobuf
implementations such as [go-protobuf](https://github.com/golang/protobuf/issues/395)
don't guarantee that fields are serialized in tag order when `oneof`
fields are used. This property is required by the Remote Execution
protocol. To work around this, we simply declare three separate fields,
where implementations should ensure only one field is set.

```protobuf
message Digest {
  string hash_other = 1;
  bytes hash_blake3zcc = 3;
  bytes hash_blake3zcc_manifest = 4;
  ...
}
```

Digests are also encoded into pathname strings used by the ByteStream
API. To distinguish BLAKE3ZCC files and manifests from other hashing
algorithms, `B3Z:` and `B3ZM:` prefixes are added to the base16 encoded
hash values, respectively.

In addition to that, a `BLAKE3ZCC` constant is added to the
DigestFunction enum, so that Buildbarn can announce support for
BLAKE3ZCC to clients.

# Changes to Buildbarn

Even though BLAKE3 only came out recently,
[one library for computing BLAKE3 hashes in Go exists](https://github.com/lukechampine/blake3).
Though this library is of good quality, it cannot be used to compute
BLAKE3ZCC hashes without making local code changes. Furthermore, this
library only provides public interfaces for converting a byte stream to
a hash value. In our case we need separate interfaces for converting
chunks to Merkle tree nodes, computing the root node for a larger Merkle
tree and obtaining Merkle tree nodes to hash values. Without such
features, we'd be unable to generate and parse manifests. We will
therefore design our own BLAKE3ZCC hashing library, which we will
package at `github.com/buildbarn/bb-storage/pkg/digest/blake3zcc`.

Buildbarn has an internal `util.Digest` data type that extends upon the
Remote Execution Digest message by storing the instance name, thereby
making it a fully qualified identifier of an object. It also has many
operations that allow computing, deriving and transforming them. Because
supporting BLAKE3ZCC and decomposition makes this type more complex, it
should first be moved out of `pkg/util` into its own `pkg/digest`
package.

In addition to being extended to support BLAKE3ZCC hashing, the Digest
data type will gain a new method:

```go
func (d Digest) ToManifest(blockSizeBytes int64) (manifestDigest Digest, parser ManifestParser, ok bool) {}
```

When called on a digest object that uses BLAKE3ZCC that may be
decomposed into multiple blocks, this function returns the digest of its
manifest. In addition to that, it returns a ManifestParser object that
may be used to extract block digests from manifest payloads or construct
them. ManifestParser will look like this:

```go
type ManifestParser interface {
	// Perform lookups of blocks on existing manifests.
	GetBlockDigest(manifest []byte, off int64) (blockDigest Digest, actualOffset int64)
	// Construct new manifests.
	AppendBlockDigest(manifest *[]byte, block []byte) Digest
}
```

One implementation of ManifestParser for BLAKE3ZCC shall be provided.

The `Digest.ToManifest()` function and its resulting ManifestParser
shall be used by a new BlobAccess decorator, called
DecomposingBlobAccess. The operations of this decorator shall be
implemented as follows:

- `BlobAccess.Get()` will simply forward the call if the provided digest
  does not correspond to a blob that can be decomposed. Otherwise, it
  will load the associated manifest from storage. It will then return a
  Buffer object that dynamically loads individual blocks from the CAS
  when accessed.
- `BlobAccess.Put()` will simply forward the call if the provided digest
  does not correspond to a blob that can be decomposed. Otherwise, it
  will decompose the input buffer into smaller buffers for every block
  and write those into the CAS. In addition to that, it will write a
  manifest object into the CAS.
- `BlobAccess.FindMissing()` will forward the call, except that digests
  of composed objects will be translated to the digests of their
  manifests. Manifests that are present will then be loaded from the
  CAS, followed by checking the existence of each of the individual
  blocks.

To be able to implement DecomposingBlobAccess, the Buffer layer will
need two minor extensions:

- `BlobAccess.Get()` needs to return a buffer that needs to be backed by
  the concatenation of a sequence of block buffers. A new buffer type
  that implements this functionality shall be added, which can be
  created by calling `NewCASConcatenatingBuffer()`.
- `BlobAccess.Put()` needs to read the input buffer in chunks equal to
  the block size. The `Buffer.ToChunkReader()` function currently takes
  a maximum chunk size argument, but provides no facilities for
  specifying a lower bound. This mechanism should be extended, so that
  `ToChunkReader()` can be used to read input one block at a time.

With all of these changes in place, Buildbarn will have basic support
for decomposing large objects in place.

# Future work

With the features proposed above, support for decomposition should be
complete enough to provide better spreading of load for sharded setups.
In addition to that, workers with lazy-loading file systems will be able
to perform I/O on large files without faulting them in entirely. There
are still two areas where performance may be improved further:

- Using BatchedStoreBlobAccess, workers currently call `FindMissing()`
  before uploading output files. This is done at the output file level,
  which is suboptimal. When decomposition is enabled, it causes workers
  to load manifests from storage, even though those could be derived
  locally. A single absent block will cause the entire output file to be
  re-uploaded. To prevent this, `BlobAccess.Put()` should likely be
  decomposed into two flavours: one for streaming uploads and one for
  uploads of local files.
- When decomposition is enabled, `FindMissing()` will likely become
  slower, due to it requiring more roundtrips to storage. This may be
  solved by adding yet another BlobAccess decorator that does short-term
  caching of `FindMissing()` results.
