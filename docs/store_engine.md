# MinIO Storage Engine

**Prompt**: Minio - review its software in the GitHub repo. What does the object store engine look like structurally  
  
Structurally, MinIO’s “engine” is basically an S3 → ObjectLayer → erasure-coded backend pipeline, implemented mostly in the `cmd` package of the GitHub repo with no external metadata database.  
  
I’ll walk it top-down and tie it back to specific code components.  

### 1. High-level layering
At a very coarse level the server looks like:  

**1. HTTP / S3 API layer**
  * Go `net/http` handlers for S3 operations (`GET / PUT / LIST`, multipart, etc.), things like `object-handlers.go`, `bucket-handlers-*.go`, `api-router.go`. You can see references to these handlers in stack traces in issues (e.g. `objectAPIHandlers.PutObjectHandler` in `object-handlers.go`). [Connection reset by peer when upload to minio](https://roadmap.ilum.cloud/boards/bugs/posts/connection-reset-by-peer-when-upload-to-minio)  

**2. ObjectLayer abstraction**
  * A central Go interface `ObjectLayer` in `cmd/object-api-interface.go`. It defines the storage engine contract — methods for bucket/object CRUD, listing, versioning, healing, etc. [cmd](https://pkg.go.dev/github.com/minio/minio/cmd), [object-api-interface.go](https://github.com/minio/minio/blob/master/cmd/object-api-interface.go)  
  * The HTTP handlers never talk to disks directly; they call `ObjectLayer` methods.  

**3. Concrete ObjectLayer implementations**
  * The main one for “real” MinIO is the **erasure-coded backend** (distributed or single-node).  
  * There’s also a **filesystem (FS) backend** (mostly for single-disk / dev), and **gateway backends** that adapt S3, GCS, HDFS, iRODS, etc. to the same `ObjectLayer` interface. [Minio Architecture Introduction](https://e-whisper.com/posts/9462/), [sia](https://pkg.go.dev/github.com/creiht/minio/cmd/gateway/sia)  

**4. Low-level storage + erasure coding**
  * Erasure coding and disk I/O live in a cluster of files like `erasure-encode.go`, `erasure-decode.go`, `erasure-object.go`, `erasure-sets.go`, `erasure-server-pool.go`, `xl-storage.go`, etc. These are all in the `cmd` package as well (see file listings from Sourcegraph/pkg.go.dev and stack traces). [bucket-targets.go](https://sourcegraph.com/github.com/minio/minio/-/blob/cmd/bucket-targets.go), [Error: Write failed. Insufficient number of disks online](https://github.com/minio/minio/discussions/17619)  

**5. Support systems**
  * **Distributed locking** via the `dsync`-style lock manager (used behind `ObjectLayer.NewNSLock`), so concurrent writers/readers see consistent state. [Minio Architecture Introduction](https://e-whisper.com/posts/9462/)  
  * **No external metadata DB** – metadata is stored alongside objects on disk (`xl.json` and friends). [Minio Architecture Introduction](https://e-whisper.com/posts/9462/)  

### 2. Cluster layout: pools → erasure sets → disks  
MinIO’s engine organizes physical storage in three nested levels (all wired up in `cmd`):  

**1. Server pools**
  * A deployment is 1+ pools, each pool defined by a set of endpoints (e.g. `http://node{1...4}/export{1...4}`).
  * `GetAllSets` and `endpoint-ellipses.go` parse these ellipses and group endpoints into balanced erasure sets.
  * The type `erasureServerPools` appears in stack traces as the high-level object that implements `ObjectLayer` over multiple pools.

**2. Erasure sets**
  * Each pool is divided into **erasure sets**: groups of N drives across one or more nodes.
  * Erasure sets are managed by `erasure-sets.go` and related files. In stack traces you see calls like `(*erasureSets).PutObject()` and then `(*erasureServerPools).PutObject()`.
  * Design docs/blogs call these “erasure sets”, where each object is striped across `K` data + `M` parity shards over the drives in the set. [Erasure Coding](https://docs.min.io/enterprise/aistor-object-store/operations/core-concepts/erasure-coding)  

**3. Disks / volumes**
  * Each MinIO process manages a group of local volumes (`/export1`, `/export2`, …).
  * The low-level disk abstraction lives in `xl-storage.go` (for erasure backend) and helper files; it handles directory layout, fsync, direct I/O, etc. (you can see types like `xlStorage` in older docs and pkg listings).
  
So structurally:  
  
> HTTP handlers → ObjectLayer (implemented by erasureServerPools)
> → erasureSets (per erasure set)
> → erasureObjects + Erasure codec
> → xlStorage (per disk)
  
All of that is in `cmd/`.  

### 3. The ObjectLayer interface (the “engine contract”)  

The `ObjectLayer` interface in `cmd/object-api-interface.go` describes what the engine must do. Some highlights from the GoDoc snippet:
* **Locking & scanning**
  * `NewNSLock(bucket, objects...)` – per-object/bucket RW locking.
  * `NSScanner(...)` – background namespace scanner for usage/healing.
* **Backend & capacity**
  * `BackendInfo() BackendInfo` – describes FS vs XL, drive counts, etc.  
  * `StorageInfo(...)` & `LocalStorageInfo(...)` – capacity and health.
* **Bucket ops**
  * `MakeBucket`, `GetBucketInfo`, `ListBuckets`, `DeleteBucket`.
* **Object ops**
  * `PutObject`, `GetObject`, `DeleteObject`, `ListObjects`, `ListObjectsV2`, `ListObjectVersions`, etc.
  * Versioning, replication, locking, lifecycle policies via other methods.

Every concrete backend (erasure, FS, gateways) must satisfy this interface. That’s the core structural boundary of the engine.

### 4. Erasure-coded backend: major pieces

Within the erasure backend, key components are:

1. **Erasure codec:** `Erasure` type
  * Implemented in files like `erasure-encode.go`, `erasure-decode.go`, `erasure-common.go`, `erasure-coding.go`. The Sourcegraph/pkg index for `cmd` lists these explicitly.
  * This does the Reed–Solomon math: splitting an object into `K` data + `M` parity shards, using CPU-optimized instructions (docs mention AVX512).

2. **Per-object engine:** `erasureObjects`
  * Stack traces show `cmd.erasureObjects.putObject()` and `cmd.erasureObjects.PutObject()` in `erasure-object.go`.
  * This layer:
    * Validates input, encryption, multipart assembly.
    * Calls `Erasure.Encode()` to generate shards.
    * Writes shard files via `xlStorage` to each disk.
    * Generates/updates the per-object metadata (`xl.json`, parts info, checksums).
    * Handles healing reads (if some shards are missing, reconstructs from parity).

3. **Erasure sets:** `erasureSets`
  * Exposed in stack traces as `(*erasureSets).PutObject()`; sits one level above `erasureObjects`.
  * Responsibilities:
    * Choose which **erasure set** (stripe of drives) an object will land on.
    * Handle load balancing across sets.
    * Aggregate storage info and health for all drives in the set.
    * Coordinate healing within the set.

4. **Server pools:** `erasureServerPools`
  * Shown in traces as `(*erasureServerPools).PutObject()`.
  * This is effectively the **cluster-level ObjectLayer implementation:**
    * Maps buckets/objects to the right pool and set.
    * Handles multi-pool features (e.g., site replication, tiering).
    * Implements the `ObjectLayer` interface for all data paths.

5. **On-disk layout & metadata**
MinIO intentionally avoids a central metadata DB; instead, each object’s metadata lives on the same drives as its data.  
  
On a single erasure set, for a given object:  

* **Data shard files:** `part.1`, `part.2`, `…`
* **Metadata file:** `xl.json`
  
Per the architecture article (and you can verify via a MinIO deployment), each disk has both a data shard and an `xl.json` for that object:  
* Paths look like `/exportN/.minio.sys/buckets/<bucket>/<object>/…` (exact directories depend on version).  
* `xl.json` includes:
  * Object name, size, timestamps.
  * Versioning info.
  * The `K`/`M` erasure parameters (`ObjectInternalInfo` has `K` and `M` fields). 
  * Checksums for each shard.
* The object is **immutable** under the XL backend: overwrites are effectively new objects plus versioning/cleanup, as noted in MinIO’s own issues discussing FS vs XL immutability.  
  
This design is what lets MinIO:  
* Heal objects in place (using `erasure-healing-*.go` files). 
* Avoid a separate metadata service (no etcd/DB for object metadata itself).
* Survive up to `M` failed disks in an erasure set.

6. **Locking and consistency (dsync)**
For concurrency, MinIO uses a distributed lock manager (often called **dsync** in the ecosystem):
* Lock operations are abstracted via `ObjectLayer.NewNSLock(...)` which returns an `RWLocker`.
* At runtime, a lock request is broadcast to all nodes; if a quorum (`N/2 + 1`) responds, the lock is granted.
* There is no single lock master; all nodes participate equally.
  
This sits in the engine "above" the erasure backnd but is essential structurally to avoid corrupted metadata and to coordinate multipart writes, deletes, and healing.  
  
7. **Gateways and alternate backends**
Finally, MinIO can implement `ObjectLayer` over other storage systems:
* Gateway types (e.g., Sia, GCS, iRODS gateways) each have a `NewGatewayLayer(...) (minio.ObjectLayer, error)` that returns a wrapper over that backend.
* Structurally they sit where `erasureServerPools` would, but internally delegate to another SDK instead of `Erasure` / `xlStorage`.

**TL;DR: what the engine “looks like”**
If you squint at the GitHub repo, the object store engine is:  
* **One big Go package** `cmd`, containing:
  * S3/HTTP handlers → `ObjectLayer` interface.  
  * A main implementation stack: `erasureServerPools` → `erasureSets` → `erasureObjects` → `Erasure` codec → `xlStorage` on each disk.
  * Supporting pieces: locking (dsync), healing, replication, metrics.  
* **No external metadata DB** – all metadata lives in `xl.json` next to data shards on disk.
* **Cluster is decomposed into pools, sets, drives**, with erasure coding across each set.
  
If you want, next step we can sketch a Rust analogue for your “Rust bucket” project that mirrors this `ObjectLayer` / erasureSets / disk layout structure but on a much smaller, v0 scope.  

**Prompt** For my “Rust Bucket” project can you suggest any architecture improvements over the minio design?  

You can absolutely borrow MinIO’s ideas but tighten up some things in Rust Bucket, especially around separation of concerns, metadata, and future “lakeFS-style” branching.  

I’ll frame this as: **what MinIO does** → **what you could do differently / better in Rust Bucket**.

### 1. Split control plane and data plane from day one
  
**What MinIO does**
* Mostly a **single monolithic process** (`minio server`) with:
  * HTTP/S3 API
  * Cluster membership discovery
  * Locking (dsync)
  * Erasure engine
  * Healing, ILM, etc.
* Internally it has clean layers (`ObjectLayer`, erasureSets, xlStorage), but it’s still one big binary.

**Rust Bucket improvement**
For you, I’d treat this as a **service-oriented design**, even if v0 is single-process:
* **Control plane** (Rust crate + service):
  * Cluster membership, node config, drive inventory.
  * Bucket policies, tenants, quotas.
  * Global metadata schema migration.
* **Data plane** (workers / storage nodes):
  * HTTP/S3 gateway → storage engine.
  * Actually reads/writes objects.

In code:
```test
crates/
  rustbucket-core/        # traits, types, error types
  rustbucket-metadata/    # metadata engine (RocksDB/SQLite/whatever)
  rustbucket-storage/     # pluggable object engines (FS, erasure)
  rustbucket-gateway/     # HTTP/S3, auth, tracing
  rustbucket-control/     # optional control-plane / admin API
  rustbucket-cli/
```
For v0 they can live in one binary, but the crates enforce **Martin Fowler SoC** and let you peel them apart later when you want real distributed control plane.

### 2. Make metadata a first-class, strongly-typed service

**What MinIO does**
* Stores object metadata as `xl.json` **files on each disk**, next to data shards.
* No external metadata DB. Great for simplicity, but:
  * Listing and scanning can mean touching lots of disks.
  * Global operations (policy queries, lineage, branch/commit semantics) are harder.
  * Schema evolution is quirky because the “schema” is effectively in JSON blobs.

**Rust Bucket improvement**
Even for a simple local filesystem backend, you can define a **metadata engine** with a clear contract:
```rust
pub trait MetadataEngine {
    async fn put_object(&self, meta: ObjectMeta) -> Result<()>;
    async fn get_object(&self, key: &ObjectKey) -> Result<Option<ObjectMeta>>;
    async fn list_objects(&self, prefix: &Prefix, opts: ListOpts)
        -> Result<ObjectList>;
    async fn begin_txn(&self) -> Result<Box<dyn MetadataTxn>>;
}
```
And implement it initially with something like:
* **SQLite / Sled / RocksDB** on local disk.
* Simple schema: `tenants`, `buckets`, `objects`, `versions`, `parts`.
Benefits:
* Fast **LIST** without touching object shards.
* Clear path to:
  * Per-bucket **branches** / tags (lakeFS-style).
  * **Lineage** and provenance (tie to your Aveniq story later).
  * Indexes for lifecycle rules, auditing, and search.
* Easier **schema migrations**: you version the schema, not ad-hoc JSON files.
  
You can still store a small `shard_manifest` on disk (per object) for healing, but treat *metadata* as a separate concern.  
  
### 3. Pluggable placement & erasure coding strategies

**What MinIO does**
* Erasure sets and erasure coding are fairly baked into the core.
* Object placement is tied to set hashing and their internal layout.

**Rust Bucket improvement**
In Rust, make **placement and redundancy policies pluggable traits:**

```rust
pub trait PlacementStrategy: Send + Sync {
    fn choose_targets(
        &self,
        key: &ObjectKey,
        cluster: &ClusterTopology,
        redundancy: &RedundancyPolicy,
    ) -> Vec<ShardTarget>;
}

pub enum RedundancyPolicy {
    Replicated { copies: u8 },
    ReedSolomon { data: u8, parity: u8 },
    // later: LRC, local-RAID, etc.
}
```
* v0: `Replicated { copies: 1 }` over local FS.
* v1: add RS with, say, 4+2.
* Later: location-aware placement, “cold” vs “hot” tiers.
  
This makes it easy to experiment with **different codes and placement strategies** without rewriting the engine.  
  
### 4. Explicit, typed “Engine” interface (better than a giant `ObjectLayer`)
MinIO’s `ObjectLayer` is huge: dozens of methods, some quite specific to S3 quirks.  

**Rust Bucket improvement**

Do a more **granular** split:

```rust
pub trait ObjectReadEngine {
    async fn get_object(&self, key: &ObjectKey, range: Option<ByteRange>)
        -> Result<ObjectStream>;
    async fn head_object(&self, key: &ObjectKey) -> Result<Option<ObjectMeta>>;
}

pub trait ObjectWriteEngine {
    async fn put_object(&self, request: PutObjectRequest)
        -> Result<PutObjectResult>;
    async fn delete_object(&self, key: &ObjectKey) -> Result<()>;
}

pub trait NamespaceEngine {
    async fn create_bucket(&self, bucket: &BucketName) -> Result<()>;
    async fn list_objects(&self, bucket: &BucketName, opts: ListOpts)
        -> Result<ObjectList>;
    // etc.
}
```
Then compose:  
```rust
pub struct Engine {
    pub reader: Arc<dyn ObjectReadEngine>,
    pub writer: Arc<dyn ObjectWriteEngine>,
    pub ns: Arc<dyn NamespaceEngine>,
}
```
* Easier **unit testing** (you can mock the read/write path differently).  
* Clearer separation of **read path vs write path** (useful when you later want stronger consistency on writes and more relaxed caching on reads).  

### 5. Events and hooks as a first-class subsystem

**What MinIO does**
* Has notification targets, but they’re not the structural center of the engine.
* For your Aveniq/lakeFS-like future, events and lineage will be critical.

**Rust Bucket improvement**
Design **event emission** into the engine contract:

```rust
pub enum ObjectEventKind {
    ObjectCreated,
    ObjectDeleted,
    ObjectVersioned,
    ObjectRestored,
    // etc.
}

pub struct ObjectEvent {
    pub kind: ObjectEventKind,
    pub key: ObjectKey,
    pub meta: ObjectMeta,
    pub tx_id: TxId,
    pub timestamp: DateTime<Utc>,
}

pub trait EventSink {
    async fn publish(&self, event: ObjectEvent) -> Result<()>;
}
```

* Engine always emits events into an `EventSink`.
* v0: a simple local log file, or in-memory channel.
* Later:
  * Kafka/NATS/Redis streams.
  * “Rust” Avro/Parquet events feeding your data lake.
  * Hook into **data labeling**, lineage, and audits.

This sets you up for Aveniq-style provenance without overcomplicating v0.  

### 6. Stronger, explicit consistency and failure semantics

MinIO’s dsync is quorum-based and “good enough”, but semantics are not always crystal-clear to the user.  

**Rust Bucket improvement**
Even before you implement distributed consensus, you can:  
* Encode `ConsistencyLevel` as a **type/enum**, and let users choose per-bucket:
```rust
pub enum ConsistencyLevel {
    SingleNode,    // v0
    Quorum,        // future, across nodes
    All,           // write-through to all targets
}
```
* Document behavior:
  * What does a successful `PUT` mean?
  * Under what conditions can you see stale reads?
* Design your metadata engine with a **transaction log/WAL** so that later you can swap the local engine for a Raft-based “metadata cluster” without changing high-level code.  
  
Even if v0 is “single-node only”, having the concept in the design is a big improvement for future evolution.  

### 7. Background jobs as an explicit subsystem, not sprinkled logic
  
MinIO has healing, scanner, lifecycle, etc., but they’re very tied to the monolith.  

**Rust Bucket improvement**

Introduce a minimal **Job Engine**:

```rust
pub enum JobKind {
    HealObject(ObjectKey),
    CompactBucket(BucketName),
    ApplyLifecyclePolicy(BucketName),
}

pub trait JobQueue {
    async fn enqueue(&self, job: JobKind) -> Result<()>;
    async fn next(&self) -> Result<Option<JobKind>>;
}
```
* v0: in-memory queue, worker thread.
* Later: durable job store per node, or shared queue.
This makes:  
* Healing logic testable and decoupled.
* ILM policies pluggable.
* Easier to build that “Aveniq Server” doing offline tasks over multiple Rust Bucket nodes.

### 8. Observability designed in (metrics + tracing)  
  
MinIO has metrics, but you can be more opinionated with modern Rust tooling.  

**Rust Bucket improvement**
* Structured logs via `tracing`:
  * Include `request_id`, `bucket`, `object_key`, `node_id`, etc.
* **Metrics** trait and implementation:
```rust
pub trait Metrics {
    fn inc_requests(&self, op: &str);
    fn observe_lat(&self, op: &str, value: Duration);
    fn inc_bytes(&self, op: &str, bytes: u64);
}
```
* Call metrics from the engine and gateway, not from ad-hoc `println!`.
* Later: export Prometheus, OpenTelemetry; drive dashboards for:
  * Per-bucket SLAs.
  * Cluster balancing and skew.
  * Healing backlog.
  
Given your telemetry & options-trading work, this will feel natural.  

### 9. Security and multi-tenancy as design constraints  
  
MinIO of course supports multi-tenancy, but for Rust Bucket you can bake it deeply into the type system.  

**Rust Bucket improvement**
* Tenant isolation:
  * `TenantId` carried everywhere: `ObjectKey { tenant: TenantId, bucket: BucketName, key: String }`.
  * No “tenantless” operations in internal code.
* **Per-tenant policy** object:
  * Encryption defaults (SSE-S3/SSE-KMS style).
  * Allowed redundancy policies.
  * Quotas and retention.
* **Audit trail**:
  * Every operation gets a `PrincipalId` and `RequestId` that flow all the way down.
  * Events and metadata carry those IDs for later compliance work.
  
This dovetails nicely with your **classified-network / ATO / STIG** world.  

### 10. Seed the branch/commit model (lakeFS future) without implementing it now
You said you want lakeFS-style features later but not in v0. You can still arrange the **metadata schema** and types with that in mind:  
* Make `ObjectKey` include a **logical path** and have a separate `BranchId` / `CommitId` concept in the metadata layer.  
* v0: every bucket has exactly one branch (`main`), and all operations target that branch implicitly.
* v1+: you expose:
  * `rb branch create`, `rb branch diff`, `rb branch commit`.
  * “Production” vs “Experiment” views of the same dataset.
  
Because metadata is in a real engine (not scattered `xl.json`), adding branches later becomes an extension, not a rewrite.  

### How I’d summarize the “Rust Bucket vs MinIO” design stance
For Rust Bucket, I’d keep the **spirit** of MinIO’s:
* S3-compatible API
* Pluggable backend (`ObjectLayer` idea)
* Erasure / replication for durability
…but improve on:  
1. **Control plane vs data plane** separation.
2. **Metadata as a proper engine** (with transactions, indexes, and schema).
3. **Pluggable placement & redundancy** via traits.
4. **Smaller, composable engine traits** instead of one god-interface.
5. **Events, jobs, observability, and multi-tenancy** as first-class components.
6. Laying the groundwork for **branch/commit semantics** (lakeFS-style) without forcing it into v0.

If you’d like, next step I can sketch a **crate layout + minimal trait definitions + a v0 flow** like “`PUT` to local FS with metadata in SQLite”, so Rust Bucket is actually buildable while still following this architecture.  

**Prompt**: Let’s do it
  
Nice, Rust Bucket time 😄  
  
I’ll give you a **buildable v0 skeleton** you can drop into a workspace:  
* Single node, local filesystem.
* Metadata in SQLite.
* Async HTTP gateway (axum) with a minimal API:
  * `PUT /v0/buckets/:bucket/objects/*key`
  * `GET /v0/buckets/:bucket/objects/*key`
  
You get clean separation via crates and traits, so we can grow into erasure coding / lakeFS-style later.  

### 1. Workspace layout
```text
rust-bucket/
  Cargo.toml        # workspace
  crates/
    rustbucket-core/
      Cargo.toml
      src/lib.rs
    rustbucket-metadata/
      Cargo.toml
      src/lib.rs
      src/sqlite_engine.rs
    rustbucket-storage/
      Cargo.toml
      src/lib.rs
      src/fs_engine.rs
    rustbucket-gateway/
      Cargo.toml
      src/main.rs
```
### 2. Root Cargo.toml (workspace)
```toml
[workspace]
members = [
    "crates/rustbucket-core",
    "crates/rustbucket-metadata",
    "crates/rustbucket-storage",
    "crates/rustbucket-gateway",
]

resolver = "2"
```
### 3. rustbucket-core: shared types + engine traits
`crates/rustbucket-core/Cargo.toml`
```toml
[package]
name = "rustbucket-core"
version = "0.1.0"
edition = "2021"

[dependencies]
async-trait = "0.1"
chrono = { version = "0.4", features = ["serde"] }
serde = { version = "1", features = ["derive"] }
thiserror = "1"
bytes = "1"
```
`crates/rustbucket-core/src/lib.rs`
```rust
pub mod types;
pub mod engine;
pub mod error;

pub use crate::types::*;
pub use crate::engine::*;
pub use crate::error::*;
```
`crates/rustbucket-core/src/types.rs`
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct BucketName(pub String);

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ObjectKey {
    pub bucket: BucketName,
    pub key: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ObjectMeta {
    pub key: ObjectKey,
    pub size: u64,
    pub created_at: DateTime<Utc>,
    pub content_type: Option<String>,
    // room for future: etag, version_id, tenant_id, tags, etc.
}

#[derive(Debug, Clone)]
pub struct PutObjectRequest {
    pub key: ObjectKey,
    pub data: bytes::Bytes,
    pub content_type: Option<String>,
}

#[derive(Debug, Clone)]
pub struct PutObjectResult {
    pub meta: ObjectMeta,
}

#[derive(Debug, Clone)]
pub struct ListOptions {
    pub prefix: Option<String>,
    pub limit: u32,
}
```
`crates/rustbucket-core/src/error.rs`
```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum RustBucketError {
    #[error("metadata error: {0}")]
    MetadataError(String),

    #[error("storage error: {0}")]
    StorageError(String),

    #[error("not found: {0}")]
    NotFound(String),

    #[error("internal error: {0}")]
    Internal(String),
}

pub type Result<T> = std::result::Result<T, RustBucketError>;
```
`crates/rustbucket-core/src/engine.rs`
```rust
use crate::{ListOptions, ObjectKey, ObjectMeta, PutObjectRequest, PutObjectResult, Result};
use async_trait::async_trait;
use bytes::Bytes;

#[async_trait]
pub trait ObjectReadEngine: Send + Sync {
    async fn get_object(&self, key: &ObjectKey) -> Result<Bytes>;
    async fn head_object(&self, key: &ObjectKey) -> Result<ObjectMeta>;
}

#[async_trait]
pub trait ObjectWriteEngine: Send + Sync {
    async fn put_object(&self, req: PutObjectRequest) -> Result<PutObjectResult>;
    async fn delete_object(&self, key: &ObjectKey) -> Result<()>;
}

#[async_trait]
pub trait NamespaceEngine: Send + Sync {
    async fn create_bucket(&self, bucket: &str) -> Result<()>;
    async fn list_objects(&self, bucket: &str, opts: ListOptions) -> Result<Vec<ObjectMeta>>;
}

#[async_trait]
pub trait MetadataEngine: Send + Sync {
    async fn put_meta(&self, meta: ObjectMeta) -> Result<()>;
    async fn get_meta(&self, key: &ObjectKey) -> Result<ObjectMeta>;
    async fn delete_meta(&self, key: &ObjectKey) -> Result<()>;
    async fn list_meta(&self, bucket: &str, opts: ListOptions) -> Result<Vec<ObjectMeta>>;
    async fn ensure_schema(&self) -> Result<()>;
}

/// Simple v0 “engine wiring” used by the gateway.
pub struct Engine {
    pub reader: std::sync::Arc<dyn ObjectReadEngine>,
    pub writer: std::sync::Arc<dyn ObjectWriteEngine>,
    pub ns: std::sync::Arc<dyn NamespaceEngine>,
}
```

### 4. `rustbucket-metadata`: SQLite metadata engine
`crates/rustbucket-metadata/Cargo.toml`
```toml
[package]
name = "rustbucket-metadata"
version = "0.1.0"
edition = "2021"

[dependencies]
rustbucket-core = { path = "../rustbucket-core" }
async-trait = "0.1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite"] }
chrono = { version = "0.4", features = ["serde"] }
```
`crates/rustbucket-metadata/src/lib.rs`
```rust
mod sqlite_engine;

pub use sqlite_engine::SqliteMetadataEngine;
```
`crates/rustbucket-metadata/src/sqlite_engine.rs`
```rust
use async_trait::async_trait;
use chrono::Utc;
use rustbucket_core::{
    ListOptions, MetadataEngine, ObjectKey, ObjectMeta, Result, RustBucketError,
};
use sqlx::{sqlite::SqlitePoolOptions, Pool, Sqlite};

#[derive(Clone)]
pub struct SqliteMetadataEngine {
    pool: Pool<Sqlite>,
}

impl SqliteMetadataEngine {
    pub async fn new(database_url: &str) -> Result<Self> {
        let pool = SqlitePoolOptions::new()
            .max_connections(5)
            .connect(database_url)
            .await
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

        let engine = Self { pool };
        engine.ensure_schema().await?;
        Ok(engine)
    }
}

#[async_trait]
impl MetadataEngine for SqliteMetadataEngine {
    async fn ensure_schema(&self) -> Result<()> {
        // Very simple v0 schema
        let query = r#"
            CREATE TABLE IF NOT EXISTS objects (
                bucket      TEXT NOT NULL,
                key         TEXT NOT NULL,
                size        INTEGER NOT NULL,
                created_at  TEXT NOT NULL,
                content_type TEXT,
                PRIMARY KEY (bucket, key)
            );
        "#;

        sqlx::query(query)
            .execute(&self.pool)
            .await
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

        Ok(())
    }

    async fn put_meta(&self, meta: ObjectMeta) -> Result<()> {
        let q = r#"
            INSERT INTO objects (bucket, key, size, created_at, content_type)
            VALUES (?1, ?2, ?3, ?4, ?5)
            ON CONFLICT(bucket, key) DO UPDATE SET
                size = excluded.size,
                created_at = excluded.created_at,
                content_type = excluded.content_type;
        "#;

        sqlx::query(q)
            .bind(&meta.key.bucket.0)
            .bind(&meta.key.key)
            .bind(meta.size as i64)
            .bind(meta.created_at.to_rfc3339())
            .bind(meta.content_type.as_deref())
            .execute(&self.pool)
            .await
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

        Ok(())
    }

    async fn get_meta(&self, key: &ObjectKey) -> Result<ObjectMeta> {
        let q = r#"
            SELECT bucket, key, size, created_at, content_type
            FROM objects
            WHERE bucket = ?1 AND key = ?2;
        "#;

        let row = sqlx::query(q)
            .bind(&key.bucket.0)
            .bind(&key.key)
            .fetch_optional(&self.pool)
            .await
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?
            .ok_or_else(|| RustBucketError::NotFound(format!("{}/{}", key.bucket.0, key.key)))?;

        let bucket: String = row.get("bucket");
        let key_str: String = row.get("key");
        let size: i64 = row.get("size");
        let created_at_str: String = row.get("created_at");
        let content_type: Option<String> = row.try_get("content_type").ok();

        let created_at = created_at_str
            .parse()
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

        Ok(ObjectMeta {
            key: ObjectKey {
                bucket: crate::bucket_name(bucket),
                key: key_str,
            },
            size: size as u64,
            created_at,
            content_type,
        })
    }

    async fn delete_meta(&self, key: &ObjectKey) -> Result<()> {
        let q = r#"
            DELETE FROM objects WHERE bucket = ?1 AND key = ?2;
        "#;

        sqlx::query(q)
            .bind(&key.bucket.0)
            .bind(&key.key)
            .execute(&self.pool)
            .await
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

        Ok(())
    }

    async fn list_meta(&self, bucket: &str, opts: ListOptions) -> Result<Vec<ObjectMeta>> {
        let mut base = String::from(
            "SELECT bucket, key, size, created_at, content_type FROM objects WHERE bucket = ?1",
        );
        if opts.prefix.is_some() {
            base.push_str(" AND key LIKE ?2 || '%'");
        }
        base.push_str(" ORDER BY key LIMIT ?3");

        let mut query = sqlx::query(&base).bind(bucket.to_string());
        if let Some(prefix) = opts.prefix {
            query = query.bind(prefix);
        }
        query = query.bind(opts.limit as i64);

        let rows = query
            .fetch_all(&self.pool)
            .await
            .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

        let mut out = Vec::new();
        for row in rows {
            let bucket: String = row.get("bucket");
            let key_str: String = row.get("key");
            let size: i64 = row.get("size");
            let created_at_str: String = row.get("created_at");
            let content_type: Option<String> = row.try_get("content_type").ok();

            let created_at = created_at_str
                .parse()
                .map_err(|e| RustBucketError::MetadataError(e.to_string()))?;

            out.push(ObjectMeta {
                key: ObjectKey {
                    bucket: crate::bucket_name(bucket),
                    key: key_str,
                },
                size: size as u64,
                created_at,
                content_type,
            });
        }

        Ok(out)
    }
}

// tiny helper; you could move this into rustbucket-core instead:
impl SqliteMetadataEngine {
    pub fn pool(&self) -> &Pool<Sqlite> {
        &self.pool
    }
}

// local helper used above
mod crate {
    use rustbucket_core::BucketName;

    pub fn bucket_name(name: String) -> BucketName {
        BucketName(name)
    }
}

use sqlx::Row;
use sqlx::Column;
use sqlx::Row as _;
```
(You may want to clean up the helper module later; I kept it local to keep core untouched.)  

### 5. `rustbucket-storage`: local filesystem backend
`crates/rustbucket-storage/Cargo.toml`
```toml
[package]
name = "rustbucket-storage"
version = "0.1.0"
edition = "2021"

[dependencies]
rustbucket-core = { path = "../rustbucket-core" }
rustbucket-metadata = { path = "../rustbucket-metadata" }
async-trait = "0.1"
tokio = { version = "1", features = ["fs", "macros", "rt-multi-thread"] }
chrono = { version = "0.4", features = ["serde"] }
bytes = "1"
```
`crates/rustbucket-storage/src/lib.rs`
```rust
mod fs_engine;

pub use fs_engine::FsStorageEngine;
```
`crates/rustbucket-storage/src/fs_engine.rs`
```rust
use async_trait::async_trait;
use bytes::Bytes;
use chrono::Utc;
use rustbucket_core::{
    ListOptions, MetadataEngine, NamespaceEngine, ObjectKey, ObjectMeta, ObjectReadEngine,
    ObjectWriteEngine, PutObjectRequest, PutObjectResult, Result, RustBucketError,
};
use std::path::{Path, PathBuf};
use std::sync::Arc;
use tokio::fs;

#[derive(Clone)]
pub struct FsStorageEngine<M: MetadataEngine + 'static> {
    root: PathBuf,
    meta: Arc<M>,
}

impl<M: MetadataEngine + 'static> FsStorageEngine<M> {
    pub fn new<P: AsRef<Path>>(root: P, meta: Arc<M>) -> Self {
        Self {
            root: root.as_ref().to_path_buf(),
            meta,
        }
    }

    fn object_path(&self, key: &ObjectKey) -> PathBuf {
        // simple v0 layout: <root>/<bucket>/<key>
        self.root.join(&key.bucket.0).join(&key.key)
    }
}

#[async_trait]
impl<M> ObjectWriteEngine for FsStorageEngine<M>
where
    M: MetadataEngine + Send + Sync,
{
    async fn put_object(&self, req: PutObjectRequest) -> Result<PutObjectResult> {
        let path = self.object_path(&req.key);

        if let Some(parent) = path.parent() {
            fs::create_dir_all(parent)
                .await
                .map_err(|e| RustBucketError::StorageError(e.to_string()))?;
        }

        fs::write(&path, &req.data)
            .await
            .map_err(|e| RustBucketError::StorageError(e.to_string()))?;

        let meta = ObjectMeta {
            key: req.key,
            size: req.data.len() as u64,
            created_at: Utc::now(),
            content_type: req.content_type,
        };

        self.meta.put_meta(meta.clone()).await?;

        Ok(PutObjectResult { meta })
    }

    async fn delete_object(&self, key: &ObjectKey) -> Result<()> {
        let path = self.object_path(key);
        // delete file (ignore "not found" at FS level, but metadata may error)
        if let Err(e) = fs::remove_file(&path).await {
            if e.kind() != std::io::ErrorKind::NotFound {
                return Err(RustBucketError::StorageError(e.to_string()));
            }
        }

        self.meta.delete_meta(key).await?;
        Ok(())
    }
}

#[async_trait]
impl<M> ObjectReadEngine for FsStorageEngine<M>
where
    M: MetadataEngine + Send + Sync,
{
    async fn get_object(&self, key: &ObjectKey) -> Result<Bytes> {
        let path = self.object_path(key);
        let data = fs::read(&path)
            .await
            .map_err(|e| match e.kind() {
                std::io::ErrorKind::NotFound => {
                    RustBucketError::NotFound(format!("{}/{}", key.bucket.0, key.key))
                }
                _ => RustBucketError::StorageError(e.to_string()),
            })?;
        Ok(Bytes::from(data))
    }

    async fn head_object(&self, key: &ObjectKey) -> Result<ObjectMeta> {
        self.meta.get_meta(key).await
    }
}

#[async_trait]
impl<M> NamespaceEngine for FsStorageEngine<M>
where
    M: MetadataEngine + Send + Sync,
{
    async fn create_bucket(&self, bucket: &str) -> Result<()> {
        let path = self.root.join(bucket);
        fs::create_dir_all(&path)
            .await
            .map_err(|e| RustBucketError::StorageError(e.to_string()))?;
        Ok(())
    }

    async fn list_objects(&self, bucket: &str, opts: ListOptions) -> Result<Vec<ObjectMeta>> {
        self.meta.list_meta(bucket, opts).await
    }
}
```

### 6. rustbucket-gateway: minimal HTTP API (axum)
`crates/rustbucket-gateway/Cargo.toml`
```toml
[package]
name = "rustbucket-gateway"
version = "0.1.0"
edition = "2021"

[dependencies]
rustbucket-core = { path = "../rustbucket-core" }
rustbucket-metadata = { path = "../rustbucket-metadata" }
rustbucket-storage = { path = "../rustbucket-storage" }

axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
bytes = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["fmt", "env-filter"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```
`crates/rustbucket-gateway/src/main.rs`
```rust
use axum::{
    body::Bytes,
    extract::{Path, State},
    http::StatusCode,
    routing::{get, put},
    Router,
};
use rustbucket_core::{
    BucketName, Engine, ObjectKey, PutObjectRequest, Result as RBResult, RustBucketError,
};
use rustbucket_metadata::SqliteMetadataEngine;
use rustbucket_storage::FsStorageEngine;
use std::{net::SocketAddr, sync::Arc};
use tracing::{error, info};
use tracing_subscriber::{fmt, EnvFilter};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    init_tracing();

    let db_url = std::env::var("RUST_BUCKET_DB_URL")
        .unwrap_or_else(|_| "sqlite://rustbucket.sqlite".to_string());
    let data_root = std::env::var("RUST_BUCKET_DATA_ROOT")
        .unwrap_or_else(|_| "./data".to_string());

    let meta_engine = Arc::new(SqliteMetadataEngine::new(&db_url).await?);
    let storage_engine = Arc::new(FsStorageEngine::new(data_root, meta_engine.clone()));

    let engine = Arc::new(Engine {
        reader: storage_engine.clone(),
        writer: storage_engine.clone(),
        ns: storage_engine.clone(),
    });

    let app = Router::new()
        .route(
            "/v0/buckets/:bucket/objects/*key",
            put(put_object_handler).get(get_object_handler),
        )
        .with_state(engine);

    let addr: SocketAddr = "0.0.0.0:8080".parse().unwrap();
    info!("Rust Bucket listening on {}", addr);
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await?;

    Ok(())
}

fn init_tracing() {
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));
    fmt().with_env_filter(filter).init();
}

async fn put_object_handler(
    State(engine): State<Arc<Engine>>,
    Path((bucket, key)): Path<(String, String)>,
    body: Bytes,
) -> (StatusCode, String) {
    let key = ObjectKey {
        bucket: BucketName(bucket.clone()),
        key: key.clone(),
    };

    let req = PutObjectRequest {
        key,
        data: body,
        content_type: None, // could read from header
    };

    match engine.writer.put_object(req).await {
        Ok(result) => (
            StatusCode::CREATED,
            format!("stored {}/{} ({} bytes)", bucket, key, result.meta.size),
        ),
        Err(e) => error_response(e),
    }
}

async fn get_object_handler(
    State(engine): State<Arc<Engine>>,
    Path((bucket, key)): Path<(String, String)>,
) -> (StatusCode, axum::body::Body) {
    let key_struct = ObjectKey {
        bucket: BucketName(bucket.clone()),
        key: key.clone(),
    };

    match engine.reader.get_object(&key_struct).await {
        Ok(bytes) => (StatusCode::OK, axum::body::Body::from(bytes)),
        Err(e) => {
            let (code, msg) = map_error(&e);
            (code, axum::body::Body::from(msg))
        }
    }
}

fn error_response(e: RustBucketError) -> (StatusCode, String) {
    let (code, msg) = map_error(&e);
    error!(error = ?e, "request failed");
    (code, msg)
}

fn map_error(e: &RustBucketError) -> (StatusCode, String) {
    match e {
        RustBucketError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
        RustBucketError::MetadataError(msg)
        | RustBucketError::StorageError(msg)
        | RustBucketError::Internal(msg) => {
            (StatusCode::INTERNAL_SERVER_ERROR, msg.clone())
        }
    }
}
```

### 7. How to run v0
From the workspace root:  
```bash
# 1) Build
cargo build

# 2) Run the gateway
RUST_BUCKET_DB_URL="sqlite://rustbucket.sqlite" \
RUST_BUCKET_DATA_ROOT="./data" \
cargo run -p rustbucket-gateway
```
In another terminal:
```bash
# Put an object
curl -X PUT --data-binary @Cargo.toml \
  http://localhost:8080/v0/buckets/test-bucket/objects/foo/bar.txt

# Get it back
curl -v http://localhost:8080/v0/buckets/test-bucket/objects/foo/bar.txt
```
The filesystem layout will look like:  
```text
data/
  test-bucket/
    foo/
      bar.txt
rustbucket.sqlite   # holds object metadata
```
If you’re good with this skeleton, the next iterations can be:  
1. Add **bucket create/list endpoints**.
2. Add **content-type** handling from headers.
3. Introduce a **PlacementStrategy** trait on top of `FsStorageEngine`.
4. Start sketching a **JobQueue** trait and a simple in-process healing/compaction job.
  
But this gives you a real, Rust-native “MinIO-lite” core with cleaner separation of concerns than MinIO’s monolith.  
