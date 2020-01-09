# Buildbarn Architecture Decision Record #1: Buffer layer

Author: Ed Schouten<br/>
Date: 2020-01-09

# Context

The `BlobAccess` interface that Buildbarn currently uses to abstract
away different kinds of backing stores for the CAS and AC (Redis, S3,
gRPC, etc.) is a bit simplistic, in that contents are always transferred
through `io.ReadCloser` handles. This is causing a couple of problems:

- There is an unnecessary amount of copying of data in memory. In likely
  the worst case (`bb_storage` with a gRPC AC storage backend), data may
  be converted from ActionResult Protobuf message → byte slice →
  `io.ReadCloser` → byte slice → ActionResult Protobuf message.

- It makes implementing a replicating `BlobAccess` harder, as
  `io.ReadCloser`s may not be duplicated. A replicating `BlobAccess`
  could manually copy blobs into a byte slice itself, but this would
  only contribute to more unnecessary copying of data.

- Tracking I/O completion and hooking into errors is not done
  accurately and consistently. `MetricsBlobAccess` currently only counts
  the amount of time spent in `Get()`, which won't always include the
  time actually spent transferring data. `ReadCachingBlobAccess` is
  capable of responding to errors returned by `Get()`, but not ones that
  occur during the actual transfer. A generic mechanism for hooking into
  I/O errors and completion is absent.

- To implement an efficient FUSE file system for `bb_worker`, we need to
  support efficient random access I/O, as userspace applications are
  capable of accessing files at random. This is not supported by
  `BlobAccess`. An alternative would be to not let a FUSE file system
  use `BlobAccess` directly, but this would lead to reduced operational
  flexibility.

- Data integrity checking (checksumming) is currently done through
  `MerkleBlobAccess`. To prevent foot-shooting, this `BlobAccess`
  decorator is injected into the configuration automatically. The
  problem is that `MerkleBlobAccess` is currently only inserted at one
  specific level of the system. When Buildbarn is configured to use a
  Redis remote data store in combination with local on-disk caching,
  there is no way to enable checksum validation for both Redis → local
  cache and local cache → consumption.

- Stretch goal: There is no framework in place to easily implement
  decomposition of larger blobs into smaller ones (e.g., using
  [VSO hashing](https://github.com/microsoft/BuildXL/blob/master/Documentation/Specs/PagedHash.md)).
  Supporting this would allow us to use data stores optimized for small
  blobs exclusively (e.g., Redis).

# Decision

The decision is to add a new abstraction to Buildbarn, called the buffer
layer, stored in Go package
`github.com/buildbarn/bb-storage/pkg/blobstore/buffer`. Below is a
massively simplified version of what the API will look like:

```go
func NewBufferFromActionResult(*remoteexecution.ActionResult) Buffer {}
func NewBufferFromByteSlice([]byte) Buffer                           {}
func NewBufferFromReader(io.ReadCloser) Buffer                       {}

type Buffer interface {
	ToActionResult() *remoteexecution.ActionResult
	ToByteSlice() []byte
	ToReader() io.ReadCloser
}
```

It can be thought of as a union/variant type that automatically does
conversions from one format to another, but only when strictly
necessary. Calling one of the `To*()` functions extracts the data from
the buffer, thereby destroying it.

To facilitate support for replication, the `Buffer` interface may
contain a `Clone()` function. For buffers created from a byte slice,
this function may be a no-op, causing the underlying slice to be shared.
For buffers created from other kinds of sources, the implementation may
be more complex (e.g., converting it to a byte slice on the spot).

To track I/O completion and to support retrying, every buffer may have a
series of `ErrorHandler`s associated with it:

```go
type ErrorHandler interface {
	OnError(err error) (Buffer, error)
	Done()
}
```

During a transmission, a buffer may call `OnError()` whenever it runs
into an I/O error, asking the `ErrorHandler` to either capture the error
(as done by `MetricsBlobAccess`), substitute the error (as done by
`ExistencePreconditionBlobAccess`) or to continue the transmission using
a different buffer. The latter may be a copy of the same data obtained
from another source (high availability).

To facilitate fast random access I/O, the `Buffer` interface may
implement `io.ReaderAt` to extract just a part of the data. It is likely
the case that only a subset of the buffer types are capable of
implementing this efficiently, but that is to be expected.

Data integrity checking could be achieved by having special flavors of
the buffer construction functions:

```go
func NewACBufferFromReader(io.ReadCloser, RepairStrategy) Buffer                {}
func NewCASBufferFromReader(*util.Digest, io.ReadCloser, RepairStrategy) Buffer {}
```

Buffers created through these functions may enforce that their contents
are valid prior to returning them to their consumer. When detecting
inconsistencies, the provided `RepairStrategy` may contain a callback
that the storage backend can use to repair or delete the inconsistent
object.

Support for decomposing large blobs into smaller ones and recombining
them may be realized by adding more functions to the `Buffer` interface
or by adding decorator types. The exact details of that are outside the
scope of this ADR.
