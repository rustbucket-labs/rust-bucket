<p align="center">
  <img src="https://github.com/rustbucket-labs/.github/blob/main/images/rust-bucket-banner.svg" alt="Rust-Bucket - High-performance, S3-compatible object storage built in Rust">
</p>

## 📌 Overview

**Rust-Bucket** is a **Rust-native object storage server** designed for **performance, operational safety, and deployment flexibility**.  
It provides **S3-style APIs** and supports environments ranging from **local laptops** to **multi-node private clusters**. Rust-Bucket delivers **software-defined, scalable storage** for unstructured data without external dependencies or cloud lock-in.  
  
Built for modern workloads, Rust-Bucket emphasizes:  
  
* **Performance** — predictable latency and efficient resource usage  
* **Security & Safety** — memory-safe Rust foundation and future encrypted object support  
* **Portability** — runs anywhere: bare metal, containers, Kubernetes, VMs, and air-gapped systems
* **Ownership & Control** — deploy, operate, and extend on your terms

Whether used for **analytics data, application backends, archive repositories**, or **internal platforms**, Rust-Bucket provides the foundation for reliable object retention and access at scale.  

## 🚀 Key Capabilities

| Capability             | Description                                                     |
| ---------------------- | --------------------------------------------------------------- |
| S3-style API surface   | Familiar object storage semantics for easy integration          |
| Lightweight deployment | Single-binary design, ideal for Dev, CI, QA, and private infra  |
| Local-first operation  | Fully self-contained — no cloud dependencies required           |
| Rust-based safety      | Memory-safe, predictable performance, low operational overhead  |
| Scalability horizon    | Built to evolve into multi-node clusters and replicated storage |

## 🏁 Quick Start

#### Requirements
* Rust 2021+
* Cargo

#### Run the Server
```bash
git clone https://github.com/<org>/rust-bucket.git
cd rust-bucket
cargo run
```

#### Store an Object
```bash
curl -X PUT --data-binary @file.txt http://localhost:8080/object/file.txt
```

#### Retrieve an Object
```bash
curl http://localhost:8080/object/file.txt -o downloaded.txt
```

## 🧱 Architecture Snapshot
```bash
rust-bucket/
├─ src/
│  ├─ main.rs            # Entrypoint
│  ├─ server.rs          # HTTP Server (Axum/Hyper)
│  ├─ handlers/          # PUT/GET route handlers
│  ├─ storage/
│  │   ├─ fs_backend.rs  # Local filesystem implementation
│  │   └─ traits.rs      # BucketStorage trait
│  ├─ models/            # S3-style types (future: Smithy generation)
│  └─ config.rs          # Runtime settings (future)
├─ docs/
│  ├─ architecture.md
│  ├─ roadmap.md
│  └─ mbse/
├─ Cargo.toml
└─ README.md
```

## 🧪 Testing
```bash
cargo test
```

## 📜 Roadmap
| Phase       | Goal                                                 |
| ----------- | ---------------------------------------------------- |
| **MVP**     | PUT/GET objects, local storage, async server         |
| **Phase-1** | Buckets, metadata, object size & MD5, object listing |
| **Phase-2** | Auth & policies, encryption, lifecycle rules         |
| **Phase-3** | Distributed clustering, replication, versioning      |
| **Phase-4** | Advanced S3-style parity                             |

## 🤝 Contributing

Contributions are welcome.
Planned documents:
* `CONTRIBUTING.md`
* `ARCHITECTURE.md`
* `DESIGN.md`

Open a PR or issue to begin

## 🏛 License
MIT (See [LICENSE](./LICENSE))

## ⭐ Support
If Rust-Bucket is useful — **star the repository** to help others discover it.

