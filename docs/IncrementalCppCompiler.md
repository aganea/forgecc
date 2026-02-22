# **forgecc**: An Incremental, Distributed C++20 Compiler

**Status**: Proposal / Architecture Draft  
**Author**: Alexandre Ganea  
**Updated**: 2026-02-22

---

**Table of Contents**

- [1. Executive Summary](#1-executive-summary)
- [2. Design Principles](#2-design-principles)
- [2b. Why a New Compiler Instead of Modifying LLVM/Clang](#2b-why-a-new-compiler-instead-of-modifying-llvmclang)
- [3. High-Level Architecture](#3-high-level-architecture)
- [4. Component Design](#4-component-design)
  - [4.1 The Distributed Content-Addressed Store](#41-the-distributed-content-addressed-store-ipfs-like)
  - [4.2 The Memoizer](#42-the-memoizer)
  - [4.3 Lexer & Preprocessor](#43-lexer--preprocessor)
  - [4.4 Parser](#44-parser)
  - [4.5 Semantic Analysis (Modular Design)](#45-semantic-analysis-modular-design)
  - [4.6a consteval/constexpr via In-Process JIT](#46a-constevalconstexpr-via-in-process-jit-key-innovation)
  - [4.6b ForgeIR — Intermediate Representation](#46b-forgeir--intermediate-representation)
  - [4.7 x86-64 Code Generation](#47-x86-64-code-generation)
  - [4.8 Runtime Loader / JIT Execution](#48-runtime-loader--jit-execution)
  - [4.9 Linker](#49-linker)
  - [4.10 Unreal Build Tool (UBT) Integration](#410-unreal-build-tool-ubt-integration-command-passing-protocol)
  - [4.11 Daemon Architecture](#411-daemon-architecture)
  - [4.12 Distributed P2P Network](#412-distributed-p2p-network-ipfs-like)
  - [4.13 Unified Protocol Design](#413-unified-protocol-design-http--messagepackjson)
- [5. Crate Structure](#5-crate-structure)
- [6. Key Data Flows](#6-key-data-flows)
- [7. Build System Integration](#7-build-system-integration)
- [8. Phased Implementation Plan](#8-phased-implementation-plan)
- [9. Risk Assessment](#9-risk-assessment)
- [10. Key Dependencies (Rust Crates)](#10-key-dependencies-rust-crates)
- [11. Lines-of-Code Estimates](#11-lines-of-code-estimates)
- [12. Parallelism Strategy (Detailed)](#12-parallelism-strategy-detailed)
- [13. Standard Library Strategy](#13-standard-library-strategy)
- [14. C++20 Modules Decision](#14-c20-modules-decision)
- [15. UE5 Codebase Analysis Findings](#15-ue5-codebase-analysis-findings-verified-against-source)
- [16. Deterministic Compilation (Required)](#16-deterministic-compilation-required)
- [17. JIT Runtime Details](#17-jit-runtime-details)
- [18. Live PGO Injection (PoC Feature)](#18-live-pgo-injection-poc-feature)
- [19. A VM for C++?](#19-a-vm-for-c)
- [20. Future Work (Beyond the PoC)](#20-future-work-beyond-the-poc)
  - [20.1 Console Development](#201-console-development-ps5-xbox-series-x-nintendo-switch)
  - [20.2 Artifact Extraction and Release Builds](#202-artifact-extraction-and-release-builds)
  - [20.3 Rust Front-End](#203-rust-front-end)
- [21. N4950 Cross-Reference (C++23 Standard Draft)](#21-n4950-cross-reference-c23-standard-draft)
- [22. Success Criteria](#22-success-criteria)

---

## 1. Executive Summary

**forgecc** is a thought experiment exploring what a C++ compiler could look like if
designed from scratch around memoization, daemon persistence, and JIT execution. It
takes the form of an incremental C++20 compiler and linker, written in Rust, running
as a persistent daemon. Its primary goal is to dramatically reduce build times for
large C++ codebases (target: Unreal Engine 5+ / Chromium) by:

- **Eliminating redundant work** through fine-grained memoization at every compilation stage
- **Persisting compilation state** across builds in a content-addressed distributed database
- **Distributing compilation knowledge** across machines via a peer-to-peer synchronized index
- **Unifying compilation and linking** in a single in-process pipeline
- **Generating debug information on demand** — no PDB or DWARF files during development;
  the daemon retains all compilation metadata (types, source maps, frame layouts) and
  serves debug info live through a built-in [DAP](https://microsoft.github.io/debug-adapter-protocol/) server
- **Runtime patching / JIT execution** — the daemon acts as an OS loader, building the
  target application in-memory progressively, compiling functions on demand

**forgecc** is, in essence, a **native JIT runtime for C++** — a daemon that compiles,
loads, patches, and debugs C++ code entirely in-memory, without producing intermediate
files. It is informed by LLVM/Clang's architecture but is a clean implementation, not
a fork. It targets a minimal but sufficient feature set: **C++20 (and C), x86-64,
Windows only** (initially). COFF/CodeView/PDB are supported for release builds only;
during development, the daemon's JIT loader and DAP debugger eliminate the need for
object files or debug info serialization.

**Notable precedents**:

- **CERN Cling** (2011–present): Interactive C++ interpreter/JIT built on LLVM/Clang.
  Used in production at CERN for high-energy physics (1 EB of data analyzed, 1000+
  scientific publications). Demonstrates that C++ JIT is viable at scale, though Cling
  is an interpreter with JIT, not a full compiler daemon.
  [https://root.cern/cling/](https://root.cern/cling/)

- **Zapcc** (2018): Caching C++ compiler daemon based on Clang. Keeps parsed headers
  in memory across compilations. Achieved 10–50x speedups on template-heavy code.
  Closest architectural precedent to **forgecc**'s daemon model, though it doesn't do JIT
  execution or runtime patching.
  [https://github.com/yrnkrn/zapcc](https://github.com/yrnkrn/zapcc)

- **Live++** (commercial): Binary hot-reload tool for C++. Patches machine code of
  running executables in-place. Used by 100+ companies including many UE5 studios.
  Demonstrates that runtime patching of C++ is production-viable on Windows.
  [https://liveplusplus.tech/](https://liveplusplus.tech/)

- **Runtime Compiled C++ (RCC++)**: Open-source runtime C++ recompilation for game
  development. Recompiles changed code and hot-swaps it into a running process.
  [https://github.com/RuntimeCompiledCPlusPlus/RuntimeCompiledCPlusPlus](https://github.com/RuntimeCompiledCPlusPlus/RuntimeCompiledCPlusPlus)

- **Circle** (Sean Baxter): Single-developer C++ compiler with novel language
  extensions. Demonstrates that a single person/small team can build a working C++
  compiler targeting real-world code.

- **SN-Systems Program Repository** (Sony, 2018–2022): LLVM-based research project
  that replaced object files with a content-addressed database (pstore). Proved that
  storing compilation results in a database with minimal stamp files for build system
  compatibility works at scale.
  [https://github.com/SNSystems/llvm-project-prepo/wiki](https://github.com/SNSystems/llvm-project-prepo/wiki)

**forgecc** differs from all of these by **unifying compilation, linking, loading,
debugging, and distributed caching into a single daemon process**.

### Non-Goals (for the proof-of-concept)

- No 32-bit support, no ARM64
- No ELF/Mach-O/DWARF (architecture allows future extension)
- No optimization passes beyond trivial ones (dead code elimination, constant folding)
- No standalone compiler — `forge-cc.exe` and the forge tool family are thin RPC
  clients (~800 lines total) that forward to the daemon; all real work happens
  in the daemon (see §7.3)
- No Objective-C/OpenCL/CUDA
- No cross-compilation
- No LLVM IR for the PoC — we define our own IR for determinism and memoization
  control; LLVM IR lowering deferred to release-mode optimization path (see §4.6b)
- No PCH — the memoization/caching system replaces PCH transparently
- No C++20 modules (UE5 has zero usage; see §15.14)
- No C++20 coroutines (UE5 has zero usage in Engine; see §15.14)
- No `std::ranges` / `std::views` (UE5 has zero usage; uses custom iterators)
- No `std::format` (UE5 has zero usage; uses `FString::Printf` / `fmt`)
- No `std::span` (UE5 has zero usage; uses `TArrayView` / `FMemoryView`)
- No `__fastcall` / `__vectorcall` / `__thiscall` calling conventions (zero usage)
- No `__declspec(novtable)` / `__declspec(property)` (zero usage)
- No WinRT / WRL support (zero usage)
- No resource compiler (.rc) support — `forge-rc.exe` exists as a driver shim but
  forwards to `llvm-rc` or `rc.exe` for now; native .rc compilation is deferred
- No HLSL shader compilation — shaders are compiled by UE5's ShaderCompileWorker,
  a completely separate pipeline outside forgecc's scope
- No memory budget optimization — this is a PoC; memory profiling and optimization
  come later

---

## 2. Design Principles

1. **Minimal code, maximum leverage**: Every component should be as small as possible.
   Prefer simplicity over generality. We don't need to handle every edge case in the
   C++ standard — we need to handle the subset that UE5/Game uses.

2. **Content-addressed everything**: Every intermediate artifact (tokens, AST nodes,
   types, IR, machine code sections) is identified by a hash of its inputs. If the
   inputs haven't changed, the output is reused. The indexing model draws inspiration
   from the [Legion Labs Merkle tree content store](https://github.com/legion-labs/legion/tree/main)
   (`lgn-content-store`), which uses copy-on-write trees, structural diffing, and
   layered indexers for efficient deduplication and change detection.

3. **No files, only database records**: Traditional compilers produce .obj files that
   the linker reads. **forgecc** skips this — the compiler writes sections directly into a
   distributed content-addressed store, and the linker reads from the same store.
   No serialization/deserialization of object files.

4. **Always-on, always-warm**: The daemon keeps hot data in memory. A full rebuild after
   a single-line change should take milliseconds, not seconds.

5. **Distributed by default (IPFS-style)**: Multiple daemon instances form a peer-to-peer
   network. Storage is distributed across machines. No central server, no centralized
   storage. Machines discover each other via seeds.

6. **No user-managed complexity**: Users should not need to manage PCH, unity builds,
   include-what-you-use, or module maps. The compiler handles caching, deduplication,
   and incremental compilation transparently.

7. **Parallel by design**: Every component must support parallelism. Unlike LLVM/Clang
   which is single-threaded per TU, **forgecc** parallelizes within a single TU where possible
   and across TUs by default.

8. **Runtime-first**: The primary execution model is runtime patching / JIT. The daemon
   loads and patches the target application in-memory. Generating a standalone .EXE/.DLL
   is a secondary "release" mode.

---

## 2b. Why a New Compiler Instead of Modifying LLVM/Clang

The natural first question is: why not build forgecc's features on top of
LLVM/Clang? Clang is an excellent, mature C++ compiler — arguably the best open-
source C++ front-end ever built. We have deep respect for the LLVM community and
the decades of work that went into it. Ideally, we would contribute these ideas
directly to LLVM. This section explains why, regrettably, the specific combination
of features forgecc needs does not fit within LLVM/Clang's current architecture
without changes so deep that they would amount to a rewrite of core subsystems.

**Prior art: projects that extended LLVM/Clang**

Several teams have explored variations of this idea. Their experiences are
instructive — not as failures, but as evidence of where the architectural
boundaries lie:

| Project | Approach | What happened |
|---|---|---|
| **Zapcc** (2018) | Forked Clang; kept AST/Sema in memory across TUs as a daemon | Achieved impressive 10–50x speedups, validating the daemon concept. Unfortunately, maintaining a Clang fork proved unsustainable — Zapcc fell behind upstream within ~2 years. The fork touched ~200 files across Clang's internals, making rebasing prohibitively expensive. |
| **Cling** (CERN, 2011–present) | Built on top of Clang/LLVM as a library; interactive C++ JIT | A remarkable success story — used in production at CERN. However, the team reports that every LLVM major version upgrade requires significant porting effort (~62 files changed, ~1,300 insertions against Clang-9). The CaaS (Compiler-as-a-Service) project is working to upstream these patches, which speaks to how much effort it takes to maintain them out-of-tree. |
| **SN-Systems pstore** (Sony, 2018–2022) | Modified LLVM's output layer to write to a content-addressed database instead of .obj files | Proved that content-addressed compilation artifacts work at scale — a key validation for forgecc's approach. The project was eventually discontinued; the modifications were difficult to maintain alongside upstream LLVM evolution. |
| **Clangd / rust-analyzer** | Built as separate tools using compiler infrastructure as libraries | Clangd works well within Clang's constraints but inherits its single-threaded, file-at-a-time model. Notably, rust-analyzer chose to reimplement parsing and type-checking independently of rustc, specifically to achieve the incrementality that rustc's architecture could not easily provide. |

These projects demonstrate that LLVM/Clang is a powerful foundation, but that
certain architectural properties — persistent daemon state, fine-grained
memoization, intra-TU parallelism — require changes deep enough that maintaining
them as patches or forks becomes the dominant engineering cost.

**What forgecc's feature set would require inside LLVM/Clang**:

1. **Content-addressed memoization at every stage** — This is the core challenge.
   Clang's AST nodes are allocated in a `BumpPtrAllocator` with pointer identity;
   they are not content-hashable by design. To add content-addressing, we would
   need to either (a) serialize every AST node to compute a hash (which adds
   overhead that could negate the memoization benefit) or (b) rework Clang's AST
   allocation model to support structural hashing. Option (b) would touch a very
   large number of files across `clang/AST/` and `clang/Sema/`. We would love to
   see content-addressable AST nodes in Clang someday, but the scope of that
   change is beyond what a single project can reasonably propose.

2. **Daemon mode with persistent state** — Clang's `CompilerInstance` is designed
   around a create-use-destroy lifecycle for each TU. Global state
   (IdentifierTable, SourceManager, Preprocessor) is not designed to be reused
   across compilations. Zapcc demonstrated that this can be made to work, but
   the cleanup logic required careful handling of Clang internals that change
   with each upstream release. The CaaS project at CERN is making progress on
   upstreaming incremental compilation support (IncrementalAction, persistent
   CompilerInstance), which is encouraging — but even their scope is narrower
   than what forgecc needs (they focus on interactive REPL, not full build-system
   integration with cross-build caching).

3. **Fine-grained parallelism within a TU** — Clang's Sema is single-threaded
   per TU, which is a reasonable design choice for a traditional compiler. Sema
   uses mutable state extensively (`Sema::CurContext`, `Sema::ExprEvalContexts`,
   delayed diagnostics). Making Sema thread-safe would require reworking most of
   `lib/Sema/` (~180,000 lines) — a change so large that it would effectively be
   a new Sema. This is not a criticism of Clang's design; single-threaded Sema
   is the right trade-off for a general-purpose compiler. forgecc's different
   trade-off (daemon with warm caches) makes intra-TU parallelism more valuable.

4. **Deterministic output for memoization** — Clang emits LLVM IR, which has
   sources of non-determinism described in §4.6b (sequential metadata IDs,
   emission-order-dependent value numbering). These are not bugs — they are
   consequences of LLVM IR's design, which optimizes for compilation speed rather
   than output reproducibility. Fixing this would require canonicalization passes
   in `llvm/lib/IR/` and `llvm/lib/Bitcode/`, touching ~50+ files. Some of these
   changes might be upstreamable (deterministic output is valuable for build
   reproducibility in general), and we would be happy to contribute them if the
   LLVM community is interested.

5. **JIT execution with runtime patching** — This is where LLVM is closest to
   forgecc's vision. ORC JIT already provides impressive infrastructure:
   COFF/x86-64 JIT linking, SEH frame support, function redirection via
   atomically-rewritable stubs, a re-optimization layer, out-of-process
   execution, and lazy compilation. forgecc's JIT concept is directly informed
   by ORC's design. The remaining gaps are the DAP debug server, native Windows
   TLS (ORC currently uses emulated TLS), build-system daemon integration, and
   — most importantly — content-addressed memoization. The JIT in forgecc is
   tightly coupled to the memoization system: every JIT'd function must be
   content-addressed, cached, and distributable. Integrating this into ORC would
   require either feeding it pre-built .obj files (reducing ORC to just a linker)
   or modifying ORC's internals to work with our store. Instead, forgecc builds
   a simpler, purpose-built runtime loader (~3,000 lines) that consumes sections
   directly from the content-addressed store, using ORC as a reference
   implementation for the hard parts (relocation application, SEH registration,
   TLS setup).

6. **Distributed P2P caching** — This is entirely new functionality that does not
   exist in LLVM. It would be a pure addition, but it needs deep integration with
   the content-addressed store (point 1), which itself requires the architectural
   changes described above.

**Estimated scope of LLVM/Clang modifications**:

| Change | Estimated files touched | Complexity |
|---|---|---|
| Content-hashable AST | ~500+ files in `clang/AST/`, `clang/Sema/` | Very high — fundamental data structure redesign |
| Daemon mode (persistent state) | ~100+ files (Zapcc touched ~200) | High — CaaS project is making progress but scope is narrower |
| Intra-TU parallelism | ~300+ files in `clang/Sema/` | Very high — Sema's threading model is deeply embedded |
| LLVM IR determinism fixes | ~50+ files in `llvm/lib/IR/`, `llvm/lib/Bitcode/` | Medium — potentially upstreamable; benefits reproducible builds in general |
| JIT + runtime patching | ~50 new files | Low — ORC provides most primitives; integration with content store is the gap |
| P2P distributed store | ~30 new files | Low — purely additive |
| **Total** | ~1,000+ files modified in a ~3M LoC codebase | |

**The fork maintenance challenge**: LLVM/Clang receives ~1,000 commits per week.
A fork touching 1,000+ files would diverge from upstream within months, making it
increasingly difficult to pull in bug fixes, new platform support, and C++ standard
updates. Zapcc's experience illustrates this — it was a well-executed fork by an
experienced team, and it still became impractical to maintain.

**The upstreaming challenge**: We would genuinely prefer to contribute these
features to LLVM. However, the changes forgecc needs are architectural (AST
allocation model, Sema threading, IR determinism), not incremental. Proposing
changes of this scope to Clang's core data structures would be a multi-year effort
requiring broad community consensus — and rightly so, since these changes would
affect every Clang user. The CaaS project's experience is informative: even with
CERN's backing and a relatively modest patch set (~62 files), upstreaming
incremental compilation support has been a years-long effort. forgecc's changes
would be an order of magnitude larger.

**What forgecc gains from a fresh start**:

| Advantage | Detail |
|---|---|
| **Pure Rust, no C++ dependency** | Eliminates the LLVM build (10–30 min), C++ FFI complexity, and mixed-language debugging. The entire compiler is one `cargo build`. |
| **Designed for memoization from day one** | Every data structure is content-hashable by construction. No retrofitting. |
| **Designed for parallelism from day one** | Rust's ownership model enforces thread safety at compile time. No global mutable state in Sema. |
| **Minimal code** | ~110K LoC for the full compiler vs. ~3M LoC for LLVM/Clang. Easier to understand, debug, and modify. |
| **No upstream dependency** | No version tracking, no merge conflicts, no waiting for upstream reviews. |
| **Subset targeting** | Only implement the C++ features UE5 actually uses. Clang must support everything — a much harder problem. |

**What forgecc gives up — and how we mitigate it**:

| Cost | Mitigation |
|---|---|
| **20+ years of Clang bug fixes** | `forge-verify` (§8b) continuously compares forgecc's output against Clang, catching corner-case divergences early. |
| **Optimization pipeline** | The hybrid path (§4.6b) lowers ForgeIR → LLVM IR for release builds, reusing LLVM's full -O2/-O3 pipeline. |
| **Platform breadth** | forgecc targets one platform (x86-64 Windows) for the PoC. The architecture supports future extension. |
| **Community and ecosystem** | Clang has hundreds of contributors, sanitizers, static analyzers, clang-tidy, clang-format. forgecc starts without these, but the content-addressed store and daemon architecture enable new tooling patterns. |
| **Standard compliance** | forgecc implements the C++ subset that UE5 uses, see §21. |

**A hybrid path could preserve the relationship with LLVM**: The design envisions
a possible `ForgeIR → LLVM IR` lowering path for release builds (§4.6b). If
pursued, forgecc would not replace LLVM but rather use it as a backend for
optimized builds — similar to how Swift (SIL → LLVM IR) and Rust (MIR → LLVM IR)
leverage LLVM today. In that scenario, the clean break would only be in the
front-end and the development-mode pipeline; improvements to LLVM's codegen or
optimization passes would benefit forgecc's release builds automatically. Whether
and when this path is implemented depends on how the project evolves.

---

## 3. High-Level Architecture

```
                                                          UBT / ninja
                                                          VS Code (DAP)
                                                          Peer daemons
                                                               │
                                                          HTTP (msgpack/json)
                                                               │
┌──────────────────────────────────────────────────────────────▼───────┐
│                           forgecc daemon                             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  HTTP Server (axum) — unified endpoint :9473                 │    │
│  │  /v1/compile, /v1/artifact/*, /v1/peers, /v1/debug/*         │    │
│  └──────────┬──────────────┬──────────────────┬─────────────────┘    │
│             │              │                  │                      │
│  ┌──────────▼──┐  ┌───────▼────────┐  ┌──────▼───────────────┐       │
│  │  Build      │  │  P2P Network   │  │  Content-Addressed   │       │
│  │  Scheduler  │──│  (seeds,       │──│  Distributed Store   │       │
│  │             │  │   IPFS-like)   │  │  (local + remote)    │       │
│  └──────┬──────┘  └────────────────┘  └──────────┬───────────┘       │
│         │                                        │                   │
│         ┌────────────────────────────┐           │                   │
│         │   Compilation Pipeline     │           │                   │
│         │   (parallel throughout)    │           │                   │
│         │                            │           │                   │
│         │  ┌─────┐  ┌──────────┐     │           │                   │
│         │  │Lexer│─>│Preproc-  │     │           │                   │
│         │  │     │  │essor     │     │           │                   │
│         │  └─────┘  └────┬─────┘     │           │                   │
│         │                │           │           │                   │
│         │         ┌──────▼──────┐    │           │                   │
│         │         │   Parser    │    │           │                   │
│         │         └──────┬──────┘    │           │                   │
│         │                │           │           │                   │
│         │    ┌───────────▼────────┐  │           │                   │
│         │    │  Sema (modular)    │  │  ┌──────┐ │                   │
│         │    │ ┌──────┐┌───────┐  │  │  │Memo- │ │                   │
│         │    │ │Lookup││Overld.│  │  │  │izer  │<┘                   │
│         │    │ ├──────┤├───────┤  │──┼─>│      │                     │
│         │    │ │Templ.││ Types │  │  │  └──────┘                     │
│         │    │ ├──────┤├───────┤  │  │                               │
│         │    │ │Layout││Mangle │  │  │                               │
│         │    │ └──────┘└───────┘  │  │                               │
│         │    └───────────┬────────┘  │                               │
│         │                │           │                               │
│         │         ┌──────▼──────┐    │                               │
│         │         │  IR Lower   │    │                               │
│         │         │  (ForgeIR)  │    │                               │
│         │         └──────┬──────┘    │                               │
│         │                │           │                               │
│         │         ┌──────▼──────┐    │                               │
│         │         │  x86-64     │    │                               │
│         │         │  CodeGen    │    │                               │
│         │         └──────┬──────┘    │                               │
│         │                │           │                               │
│         └────────────────┼───────────┘                               │
│                          │                                           │
│              ┌───────────┼───────────┐                               │
│              │           │           │                               │
│       ┌──────▼──────┐ ┌─▼────────┐ ┌▼───────────┐                    │
│       │   Linker    │ │ Runtime  │ │    DAP     │                    │
│       │ (release    │ │ Loader / │ │  Debugger  │<── VS Code         │
│       │  mode only) │ │ JIT      │ │  Server    │                    │
│       └─────────────┘ └──────────┘ └────────────┘                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Component Design

### 4.1 The Distributed Content-Addressed Store (IPFS-like)

This is the heart of **forgecc**. Every compilation artifact is stored as a record keyed by
the hash of its inputs. Storage is distributed across all daemon instances in the
network — there is no central server.

```
Key:   blake3(input_content + compiler_flags + dependency_hashes)
Value: the artifact (tokens, AST fragment, IR, machine code section, etc.)
```

**Local storage**: An embedded key-value store per machine. Candidates:
- [redb](https://github.com/cberner/redb) — pure Rust, ACID, MVCC, zero-copy reads

**Distributed layer (IPFS-like)**:
- Each daemon maintains a local store and a **distributed hash table (DHT)** index
- When a daemon produces an artifact, it stores it locally and announces the key to peers
- When a daemon needs an artifact, it checks: local store → DHT → compile locally
- Data is fetched on demand from the peer that has it (content-addressed, so any peer
  with the same key has the same data)
- **Seeds**: Daemons bootstrap into the network via a seed list (IP:port pairs).
  No mDNS, no central discovery server. Seeds can be configured in a config file
  or environment variable.
- **No centralized storage**: Each machine stores what it compiles. Popular artifacts
  (e.g., frequently-included headers) naturally replicate across machines as they're
  requested.

**Record types stored**:

| Record Type | Key Inputs | Value |
|---|---|---|
| `SourceFile` | file path + content hash | raw source bytes |
| `TokenStream` | source hash + predefines | serialized token stream |
| `PreprocResult` | token hash + include graph hash | preprocessed token stream |
| `ASTFragment` | preproc hash + parse context | serialized AST |
| `TypeChecked` | AST hash + type context | type-annotated AST |
| `ForgeIR` | sema hash + codegen options | IR for a function/global |
| `MachineCode` | IR hash + target options | x86-64 bytes + relocations |
| `Section` | machine code hash + section name | COFF section content |
| `DebugInfo` | sema hash + debug options | Source maps, type info, frame layouts (internal format for DAP; serialized to CodeView only for release builds) |

### 4.2 The Memoizer

The memoizer wraps every pipeline stage. Pseudocode:

```rust
fn memoized<I: Hash, O: Serialize + DeserializeOwned>(
    store: &ContentStore,
    stage: Stage,
    input: &I,
    compute: impl FnOnce(&I) -> O,
) -> O {
    let key = Key::new(stage, input.content_hash());
    // Check local store first, then distributed peers
    if let Some(cached) = store.get(&key) {
        return cached.deserialize();
    }
    let result = compute(input);
    store.put(&key, &result.serialize());
    result
}
```

**The fundamental memoization invariant**: For memoization to be worthwhile at any
granularity level, the cost of computing and validating the cache key (the "signature")
must be **significantly less** than the cost of executing the computation it guards.
If checking the signature takes as long as just re-doing the work, the cache is
overhead, not optimization. This invariant must hold not just on average but in the
common case — the "change one file, rebuild" inner loop that developers hit hundreds
of times per day. Every design decision below is evaluated against this invariant.

**Granularity levels** (from coarse to fine):

1. **Translation unit level**: Hash(source + all includes + flags) → full compiled output.
   This is what ccache/sccache do. **forgecc** does this as the outermost cache.

2. **Header level**: Hash(header content + its transitive includes) → preprocessed + parsed
   result. This **replaces PCH entirely**. Every header is automatically cached at the
   granularity the compiler determines is optimal. Users never manage PCH or unity builds.

3. **Function level**: Hash(function AST + type context) → IR → machine code.
   If a header change doesn't affect a function's types, the function isn't recompiled.

4. **Section level**: Hash(machine code + relocations) → COFF section.
   Identical sections across TUs are stored once (COMDAT-like deduplication for free).

**Why this replaces PCH**: In traditional compilers, PCH is a user-managed optimization
where the developer chooses which headers to precompile. This is fragile (wrong PCH
invalidates everything) and requires manual maintenance. **forgecc**'s approach:
- Every header's parsed/type-checked result is cached automatically
- The cache key includes the exact macro state at the point of inclusion
- Headers included with the same macro state share a single cached result
- No `SharedPCHHeaderFile`, no `/Yu`/`/Yc` flags, no unity builds

#### Fine-Grained Dependency Tracking

This is one of the most impactful optimizations **forgecc** can implement, and it is
something no mainstream C++ compiler does today. The idea: after the first parse of a
file, record a **dependency signature** — the minimal set of external inputs that
actually affected the compilation result. On subsequent builds, check only those
dependencies instead of re-hashing the entire transitive include graph.

**What the dependency signature captures**:

- **Macro dependencies**: Which `#ifdef`/`#if defined()` macros were tested, and their
  values. If a file tests `#ifdef _DEBUG` and `#ifdef UE_BUILD_DEVELOPMENT`, only those
  two macros are dependencies — not the hundreds of other `-D` flags on the command line.

- **Symbol dependencies**: Which names were looked up from other TUs/headers, and the
  hash of their declarations. If `Foo.cpp` uses `class Bar` from `Bar.h`, the dependency
  is `hash(Bar's declaration)`, not `hash(entire Bar.h)`.

- **Type layout dependencies**: Which type layouts were queried (for `sizeof`, member
  access, vtable dispatch). If `Bar` adds a new private method that doesn't change its
  layout, `Foo.cpp` doesn't need recompilation.

- **Include dependencies**: Which headers were actually included (after `#ifdef` guards
  resolved), and the hash of their content.

**Data structure** (stored per TU in the content-addressed store):

```rust
struct DependencySignature {
    /// Macros tested and their values at point of use
    macro_deps: BTreeMap<InternedString, Option<TokenStream>>,
    /// External symbols referenced and hash of their declarations
    symbol_deps: BTreeMap<QualifiedName, Blake3Hash>,
    /// Type layouts depended on (for sizeof, member access)
    layout_deps: BTreeMap<QualifiedName, Blake3Hash>,
    /// Headers included and their content hashes
    include_deps: BTreeMap<PathBuf, Blake3Hash>,
}
```

**On rebuild**: instead of re-parsing the file, check whether any dependency in the
signature has changed. If none have, skip the file entirely — not even re-lexing is
needed. This is strictly more precise than the TU-level hash (which re-hashes all
includes and flags).

**Expected impact**: For a typical "change one .cpp file" scenario in UE5, this could
reduce the set of TUs that need any work from hundreds (due to shared headers) to just
the one file that changed. For header changes, it narrows recompilation to only TUs
that actually use the changed declarations — not all TUs that transitively include
the header.

**Implementation note**: This is a Phase 1–3 feature. The dependency signature is built
incrementally during lexing (macro deps, include deps), parsing (symbol deps), and
sema (layout deps), then stored in the content-addressed store alongside the
compilation result. Estimated cost: ~500–800 additional lines in the memoizer.

#### Template Instantiation Memoization (The Hard Problem)

Template instantiation memoization is one of the hardest problems in the entire
**forgecc** design. Unlike TU-level or header-level caching where the inputs are
relatively self-contained (source text + flags + includes), a template instantiation's
result depends on a **large, open-ended set of contextual information** — name lookup
results, overload resolution outcomes, ADL-discovered functions, concept satisfaction,
SFINAE success/failure, partial specialization selection, and more. Getting this wrong
means either **incorrect caching** (returning stale results after a change) or
**excessive invalidation** (recomputing instantiations that haven't actually changed).

##### Why this matters for UE5

UE5's template-heavy code (TArray, TMap, TSet, TSharedPtr, TFunction, TDelegate,
TVariant, TOptional, plus MSVC STL headers) means the same templates are instantiated
across hundreds of TUs with the same or similar arguments. A single `TArray<AActor*>`
instantiation produces ~2,000–5,000 lines of IR. If we can memoize it, we avoid
repeating that work across every TU that uses it. The potential savings are enormous —
but only if the signature check is fast enough.

##### Exhaustive dependency list

The result of instantiating a template `T<Args...>` depends on all of the following:

**A. The template definition**

| Dependency | Why it matters |
|---|---|
| Template body AST hash | Any change to the template source code |
| Base class template definitions (recursive) | e.g. `_Compressed_pair` inside `std::vector` |
| Default template argument expressions | `template<class T, class Alloc = allocator<T>>` |
| Member function template bodies | Each may be instantiated on demand |
| Friend declarations | Friends can inject names into enclosing namespaces |

**B. The template arguments (deep hash)**

| Dependency | Why it matters |
|---|---|
| Full type definition of each type argument | Layout, members, bases, virtual functions — not just the name |
| Nested types of arguments (`T::value_type`, `T::iterator`) | Dependent name resolution |
| Member functions of arguments | If template calls `t.foo()`, the overload set of `foo` matters |
| Base classes of arguments | Affects member lookup, conversions, concept satisfaction |
| Friend functions of arguments | Found via ADL |
| NTTP values | `array<int, 5>` — the `5` is part of the key |
| Template template argument definitions | `template<template<class> class C>` — `C`'s definition matters |

**C. Two-phase name lookup results**

This is the hardest category. C++ templates use two-phase lookup (N4950 §13.8):

*Phase 1 — non-dependent names (resolved at template definition point):*

| Dependency | Why it matters |
|---|---|
| Non-dependent name resolutions | Names that don't depend on `T` are resolved once when the template is first parsed |
| Non-dependent operator overloads | `a + b` where both types are known at definition |
| Using declarations visible at definition | `using std::swap;` |
| Namespace-scope names visible at definition | All names from enclosing/used namespaces |

*Phase 2 — dependent names (resolved at instantiation point):*

| Dependency | Why it matters |
|---|---|
| ADL-found functions for dependent calls | `swap(a, b)` where `a` is type `T` — ADL searches `T`'s associated namespaces |
| ADL-found operators for dependent expressions | `a == b`, `os << a` where `a` is dependent |
| Associated namespaces of all argument types | The set of namespaces ADL searches |
| Associated namespaces of base classes of arguments | ADL also searches base class namespaces |
| Associated namespaces of template arguments of arguments | `vector<MyClass>` → `MyClass`'s namespace is associated |

**D. Overload resolution outcomes**

Every call expression involving dependent types produces an overload resolution:

| Dependency | Why it matters |
|---|---|
| Full candidate set (all visible overloads) | Adding/removing an overload changes the winner |
| Implicit conversion sequences for each candidate | Whether `T` is convertible to parameter types |
| Concept constraints on candidates | Constrained candidates may become more/less viable |
| Partial ordering of function templates | Which function template is more specialized |
| User-defined conversion operators/constructors | `explicit`/non-`explicit` conversions |
| Deleted functions in candidate set | A deleted overload can make the call ill-formed |
| Default argument expressions | Affect viable candidate determination |

**E. Concept satisfaction and SFINAE**

| Dependency | Why it matters |
|---|---|
| Concept definition hashes | The concept body itself (e.g. `CSameAs`) |
| Atomic constraint satisfaction results | Each `requires` sub-expression evaluated against `T` |
| Nested requires-expression validity | `requires { t.foo(); }` — depends on `T::foo` existing |
| Subsumption ordering | Which constrained overload is more constrained |
| SFINAE substitution success/failure | `enable_if_t<is_integral_v<T>>` — depends on the trait |
| `void_t` detection idiom results | Whether an expression is well-formed |
| `decltype` on dependent expressions | The deduced type of `decltype(t + u)` |

**F. Specialization selection**

| Dependency | Why it matters |
|---|---|
| Set of visible partial specializations | Adding a partial spec can change which is selected |
| Set of visible explicit specializations | An explicit spec replaces the primary template entirely |
| Partial ordering of specializations | Which partial spec is "more specialized" |

**G. Implicit special member functions**

| Dependency | Why it matters |
|---|---|
| Defaulted/deleted status of `T`'s special members | Whether `T` is copyable, movable, destructible |
| Trivially copyable/destructible status | Affects `memcpy` optimization, `constexpr` eligibility |
| Aggregate status of `T` | Whether aggregate initialization applies |

**H. Instantiation-point context**

| Dependency | Why it matters |
|---|---|
| `#pragma pack` state at instantiation point | Affects layout of instantiated class templates |
| `[[no_unique_address]]` on members | Affects class layout |
| Compiler flags (`-D`, `-std`) | Feature-test macros, `__cplusplus` value |

##### Memoization strategy: phased approach

Given the complexity above, we use a **three-tier strategy** that respects the
fundamental memoization invariant (signature cost << computation cost):

**Tier 1 — Structural key (Phase 3, day one)**

```rust
struct TemplateInstantiationKey {
    /// Hash of the template definition AST (body, bases, defaults, friends)
    template_def_hash: Blake3Hash,
    /// Deep hash of all template arguments (full type definitions, not just names)
    template_args_hash: Blake3Hash,
    /// Hash of the "instantiation context": all declarations visible in the
    /// associated namespaces of the template arguments at the point of instantiation.
    /// This is the conservative over-approximation that makes ADL safe.
    instantiation_context_hash: Blake3Hash,
    /// Active #pragma pack state, relevant compiler flags
    context_flags_hash: Blake3Hash,
}
```

**Cost analysis**: Computing `template_def_hash` and `template_args_hash` is cheap —
these are hashes of AST nodes that are already in memory (or in the store) from
earlier pipeline stages. The expensive part is `instantiation_context_hash`, which
requires enumerating all declarations in the associated namespaces. However:

- Associated namespaces are typically small (1–3 namespaces for most UE5 types)
- The declarations in those namespaces are already parsed and hashed (from header
  caching) — we're combining existing hashes, not re-parsing
- We can cache the "namespace content hash" per namespace and invalidate it only
  when a declaration is added/removed/changed in that namespace

**Estimated signature cost**: ~1–5 μs per instantiation (hash lookups + combine).
**Estimated instantiation cost**: ~50–500 μs for a typical class template (type
checking all members, overload resolution, codegen). **Ratio: 50–500x**, well within
the memoization invariant.

**Trade-off**: This over-invalidates. Adding an unrelated function to `namespace game`
invalidates all instantiations whose arguments have `game` as an associated namespace.
In practice this is acceptable because (a) most UE5 types are in the global namespace
or a small set of UE namespaces, and (b) namespace-level changes are relatively rare
compared to function-body changes.

**Tier 2 — Recorded dependency signature (Phase 7+, optimization)**

During the first instantiation, **instrument** every lookup, overload resolution, and
concept check to record exactly which external facts were consulted:

```rust
struct TemplateInstantiationSignature {
    /// Tier 1 key (for initial lookup)
    key: TemplateInstantiationKey,

    /// Phase 1 (definition-time) name resolutions — shared across all instantiations
    /// of the same template, computed once when the template is first parsed
    nondependent_lookups: BTreeMap<QualifiedName, Blake3Hash>,

    /// Phase 2 (instantiation-time) dependent name resolutions
    dependent_name_resolutions: BTreeMap<QualifiedName, Blake3Hash>,

    /// ADL results: (unqualified function name, argument type hashes) → winning candidate hash
    adl_results: BTreeMap<(InternedString, Vec<Blake3Hash>), Blake3Hash>,

    /// Overload resolution outcomes: call site → selected candidate hash
    overload_winners: BTreeMap<CallSiteId, Blake3Hash>,

    /// Concept satisfaction: (concept hash, argument hashes) → satisfied?
    concept_results: BTreeMap<(Blake3Hash, Vec<Blake3Hash>), bool>,

    /// SFINAE: substitution site → success/failure
    sfinae_results: BTreeMap<SubstitutionSiteId, bool>,

    /// Specialization selection: which partial/explicit spec was chosen
    specialization_choices: BTreeMap<Blake3Hash, SpecializationChoice>,

    /// Type properties queried: sizeof, alignof, is_trivially_copyable, etc.
    type_properties: BTreeMap<Blake3Hash, TypePropertySet>,
}
```

**Cost analysis**: On the first instantiation, recording these dependencies adds
~10–20% overhead (an extra hash per lookup/resolution). On subsequent builds, the
**validation** cost is: iterate the recorded dependencies, check each hash against
the current value. This is O(number of recorded dependencies), typically 10–100
lookups for a class template instantiation.

**Estimated validation cost**: ~5–20 μs (10–100 hash comparisons).
**Estimated re-instantiation cost**: ~50–500 μs.
**Ratio: 5–50x**, still well within the invariant, and with far fewer false
invalidations than Tier 1.

**Key insight**: The Tier 2 signature is only computed once (during the first
instantiation) and then stored alongside the cached result. On subsequent builds,
we only *validate* it (check if any recorded dependency changed), which is much
cheaper than computing it from scratch. This is analogous to how build systems
record file dependencies during compilation and check mtimes on rebuild — the
recording happens as a side effect of the work, not as a separate pass.

**Tier 3 — Namespace content hash caching (cross-cutting optimization)**

The most expensive part of Tier 1's `instantiation_context_hash` is enumerating
associated namespaces. We optimize this with a **per-namespace content hash** that
is maintained incrementally:

```rust
struct NamespaceContentHash {
    /// Hash of all declaration signatures in this namespace
    decl_hash: Blake3Hash,
    /// Monotonic version counter — bumped on any add/remove/change
    version: u64,
}
```

When a header is re-parsed and a namespace gains/loses/changes a declaration, the
namespace's content hash is updated. Template instantiation keys that reference this
namespace can do a cheap version check (`version == cached_version`) before falling
back to full hash comparison. This makes the common case (nothing changed in the
namespace) essentially free — a single integer comparison.

##### ADL: the specific hard case

ADL is the single most dangerous dependency for memoization correctness. Consider:

```cpp
namespace game {
    struct Actor { /* ... */ };
}

// In TU A (compiled first):
template<class T> void process(T& x) { serialize(x); }  // ADL finds game::serialize
void foo() { game::Actor a; process(a); }

// Later, someone adds to game namespace:
namespace game {
    void serialize(Actor& a) { /* new implementation */ }
}
```

Adding `game::serialize` changes the ADL result for `process<game::Actor>`. The
memoized instantiation of `process<game::Actor>` must be invalidated.

**Tier 1 handles this correctly** because `instantiation_context_hash` includes all
declarations in `namespace game`. Adding `serialize` changes the hash.

**Tier 2 handles this precisely** because `adl_results` records that the call to
`serialize(x)` resolved to a specific candidate (or failed). On rebuild, we check
whether the same unqualified lookup with the same argument types still produces the
same winner.

**Worst case for ADL**: Types in the global namespace. The global namespace can
contain thousands of declarations, making the namespace content hash expensive to
compute. Mitigation: for the global namespace specifically, we partition the content
hash by declaration name prefix or use a Merkle tree structure so that adding one
function only invalidates the relevant subtree.

##### Signature cost budget

To maintain the memoization invariant, we enforce these cost budgets:

| Granularity | Max signature cost | Typical computation cost | Min ratio |
|---|---|---|---|
| TU level | ~100 μs (hash source + includes) | ~100–500 ms (full compile) | 1000x |
| Header level | ~10–50 μs (hash content + macro state) | ~1–10 ms (parse + type-check) | 50x |
| Function level | ~1–5 μs (hash AST + type context) | ~10–100 μs (IR + codegen) | 10x |
| Template instantiation (Tier 1) | ~1–5 μs (combine existing hashes) | ~50–500 μs (full instantiation) | 50x |
| Template instantiation (Tier 2) | ~5–20 μs (validate recorded deps) | ~50–500 μs (full instantiation) | 5x |

If profiling shows a granularity level violating its ratio, we either (a) make the
signature cheaper (coarser hash, fewer deps checked), or (b) drop to a coarser
granularity level for that case. The system must **never** spend more time checking
the cache than it would spend just doing the work.

**Escape hatch**: For pathological cases (deeply nested template instantiations,
templates with hundreds of dependent calls, types in the global namespace with
thousands of declarations), the memoizer falls back to Tier 1 or even TU-level
caching. The instrumentation in Tier 2 can detect when the dependency set is
growing too large and abort recording, falling back to the structural key.

##### Implementation plan

- **Phase 3** (Sema): Implement Tier 1 (structural key). This is sufficient for
  correctness and provides significant speedup for the common case.
- **Phase 7+** (Optimization): Implement Tier 2 (recorded signatures) for the
  most-instantiated templates. Profile first to identify which templates benefit most.
- **Ongoing**: Implement Tier 3 (namespace content hash caching) as a cross-cutting
  optimization that benefits all tiers.

Estimated cost: ~1,500 lines for Tier 1, ~2,500 lines for Tier 2, ~500 lines for
Tier 3. Total: ~4,500 lines across `forge-sema` and `forge-store`.

#### Forward Memoization vs. Query-Based Incrementality (salsa-rs)

forgecc uses **forward (push-based) memoization**: the pipeline runs stages in
order (lex → preprocess → parse → sema → IR → codegen), each stage hashes its
inputs, checks the content-addressed store, and either returns a cached result or
computes and stores a new one. This is a fundamentally different model from
**query-based (pull-based) incrementality** as implemented by
[salsa-rs](https://github.com/salsa-rs/salsa), which powers rust-analyzer.

**How salsa works**: The program is expressed as a graph of tracked functions
(queries). Each query declares its dependencies implicitly — salsa records which
other queries it called during execution. When an input changes, salsa walks the
dependency graph top-down using a "red-green" algorithm: it marks potentially
affected queries as "red" (may need re-execution), then verifies each by checking
whether its recorded dependencies actually changed. If a query's inputs are
unchanged (or changed but produced the same output — "backdating"), the query
stays "green" and its cached result is reused. Execution is demand-driven: nothing
runs until a root query is requested.

| | Forward Memoization (forgecc) | Query-Based (salsa-rs) |
|---|---|---|
| **Execution model** | Push: pipeline runs stages in fixed order | Pull: queries execute on demand, top-down |
| **Dependency tracking** | Explicit: each stage declares its input hash upfront | Implicit: salsa records dependencies during execution |
| **Granularity control** | Manual: we choose granularity levels (TU, header, function, template) | Automatic: every tracked function is a memoization point |
| **Cache key** | Content hash of inputs (blake3) | Revision counter + dependency graph walk |
| **Invalidation** | Content-based: if the hash matches, the result is valid regardless of history | Revision-based: if any transitive input's revision changed, re-verify |
| **Determinism** | Guaranteed: same inputs → same hash → same result, always | Guaranteed within a session; cross-session requires careful input identity |
| **Distribution** | Natural: content hashes are globally unique, any peer can serve a cached result | Hard: revision numbers are local to a single database instance |
| **Persistence** | Natural: content-addressed store is inherently persistent and shareable | Possible but not native; salsa's database is in-memory, persistence requires serialization |
| **Parallelism** | Coarse: stages run in parallel across TUs; within a TU, pipeline is sequential | Fine: salsa supports concurrent query execution with cycle detection |
| **Overhead per cache check** | ~1–5 μs (hash lookup in store) | ~0.1–1 μs (revision comparison, no hashing) |
| **Overhead per cache miss** | Hash computation + store write (~5–20 μs) | Execution + dependency recording (~1–5 μs recording overhead) |
| **False re-executions** | Low: content-based — if the output bytes are identical, it's a hit | Lower: backdating means even if an input changed, if the output didn't, downstream stays green |
| **Incremental precision** | Depends on granularity tier chosen (Tier 1 over-invalidates, Tier 2 is precise) | High: automatic fine-grained tracking of every query dependency |
| **Code complexity** | Lower: each stage is a standalone function with explicit hash-in/hash-out | Higher: requires structuring the entire program as tracked queries with a database |
| **Bootstrapping cost** | Low: works with any function signature | Moderate: requires salsa macros, database setup, tracked struct definitions |

**Why forgecc uses forward memoization instead of salsa**:

1. **Distribution is the killer feature.** Content hashes are globally meaningful —
   if machine A computes `blake3(inputs) → result`, machine B can use that result
   without knowing anything about machine A's revision history. salsa's revision
   numbers are local to a single database instance. Making salsa distributed would
   require mapping local revisions to content hashes anyway, at which point we've
   rebuilt forward memoization on top of salsa.

2. **Persistence is free.** The content-addressed store is inherently persistent —
   shut down the daemon, restart it, and all cached results are still valid because
   they're keyed by content, not by ephemeral revision numbers. salsa's database
   is in-memory; persisting it requires serializing the entire dependency graph and
   re-validating it on reload (since file contents may have changed while the
   process was down).

3. **C++ compilation is coarse-grained enough.** salsa shines when the computation
   graph has thousands of fine-grained interdependent queries (like rust-analyzer's
   type inference, where changing one function signature can ripple through hundreds
   of call sites). C++ compilation is more pipeline-shaped: lex → preprocess →
   parse → sema → IR → codegen. The natural memoization points (TU, header,
   function, template instantiation) are well-defined and relatively few per TU.
   The overhead of salsa's automatic dependency tracking doesn't pay for itself
   when the granularity is already coarse.

4. **Backdating is less valuable for C++.** salsa's backdating (detecting that a
   re-executed query produced the same output as before) is powerful for IDE
   scenarios where the user is typing and most keystrokes don't change the semantic
   result. forgecc targets batch compilation where inputs change between builds,
   not keystroke-by-keystroke. Our content-based approach achieves the same effect:
   if the output hash is the same, downstream stages see no change.

5. **Simpler mental model.** Each pipeline stage is a pure function:
   `hash(inputs) → check store → compute or return cached`. There is no global
   dependency graph to reason about, no cycle detection, no revision bookkeeping.
   This makes the system easier to debug, profile, and extend.

**Where salsa would win**: If forgecc ever adds an IDE mode (language server),
salsa's demand-driven model with fine-grained invalidation would be superior for
interactive use cases. The architecture does not preclude adding a salsa-based
layer on top of the content-addressed store for IDE features — salsa queries could
use the store as their backing cache, getting the best of both worlds.

**Cost comparison**: Forward memoization adds ~500–800 lines to the memoizer
(§4.2). A salsa-based approach would require restructuring the entire pipeline
into tracked queries (~2,000–3,000 lines of boilerplate) plus the salsa dependency
itself. The forward approach is smaller, simpler, and directly supports the
distributed use case that is core to forgecc's value proposition.

### 4.3 Lexer & Preprocessor

The lexer and preprocessor are tightly coupled in C++ because the preprocessor operates
on tokens, and macro expansion can produce new tokens.

**Key design decisions**:

- **Lazy lexing**: Lex on demand as the preprocessor or parser requests tokens.
- **Phases of translation (N4950 §5.2)**: The lexer implements the standard's phases:
  - **Phase 1**: UTF-8 input decoding, CR/CRLF → LF normalization, BOM (`U+FEFF`)
    stripping at start of file
  - **Phase 2**: Backslash-newline line splicing (must be reverted inside raw string
    literals per the standard)
  - **Phase 3**: Decomposition into preprocessing tokens and whitespace
  - **Phase 5–6**: Adjacent string literal concatenation with encoding prefix
    compatibility checking (e.g., `u8"a" "b"` → `u8"ab"`, but `u"a" U"b"` is
    ill-formed). This includes user-defined string literal concatenation where
    `ud-suffix`es must match.
- **Include-once optimization**: Track `#pragma once` and include guards. If header
  content hash unchanged, skip re-lexing entirely.
- **Automatic header caching**: Any header's **fully parsed and type-checked result**
  is cached in the store — not just the preprocessed token stream, but the complete
  AST and type information. This is closer to an "automatic serialized AST per header"
  than traditional PCH. The key difference from PCH: the cache is per-header (not
  per-project), keyed by content + macro state, and requires zero user configuration.

**Literal support** (N4950 §5.13):

- **Integer literals**: decimal, octal, hex, binary; digit separators (`1'000'000`);
  suffixes `u`/`U`, `l`/`L`, `ll`/`LL`, `z`/`Z` (C++23 `size_t` suffix — needed
  for MSVC STL headers)
- **Floating-point literals**: decimal and **hexadecimal** (`0xC.68p+2`); digit
  separators; suffixes `f`/`F`, `l`/`L`; extended float suffixes (`f16`, `f32`,
  `f64`, `f128`, `bf16`) recognized but treated as conditionally-unsupported
- **Character literals**: all encoding prefixes (`u8`, `u`, `U`, `L`); escape
  sequences (simple, octal, hex, `\u`, `\U`, `\N{name}`, `\o{}`/`\x{}`);
  multicharacter literals
- **String literals**: all encoding prefixes; **raw string literals** (`R"delim(...)delim"`)
  with special lexer handling (line splicing reverted, delimiter matching); adjacent
  string concatenation (phase 5–6)
- **User-defined literals** (N4950 §5.13.8): `ud-suffix` recognition for integer,
  floating-point, string, and character literals (e.g., `123_km`, `"hello"s`,
  `100ms`). The lexer produces a `UserDefinedLiteral` token; Sema resolves it as a
  call to `operator""` (see §4.5 overload module)
- **Boolean literals**: `true`, `false`
- **Pointer literal**: `nullptr`

**Preprocessor features needed** (based on UE5/Game codebase analysis + N4950 §15):

- `#include` / `#include_next`
- `#define` / `#undef` (object-like and function-like macros, variadic)
- `#if` / `#ifdef` / `#ifndef` / `#elif` / `#elifdef` / `#elifndef` / `#else` / `#endif`
- `#pragma once`, `#pragma push_macro`, `#pragma pop_macro`
- `#pragma comment(lib, ...)`, `#pragma comment(linker, ...)`
- `#pragma pack(push, N)`, `#pragma pack(pop)`, `#pragma pack(N)` — **critical** for
  MSVC struct layout compatibility; heavily used in UE5 and MSVC STL headers
- `#pragma warning(push)`, `#pragma warning(pop)`, `#pragma warning(disable: N)` —
  heavily used in UE5 and MSVC STL headers to suppress warnings
- `#line` (N4950 §15.7) — used by code generators including UHT-generated `.gen.cpp`
- `#error` (N4950 §15.8) — used in UE5 headers for platform/config validation
- `#warning` — non-standard but supported by MSVC and clang-cl
- Null directive `#` (bare `#` on a line, N4950 §15.10)
- `__has_include`, `__has_cpp_attribute`
- `__has_builtin` — clang extension, queried by MSVC STL and UE5 headers to detect
  compiler built-in availability (e.g., `__has_builtin(__builtin_addressof)`)
- `__has_feature`, `__has_extension` — clang extensions, queried by third-party headers
  and some UE5 code for feature detection
- `_Pragma`
- `__VA_ARGS__`, `__VA_OPT__`
- `__FILE__`, `__LINE__`, `__COUNTER__`, `__FUNCTION__`, `__FUNCSIG__`
- MSVC-specific: `__declspec(...)`, `__forceinline`, `__pragma`
- UE-specific: `UCLASS()`, `UPROPERTY()`, `GENERATED_BODY()` — standard macros, but
  the compiler must handle UHT annotations without issues

**Predefined macros** (N4950 §15.11 + MSVC/clang-cl extensions):

- Standard: `__cplusplus` (must be `202002L` for C++20), `__STDC_HOSTED__`,
  `__STDCPP_DEFAULT_NEW_ALIGNMENT__`, `__STDCPP_THREADS__`
- Feature-test macros: `__cpp_concepts`, `__cpp_consteval`, `__cpp_designated_initializers`,
  `__cpp_structured_bindings`, `__cpp_three_way_comparison`, `__cpp_if_constexpr`,
  `__cpp_fold_expressions`, `__cpp_deduction_guides`, etc. — queried by MSVC STL
  headers via `#if __cpp_lib_*` / `#if __cpp_*` to enable/disable features
- MSVC-specific: `_MSC_VER`, `_MSC_FULL_VER`, `_MSVC_LANG`, `_WIN32`, `_WIN64`,
  `_M_X64`, `_M_AMD64`, `_DEBUG` / `NDEBUG`
- Clang-cl-specific: `__clang__`, `__clang_major__`, `__clang_minor__`

**C language support**: The preprocessor and lexer must handle both C and C++ modes.
Based on our analysis, UE5's Engine/Source/ThirdParty contains **4,335 .c files**
(libcurl, zlib, FreeType, Bink Audio, Rad Audio, etc.). These are third-party libraries
that must compile as C.

### 4.4 Parser

Recursive descent parser (like Clang), producing an AST. **Parsing is inherently
sequential** within a TU because C++ is context-dependent: whether a name refers to a
type or a value affects how subsequent tokens are parsed (e.g., `T * p;` is a pointer
declaration if `T` is a type, or a multiplication if `T` is a value). `typedef`,
`using`, class/struct/enum declarations, and template declarations all change the
parsing context for everything that follows. This means the parser must process
top-level declarations in order — there is no practical way to parallelize C++ parsing
within a single file. (Parallelism across TUs is unaffected.)

**C++20 features needed** (based on UE5/Game codebase analysis + N4950 cross-reference):

- Classes, structs, unions, enums (including `enum class`)
- **Bit-fields** — used in UE5 for flags, compact representations; affects parser
  (member-declarator with `:` width), Sema (layout computation), and codegen
- **Non-static data member initializers (NSDMI)** — default member initializers
  (`int x = 42;` in class body); heavily used in UE5
- Templates (class, function, variable, alias templates)
- **Template template parameters** — used in MSVC STL headers
- Template specialization (partial and full)
- **Explicit template instantiation** (`template class X<int>;`) and **`extern template`**
  (`extern template class X<int>;`) — used in UE5 Core for compile-time control
  (StringFormatter, TokenStream, SizedHeapAllocator for ANSICHAR/WIDECHAR)
- **Class template argument deduction (CTAD)** and deduction guides (N4950 §13.3.1) —
  used in MSVC STL (`std::vector v = {1, 2, 3};`, `std::pair p = {1, "hello"};`);
  requires parsing deduction-guide declarations and synthesizing deduction candidates
- **Concepts and requires-clauses** — UE5 uses these! Found in `Core/Public/Concepts/`
  (`CSameAs`, `CDerivedFrom`, `CConvertibleTo`, etc.), `MassEntity`, `CoreUObject`,
  `TypedElementFramework`, and many more. ~200+ files with concept/requires usage.
- Lambdas (including generic lambdas, init-capture)
- `constexpr` / `constinit` / `consteval`
  - `constinit` (N4950 §9.2.1) — guarantees constant initialization of variables;
    used for static initialization safety
  - `consteval` — used in D3D12RHI, VerseVM, CoreUObject (~20 files)
- Structured bindings (N4950 §9.6)
- `if constexpr` / `if consteval` (N4950 §8.5.2)
- Fold expressions
- Designated initializers
- **Aggregate initialization** including C++20 parenthesized aggregate init
  (`T(args...)` for aggregates)
- **Brace-enclosed initializer lists** / `std::initializer_list` construction —
  complex rules for narrowing conversions, brace elision, copy-list-init vs.
  direct-list-init (N4950 §9.4.5)
- Three-way comparison (`<=>`) — used in abseil, D3D12RHI
- **Defaulted comparison operators** (`= default` for `operator==` and `operator<=>`,
  N4950 §11.10) — compiler must synthesize member-wise comparison logic
- `using enum`
- `static_assert` with optional message (N4950 §9.1)
- `alignas` / `alignof` (N4950 §9.12.2) — used in UE5 for SIMD alignment
- **`extern "C"` / linkage specifications** (N4950 §9.11) — used extensively in UE5
  for C interop and DLL exports; affects name mangling and calling conventions
- **`explicit(bool)`** conditional explicit (N4950 §9.2.3) — `explicit(condition)`
  where condition is a constant expression; used in MSVC STL headers (`std::pair`,
  `std::tuple`, `std::optional` constructors)
- **`noexcept` specifier and operator** (N4950 §14.5, §7.6.2.7) — `noexcept` is part
  of the function type in C++17+; affects overload resolution and type system
- **`typeid` and RTTI** — `dynamic_cast`, `typeid` used in some UE5 paths; requires
  vtable RTTI info generation
- Attributes (`[[nodiscard]]`, `[[nodiscard("reason")]]`, `[[likely]]`,
  `[[unlikely]]`, `[[no_unique_address]]`, `[[deprecated("msg")]]`,
  `[[fallthrough]]`, `[[maybe_unused]]`) — attribute argument parsing needed for
  string-argument attributes
- **User-defined literals** — parser must recognize `ud-suffix` on literals (see §4.3)
- MSVC extensions: `__declspec`, `__forceinline`, `__int64`, SEH (`__try/__except/__finally`),
  `__uuidof` (COM interface identification, ~70 files), `__assume` (optimizer hint),
  `_alloca` (stack allocation)
- C language parsing mode (for .c files)

**NOT needed** (confirmed from UE5 codebase scan — see §15.14):
- Coroutines (`co_await`, `co_yield`, `co_return`) — **zero** occurrences in Engine
- C++20 modules (`export module`, `import`) — **zero** occurrences
- `std::ranges` / `std::views` — **zero** occurrences (UE uses custom iterators)
- `std::format` — **zero** occurrences (UE uses `FString::Printf` / `fmt`)
- `std::span` — **zero** occurrences (UE uses `TArrayView` / `FMemoryView`)
- `__fastcall` / `__vectorcall` / `__thiscall` — **zero** occurrences
- `__declspec(novtable)` / `__declspec(property)` — **zero** occurrences

### 4.5 Semantic Analysis (Modular Design)

This is the largest component. **Critical design requirement**: Sema must NOT be
monolithic. LLVM's Sema is notoriously hard to navigate and debug due to massive
source files (tens of thousands of lines). **forgecc**'s Sema is split into small,
focused modules.

**Modular Sema architecture**:

```
forge-sema/
├── src/
│   ├── lib.rs              # Public API, orchestration (~200 lines)
│   │
│   ├── context.rs          # Shared semantic context (type tables, scope stack)
│   │
│   ├── lookup/             # Name lookup (separate crate-like module)
│   │   ├── mod.rs          # Public interface
│   │   ├── unqualified.rs  # Unqualified name lookup
│   │   ├── qualified.rs    # Qualified name lookup (::)
│   │   ├── adl.rs          # Argument-dependent lookup
│   │   ├── using.rs        # Using declarations/directives
│   │   └── two_phase.rs    # Two-phase name lookup for templates (N4950 §13.8):
│   │                       #   non-dependent names resolved at definition,
│   │                       #   dependent names resolved at instantiation
│   │
│   ├── types/              # Type system
│   │   ├── mod.rs
│   │   ├── repr.rs         # Type representation (~500 lines max)
│   │   ├── equivalence.rs  # Type comparison, compatibility
│   │   ├── conversion.rs   # Implicit/explicit conversions, narrowing detection
│   │   ├── deduction.rs    # Template argument deduction, auto, CTAD
│   │   ├── qualifiers.rs   # const, volatile, ref qualifiers
│   │   └── noexcept.rs     # noexcept as part of function type (C++17+),
│   │                       #   noexcept operator evaluation (N4950 §7.6.2.7)
│   │
│   ├── overload/           # Overload resolution
│   │   ├── mod.rs
│   │   ├── candidates.rs   # Candidate set construction (including built-in
│   │   │                   #   operator candidates per N4950 §12.5)
│   │   ├── ranking.rs      # Implicit conversion ranking
│   │   ├── resolution.rs   # Best viable function selection
│   │   ├── literals.rs     # User-defined literal operator resolution:
│   │   │                   #   literal-operator-id lookup, literal operator
│   │   │                   #   templates (N4950 §12.6)
│   │   └── address_of.rs   # Address of overloaded function (N4950 §12.3)
│   │
│   ├── templates/          # Template instantiation
│   │   ├── mod.rs
│   │   ├── instantiate.rs  # On-demand instantiation with memoization
│   │   │                   #   (class, function, variable, alias templates)
│   │   ├── specialize.rs   # Partial/full specialization matching
│   │   ├── sfinae.rs       # Substitution failure handling
│   │   ├── ctad.rs         # Class template argument deduction (N4950 §13.3.1):
│   │   │                   #   deduction guide synthesis, implicit guides from
│   │   │                   #   constructors, aggregate deduction guides (C++20)
│   │   └── nttp.rs         # Non-type template parameters: class-type NTTPs
│   │                       #   (C++20 structural types), template template params
│   │
│   ├── concepts/           # C++20 concepts
│   │   ├── mod.rs
│   │   ├── satisfaction.rs # Concept satisfaction checking
│   │   └── constraints.rs  # Requires-clause evaluation
│   │
│   ├── consteval/          # Constant expression evaluator (JIT-based, see §4.6a)
│   │   ├── mod.rs          # evaluate_constexpr(expr) -> Value
│   │   ├── compiler.rs     # Drives IR lowering + codegen for consteval functions
│   │   ├── sandbox.rs      # Memory sandbox (custom allocator, UB detection)
│   │   └── builtins.rs     # Compiler built-in constexpr functions:
│   │                       #   __builtin_is_constant_evaluated() (used by UE5
│   │                       #   via UE_IF_CONSTEVAL), __builtin_addressof,
│   │                       #   __builtin_unreachable, __builtin_expect
│   │
│   ├── decl/               # Declaration processing
│   │   ├── mod.rs
│   │   ├── variables.rs    # Including constinit validation
│   │   ├── functions.rs    # Including explicit(bool), noexcept spec
│   │   ├── classes.rs      # Including bit-fields, NSDMI, aggregate detection
│   │   ├── enums.rs
│   │   ├── namespaces.rs
│   │   ├── linkage.rs      # extern "C" / linkage specifications (N4950 §9.11):
│   │   │                   #   affects name mangling and calling conventions
│   │   └── static_assert.rs # static_assert with optional message
│   │
│   ├── expr/               # Expression type-checking
│   │   ├── mod.rs
│   │   ├── binary.rs       # Including three-way comparison (<=>)
│   │   ├── unary.rs        # Including alignof, sizeof, noexcept operator, typeid
│   │   ├── call.rs
│   │   ├── member.rs
│   │   ├── cast.rs         # Including dynamic_cast (requires RTTI)
│   │   ├── lambda.rs
│   │   └── init.rs         # Brace-enclosed init lists, std::initializer_list
│   │                       #   construction, narrowing checks, brace elision
│   │
│   ├── stmt/               # Statement checking
│   │   ├── mod.rs
│   │   └── control_flow.rs
│   │
│   ├── bindings.rs         # Structured binding declarations (N4950 §9.6):
│   │                       #   tuple-like (std::tuple_size/get<>), array, and
│   │                       #   data member decomposition
│   │
│   ├── comparisons.rs      # Defaulted comparison operators (N4950 §11.10):
│   │                       #   synthesize member-wise ==, <=>, rewrite rules
│   │
│   ├── rtti.rs             # RTTI support: typeid, dynamic_cast, vtable RTTI
│   │                       #   info generation
│   │
│   ├── msvc/               # MSVC-specific
│   │   ├── mod.rs
│   │   ├── layout.rs       # MSVC class layout computation (including
│   │   │                   #   #pragma pack effects, __declspec(align),
│   │   │                   #   __declspec(empty_bases))
│   │   ├── mangle.rs       # Microsoft name mangling
│   │   ├── seh.rs          # SEH __try/__except/__finally
│   │   ├── abi.rs          # Calling conventions (__cdecl, __stdcall),
│   │   │                   #   vtable layout, COM (__uuidof)
│   │   ├── declspec.rs     # __declspec variants: dllexport/dllimport,
│   │   │                   #   noinline, thread, selectany, noreturn,
│   │   │                   #   allocate, restrict, safebuffers, code_seg
│   │   └── intrinsics.rs   # MSVC compiler intrinsics: _BitScanForward/Reverse,
│   │                       #   _InterlockedCompareExchange*, __debugbreak,
│   │                       #   _ReturnAddress, __cpuid, _byteswap_*,
│   │                       #   __assume, _alloca
│   │
│   └── access.rs           # Access control (public/private/protected/friend)
```

**Design rules for Sema**:
- No source file exceeds **1,000 lines**
- Each module has a clear, narrow responsibility
- Modules communicate through the shared `SemaContext` (type tables, scope stack)
- Each module is independently testable
- Memoization hooks at every module boundary

**Parallelism in Sema**: Sema has a **sequential spine** and **parallel fan-out**:

- **Sequential (the "spine")**: Top-level declaration signatures must be processed in
  declaration order, because each declaration can reference types and names introduced
  by earlier declarations. This is the same ordering constraint as parsing — a function
  signature can use a class declared above it, so signatures are resolved top-to-bottom.
- **Parallel (the "fan-out")**: Once all signatures in a TU are resolved, **function
  bodies** can be type-checked in parallel — each body only reads from the shared type
  table (populated during the sequential phase) and does not modify it. This is a
  read-heavy workload ideal for shared-memory multithreading (`RwLock` / `DashMap`).
  **Template instantiations** are also parallelized (each instantiation is independent
  once the template definition and arguments are known).
- The `SemaContext` uses concurrent data structures (lock-free maps for type tables,
  read-write locks for scope stacks) to support the parallel phase.

**MSVC ABI compatibility** (critical for UE5):
- Name mangling scheme (Microsoft C++ mangling, not Itanium)
- Class layout (including virtual bases, vtable pointer placement)
- Virtual function table layout and thunks
- Exception handling (SEH-based) — **22 files** in Engine/Source/Runtime use
  `__try/__except/__finally`, mostly in crash handling and D3D code
- Calling conventions (`__cdecl`, `__stdcall`, `__fastcall`, `__vectorcall`)

### 4.6a `consteval`/`constexpr` via In-Process JIT (Key Innovation)

Traditional compilers implement `consteval`/`constexpr` evaluation by building an
**AST interpreter** — a mini virtual machine that walks the AST and evaluates
expressions. Clang's implementation (`ExprConstant.cpp`) is ~17,000 lines and one
of the most complex files in the entire codebase. It must reimplement every expression,
statement, and type operation that can appear in a `constexpr` context.

**forgecc takes a radically different approach**: instead of interpreting the AST,
**compile the `consteval` function to native x86-64 code, execute it in the daemon's
own process, and capture the result.**

> **Implementation dependency**: This approach requires the full compilation pipeline
> (forge-ir → forge-codegen → JIT loader) to be operational. During Sema (Phase 3),
> `consteval` contexts are detected and recorded as deferred evaluations. The actual
> compilation and execution happens in **Phase 6**, after the JIT infrastructure from
> Phase 5 is available. The generated native code is mapped into the daemon's own
> address space and executed in-process to produce compile-time constants.

```
consteval int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

constexpr int x = factorial(10);  // needs compile-time evaluation
```

**Flow**:

```
1. Sema encounters factorial(10) in a constant expression context
       │
       ▼
2. forge-sema requests consteval compilation of factorial()
       │
       ▼
3. forge-ir: Lower factorial() to ForgeIR (same pipeline as normal code)
       │
       ▼
4. forge-codegen: Compile to x86-64 machine code (just this function + callees)
       │
       ▼
5. Map the machine code into the daemon's own address space (VirtualAlloc + PAGE_EXECUTE)
       │
       ▼
6. Call the function pointer: result = compiled_factorial(10)
       │
       ▼
7. Capture the return value (3628800)
       │
       ▼
8. Sema uses the result as a compile-time constant
```

**Why this is advantageous**:

1. **Massively reduces Sema complexity**: No need for a ~17,000-line AST interpreter.
   The consteval module becomes ~800 lines of "compile and call" orchestration instead.

2. **Correctness for free**: The compiled code behaves exactly like runtime code because
   it *is* runtime code. No risk of interpreter bugs where compile-time evaluation
   disagrees with runtime behavior.

3. **Performance**: Native execution is orders of magnitude faster than AST interpretation.
   For heavy `constexpr` computation (compile-time string processing, compile-time
   hashing, etc.), this matters.

4. **Reuses existing infrastructure**: The JIT compilation pipeline (`forge-ir` →
   `forge-codegen` → map into memory) is the same one used for the runtime loader.
   No new compilation infrastructure needed.

5. **C++20/23/26 constexpr features for free**: As the language adds more features to
   `constexpr` (dynamic allocation, virtual calls, try-catch), we don't need to update
   an interpreter — the regular compiler already handles them.

**Handling transitive calls**: `consteval` functions can call other `constexpr`
functions, use `std::` algorithms, etc. We compile the entire transitive closure of
called functions. This is the same lazy compilation model as the runtime JIT — when
a called function hasn't been compiled yet, compile it on demand.

**Undefined behavior detection**: The C++ standard requires that `constexpr` evaluation
detect UB and report it as a compile error. Native execution doesn't trap on all UB.
Solution: compile `consteval` code with **lightweight sanitizer instrumentation**:

- Signed integer overflow → overflow-checking arithmetic instructions
- Null/dangling pointer dereference → insert null checks before dereferences
- Out-of-bounds array access → insert bounds checks
- Division by zero → insert zero-check before division
- Shift by negative/too-large amount → insert range checks

This is far simpler than a full interpreter because we're just inserting a few extra
IR instructions during lowering, not reimplementing every language feature.

**Memory allocation in `constexpr`** (C++20): `new`/`delete` are allowed in `constexpr`
contexts, but all allocations must be freed before evaluation completes. Solution: use
a **sandboxed allocator** for `consteval` execution — a bump allocator that tracks all
allocations. After the function returns, verify all allocations were freed. If not,
report a compile error.

**Memoization**: `consteval` results are cached in the content-addressed store:
`Key = blake3(function_hash + argument_values) → Value = result`. If the same
`consteval` function is called with the same arguments in a different TU, the result
is reused without re-execution.

### 4.6b ForgeIR — Intermediate Representation

**forgecc uses a custom SSA-based IR (ForgeIR)**, not MLIR or LLVM IR. Here is why,
and how the alternatives compare:

| | Custom ForgeIR | MLIR-based | LLVM IR |
|---|---|---|---|
| **Simplicity** | Simpler to start | More infrastructure upfront | Moderate — well-documented but large spec |
| **Extensibility** | Must build everything | Dialects, passes, transformations for free | Fixed instruction set; extending requires forking |
| **Parallelism** | Must implement | MLIR has parallel pass infrastructure | No built-in parallel pass infrastructure |
| **Optimization** | Must build pass manager | Can leverage existing MLIR passes | Full LLVM optimization pipeline available |
| **Debugging** | Must build printer/verifier | Built-in verification, printing, testing | Mature tools: `llvm-dis`, `opt`, `llc`, verifier |
| **Code size** | Smaller | Larger (MLIR dependency) | Large (LLVM dependency — millions of LoC) |
| **Rust interop** | Native | FFI to C++ MLIR libraries (or Rust MLIR bindings) | FFI to C++ LLVM (`inkwell` / `llvm-sys` crates) |
| **Memoization** | Full control over hashing and serialization | Full control | **Hard** — non-deterministic metadata IDs, pointer identity, global numbering make content-addressing difficult |
| **Determinism** | Full control over ordering | Full control | **Difficult** — many sources of non-determinism (metadata `!0`/`!1` numbering, type uniquing, unnamed value `%0`/`%1` sequencing depend on emission order) |
| **Incremental granularity** | Function-level (our design) | Function-level (our design) | **Module-level** — LLVM's unit of compilation is a `Module`; splitting below module level is non-trivial |
| **JIT** | Must build loader | Must build loader | **LLVM ORC JIT available** — mature, handles relocations, exception tables, TLS |
| **Codegen quality (-O0)** | Our codegen (sufficient) | Our codegen (sufficient) | LLVM's codegen (higher quality but overkill for -O0) |
| **Codegen quality (-O2+)** | Not available | Not available | **Full LLVM optimization pipeline** — critical for release builds |
| **SIMD intrinsics** | Must map each intrinsic manually | Must map each intrinsic | **All x86 SIMD intrinsics already defined** in LLVM IR |
| **Build complexity** | Pure Rust, no external deps | C++ FFI, LLVM/MLIR build | C++ FFI, full LLVM build (~10–30 min) |

**Why not LLVM IR**: LLVM IR conflicts with forgecc's two core differentiators:

1. **Memoization**: LLVM IR assigns sequential numeric IDs to unnamed values
   (`%0`, `%1`, ...) and metadata nodes (`!0`, `!1`, ...). These IDs depend on
   emission order, which can vary. Our content-addressed store requires
   byte-identical output for identical inputs — LLVM IR makes this hard to
   guarantee without careful canonicalization.

2. **Fine-grained incrementality**: LLVM's unit of compilation is a `Module`.
   Memoizing at function level (which is what makes forgecc fast on incremental
   rebuilds) requires splitting/merging LLVM modules per function — this adds
   significant complexity and fights LLVM's design.

LLVM IR does give us a mature JIT (ORC), all SIMD intrinsics for free, and
production-quality codegen for release builds. We capture these benefits through
the **hybrid path** described below, without giving up control over the dev pipeline.

**Why not MLIR**: MLIR is a C++ library. Adding a C++ dependency to a Rust project
adds significant build complexity. MLIR's abstractions (dialects, passes) are
powerful but unnecessary for the PoC — our IR is simple enough that a custom
implementation is smaller and faster to iterate on.

**Hybrid path for release builds**: ForgeIR is used for the development pipeline
(where we need full control over determinism, memoization, and function-level
granularity). For release builds with -O2+ optimizations, ForgeIR lowers to
LLVM IR to leverage the full LLVM optimization pipeline. This is the same pattern
used by Swift (SIL → LLVM IR) and Rust (MIR → LLVM IR). The release-mode path:
`ForgeIR → LLVM IR → LLVM opt → LLVM codegen → .obj`. This path is deferred —
the PoC focuses on -O0 / JIT where our own codegen is sufficient.

ForgeIR uses MLIR-compatible concepts (operations, regions, blocks) so that
migration to MLIR is straightforward if the need arises post-PoC.

**ForgeIR design**:

```
; Example: int add(int a, int b) { return a + b; }
func @"?add@@YAHHH@Z"(i32 %a, i32 %b) -> i32 {
entry:
    %result = add i32 %a, %b
    ret i32 %result
}
```

**Minimal pass manager**: Only these passes for the prototype:
- **Dead code elimination** (remove unreachable basic blocks)
- **Constant folding** (evaluate constant expressions at compile time)
- **Stack slot allocation** (convert SSA to stack-based before register allocation)
- **Register allocation** (linear scan — simple and fast, good enough for -O0)

### 4.7 x86-64 Code Generation

Generates machine code from ForgeIR. Since we're targeting unoptimized code, this is
essentially a 1:1 lowering.

**SIMD intrinsics support**: This is critical. Based on codebase analysis, UE5 uses
SIMD extensively:
- **383 matches** in `UnrealMathSSE.h` alone (the core math library)
- SSE2 baseline (`<emmintrin.h>`), optional SSE4.1 (`<smmintrin.h>`), AVX/AVX2 (`<immintrin.h>`)
- Key types: `__m128`, `__m128d`, `__m128i`, `__m256`, `__m256d`
- Key intrinsics: `_mm_add_ps`, `_mm_mul_ps`, `_mm_shuffle_ps`, `_mm_castps_si128`, etc.
- UE5 wraps these in `VectorRegister4Float`, `VectorRegister4Double` abstractions

**Strategy**: Map each SIMD intrinsic to a ForgeIR intrinsic operation, then lower
directly to the corresponding x86-64 instruction. No optimization of SIMD code —
just faithful 1:1 translation.

**Inline assembly (`__asm`)**: Based on analysis, only **~23 files** use `__asm` in
Engine/Source/Runtime, mostly in:
- Audio codecs (Bink Audio, Rad Audio) — third-party, likely can be linked as prebuilt
- Platform detection (`cpux86.cpp`)
- Floating point state manipulation
- LongJump (AutoRTFM)

**Strategy**: Support basic `__asm` blocks for x86-64. For the PoC, if specific `__asm`
blocks are too complex, emit a call to a precompiled stub.

**Output**: Not .obj files, but database records:
- `Section` records (`.text`, `.data`, `.bss`, `.rdata`, `.pdata`, `.xdata`)
- `Relocation` records
- `Symbol` records (mangled names, visibility)
- `DebugInfo` records (source maps, type info, frame layouts — kept in internal format
  for DAP; serialized to CodeView only in release mode)

### 4.8 Runtime Loader / JIT Execution

**This is a key innovation.** Instead of the traditional compile → link → run cycle,
**forgecc**'s daemon acts as an OS loader:

1. **Process creation**: The daemon creates a suspended process (or uses itself as host)
2. **Section mapping**: Maps compiled sections into the target process's address space
3. **Lazy compilation**: Functions that haven't been compiled yet get a **stub** that
   traps back to the daemon, triggers compilation, patches the call site, and resumes
4. **Incremental patching**: When source changes, only affected functions are recompiled
   and patched in the running process

This is conceptually similar to **LLVM ORC JIT** but integrated with the compiler.
Prior art: the [in-process clang branch](https://github.com/aganea/llvm-project/tree/feat/clang_build_multiple_files_in_process)
explored compiling multiple files in-process within a single clang invocation, and
encountered related challenges (TLS via `LdrpHandleTlsData`, global state cleanup,
etc.) that inform this design.

```
┌─────────────────────────────────────────────────┐
│                 Target Process                  │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ main()   │  │ Foo::bar │  │ stub: Baz()  │   │
│  │ compiled │  │ compiled │  │ → trap to    │   │
│  │          │──│          │──│   daemon     │   │
│  └──────────┘  └──────────┘  └───────┬──────┘   │
│                                      │          │
└──────────────────────────────────────┼──────────┘
                                       │
                    ┌──────────────────▼──────────┐
                    │        forgecc Daemon       │
                    │                             │
                    │  1. Compile Baz()           │
                    │  2. Map into target process │
                    │  3. Patch stub → real code  │
                    │  4. Resume execution        │
                    └─────────────────────────────┘
```

**Relationship to UE5's Live Coding**: UE5 already has Live Coding (via Live++) which
patches running code. However, Live Coding is limited:
- Only function body changes in .cpp files
- No header changes, no new members, no constructor/destructor changes
- No UCLASS/USTRUCT/UFUNCTION changes

**forgecc**'s approach is more fundamental — the entire application is JIT-compiled, so
**any** change can be patched (including header changes, new members, layout changes)
by recompiling and re-linking affected code.

**Debugging: DAP-first approach (no PDB generation)**

Since the daemon always reconstructs the executable in-memory and acts as the loader,
it has *perfect* knowledge of everything a debugger needs: source locations, types,
variable locations, stack frame layouts. Serializing this into PDB format and then
having a separate debugger deserialize it is redundant work.

Instead, **the daemon itself IS the debugger**. It implements the
[Debug Adapter Protocol (DAP)](https://microsoft.github.io/debug-adapter-protocol/),
allowing VS Code (or any DAP client, including Visual Studio via extensions) to debug
the JIT-compiled application directly.

**What the daemon provides as a DAP server**:
- **Breakpoints**: Patch `int3` instructions into JIT'd code at the requested source line
- **Single-stepping**: Use hardware debug registers (`DR0-DR3`) or `int3` at next source line
- **Variable inspection**: Read target process memory via `ReadProcessMemory`, interpret
  using the daemon's own type information from Sema (no CodeView serialization needed)
- **Stack unwinding**: The daemon knows every frame layout because it generated the code
- **Source mapping**: Direct — the daemon has the source map from compilation
- **Watch expressions**: Evaluate using the same consteval JIT infrastructure (§4.6a)
- **Conditional breakpoints**: Compile the condition, JIT it, evaluate at breakpoint

**What this saves vs. PDB generation**:
- No PDB writer needed (~2,000-3,000 lines eliminated)
- No CodeView serialization needed (~1,500 lines eliminated)
- No need to keep PDB synchronized with live-patched code (PDB is well-documented
  in LLVM, but the synchronization overhead with JIT-patched code is the real cost)
- Debug info stays in Sema's internal format — zero serialization overhead

**What DAP costs**:
- DAP server implementation (~1,500 lines) — well-documented JSON-over-stdio protocol
- Process debugging primitives (~1,000 lines) — `WaitForDebugEvent`, `ReadProcessMemory`,
  `WriteProcessMemory`, `SetThreadContext`, hardware breakpoints
- Stack unwinding logic (~500 lines) — simpler than PDB because we control frame layout

**Net savings**: ~1,500-2,000 lines, plus complete avoidance of PDB format complexity.

**The application requires the daemon to run**: This is a fundamental architectural
property. The target application can only start if the daemon is up, because the daemon
creates the process, maps code into it, and manages its execution. The daemon
reconstructs the executable from the content-addressed store on startup (which should
be fast due to caching). This is analogous to how a JVM or .NET CLR must be running
for managed applications to execute.

**Release mode**: For shipping builds (not day-to-day development), the daemon can
still generate standalone PE/DLL + PDB files via the linker (§4.9). But for the PoC,
the DAP-based debugging path is the primary focus.

### 4.9 Linker

The linker reads from the same content-addressed store the compiler writes to. It
**never reads .obj files from disk**. When the build system sends a link command with
`.obj` paths (e.g., `POST /v1/link`), the daemon resolves each path through the
**build index** (see §4.11) to find the corresponding content key, then retrieves
sections, symbols, and relocations from the store. In JIT mode, the daemon responds
to the build system immediately after linking completes; JIT injection into the target
process happens asynchronously and does not block the build system.

**Two modes**:

1. **JIT mode** (primary, PoC focus): No traditional linking step needed — sections are
   mapped directly into the target process by the runtime loader. Symbol resolution
   happens in-memory. The daemon acts as both linker and loader.

2. **Release mode** (secondary, for shipping builds): Generate PE executables and DLLs
   for distribution.
   - Resolve symbol references between translation units
   - COMDAT deduplication (already partially done by content-addressed store)
   - Apply relocations
   - Generate PE/DLL
   - Generate PDB (only needed for release mode — DAP handles dev debugging)
   - Handle `#pragma comment(lib, ...)` and default libraries
   - Read import libraries (.lib) for Windows SDK / third-party DLLs
   - For interop with external linkers (link.exe, lld), real `.obj` files can be
     materialized from the store on demand (future work, see §4.13)

**UE5 DLL structure**: The Game editor target builds **~190 modules** (see §15.5),
each as a separate DLL. The `*_API` macro pattern (`CORE_API`, `ENGINE_API`, etc.)
controls `__declspec(dllexport/dllimport)`. In JIT mode, we can ignore DLL boundaries
entirely and treat everything as a single address space.

**Incremental linking** (release mode): When only a few functions change:
1. Detect which sections changed (via content hash comparison)
2. If changed sections fit in their old locations, patch in place
3. Otherwise, re-layout only affected sections
4. Update PDB incrementally

**Note**: PDB generation and CodeView serialization are only needed for release mode.
During development, the DAP-based debugger (§4.8) provides debugging without PDB.
This means PDB support can be deferred to a later phase.

### 4.10 Unreal Build Tool (UBT) Integration (Command-Passing Protocol)

**No MSVC shim.** Instead, **forgecc** integrates with UBT via a direct command-passing
protocol. UBT already structures compilation as a set of `VCCompileAction` objects
(see `VCCompileAction.cs`) containing:

- Source file path
- Object file output path
- Include paths (user + system)
- Preprocessor definitions
- Force-include files
- Architecture
- PCH settings
- Compiler flags

**Protocol design**: UBT sends these structured actions to the **forgecc** daemon via
**HTTP** (see §4.13 for the unified protocol design). The same HTTP endpoint serves
both local builds (`http://localhost:9473`) and remote builds (`http://peer:9473`).
The request/response uses **MessagePack** by default (compact, fast) with **JSON**
available for debugging:

```
POST /v1/compile
Content-Type: application/msgpack
Accept: application/msgpack

// Decoded request body (shown as JSON for readability):
{
    "source_file": "Game/Source/Game/Foo.cpp",
    "output": "build/Game/Foo.obj",
    "include_paths": ["Engine/Source/Runtime/Core/Public", ...],
    "definitions": ["UE_BUILD_DEVELOPMENT=1", "WITH_EDITOR=1", ...],
    "force_includes": ["SharedPCH.Engine.h"],
    "flags": { "cpp_standard": "c++20", "exceptions": true, "rtti": true }
}

// Response:
{
    "status": "success",
    "content_key": "blake3:abc123...",
    "diagnostics": []
}
```

**Compile output — shallow .obj files**: The build system specifies an `output` path
where it expects an `.obj` file (e.g., `build/Game/Foo.obj`). When compilation
completes, the daemon: (1) stores the result in the content-addressed store,
(2) updates the **build index** (see §4.11) mapping `output` → `content_key`, and
(3) kicks off an **async background write** of a shallow `.obj` to disk. The HTTP
response is sent immediately after steps 1–2; the file write is non-blocking.

The shallow `.obj` is a valid COFF file with a proper header, but containing only a
small `.forgeid` section with metadata (source file path, flags hash, content key).
No `.text`, `.data`, or other real sections. It is a few hundred bytes and exists
solely to satisfy the build system's timestamp-based dependency tracking. The daemon's
own pipeline never reads these files back — the `content_key` in the HTTP response
(and the build index) is the canonical reference to the compilation result.

This approach is inspired by
[SN-Systems' Program Repository](https://github.com/SNSystems/llvm-project-prepo/wiki)
(Sony, 2018–2022), which proved that storing compilation results in a database while
writing minimal stamp files for build system compatibility is practical at scale.

**UBT integration uses clang-cl mode.** The Game project already uses clang-cl.
Clang-cl mode uses fewer MSVC-specific flags, has cleaner
semantics, and maps naturally to **forgecc**'s internal representation.

UBT modifications needed: A new `ForgeccToolChain.cs` that either (a) points
`CompilerPath` at `forge-cc.exe` (quick start — UBT spawns it like clang-cl), or
(b) sends `ForgeccCompileAction` objects directly to the daemon via HTTP (optimized
— bypasses process spawning). See §7.1 for details.

### 4.11 Daemon Architecture

The daemon is a long-running process that owns the local store and compilation pipeline.

```
forge-daemon
    │
    ├── HTTP Server (unified protocol — serves drivers, ninja, UBT, peers, DAP)
    │   ├── /v1/compile      — compile actions (forge-cc / clang-cl equivalent)
    │   ├── /v1/link         — link actions (forge-link / lld-link equivalent)
    │   ├── /v1/lib          — static library actions (forge-lib / llvm-lib equivalent)
    │   ├── /v1/rc           — resource compile actions (forge-rc / llvm-rc equivalent)
    │   ├── /v1/ml           — assembly actions (forge-ml / llvm-ml equivalent)
    │   ├── /v1/artifact/    — content-addressed artifact get/put (local + P2P)
    │   └── /v1/peers        — peer discovery (gossip protocol)
    ├── Build Index (output path → content key, persisted in redb)
    ├── P2P Network (seed-based discovery, DHT)
    ├── Build Scheduler (parallel task graph)
    ├── File Watcher (pre-compilation on save)
    ├── Runtime Loader (JIT execution)
    ├── Compilation Thread Pool
    └── Content-Addressed Store (local redb + distributed DHT)
```

**Build model**:

1. UBT sends compile actions to daemon via HTTP (`POST /v1/compile`)
2. Daemon checks each source file against the store:
   - Content hash unchanged? → skip entirely
   - Content hash changed? → determine what needs recompilation (fine-grained)
3. Daemon compiles changed units (parallel, using thread pool)
4. In JIT mode: patches running process
5. In release mode: links incrementally
6. Daemon returns results to UBT

**File watching**: The daemon watches source directories. When a file is saved, it
begins pre-compilation immediately, before UBT even sends the compile action. By the
time the user triggers a build, the work may already be done.

**UBT bypass (future optimization)**: Eventually, the daemon could cache UBT's
command lines and use the file watcher to detect changes, making UBT invocations
unnecessary for incremental builds. UBT would still be needed for the initial
"Makefile" generation (module discovery, dependency graph), but subsequent rebuilds
could be driven entirely by the daemon's file watcher + cached command lines.

**Parallelism model**: The daemon uses a work-stealing thread pool (rayon). Parallelism
occurs at multiple levels:
- **Across TUs**: Different translation units compile in parallel (like today)
- **Within a TU**: Top-level declarations, template instantiations, and function bodies
  can be processed in parallel
- **Across pipeline stages**: Pipelining — TU1's codegen can run while TU2 is still parsing
- **Across machines**: Via the distributed store, machines share work

**Build index**: The daemon maintains an in-memory **build index** that maps output
paths (e.g., `build/Game/Foo.obj`) to content keys in the store. This is the bridge
between the build system's file-oriented world and **forgecc**'s content-addressed
world. When a compilation completes, the daemon updates the build index and kicks off
an async background write of the shallow `.obj` to disk (non-blocking — the HTTP
response is sent immediately). When the daemon later receives a link command with
`.obj` paths on the command line, it resolves each path through the build index to
find the corresponding content key — it never reads the `.obj` files from disk.

The build index is persisted to the local redb store as a dedicated table. Since redb
is ACID, writes are safe even if the daemon crashes mid-update. On restart, the daemon
loads the build index from redb — recovery is instant, no recompilation needed. This
is essentially the daemon's equivalent of a `.ninja_log` file, but stored in the same
database as all other compilation artifacts.

**Workspace binding**: Each daemon instance is bound to a **workspace root** — a
specific directory tree (git repo, Perforce workspace, or arbitrary source tree):

```toml
# forgecc.toml
[workspace]
root = "C:/src/Game"    # or auto-detected from .git / P4CONFIG
```

All paths in the build index are stored **relative to the workspace root** (consistent
with the determinism requirements in §16). This means the build index is portable if
the workspace moves, and different developers with different checkout paths produce
identical content keys. One daemon per workspace — if you have two projects, you run
two daemons.

### 4.12 Distributed P2P Network (IPFS-like)

Multiple daemon instances form a peer-to-peer network with no central authority.

**Discovery**: Seed-based. Each daemon is configured with a list of seed peers:

```toml
# forgecc.toml
[network]
seeds = ["192.168.1.10:9473", "192.168.1.20:9473"]
listen = "0.0.0.0:9473"
```

Port 9473 is unassigned by IANA and spells "XFCC" (heX ForgeCC) on a phone keypad.
Any unassigned port works; this is configurable.

**Protocol**: The P2P protocol uses the same **unified HTTP + MessagePack/JSON**
transport described in §4.13. Every daemon exposes the same HTTP API on the same port,
serving both build system clients and peer daemons. The P2P-specific endpoints are:

- `GET /v1/artifact/{blake3_hash}` — fetch an artifact by content hash
- `PUT /v1/artifact/{blake3_hash}` — announce/store an artifact
- `GET /v1/peers` — list known peers (for gossip protocol bootstrap)
- `POST /v1/peers` — register as a peer

1. On startup, daemon connects to seeds and exchanges peer lists (gossip protocol)
2. Each daemon maintains a **distributed hash table (DHT)** mapping content keys to
   peer addresses
3. When daemon A compiles something, it announces the key to the DHT
4. When daemon B needs an artifact:
   - Check local store → found? Use it.
   - Check DHT → found on peer C? Request data from C via `GET /v1/artifact/{hash}`.
   - Not found anywhere? Compile locally, announce to DHT.

**Data transfer**: HTTP/2 over a single persistent TCP connection per peer (see §4.13
for details on HTTP version choice). Metadata (DHT announcements, peer lists) uses
MessagePack. Artifact bodies are transferred as raw binary (`application/octet-stream`)
with MessagePack metadata headers. Content-addressed, so integrity is verified by hash.
Multiple artifact fetches from the same peer run as concurrent HTTP/2 streams on one
connection.

**Consistency**: No strong consistency needed. If two machines compile the same thing
independently, they produce the same output (deterministic compilation). The DHT is
eventually consistent.

### 4.13 Unified Protocol Design (HTTP + MessagePack/JSON)

**forgecc** uses a single transport for **all** communication: build system to daemon,
daemon to daemon, and (future) console client to daemon. This is a deliberate
architectural choice — one code path, one port, one protocol.

**Transport: HTTP/2 over TCP**

All communication uses **HTTP/2** over TCP. This includes local communication
(`http://localhost:9473`) and remote communication (`http://peer:9473`).

**Why HTTP/2** (not HTTP/1.1 or HTTP/3):

HTTP/2's key feature is **multiplexing** — many concurrent request/response exchanges
over a **single TCP connection**, with no head-of-line blocking between streams. This
maps perfectly to our workload:

- **Build system → daemon**: UBT/ninja sends hundreds of compile actions in parallel.
  With HTTP/1.1, this would require hundreds of TCP connections (or serialize requests,
  killing parallelism). With HTTP/2, **one connection** handles all of them as
  concurrent streams. The daemon receives them in parallel and responds as each
  completes — out of order, no blocking.
- **Daemon → peer daemons**: When fetching artifacts from a peer, the daemon may
  request dozens concurrently. HTTP/2 multiplexes all of them on a single connection
  per peer — no connection pool management, no per-request TCP handshake overhead.
- **Connection lifecycle**: HTTP/2 connections are long-lived. The build system opens
  one connection to the daemon at startup and reuses it for the entire build session.
  Peer connections are established on first contact and kept alive for the daemon's
  lifetime.

**Why not HTTP/1.1**: HTTP/1.1 allows only one in-flight request per connection
(pipelining exists but is effectively unused — no browser or major client implements
it reliably). To get parallelism, you'd need a pool of many TCP connections. This
works but adds complexity (pool sizing, connection management) and wastes resources
(each TCP connection has kernel buffers, file descriptors, etc.).

**Why not HTTP/3 (QUIC)**: HTTP/3 runs over QUIC (UDP) and eliminates TCP-level
head-of-line blocking on lossy networks. This matters for the internet and mobile, but
on a LAN or localhost, TCP packet loss is effectively zero — QUIC's advantage
disappears. Additionally, the Rust server-side ecosystem for HTTP/3 is less mature
(`axum` doesn't support it natively yet), and QUIC adds complexity (TLS 1.3 is
mandatory, UDP may be blocked by corporate firewalls). HTTP/3 is out of scope; if
WAN-distributed builds become a use case post-PoC, it is a straightforward protocol
upgrade.

**Why TCP even for local IPC**: Windows implements a
[TCP Loopback Fast Path](https://techcommunity.microsoft.com/blog/networking/windows-tcp-loopback-fast-path-is-now-on-by-default/3023562)
(enabled by default since Windows 10 1607) that bypasses the full network stack for
`localhost` connections. Loopback TCP does a direct memory copy between send/receive
buffers — no Winsock Kernel, no AFD driver, no TCP/IP driver. In practice, localhost
TCP throughput is within ~5–10% of shared memory, while giving us a **single code
path** for local and remote communication. No named pipes, no Unix domain sockets, no
separate IPC mechanism to maintain.

**Connection model summary**:

```
UBT / ninja ──── 1 HTTP/2 connection ────▶ forge-daemon (:9473)
  (or forge-cc     (hundreds of concurrent       │
   thin drivers)    compile/link/lib streams)    │
                                               ├── 1 HTTP/2 connection ──▶ peer A (:9473)
                                               │   (concurrent artifact   
                                               │    get/put streams)      
                                               │
                                               └── 1 HTTP/2 connection ──▶ peer B (:9473)
```

**Wire format: MessagePack (default) + JSON (debug)**

Every endpoint supports two serialization formats, selected via standard HTTP content
negotiation:

| Header | MessagePack | JSON |
|---|---|---|
| `Content-Type` (request body) | `application/msgpack` | `application/json` |
| `Accept` (desired response) | `application/msgpack` | `application/json` |

**MessagePack** ([msgpack.org](https://msgpack.org)) is the default. It is a binary
format that is schema-less like JSON but:
- ~30–50% smaller on the wire
- ~5–10x faster to serialize/deserialize
- Supports binary data natively (no base64 encoding)
- Trivially interoperable with serde in Rust (via `rmp-serde`)

**JSON** is available for debugging and tooling. Any endpoint can be called with
`curl` by setting `Accept: application/json`:

```bash
# Debug: inspect a compile request as JSON
curl -H "Accept: application/json" http://localhost:9473/v1/status

# Debug: submit a compile action as JSON
curl -X POST -H "Content-Type: application/json" -H "Accept: application/json" \
     -d '{"source_file": "foo.cpp", ...}' http://localhost:9473/v1/compile
```

**Format selection rules**:
1. If `Content-Type` header is present, the daemon parses the request body in that
   format. If absent, the daemon assumes MessagePack.
2. If `Accept` header is present, the daemon responds in the requested format.
   If absent, the daemon responds in the same format as the request.
3. Artifact bodies (`GET /v1/artifact/{hash}`) are always raw binary
   (`application/octet-stream`), regardless of content negotiation — only the
   metadata envelope (headers, error responses) uses MessagePack/JSON.

**Why not gRPC / Cap'n Proto / FlatBuffers**: These require schema compilation (`.proto`
files, code generation) which adds build complexity. MessagePack + serde gives us
schema-from-code (the Rust structs ARE the schema), zero code generation, and the
JSON fallback for free. gRPC's framing also makes `curl` debugging harder — our
approach lets you `curl` any endpoint with `Accept: application/json` and get readable
output.

**Endpoint summary** (all served on the same port):

| Endpoint | Method | Purpose | Used by |
|---|---|---|---|
| `/v1/compile` | POST | Submit compile action(s) | `forge-cc.exe`, UBT, ninja |
| `/v1/link` | POST | Submit link action (.obj paths resolved via build index) | `forge-link.exe`, UBT, ninja |
| `/v1/lib` | POST | Submit static library creation action | `forge-lib.exe`, ninja |
| `/v1/rc` | POST | Submit resource compilation action | `forge-rc.exe`, ninja |
| `/v1/ml` | POST | Submit assembly action | `forge-ml.exe`, ninja |
| `/v1/status` | GET | Daemon status, build progress | Build system, CLI tools |
| `/v1/artifact/{hash}` | GET | Fetch artifact by content hash | Peers, `forge-verify` |
| `/v1/artifact/{hash}` | PUT | Store/announce artifact | Peers |
| `/v1/peers` | GET | List known peers | Peers (gossip) |
| `/v1/peers` | POST | Register as peer | Peers (gossip) |
| `/v1/debug/*` | various | DAP-over-HTTP tunnel (see §4.8) | Non-VS Code debugger clients |

**Future: real .obj materialization** (deferred, not needed for PoC): The
`/v1/artifact/{hash}?format=coff` endpoint could reconstruct a full COFF `.obj` file
from the store. This would be useful for `forge-verify` (diffing against reference
compilers), external linker interop (link.exe, lld), and inspection with
dumpbin/llvm-objdump.

**Note on DAP transport**: VS Code's standard DAP integration uses **JSON-over-stdio**
(VS Code launches the debug adapter as a child process). The daemon supports this as
its primary DAP transport — a thin `forge-dap` wrapper process connects to the
daemon's HTTP API internally and bridges to stdio for VS Code. The `/v1/debug/*` HTTP
endpoints provide an alternative for clients that prefer HTTP (e.g., custom IDE
integrations, web-based debuggers).

---

## 5. Crate Structure

```
forgecc/
├── Cargo.toml                    # Workspace root
├── crates/
│   ├── forge-daemon/             # The main daemon binary
│   │   ├── src/
│   │   │   ├── main.rs           # Entry point, daemon lifecycle
│   │   │   ├── scheduler.rs      # Build scheduler, parallel task graph
│   │   │   ├── watcher.rs        # File system watcher
│   │   │   ├── loader.rs         # Runtime loader / JIT execution
│   │   │   └── determinism.rs    # --verify-determinism mode
│   │   └── Cargo.toml
│   │
│   ├── forge-net/                # HTTP server + P2P networking
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── server.rs         # axum HTTP server (unified: UBT + peers + debug)
│   │   │   ├── protocol.rs       # MessagePack/JSON content negotiation
│   │   │   ├── peer.rs           # Peer connection management
│   │   │   ├── dht.rs            # Distributed hash table
│   │   │   ├── gossip.rs         # Peer discovery gossip protocol
│   │   │   └── transfer.rs       # Content transfer (get/put artifacts)
│   │   └── Cargo.toml
│   │
│   ├── forge-store/              # Content-addressed distributed store
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── local.rs          # Local redb store
│   │   │   ├── distributed.rs    # Distributed layer (local + DHT + remote fetch)
│   │   │   ├── keys.rs           # Key computation (BLAKE3 hashing)
│   │   │   └── records.rs        # Record type definitions & serialization
│   │   └── Cargo.toml
│   │
│   ├── forge-lex/                # Lexer (C and C++)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── token.rs          # Token types
│   │   │   ├── lexer.rs          # Tokenizer
│   │   │   └── literals.rs       # Number/string/char literal parsing
│   │   └── Cargo.toml
│   │
│   ├── forge-pp/                 # Preprocessor
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── directives.rs     # #include, #define, #if, etc.
│   │   │   ├── macros.rs         # Macro expansion engine
│   │   │   ├── include.rs        # Include resolution & automatic caching
│   │   │   └── builtins.rs       # __FILE__, __LINE__, predefined macros
│   │   └── Cargo.toml
│   │
│   ├── forge-parse/              # Parser (C and C++)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── ast.rs            # AST node definitions
│   │   │   ├── parser.rs         # Recursive descent parser (orchestration)
│   │   │   ├── expr.rs           # Expression parsing
│   │   │   ├── stmt.rs           # Statement parsing
│   │   │   ├── decl.rs           # Declaration parsing
│   │   │   ├── types.rs          # Type parsing
│   │   │   └── templates.rs      # Template parsing
│   │   └── Cargo.toml
│   │
│   ├── forge-sema/               # Semantic analysis (MODULAR — see §4.5)
│   │   ├── src/
│   │   │   ├── lib.rs            # Public API (~200 lines)
│   │   │   ├── context.rs        # Shared semantic context
│   │   │   ├── lookup/           # Name lookup (6 files, ~2500 lines total)
│   │   │   ├── types/            # Type system (6 files, ~3500 lines total)
│   │   │   ├── overload/         # Overload resolution (5 files, ~3000 lines)
│   │   │   ├── templates/        # Template instantiation (5 files, ~4000 lines)
│   │   │   ├── concepts/         # C++20 concepts (2 files, ~1000 lines)
│   │   │   ├── consteval/        # Constant evaluator (2 files, ~2000 lines)
│   │   │   ├── decl/             # Declaration processing (7 files, ~4000 lines)
│   │   │   ├── expr/             # Expression checking (7 files, ~5000 lines)
│   │   │   ├── stmt/             # Statement checking (2 files, ~500 lines)
│   │   │   ├── msvc/             # MSVC-specific (6 files, ~4000 lines)
│   │   │   ├── bindings.rs       # Structured binding decomposition (~500 lines)
│   │   │   ├── comparisons.rs    # Defaulted comparison synthesis (~500 lines)
│   │   │   ├── rtti.rs           # typeid, dynamic_cast support (~300 lines)
│   │   │   └── access.rs         # Access control (~500 lines)
│   │   └── Cargo.toml
│   │   # Total: ~55 files, ~34,500 lines, no file > 1000 lines
│   │
│   ├── forge-ir/                 # Intermediate representation
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── types.rs          # IR type system
│   │   │   ├── instructions.rs   # IR instruction set (including SIMD intrinsics)
│   │   │   ├── builder.rs        # IR construction helpers
│   │   │   ├── printer.rs        # Human-readable IR dump
│   │   │   └── passes/
│   │   │       ├── mod.rs        # Pass manager
│   │   │       ├── dce.rs        # Dead code elimination
│   │   │       └── constfold.rs  # Constant folding
│   │   └── Cargo.toml
│   │
│   ├── forge-codegen/            # x86-64 code generation
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── lower.rs          # IR → x86-64 instruction selection
│   │   │   ├── regalloc.rs       # Register allocation (linear scan)
│   │   │   ├── emit.rs           # Machine code emission
│   │   │   ├── encoding.rs       # x86-64 instruction encoding
│   │   │   ├── simd.rs           # SSE/AVX intrinsic lowering
│   │   │   └── relocations.rs    # Relocation generation
│   │   └── Cargo.toml
│   │
│   ├── forge-debug/              # DAP debugger server
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── dap_server.rs     # DAP protocol implementation (JSON-over-stdio)
│   │   │   ├── breakpoints.rs    # Breakpoint management (int3 patching)
│   │   │   ├── stepping.rs       # Single-step, step-over, step-out
│   │   │   ├── variables.rs      # Variable inspection (ReadProcessMemory + type info)
│   │   │   ├── stack.rs          # Stack unwinding (using codegen frame layout info)
│   │   │   └── process.rs        # WaitForDebugEvent, debug primitives
│   │   └── Cargo.toml
│   │
│   ├── forge-link/               # Linker (release mode only — not needed for dev builds)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── resolver.rs       # Symbol resolution
│   │   │   ├── layout.rs         # Section layout
│   │   │   ├── relocate.rs       # Relocation application
│   │   │   ├── pe.rs             # PE/COFF executable output
│   │   │   ├── dll.rs            # DLL generation (for UE5 module structure)
│   │   │   ├── pdb.rs            # PDB generation (release mode only)
│   │   │   ├── codeview.rs       # CodeView debug info (release mode only)
│   │   │   └── imports.rs        # Import library reading (.lib)
│   │   └── Cargo.toml
│   │
│   ├── forge-common/             # Shared utilities
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── diagnostics.rs    # Error/warning reporting
│   │   │   ├── source_map.rs     # Source location tracking
│   │   │   ├── intern.rs         # String interning (lock-free)
│   │   │   └── hash.rs           # BLAKE3 hashing utilities
│   │   └── Cargo.toml
│   │
│   ├── forge-driver/             # Thin driver binaries (see §7.3)
│   │   ├── src/
│   │   │   ├── main.rs           # Entry point: argv[0] detection, dispatch
│   │   │   ├── detect.rs         # Tool name detection (forge-cc, forge-link, etc.)
│   │   │   ├── args_cc.rs        # clang-cl flag parsing → CompileAction
│   │   │   ├── args_link.rs      # lld-link flag parsing → LinkAction
│   │   │   ├── args_lib.rs       # llvm-lib flag parsing → LibAction
│   │   │   ├── args_rc.rs        # llvm-rc flag parsing → RcAction
│   │   │   ├── args_ml.rs        # llvm-ml flag parsing → MlAction
│   │   │   ├── client.rs         # HTTP/2 client (reqwest) → daemon RPC
│   │   │   └── autostart.rs      # Daemon auto-start (spawn, wait for /v1/status)
│   │   └── Cargo.toml
│   │   # Build produces forge-daemon.exe + symlinks: forge-cc, forge-link,
│   │   # forge-lib, forge-rc, forge-ml
│   │
│   └── forge-verify/             # Verification CLI tool (see §8b)
│       ├── src/
│       │   ├── main.rs           # CLI entry point, argument parsing
│       │   ├── daemon_client.rs  # HTTP client to connect to forge-daemon
│       │   ├── reference.rs      # Invoke clang-cl / MSVC as reference compiler
│       │   ├── diff_sections.rs  # Compare COFF sections, symbols, relocations
│       │   ├── diff_debug.rs     # Invoke llvm-debuginfo-analyzer --compare
│       │   └── diff_exec.rs      # Link, run, compare stdout/stderr/exit code
│       └── Cargo.toml
│
└── tests/
    ├── level0_handcrafted/       # Hand-crafted per-feature tests (see §8a)
    ├── level1_stb/               # stb library compilation tests
    ├── level2_lmdb/              # LMDB compilation tests
    ├── level3_json/              # nlohmann/json compilation tests
    ├── level4_imgui/             # Dear ImGui compilation tests
    ├── level5_gtest/             # Google Test compilation tests
    ├── level6_lit/               # Selected LLVM lit tests
    └── integration/              # End-to-end build tests
```

---

## 6. Key Data Flows

### 6.1 First Build (Cold Cache)

```
Source files ──▶ Lexer ──▶ Preprocessor ──▶ Parser ──▶ Sema ──▶ IR ──▶ CodeGen
                  │            │               │         │        │        │
                  ▼            ▼               ▼         ▼        ▼        ▼
              Store:Token  Store:Preproc   Store:AST  Store:   Store:  Store:Section
              Stream       Result          Fragment   Type     IR      + Store:Debug
                                                      Checked
```

Every stage writes its output to the store before passing it to the next stage.
All stages run in parallel across TUs, and within TUs where possible.

### 6.2 Incremental Build (Warm Cache)

```
Source file changed
       │
       ▼
  Hash source ──▶ Store lookup: TokenStream? ──YES──▶ skip lexing
       │                                                   │
       NO                                                  ▼
       │                                         Store lookup: PreprocResult
       ▼                                         with same deps? ──YES──▶ skip
  Lex file ──▶ Preprocess ──▶ ... each stage checks store first
```

### 6.3 JIT Execution Flow

```
User: "Run"
       │
       ▼
  Daemon creates target process (suspended)
       │
       ▼
  Map already-compiled sections into process
       │
       ▼
  Install stubs for uncompiled functions
       │
       ▼
  Resume process
       │
       ▼
  Function called ──▶ stub traps ──▶ daemon compiles ──▶ patch & resume
```

### 6.4 Distributed Build

```
Machine A                              Machine B
┌──────────┐                          ┌──────────┐
│ Daemon A │◄── DHT sync (TCP) ─────▶│ Daemon B │
│          │                          │          │
│ Store:   │                          │ Store:   │
│ has X123 │◄── get(X123) ────────────│ needs    │
│          │─── bytes ──────────────▶│ X123     │
└──────────┘                          └──────────┘
```

### 6.5 AST Immutability

A key design goal for **forgecc**'s AST is **immutability after construction**. In
LLVM/Clang, AST nodes are mutable — Sema modifies them in place during type-checking,
template instantiation, and implicit conversion insertion. This makes the AST
difficult to share across threads and impossible to content-address reliably.

**forgecc**'s approach: once an AST node is constructed, it is **immutable**. Sema
produces new, annotated AST nodes rather than modifying existing ones. This enables:

- **Safe sharing across threads**: Immutable AST nodes can be read concurrently
  without locks, enabling the parallel Sema design (§4.5)
- **Content-addressable AST**: Since nodes don't change, their hash is stable and
  can be used as a cache key in the content-addressed store
- **Efficient caching**: Unchanged subtrees from a previous compilation can be
  reused directly — no need to deep-copy and re-verify
- **Simpler debugging**: Immutable data structures are easier to reason about and
  reproduce bugs in

Rust's ownership model naturally encourages this pattern. `Arc<AstNode>` provides
shared immutable access, and the borrow checker prevents accidental mutation. Where
Sema needs to "modify" a node (e.g., adding type annotations), it creates a new node
that references the original, forming a layered structure.

### 6.6 Team and CI: Weekly Aggregate Savings

A central insight behind **forgecc** is that **full invalidation of the build is rare**.
On a shared branch, most commits touch a small number of files. Clean rebuilds or
cache wipes are the exception. So the **daily** reality is: each developer and each
build machine only needs to compile a **delta** — the TUs that actually changed or
that depend on changed code. With fine-grained dependency signatures (§4.2), that
delta is minimal. With a **shared content-addressed store** over P2P (§4.12), each
unique (TU, input context) is compiled **once** across the entire team and CI; every
other daemon fetches the artifact by hash.

**Implication for weekly savings:** The benefit is not only "one full rebuild is 2×
faster" but "**across the whole team and all build machines, we compile much less
in total**." Aggregate compilation work in a week can drop by a large factor because
work is shared instead of repeated.

**Illustrative model (UE-style team):**

| Role | Count | Builds per day | Today: TUs recompiled per build (approx.) | Today: TU-minutes per build (approx.) |
|------|-------|----------------|------------------------------------------|---------------------------------------|
| Developer | 20 | 20 | 50–200 | 25–200 |
| Build machine (CI) | 5 | 50 | 100–500 | 50–300 |

- **Without forgecc:** Each developer does ~20 × 100 ≈ 2,000 TU-minutes/day; 20 devs
  → 40,000 TU-minutes/day. CI: 5 × 50 × 150 ≈ 37,500 TU-minutes/day. **Total ~77,500
  TU-minutes/day** → **~388k TU-minutes/week** (single-threaded equivalent).
- **With forgecc (P2P + fine-grained incremental):** Many of those "TUs" are the
  **same** (same file, same flags, same header state). The first daemon to need a
  (TU, context) compiles it; all others get a cache hit from the P2P store. If
  **20%** of the week's logical compilations are unique and **80%** are served from
  peers, total **actual** compilation is 20% of 388k ≈ **78k TU-minutes/week**.
- **Result:** Roughly **5× less aggregate compilation work** in the week. The exact
  factor depends on how aligned the team's changes are (same branch, similar -D
  flags, similar includes). Conservative estimates still yield **3–5×** reduction in
  total CPU-time spent compiling across the org.

**Takeaways:**

- **Weekly savings** are in **aggregate**: fewer CPU-hours compiling, faster feedback
  for everyone (incremental + cache hits from peers), and less load on build machines.
- **First build of the week** (or after a big merge) does more work; **subsequent
  builds** are mostly cache hits from peers. The system naturally favors the common
  case: small deltas, shared codebase.

### 6.7 Distribution and Caching Out of the Box

Today, getting **distribution + caching** for C++ builds requires assembling and
operating several pieces: a cache (sccache, ccache, Reclient, BuildBuddy), a
distributed build layer (Incredibuild, FASTBuild, Reclient), cache servers or
cluster endpoints, auth, and tuning. **forgecc** provides equivalent behavior
**out of the box** with no separate servers and no .obj shipping.

| Concern | Today | With forgecc |
|---------|--------|----------------|
| **Cache** | sccache / ccache / Reclient, separate from the compiler | Daemon **is** the cache; content-addressed store is built in (§4.1, §4.2). |
| **Distribution** | Incredibuild, FASTBuild, Reclient, distcc, etc. | P2P between daemons (§4.12); no central job scheduler. |
| **What is shared** | Whole .obj (or equivalent) per TU | Content-addressed **sections**; same hash ⇒ one logical copy across the network. |
| **Warm cache** | Dev A's cache ≠ Dev B's; CI cache often separate. | Same codebase + same inputs ⇒ same hashes ⇒ any peer can serve. |
| **Setup** | Servers, auth, config, cache sizing, invalidation. | Run the daemon; point at seeds (or same LAN). No dedicated cache servers. |
| **Network load** | 15–18 GB (e.g. UE Editor) of .obj in/out on cache miss or distribute. | Fetch only **missing** content by hash; heavy dedup (header-derived code shared). |

**Value proposition:** Teams get **shared caching and distribution** by running
**forgecc** daemons and connecting them (e.g. via seeds). No separate distribution
or cache infrastructure to deploy or maintain.

---

## 7. Build System Integration

### 7.1 Unreal Build Tool (UBT) Integration (Primary)

Two approaches, used in sequence:

**Quick start (spawn driver)**: A new `ForgeccToolChain.cs` that points
`CompilerPath` at `forge-cc.exe` (see §7.3). UBT spawns it exactly like
`clang-cl.exe` — no changes to UBT's action execution model, only the toolchain
path changes (~5 lines). This is the initial integration path.

**Optimized (direct HTTP)**: `ForgeccToolChain.cs` constructs
`ForgeccCompileAction` objects and sends them to the daemon via HTTP
(`POST /v1/compile`) directly, bypassing process spawning entirely. This
eliminates per-action process creation overhead for UE5's hundreds of compile
actions. The data sent is the same either way:

- Source file path
- Include paths (user + system + VC include paths)
- Preprocessor definitions
- Force-include files
- Language standard (C++20)
- Architecture (x64)
- PCH settings → **ignored by forgecc** (automatic caching replaces PCH)

Both approaches report results back to UBT's action graph (success/failure +
diagnostics).

### 7.2 Native Mode

For maximum performance, projects can use **forgecc**'s native build description:

```toml
# forgecc-build.toml
[project]
name = "game"
standard = "c++20"

[targets.editor]
type = "executable"
sources = ["game/Source/**/*.cpp"]
include_paths = ["unreal/Engine/Source/Runtime/Core/Public", ...]
definitions = ["UE_BUILD_DEVELOPMENT=1", "WITH_EDITOR=1"]
```

### 7.3 Thin Driver Binaries (The Forge Tool Family)

**forgecc** provides a family of thin driver binaries that present a standard
compiler/linker interface to build systems. Each binary is the **same Rust
executable**, distinguished by `argv[0]` — the same pattern clang uses for
`clang`, `clang++`, and `clang-cl` (a single binary with symlinks/copies).

```
forge-daemon.exe     ← main binary (the daemon)
forge-cc.exe         ← symlink/copy, clang-cl-compatible compiler driver
forge-link.exe       ← symlink/copy, lld-link-compatible linker driver
forge-lib.exe        ← symlink/copy, llvm-lib-compatible librarian driver
forge-rc.exe         ← symlink/copy, llvm-rc-compatible resource compiler driver
forge-ml.exe         ← symlink/copy, llvm-ml-compatible assembler driver
```

**argv[0] detection**: On startup, the binary extracts the filename from `argv[0]`,
strips `.exe`, lowercases on Windows, and matches against the known tool names. This
determines which command-line format to parse and which daemon endpoint to call.
`--driver-mode=X` overrides the name-based detection (same as clang).

**Each driver**:
1. Parses the command line in the format of its corresponding LLVM/MSVC tool
2. Connects to the local daemon (`localhost:9473`) via HTTP/2
3. POSTs the action to the appropriate `/v1/*` endpoint
4. Streams diagnostics (warnings/errors) to stderr
5. Writes output files to the paths the build system expects
6. Exits with 0 (success) or 1 (failure)

**Daemon auto-start**: If the driver cannot connect to the daemon, it starts
`forge-daemon.exe` as a background process (detached, writes PID file), waits
for it to become ready (poll `/v1/status`), then proceeds. This means developers
never need to manually start the daemon — the first build invocation starts it
automatically.

**Tool-to-endpoint mapping**:

| Driver binary | Emulates | Daemon endpoint | Flags accepted |
|---|---|---|---|
| `forge-cc.exe` | `clang-cl.exe` | `/v1/compile` | `/c`, `/Fo`, `/I`, `/D`, `-std:c++20`, etc. |
| `forge-link.exe` | `lld-link.exe` | `/v1/link` | `/OUT:`, `/DLL`, `/LIBPATH:`, `.obj` paths |
| `forge-lib.exe` | `llvm-lib.exe` | `/v1/lib` | `/OUT:`, `.obj` paths |
| `forge-rc.exe` | `llvm-rc.exe` | `/v1/rc` | `/fo`, `.rc` path |
| `forge-ml.exe` | `llvm-ml64.exe` | `/v1/ml` | `/c`, `/Fo`, `.asm` path |

**Implementation**: The driver is a single Rust crate (`forge-driver`, ~800 lines)
that compiles to one binary. The build system produces the binary as `forge-daemon.exe`
and creates symlinks/copies for each tool name (via `build.rs` or a post-build step).

### 7.4 Ninja Daemon Mode (Zero Process Spawning)

For maximum build performance, **forgecc** includes a small patch to
[ninja](https://ninja-build.org/) that eliminates process spawning entirely for
forge-family tools. When ninja detects that a build rule's command uses a forge
tool, it routes the action directly to the daemon via RPC instead of calling
`CreateProcess()`.

**Detection**: When ninja loads `build.ninja`, for each `rule`, it checks whether
the command's tool path ends with a forge-family binary name (`forge-cc.exe`,
`forge-link.exe`, `forge-lib.exe`, `forge-rc.exe`, `forge-ml.exe`). This is a
simple string suffix check on the first token of the command.

**RPC dispatch**: For matched rules, instead of spawning a process, ninja:
1. Opens a single HTTP/2 connection to the daemon (on first use, kept alive for
   the entire build session)
2. Parses the command line into the tool type + arguments
3. POSTs to the appropriate `/v1/*` endpoint as a new HTTP/2 stream
4. Streams stdout/stderr back from the response body
5. Reports success/failure to ninja's build graph as if the process had exited

**Non-forge rules**: Any rule whose tool is NOT a forge-family binary (e.g.,
`rc.exe`, `midl.exe`, `mt.exe`, custom build steps) continues to spawn processes
normally. Ninja's existing process pool handles these unchanged.

**Fallback**: If the daemon is not reachable, ninja falls back to spawning the
thin driver binary (which will auto-start the daemon). The system is always
correct, just slower without the daemon running.

**Why this matters**: A typical UE5 editor build has ~5,000+ compile actions.
Spawning 5,000 processes (even thin ones) has measurable overhead on Windows
(~1–5 ms per `CreateProcess` call = 5–25 seconds total). With ninja daemon mode,
all 5,000 actions are HTTP/2 streams on a single TCP connection — zero process
creation, zero context switches, zero file system overhead from process startup.

**Estimated ninja patch size**: ~300–500 lines of C++ in ninja's `subprocess.cc`
/ `build.cc`:
- Tool detection in rule parsing (~50 lines)
- HTTP/2 client using a lightweight C library (e.g., nghttp2 or libcurl with
  HTTP/2) (~200 lines)
- Response streaming and exit code mapping (~100 lines)
- Connection lifecycle management (~50 lines)

### 7.5 CMake Compatibility

CMake must see a real compiler executable for configuration. `forge-cc.exe`
satisfies this by identifying as clang-cl.

**Version string**: `forge-cc.exe --version` outputs:
```
clang version 17.0.0 (forgecc 0.1.0)
Target: x86_64-pc-windows-msvc
Thread model: posix
```

This causes CMake to set `CMAKE_CXX_COMPILER_ID=Clang` and
`CMAKE_CXX_SIMULATE_ID=MSVC`, inheriting all of CMake's existing clang-cl
support. No CMake changes needed.

**What CMake does during configuration** (all handled by the thin driver):

1. **`--version`** — parseable version string (see above)
2. **`try_compile`** — compiles a small test program and produces a `.obj`.
   The driver forwards to the daemon, which produces a valid COFF `.obj`.
3. **Feature detection** — compiles snippets to detect C++ standard support
   (`-std:c++20`, etc.). Works because we accept clang-cl flags.
4. **ABI detection** — compiles `CMakeCXXCompilerABI.cpp` and inspects the
   `.obj` for ABI information. Our `.obj` must be valid COFF with correct
   section names and symbol types.

**Flow**: CMake invokes `forge-cc.exe` during configure (spawning the thin
driver). The generated `build.ninja` references `forge-cc.exe` as the compiler.
When `ninja` runs the build, it detects the forge tool and switches to daemon
mode (§7.4), bypassing process spawning for all compile/link actions.

---

## 8. Phased Implementation Plan

> **Note on time estimates**: The week ranges below assume **one experienced systems
> programmer working full-time, without AI assistance**. With AI-assisted development
> (vibe coding with an AI agent), realistic speedups are roughly **2–4x** depending
> on the component:
>
> | Component type | AI leverage | Examples |
> |---|---|---|
> | Boilerplate / table-driven / well-documented formats | 2–4x faster | Lexer, x86-64 encoding tables, DAP protocol, COFF/PE, MSVC name mangling |
> | Design-heavy but with good references | 1.5–2x faster | Parser, IR design, codegen lowering, linker |
> | Subtle correctness / deep language semantics | 1–1.5x faster | Sema (templates, overload resolution, concepts), JIT patching, distributed consistency |
>
> **Adjusted total**: The ~108 manual weeks could realistically compress to
> **~36–54 weeks** for one person working full-time with AI assistance. The largest
> gains come from the lexer/preprocessor, codegen, and linker phases; the smallest
> from Sema and JIT runtime correctness work.

### Phase 0: Infrastructure (Weeks 1–4)
- [ ] Cargo workspace setup
- [ ] `forge-store`: Content-addressed local store on redb, build index table
- [ ] `forge-common`: Diagnostics, source map, string interning, hashing
- [ ] `forge-daemon`: Basic daemon lifecycle, HTTP server (axum), workspace binding
- [ ] `forge-net`: Seed-based peer discovery, basic DHT
- [ ] **CLI / HTTP compile interface**: `POST /v1/compile` and `POST /v1/link`
  endpoints accepting compile/link actions — this is the primary way to feed work
  into the daemon from day one (UBT integration comes later; during early phases,
  a simple CLI wrapper or `curl` submits actions directly)
- [ ] Determinism infrastructure: deterministic hasher, `--verify-determinism` mode

**Milestone**: Daemon starts, accepts compile actions via HTTP, peers discover each
other, store get/put works locally and across network. All data structures use
deterministic ordering.

### Phase 1: Frontend — Lexer & Preprocessor (Weeks 5–12)
- [ ] `forge-lex`: Full C/C++20 lexer including:
  - [ ] All literal types (integer, float, char, string) with all encoding prefixes
  - [ ] Raw string literals with delimiter matching and line-splice reversion
  - [ ] User-defined literal token recognition (`ud-suffix`)
  - [ ] Hexadecimal floating-point literals
  - [ ] Phase 1–2: UTF-8 decoding, BOM stripping, line splicing
  - [ ] Phase 5–6: Adjacent string literal concatenation with encoding compatibility
- [ ] `forge-pp`: Preprocessor with all required directives including:
  - [ ] `#pragma pack`, `#pragma warning` (critical for MSVC STL compatibility)
  - [ ] `#line`, `#error`, `#warning`, null directive
  - [ ] `__has_builtin`, `__has_feature`, `__has_extension`
  - [ ] All predefined macros (`__cplusplus`, `_MSC_VER`, `__cpp_*` feature-test macros)
- [ ] Automatic header caching (replaces PCH)
- [ ] Memoization of preprocessed output in the store
- [ ] Basic Windows SDK header compatibility (start testing against `<windows.h>`
  and common system headers — these are needed by every target level)

**Milestone**: Can preprocess UE5 header files correctly. Headers are automatically
cached. No PCH configuration needed. Target level I (hand-crafted tests) preprocesses
correctly.

### Phase 2: Parser + Driver Binaries (Weeks 13–20)
- [ ] `forge-parse`: Recursive descent parser for C and C++20
- [ ] AST definition covering UE5's usage patterns
- [ ] Concepts/requires-clause parsing
- [ ] `extern "C"` / linkage specification parsing
- [ ] `explicit(bool)` conditional explicit parsing
- [ ] `constinit` declaration parsing
- [ ] CTAD / deduction guide declaration parsing
- [ ] Bit-field member declarations
- [ ] `static_assert` with optional message
- [ ] `alignas` / `alignof` parsing
- [ ] `noexcept` specifier and operator parsing
- [ ] Attribute argument parsing (string arguments for `[[nodiscard("msg")]]`, etc.)
- [ ] Error recovery
- [ ] `forge-driver`: Thin driver binaries (see §7.3):
  - [ ] argv[0] detection and tool dispatch
  - [ ] clang-cl flag parsing (`forge-cc.exe`)
  - [ ] HTTP/2 client → daemon RPC
  - [ ] Daemon auto-start
  - [ ] `--version` output for CMake compatibility (see §7.5)
  - [ ] Symlink/copy generation for forge-cc, forge-link, forge-lib, forge-rc, forge-ml

**Milestone**: Can parse UE5/Game source files into an AST. Target level I parses
correctly. `forge-cc.exe --version` works with CMake; `cmake -G Ninja` generates
a valid `build.ninja` referencing forge-cc.

### Phase 3: Semantic Analysis (Weeks 21–38)
- [ ] `forge-sema`: Modular Sema (see §4.5 for structure)
- [ ] Name lookup (including two-phase lookup for templates), type checking (parallel
  where possible)
- [ ] Template instantiation with Tier 1 memoization (structural key — see §4.2)
- [ ] CTAD: deduction guide synthesis, implicit guides from constructors, aggregate
  deduction guides
- [ ] Template template parameters, class-type NTTPs (structural types)
- [ ] Overload resolution (including built-in operator candidates, user-defined literal
  operator resolution, address of overloaded function)
- [ ] Concept satisfaction checking
- [ ] `noexcept` as part of function type; `noexcept` operator evaluation
- [ ] `extern "C"` linkage: name mangling suppression, calling convention
- [ ] `explicit(bool)` conditional explicit evaluation
- [ ] `constinit` validation (must be constant-initialized)
- [ ] Structured binding decomposition (tuple-like, array, data member)
- [ ] Defaulted comparison operators (`= default` for `==` and `<=>`)
- [ ] Brace-enclosed initializer lists / `std::initializer_list` construction /
  narrowing conversion detection / aggregate initialization rules
- [ ] `static_assert` evaluation
- [ ] `alignas` / `alignof` evaluation
- [ ] RTTI support: `typeid`, `dynamic_cast`
- [ ] MSVC class layout and name mangling (including `#pragma pack` effects)
- [ ] `constexpr` basic evaluation (literal folding, simple expressions)
- [ ] `consteval` context detection — Sema records deferred `consteval` calls for
  later JIT evaluation (requires IR + codegen + JIT from Phases 4–5; see §4.6a)
- [ ] SEH support (`__try/__except/__finally`)
- [ ] MSVC STL header compatibility (progressively — needed for levels IV–VI;
  link against vcruntime.lib, ucrt.lib)

**Milestone**: Can type-check UE5/Game source files. No Sema file > 1000 lines.
`consteval` contexts are detected and recorded; full evaluation deferred to Phase 6.

### Phase 4: IR & Code Generation (Weeks 39–50)
- [ ] `forge-ir`: SSA-based IR with SIMD intrinsic support
- [ ] Minimal passes (DCE, constant folding)
- [ ] `forge-codegen`: x86-64 lowering, register allocation, emission
- [ ] SSE/AVX intrinsic lowering
- [ ] COFF section generation
- [ ] Debug metadata generation (source maps, type info, frame layouts — stored in
  internal format for DAP; CodeView serialization deferred to Phase 8)
- [ ] Basic `__asm` block support
- [ ] **Target levels I–II**: stb libraries compile and produce correct output

**Milestone**: Can compile simple C/C++ programs to working x86-64 code. Levels I–II
pass `forge-verify`.

### Phase 5: Runtime Loader / JIT + Ninja Patch (Weeks 51–58)
- [ ] `forge-daemon/loader.rs`: Process creation, section mapping
- [ ] Lazy compilation stubs
- [ ] Incremental patching (function-level)
- [ ] Import library parsing (.lib) for Windows SDK / third-party DLLs
- [ ] Static initializer ordering and execution
- [ ] Exception handling registration (`RtlAddFunctionTable`)
- [ ] **Ninja daemon mode patch** (see §7.4):
  - [ ] Fork ninja, add forge tool detection in rule parsing
  - [ ] HTTP/2 client (nghttp2 or libcurl) for daemon RPC
  - [ ] Response streaming and exit code mapping
  - [ ] Fallback to process spawning when daemon is unreachable
- [ ] **Target level III**: LMDB compiles, links, and runs via JIT

**Milestone**: Can JIT-compile and run a simple C++ program. Changes are patched live.
Ninja daemon mode eliminates process spawning for forge tools.
Target level III passes `forge-verify`.

### Phase 6: `consteval`/`constexpr` via JIT (Weeks 59–61)
- [ ] `forge-sema/consteval`: Wire deferred `consteval` calls to the JIT pipeline
- [ ] Compile `consteval` functions to native code via forge-ir → forge-codegen
- [ ] Map compiled code into daemon's address space (`VirtualAlloc` + `PAGE_EXECUTE`)
- [ ] Execute and capture results; feed back into Sema as compile-time constants
- [ ] Lightweight sanitizer instrumentation (overflow, null deref, bounds checks)
- [ ] Sandboxed allocator for `constexpr` `new`/`delete` (C++20)
- [ ] Memoization of `consteval` results: `blake3(function_hash + args) → result`
- [ ] **Target levels IV–VI**: nlohmann/json, Dear ImGui, Google Test

**Milestone**: `consteval` functions compile and execute in-process. Results are
cached and reused across TUs. No AST interpreter needed. Levels IV–VI pass
`forge-verify`.

### Phase 7: DAP Debugger (Weeks 62–65)
- [ ] `forge-debug`: DAP protocol server (JSON-over-stdio)
- [ ] Breakpoint management (int3 patching)
- [ ] Single-stepping (hardware debug registers)
- [ ] Variable inspection (ReadProcessMemory + Sema type info)
- [ ] Stack unwinding
- [ ] VS Code launch.json integration

**Milestone**: Can debug JIT-compiled programs from VS Code. No PDB needed.

### Phase 8: Linker — Release Mode (Weeks 66–71)
- [ ] `forge-link`: Symbol resolution, section layout
- [ ] PE executable generation
- [ ] DLL generation (for UE5 module structure)
- [ ] Import library reading
- [ ] Incremental linking
- [ ] PDB generation (release mode only — deferred priority)
- [ ] **Target level VII**: Selected LLVM lit tests pass

**Milestone**: Can link multi-file C++ programs into working PE executables and DLLs.
Target level VII provides regression coverage.

### Phase 9: Real-Time Content Project (Weeks 72–75)
- [ ] **Target level VIII**: Compile raylib + a small real-time demo
- [ ] Multi-TU linking with D3D11/Win32 interop
- [ ] JIT-compile and run the demo via the daemon
- [ ] Test hot-patching: change a shader parameter or game logic function, see the
  result in the running application within seconds
- [ ] Fix issues found with DLL loading, SIMD, threads, game loop patterns

**Milestone**: A real-time graphical application builds, runs, and hot-reloads via
the daemon. This validates the full pipeline (compile → link → JIT → patch) on a
real-world content project before tackling UE5.

### Phase 10: forge-demo — Full-Feature Showcase (Weeks 76–78)
- [ ] **Target level IX**: Write and compile **forge-demo** (~2–5k LoC C++20 app)
- [ ] C++20 feature coverage: `concept`, `requires`, structured bindings, `if constexpr`,
  `auto` return types, `consteval` lookup tables (sin/cos, color palettes)
- [ ] SIMD math: SSE/AVX vector/matrix operations (`__m128`, `_mm_mul_ps`)
- [ ] SEH: `__try/__except` around D3D calls
- [ ] STL usage: `std::span`, `std::array`, `std::vector`, `<algorithm>`, `<ranges>`
- [ ] DLL plugin: `plugin.dll` loaded at runtime — validates DLL generation and import
- [ ] DAP debugging: set breakpoints in the running demo, inspect variables mid-frame
- [ ] JIT hot-patching demo: change gravity/color/logic in `physics.cpp` or `scene.cpp`,
  see the result in the running application within ~1 second
- [ ] Distributed compilation: compile forge-demo across two machines via P2P

**Milestone**: A single small project exercises every compiler phase. The hot-patching
demo provides a compelling visual demonstration of **forgecc**'s value proposition.
All C++20 features, SIMD, SEH, DLL linking, DAP, and distributed compilation work
correctly on this project before moving to UE5.

### Phase 11: UE5/Game Integration (Weeks 79–96)
- [ ] UBT integration (`ForgeccToolChain.cs`) — replaces the CLI/curl workflow with
  native UBT support; compile and link actions flow through UBT's action graph
- [ ] Handle UE5-specific patterns (UHT-generated code, `*_API` DLL exports, etc.)
- [ ] Remaining Windows SDK / MSVC STL edge cases found during UE5 builds
- [ ] Deterministic compilation verification on UE5 codebase
- [ ] **Progressive UE5 testing** (target level X):
  - [ ] Single UE5 module (e.g., Core) compiles successfully
  - [ ] Small set of modules (Core + CoreUObject + Engine) compile and link
  - [ ] UE5 Lyra demo builds and runs
  - [ ] Full Game editor builds and runs
- [ ] Fix edge cases found at each progressive stage

**Milestone**: Game editor builds and runs via the daemon. Debugging works via DAP.

### Phase 12: Distributed Features & Memoization Optimization (Weeks 97–108)
- [ ] Full DHT implementation with replication
- [ ] Cross-machine artifact sharing
- [ ] Benchmark: two machines sharing compilation work
- [ ] File watching and pre-compilation
- [ ] Template instantiation memoization Tier 2 (recorded dependency signatures — see §4.2):
  profile UE5 builds to identify most-instantiated templates, instrument dependency
  recording, validate signature cost stays within budget
- [ ] Namespace content hash caching (Tier 3) for associated-namespace lookups

**Milestone**: Two machines share compilation work via P2P network.

---

### 8a. Incremental Target Codebases

Rather than jumping from "Hello World" to UE5, **forgecc** is validated against a ladder
of real-world open-source C/C++ projects of increasing complexity. Each **target level**
(numbered with Roman numerals I–X) serves as a milestone gate: the compiler must
produce correct, runnable output for a given target level before moving to the next.
(Target levels are distinct from implementation phases — phases describe *what we
build*, target levels describe *what we validate against*.)

| Target Level | Project | ~LoC | Why | Key features exercised |
|---|---|---|---|---|
| I | Hand-crafted test suite | <1k | Controlled tests for each language feature | All, incrementally |
| II | [**stb libraries**](https://github.com/nothings/stb) (stb_image.h, etc.) | ~7k per lib | Pure C, single-file, no dependencies, widely used | C99, macros, function pointers |
| III | [**LMDB**](https://github.com/LMDB/lmdb) | ~12k | Pure C, real-world database, COFF/PE linking exercise | C99, Win32 API, threads |
| IV | [**nlohmann/json**](https://github.com/nlohmann/json) (single-header) | ~25k | Header-only C++11/17, heavy templates, tests memoization | Templates, SFINAE, exceptions, STL |
| V | [**Dear ImGui**](https://github.com/ocornut/imgui) | ~70k | C++11, moderate complexity, Win32/DX backends, good for JIT testing | Classes, vtables, function pointers, Win32 |
| VI | [**Google Test**](https://github.com/google/googletest) | ~30k | C++14, templates, macros, used by many projects | Templates, macros, exceptions, RTTI |
| VII | [**LLVM lit test suite**](https://github.com/llvm/llvm-test-suite) (selected C/C++ tests) | varies | Known-good reference tests with expected outputs | Wide coverage |
| VIII | [**raylib**](https://github.com/raysan5/raylib) + small real-time demo | ~100k | Real-time rendering, game loop, Win32/D3D, multi-TU linking, hot-reload testing | D3D interop, DLL loading, game loop, SIMD, threads, JIT patching of running app |
| IX | **forge-demo** (purpose-built C++20 showcase) | ~2–5k | Smallest project that exercises *every* phase; interactive real-time app designed for the "wow" demo | C++20 concepts, `consteval` tables, `std::span`/`std::array`, SIMD math, Win32/D3D11, SEH, DLL plugin, DAP debugging, hot-patching live |
| X | **UE5 Lyra demo** / Game editor | ~millions | Full target | Everything |

**Target level IX — forge-demo**: No single small open-source project exercises all
compiler phases, because real-world projects target broad compiler support and avoid
bleeding-edge C++20 features. **forge-demo** is a purpose-built ~2–5k LoC interactive
application (Win32 + D3D11) that intentionally uses every feature the compiler must
handle — a compiler stress test disguised as a demo app. Suggested structure:

```
forge-demo/
  main.cpp       — Win32 window creation, message loop, D3D11 init
  renderer.cpp   — D3D11 rendering, shader loading, draw calls
  scene.cpp      — Scene graph, entity system using C++20 concepts
  math.cpp       — SIMD vector/matrix math (SSE/AVX), consteval lookup tables
  physics.cpp    — Particle/physics sim (the "change this and see it live" target)
  plugin.dll     — Optional DLL plugin to test DLL linking and loading
```

| Phase | Feature needed | How forge-demo exercises it |
|---|---|---|
| 1 | Preprocessor, Windows SDK headers | `#include <windows.h>`, `#include <d3d11.h>`, conditional compilation |
| 2 | C++20 parser | `concept`, `requires`, structured bindings, `if constexpr`, `auto` return types |
| 3 | Templates, concepts, `consteval`, STL, SEH | `concept Renderable`, `consteval` color/math tables, `std::span`/`std::array`, `__try/__except` around D3D calls |
| 4 | x86-64 codegen, SIMD | `__m128`/`_mm_mul_ps` for transforms, hand-written SIMD math |
| 5 | JIT, live patching, DLL imports | Entire app runs via JIT; changing `physics.cpp` patches the running app live |
| 6 | `consteval` via JIT | `consteval` sin/cos tables, color palettes, vertex data |
| 7 | DAP | Set breakpoints in the running demo, inspect variables mid-frame |
| 8 | PE/DLL linking | Release build produces standalone `.exe` + `plugin.dll` |
| 9 | Real-time + hot-reload | This *is* the real-time hot-reload demo |
| 12 | Distributed | Compile forge-demo across two machines |

The **demo scenario for JIT hot-patching**: the app is running and rendering a scene;
the developer changes the gravity constant in `physics.cpp` or the particle color
function in `scene.cpp`, and within a second the running application reflects the
change — no restart, no visible recompile cycle. This is the "wow" moment that
demonstrates **forgecc**'s value proposition.

**How target levels map to implementation phases** (each target level is an explicit
milestone gate within its phase):
- **Target levels I–II** gate Phase 4 (codegen) — pure C, simple linking
- **Target level III** gates Phase 5 (JIT loader) — threads, Win32 API
- **Target levels IV–VI** gate Phase 6 (consteval) — heavy C++ templates, STL headers
- **Target level VII** gates Phase 8 (release linker) — regression coverage
- **Target level VIII** bridges Phase 8 → Phase 10 — real-time rendering, game loop,
  D3D interop, multi-TU linking, and the first real test of JIT hot-patching a running
  graphical application
- **Target level IX** gates Phase 11 — the only project that exercises *every* phase
  (C++20 concepts, `consteval`, SIMD, SEH, DLL plugin, DAP debugging, hot-patching);
  serves as the comprehensive validation gate before UE5
- **Target level X** is progressive within Phase 11: single UE5 module → small set →
  Lyra → full Game editor

### 8b. Verification Tooling: `forge-verify`

To validate correctness against existing compilers, **forgecc** includes a thin CLI tool
called **`forge-verify`** (~1,500 LoC). Rather than building a full comparison tool
from scratch, it leverages
[**`llvm-debuginfo-analyzer`**](https://llvm.org/docs/CommandGuide/llvm-debuginfo-analyzer.html)
(the successor to llvm-diva, already in LLVM trunk) for debug info comparison, and
adds execution-level and structural comparison on top.

**`forge-verify` workflow**:

1. Connects to the **forgecc** daemon via HTTP (`POST /v1/compile`)
2. Requests compilation of a source file (or set of files)
3. Materializes a temporary `.obj` file from the store (via
   `/v1/artifact/{hash}?format=coff` — see §4.13)
4. Compares against a reference `.obj` produced by MSVC or clang-cl using:
   - **`llvm-debuginfo-analyzer --compare`**: debug info (CodeView) correctness
   - **`llvm-objdump --disassemble`**: instruction-level comparison (structural, not
     byte-identical — register allocation will differ)
   - **Custom section diff**: compare section names, sizes, relocation counts, symbol tables
   - **Execution diff**: link and run both executables, compare stdout/stderr/exit code

**Test case structure**:

```
tests/
  level0_handcrafted/
    hello_world.cpp          # source
    hello_world.expected     # expected stdout/stderr/exit code
  level1_stb/
    stb_image_test.cpp
    stb_image_test.expected
  level2_lmdb/
    ...
```

**Usage in CI**: `forge-verify` is the primary unit-test harness. Each target codebase
level (§8a) has a corresponding test directory. CI runs `forge-verify` against all
levels that the current phase supports, comparing output against clang-cl as the
reference compiler.

**Usage for development**: During development, `forge-verify --single foo.cpp` compiles
a single file with both **forgecc** and clang-cl, then diffs the results — useful for
debugging individual compilation issues.

---

## 9. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| C++ corner cases in Sema | Very High | High | Focus on UE5/Game's subset; fail gracefully on unsupported features; incremental target codebases (§8a) |
| MSVC ABI incompatibility | High | Critical | Extensive testing against MSVC-compiled code; `forge-verify` (§8b) + `llvm-debuginfo-analyzer` |
| JIT execution stability | High | High | Hybrid mode as fallback; JIT is optional |
| Performance of unoptimized code | Medium | Medium | UE5 debug builds are already unoptimized; acceptable |
| SIMD intrinsic coverage | Medium | High | Map intrinsics incrementally; start with SSE2, add SSE4/AVX as needed |
| Windows SDK header complexity | High | Medium | Test incrementally; many headers are already handled by preprocessor |
| Distributed store consistency | Low | Low | Content-addressed = deterministic; eventual consistency is fine |
| Parallelism correctness | Medium | High | Rust's ownership model helps; extensive testing with thread sanitizer |

---

## 10. Key Dependencies (Rust Crates)

| Crate | Purpose | Link |
|---|---|---|
| `redb` | Embedded key-value database (local store) | [crates.io/crates/redb](https://crates.io/crates/redb) |
| `blake3` | Fast content hashing | [crates.io/crates/blake3](https://crates.io/crates/blake3) |
| `rayon` | Parallel compilation (work-stealing thread pool) | [crates.io/crates/rayon](https://crates.io/crates/rayon) |
| `tokio` | Async I/O for daemon HTTP server and P2P networking | [crates.io/crates/tokio](https://crates.io/crates/tokio) |
| `axum` | HTTP server framework (lightweight, tokio-native) | [crates.io/crates/axum](https://crates.io/crates/axum) |
| `reqwest` | HTTP client (for P2P peer communication) | [crates.io/crates/reqwest](https://crates.io/crates/reqwest) |
| `notify` | File system watching | [crates.io/crates/notify](https://crates.io/crates/notify) |
| `windows` | Windows API bindings (process creation, VirtualAlloc, debug API) | [crates.io/crates/windows](https://crates.io/crates/windows) |
| `object` | Reading import libraries and COFF files | [crates.io/crates/object](https://crates.io/crates/object) |
| `serde` / `rmp-serde` | Serialization: serde for derive, rmp-serde for MessagePack wire format | [crates.io/crates/rmp-serde](https://crates.io/crates/rmp-serde) |
| `serde_json` | JSON wire format (debug/curl fallback) | [crates.io/crates/serde_json](https://crates.io/crates/serde_json) |
| `clap` | CLI argument parsing | [crates.io/crates/clap](https://crates.io/crates/clap) |
| `tracing` | Structured logging | [crates.io/crates/tracing](https://crates.io/crates/tracing) |
| `dashmap` | Concurrent hash maps (for Sema type tables) | [crates.io/crates/dashmap](https://crates.io/crates/dashmap) |
| `crossbeam` | Lock-free data structures | [crates.io/crates/crossbeam](https://crates.io/crates/crossbeam) |

---

## 11. Lines-of-Code Estimates

| Crate | Estimated LoC | Notes |
|---|---|---|
| `forge-common` | 2,000 | Diagnostics, source map, interning |
| `forge-store` | 4,500 | Local store + distributed layer + dependency signature tracking (+500) + namespace content hash caching + template instantiation cache (+1000) |
| `forge-net` | 5,000 | HTTP server (axum), MessagePack/JSON content negotiation, P2P networking, DHT, gossip |
| `forge-lex` | 5,000 | Lexer + literal parsing (C + C++); raw strings, UDL, all encoding prefixes, hex floats |
| `forge-pp` | 7,500 | Preprocessor + macro expansion; #pragma pack/warning, feature-test macros, predefined macros |
| `forge-parse` | 16,000 | Parser + AST definitions (C + C++); CTAD, extern "C", explicit(bool), bit-fields |
| `forge-sema` | 34,500 | Modular Sema (~52 files, none > 1000 lines); two-phase lookup, CTAD, UDL resolution, noexcept types, defaulted comparisons, structured bindings, brace init, RTTI, MSVC __declspec/intrinsics; template instantiation memoization instrumentation (+3000); consteval via JIT saves ~3k lines vs interpreter |
| `forge-ir` | 5,000 | IR + minimal passes |
| `forge-codegen` | 14,000 | x86-64 lowering + encoding + SIMD |
| `forge-debug` | 3,000 | DAP server + breakpoints + variable inspection |
| `forge-link` | 6,000 | Linker + PE + DLL (PDB deferred to release mode) |
| `forge-daemon` | 5,500 | Daemon + scheduler + loader/JIT + determinism verification (HTTP server moved to forge-net) |
| `forge-driver` | 800 | Thin driver binaries: argv[0] detection, clang-cl/lld-link/llvm-lib/llvm-rc/llvm-ml flag parsing, HTTP client, daemon auto-start (see §7.3) |
| `forge-verify` | 1,500 | Verification CLI: daemon client, reference compiler diff, execution diff (see §8b) |
| **Total** | **~110,300** | + ~300–500 lines C++ ninja patch (external, see §7.4) |

---

## 12. Parallelism Strategy (Detailed)

This is a core differentiator from LLVM/Clang. Every component is designed for
parallelism:

### 12.1 Across Translation Units
Like existing build systems, but managed by the daemon's scheduler instead of
make/ninja. The daemon has full knowledge of the dependency graph and can schedule
optimally.

### 12.2 Within a Translation Unit

Intra-TU parallelism follows a **sequential spine → parallel fan-out** model.
The first three stages are inherently sequential due to C++'s context-dependent
grammar and declaration-order semantics; the later stages fan out to all cores:

- **Lexer/Preprocessor** (sequential): Token order matters. The preprocessor must
  process `#include` directives in order because each header is preprocessed in the
  macro context established by everything before it. Different `#include`d files can
  only be preprocessed in parallel if they are guarded (`#pragma once` / include
  guards) and don't depend on external macro state.
- **Parser** (sequential): C++ parsing is context-dependent — whether a name is a
  type or a value affects how subsequent tokens are parsed. `typedef`, `using`,
  class/struct/enum declarations all change the parsing context. Top-level
  declarations must be parsed in order.
- **Sema — signatures** (sequential): Declaration signatures must be resolved
  top-to-bottom because each can reference types introduced by earlier declarations.
- **Sema — function bodies** (parallel): Once all signatures are resolved, function
  bodies can be type-checked concurrently. Each body only reads from the shared type
  table (`DashMap`, `RwLock`) — a read-heavy workload ideal for multithreading.
  Template instantiations are also parallelized (each is independent).
- **IR lowering** (parallel): Each function is lowered independently → trivially
  parallel via rayon.
- **CodeGen** (parallel): Each function is compiled independently → trivially
  parallel via rayon.

The sequential spine (lex → parse → sema signatures) produces the work items
(function bodies, template instantiations) that then fan out across all cores.
For a large TU with hundreds of functions, the parallel phase dominates.

### 12.3 Pipeline Parallelism
Different TUs can be at different pipeline stages simultaneously:
```
Time →
TU1: [lex] [parse] [sema] [ir] [codegen]
TU2:       [lex]   [parse] [sema] [ir] [codegen]
TU3:               [lex]   [parse] [sema] [ir] [codegen]
```

### 12.4 Intra-TU Parallelism — Realistic Scaling Analysis

For a TU that takes 2–3 minutes on a single core today (without caching):

| Stage | Parallelism potential | Estimated savings (8 cores) |
|---|---|---|
| Preprocessing | Sequential; limited parallelism for guarded includes | Minimal |
| Parsing | Sequential (context-dependent grammar) | None |
| Sema — signatures | Sequential (declaration order) | None |
| Sema — bodies | Function bodies + template instantiations in parallel | **30–50%** of Sema time |
| IR lowering | Per-function, trivially parallel | **60–80%** |
| CodeGen | Per-function, trivially parallel | **60–80%** |

**Realistic overall estimate**: For a 2–3 minute TU, intra-TU parallelism alone (no
caching) could bring it down to **1–1.5 minutes** on 8 cores. The bigger win comes
from **memoization** — if 90% of headers haven't changed, the TU might take
**5–15 seconds** on a warm cache regardless of parallelism.

**Honest assessment**: Intra-TU parallelism is a **moderate** win. The
**memoization/caching** is the transformative win. Parallelism across TUs (which we
already have) plus caching is where 90% of the speedup comes from.

### 12.5 Across Machines
Via the distributed store. If machine A has already compiled a header that machine B
needs, B fetches the result instead of recompiling.

**Distributed TU compilation** (compiling a single TU across multiple machines) is
theoretically possible — individual functions could be farmed out for codegen. However,
the overhead of serializing AST/IR, transferring it, and collecting results likely
exceeds the benefit for a single TU. The better model is: **distribute TUs across
machines** (coarse-grained), not functions within a TU (fine-grained).

### 12.6 Unity/Jumbo Files — No Longer Necessary

With **forgecc**'s caching, unity files should become **unnecessary**. Unity files
exist because:

1. Compiler startup cost per TU is high (loading headers, PCH) — **forgecc** eliminates
   this via the daemon's warm cache
2. Redundant header parsing across TUs — **forgecc**'s header-level memoization
   eliminates this
3. Linker overhead from many .obj files — **forgecc**'s JIT mode has no linker step

If caching works as designed, compiling 100 separate TUs should be as fast as (or
faster than) compiling them as 10 unity files, because the cache eliminates the
redundant work that unity files were designed to avoid.

---

## 13. Standard Library Strategy

**We do NOT compile the MSVC STL.** The STL headers are `#include`d in source files
and processed by our preprocessor/parser/sema like any other header. But we **link
against** the precompiled MSVC runtime libraries:

- `vcruntime.lib` — C runtime basics
- `ucrt.lib` — Universal C runtime
- `msvcrt.lib` / `msvcprt.lib` — C++ runtime, STL implementation

This means:
- Our preprocessor/parser/sema must handle MSVC STL headers (which use heavy template
  metaprogramming and MSVC extensions)
- Our linker must read MSVC import libraries (.lib)
- Our generated code must be ABI-compatible with MSVC-compiled code in these libraries

The UE5 editor target uses DLLs, so we link against the DLL versions of the CRT
(`/MD` flag equivalent).

---

## 14. C++20 Modules Decision

**forgecc does not implement C++20 modules.** The reasons:

- UE5/Game has **zero** usage of C++20 modules (`export module`, `import`)
- UE5's "modules" are its own build system concept (`.Build.cs`, `Private`/`Public`
  directories), not C++20 modules
- C++20 modules exist primarily for build time improvement — **forgecc** achieves the
  same through its memoization/caching system, making modules redundant
- C++20 modules add significant compiler complexity (module dependency scanning,
  BMI generation, build ordering constraints) for no benefit in our target codebase

The architecture naturally supports modules (the content-addressed store handles
BMI caching), so adding them post-PoC is straightforward if a target codebase
requires them.

---

## 15. UE5 Codebase Analysis Findings (Verified Against Source)

> The findings below are based on a thorough automated scan of
> `C:\src\git\UnrealEngine\Engine\Source\` (Runtime, Editor, ThirdParty).

### 15.1 C Code
- **4,335 .c files** in Engine/Source (all third-party: zlib, libpng, FreeType,
  Bink Audio, Rad Audio, ICU, etc.)
- Engine code itself is C++ only
- **forgecc** must support C compilation for these third-party libraries

### 15.2 Inline Assembly (`__asm`)
- **~20 files** in Engine/Source/Runtime use MSVC-style `__asm` / `_asm`
- Key locations: RadAudio `rrCore.h`, BinkAudio `rrCore.h` / `win32_ticks.cpp`,
  MemPro `MemPro.cpp`, AutoRTFM `LongJump.cpp`
- GNU-style `__asm__` also used in cross-platform code (Unix, ARM, x86 breakpoints)
- Most are in third-party SDK code that could potentially be linked as prebuilt
- **Strategy**: Support basic `__asm` blocks; complex cases fall back to prebuilt stubs

### 15.3 SIMD Intrinsics
- **Heavily used** — hundreds of files, thousands of intrinsic calls
- `UnrealMathSSE.h` alone has 383 SIMD-related lines (core math library)
- SSE2 baseline, optional SSE4.1, AVX, AVX2
- Key files: `UnrealMathSSE.h`, `UnrealPlatformMathSSE.h`, `UnrealPlatformMathSSE4.h`,
  `AES.cpp` (133 matches), `SecureHash.cpp` (109 matches), `radfft.cpp` (154 matches)
- **Critical**: Must support SSE/AVX intrinsics from day one

### 15.4 SEH (`__try/__except/__finally`)
- **~20 files** in Engine/Source/Runtime
- Concentrated in: crash handling (`WindowsPlatformCrashContext.cpp`), D3D11/D3D12
  device creation (delay-load exception handling), thread management
  (`WindowsRunnableThread.cpp`), stack walking (`WindowsPlatformStackWalk.cpp`),
  rendering thread (`RenderingThread.cpp`), shader compilation (`ShaderCore.cpp`),
  race detector (`RaceDetectorWindows.cpp`)
- **Important** for Windows stability, but not in hot paths

### 15.5 C++20 Feature Usage (Detailed)

| Feature | Files | Usage Level | Notes |
|---|---|---|---|
| **Concepts/requires** | ~95+ | **Critical** | UE defines 20+ concepts in `Core/Public/Concepts/`; used in MassEntity, TypedElementFramework, Chaos, CoreUObject. `UE_REQUIRES` macro wraps `requires` with SFINAE fallback. |
| **`if constexpr`** | ~91+ | **Critical** | Pervasive: TypedElementFramework, SlateCore, RenderCore, MovieScene, Net/Iris, VerseCompiler, VulkanRHI, DelegateSignatureImpl (72 uses), Array.h (31 uses) |
| **Structured bindings** | ~33+ | **Moderate** | RenderGraphBuilder, ActorDescContainer, MaterialExpressions, VerseVM |
| **consteval** | ~6 | **Rare** | Via `UE_IF_CONSTEVAL` macro (expands to `if consteval` or `__builtin_is_constant_evaluated()`); Platform.h, StringView.h, NameTypes.h |
| **constinit** | ~14 | **Rare** | Mostly CoreUObject: UObjectGlobals.h, Object.h, Class.h, ObjectMacros.h |
| **Three-way comparison (`<=>`)** | ~10 | **Rare** | GuardedInt.h, GenericPlatformMath.h, EdGraphPin.h, RenderGraphUtils.h |
| **Designated initializers** | ~6 | **Rare** | Net/Iris, AppleDynamicRHI, ReplicationSystem |
| **`using enum`** | ~4 | **Rare** | NavigationSystem.h, ISpatialAcceleration.h |
| **User-defined literals** | ~4 | **Rare** | Delegate.h (`operator""_intseq`), StringView.h (`operator""_PrivateSV`) |
| **Fold expressions** | ~1 | **Rare** | Solaris/uLangCore Cases.h |
| **Lambdas with template params** (`[]<`) | ~5 | **Rare** | Core tests, VerseVM, GeometryCore |
| **CTAD (deduction guides)** | ~2 | **Rare** | StringBuilderTest.cpp only |
| **char8_t** | ~6 | **Rare** | VerseGrammar.h, GenericPlatform.h, StringView.h, StringConv.h |
| **Coroutines** | **0** | **Not used** | Zero in Engine; only in ThirdParty docs/comments |
| **C++20 modules** | **0** | **Not used** | Zero usage |
| **`std::ranges` / `std::views`** | **0** | **Not used** | UE uses custom iterators |
| **`std::format`** | **0** | **Not used** | UE uses `FString::Printf` / `fmt` |
| **`std::span`** | **0** | **Not used** | UE uses `TArrayView` / `FMemoryView` |
| **`std::expected`** | **0** | **Not used** | Only in ThirdParty (WebRTC) |

### 15.6 MS Extensions (Detailed)

| Extension | Usage | Priority |
|---|---|---|
| `__declspec(dllexport/dllimport)` | Pervasive via `*_API` macros | **Critical** |
| `__forceinline` | 200+ files via `FORCEINLINE` macro | **Critical** |
| `__declspec(noinline)` | WindowsPlatform.h (`FORCENOINLINE`), MemPro, AutoRTFM | **High** |
| `__declspec(align(N))` | ~8 files (mostly third-party); `alignas` preferred (~75 files) | **Medium** |
| `__declspec(thread)` | syms_base.h, RadAudio, MemPro, FramePro | **High** |
| `__declspec(selectany)` | Platform.h (`UE_SELECT_ANY`), RadAudio, BinkAudio | **High** |
| `__declspec(empty_bases)` | WindowsPlatform.h | **Medium** |
| `__declspec(noreturn)` | MicrosoftPlatformCodeAnalysis.h | **Medium** |
| `__declspec(restrict)` / `noalias` | MemoryArena.h, MemPro | **Low** |
| `__declspec(code_seg)` | WindowsPlatform.h | **Low** |
| `__declspec(allocate)` | syms_base.h (`.roglob`), RaceDetector (`.CRT$XCT`) | **Medium** |
| `__declspec(safebuffers)` | Instrumentation/Defines.h | **Low** |
| `__declspec(deprecated)` | LZ4 only | **Low** |
| `__declspec(novtable)` | **Not used** | Skip |
| `__declspec(property)` | **Not used** | Skip |
| `__pragma` | ~12 files; MSVCPlatformCompilerPreSetup.h (warning macros) | **High** |
| `__int64` | ~9 files (RadAudio, BinkAudio, SymsLib — third-party) | **Medium** |
| `__cdecl` / `__stdcall` | Common (Windows API, COM, CRT callbacks) | **High** |
| `__fastcall` / `__vectorcall` / `__thiscall` | **Not used** | Skip |
| `__assume` | ~6 files; `UE_ASSUME` macro in Platform.h | **Medium** |
| `_alloca` / `alloca` | ~15 files; stack allocation in Renderer, Core, BinkAudio | **Medium** |

### 15.7 MSVC Intrinsics

| Intrinsic | Used | Key Locations |
|---|---|---|
| `_BitScanForward` / `_BitScanForward64` | Yes | LZ4, RadAudio, D3D12 |
| `_BitScanReverse` / `_BitScanReverse64` | Yes | Same |
| `_InterlockedCompareExchange*` | Yes | WindowsPlatformAtomics.h |
| `__debugbreak` | Yes | Solaris, RadAudio, VerseGrammar |
| `_ReturnAddress` / `_AddressOfReturnAddress` | Yes | MSVCPlatform.h, RaceDetector |
| `__cpuid` / `__cpuidex` | Yes | RadAudio cpux86.cpp, VVMAtomics.h |
| `_byteswap_ushort` / `_ulong` / `_uint64` | Yes | RadAudio, BinkAudio |

### 15.8 Preprocessor Patterns

| Pattern | Usage | Notes |
|---|---|---|
| `#pragma once` | **Dominant** | Used in virtually all UE headers |
| `#pragma pack` | ~37 files | SymsLib, TraceLog, RenderCore, Oodle, D3D11/D3D12 |
| `#pragma warning` | ~80 files | MSVCPlatformCompilerSetup.h alone has ~97 uses |
| `#pragma comment(lib, ...)` | ~17 files | psapi, pdh, version, Shlwapi, Dbghelp, Ws2_32, d3d12, etc. |
| `__has_include` | ~60 files | Platform.h, ThirdParty (abseil, WebRTC, Vulkan, SDL3) |
| `__has_cpp_attribute` | ~60 files | Mostly ThirdParty (simde, hedley, abseil, Catch2) |
| `__VA_ARGS__` | 100+ files | PreprocessorHelpers.h, LogMacros, delegates, profiling |
| `__VA_OPT__` | ~10 files | Mostly ThirdParty (fmt, Catch2) |
| `_Pragma` | ~60 files | Mostly ThirdParty (simde, hedley, abseil, WebRTC) |
| Token pasting (`##`) | 60+ files | PreprocessorHelpers, Stats, Delegates, LogMacros |

### 15.9 Template & Metaprogramming Complexity

**Most template-heavy areas** (in order of complexity):
1. `Runtime/Core/Public/Templates/` — 80+ headers: traits, SFINAE, concepts, utilities
2. `Runtime/Core/Public/Delegates/` — variadic delegate signatures, policy-based design
3. `Runtime/Core/Public/Containers/` — TArray, TMap, TSet, allocators, iterators
4. `Runtime/Core/Public/Misc/` — TVariant, TOptional, ExpressionParser, StringBuilder
5. `Runtime/Core/Public/Concepts/` — 20+ C++20 concept definitions

**Key patterns**:
- `TEnableIf` / `std::enable_if_t` for SFINAE (with `UE_REQUIRES` C++20 fallback)
- `TAnd`, `TOr`, `TNot` for trait composition
- Heavy variadic template usage in delegates, tuples, structured logging
- `extern template` for compile-time control (StringFormatter, TokenStream, etc.)
- Explicit template instantiation for common char types (ANSICHAR, WIDECHAR)
- `decltype(auto)` for perfect forwarding (~20 files in Core)
- Variable templates (`TModels_V`, `TIsVariant_V`, etc.)
- `static_assert` — 100+ files in Runtime/Core

### 15.10 DLL Structure
- Game editor builds **~190 game modules** as DLLs (see `GameEditor.Target.cs`)
- Engine adds hundreds more (Core, Engine, RHI, Slate, etc.)
- Each module uses `*_API` macros for dllexport/dllimport
- `FModuleManager` handles dynamic loading
- Live Coding (Live++) patches running DLLs — limited to function body changes
- **Circular dependencies**: UE5 uses a two-step lib creation process for modules
  with circular dependencies — first create an import `.lib` (with exports only),
  then build the real DLL. **forgecc** must handle this in release mode; in JIT mode,
  circular dependencies are resolved naturally since all symbols live in one address space

### 15.11 COM Usage
- **~70+ files** in Runtime use COM interfaces (IUnknown, HRESULT, etc.)
- Main areas: D3D11, D3D12, Windows WASAPI, WMF, TextStore, UIA
- `COMPointer.h` in Core provides UE's COM smart pointer
- `__uuidof` used for COM interface identification
- **Must support**: `IUnknown`, `HRESULT`, `__uuidof`, COM calling conventions

### 15.12 Resource Compiler (.rc) Files
- **137 .rc files** in Engine/Source (44 in Runtime/Launch, rest in ThirdParty)
- Primary: `Runtime/Launch/Resources/Windows/PCLaunch.rc`
- **Out of scope for forgecc** — .rc files are compiled by `rc.exe`, not the C++ compiler

### 15.13 UBT Compiler Invocation
- `VCToolChain.cs` (~3,979 lines) handles both MSVC and clang-cl modes
- `VCCompileAction.cs` structures compilation as action objects with all necessary metadata
- **Clang-cl mode is the better integration point** for **forgecc**

### 15.14 Features Confirmed NOT Used (Safe to Skip)

These features have **zero usage** in Engine/Source and can be safely excluded from
the PoC without risk:

| Feature | Verified | Notes |
|---|---|---|
| C++20 Coroutines | Zero in Engine | Only in ThirdParty docs/comments (asio) |
| C++20 Modules | Zero | UE "modules" are build system concept |
| `std::ranges` / `std::views` | Zero | UE uses custom iterators/algorithms |
| `std::format` | Zero | UE uses `FString::Printf`, `fmt` |
| `std::span` | Zero | UE uses `TArrayView`, `FMemoryView` |
| `std::expected` | Zero in Engine | Only in ThirdParty (WebRTC) |
| `__declspec(novtable)` | Zero | |
| `__declspec(property)` | Zero | |
| `__fastcall` / `__vectorcall` / `__thiscall` | Zero | Only `__cdecl` and `__stdcall` used |
| WinRT (`<winrt/>`) | Zero | |
| WRL (`<wrl/>`) | Zero | |

---

## 16. Deterministic Compilation (Required)

Deterministic compilation is **critical** for the distributed content-addressed store.
If two machines compile the same inputs and get different outputs, the store's
content-addressing breaks — the same key would map to different values.

Reference: [Deterministic builds with clang and lld](https://blog.llvm.org/2019/11/deterministic-builds-with-clang-and-lld.html)

**What forgecc must guarantee**:

1. **No timestamp embedding**: `__TIME__`, `__DATE__`, `__TIMESTAMP__` must either be
   errors or expand to fixed values. The Game project should not use these (to verify).

2. **No non-deterministic iteration order**: All internal data structures that affect
   output must use deterministic iteration (sorted maps, not hash maps with random seeds).
   Rust's `HashMap` uses RandomState by default — we must use `BTreeMap` or a
   deterministic hasher for any map that affects output ordering.

3. **No pointer-value-dependent output**: Addresses of AST nodes, types, etc. must not
   leak into output. Use indices or content hashes instead.

4. **Path independence**: Build outputs must not contain absolute paths (or must normalize
   them). This is needed for the distributed store — machine A and machine B may have
   different checkout paths.
   - Use relative paths internally, **relative to the branch root** (e.g., the
     repository checkout root), not relative to the current working directory
   - Normalize all paths to forward slashes and consistent casing
   - For debug info, use a configurable source root prefix
   - Different developers may have different branch root paths (e.g.,
     `C:\src\project` vs. `D:\dev\project`) — the content-addressed store must
     produce identical hashes regardless of the absolute branch root location

5. **Thread-determinism**: Even though compilation is parallel, the output must be
   independent of thread scheduling. This means:
   - No output in "order threads finish"
   - Merge parallel results in a deterministic order (e.g., sorted by symbol name)
   - Use deterministic work-stealing (seed the work queue deterministically)

6. **No `__COUNTER__` across TUs**: `__COUNTER__` is inherently TU-local and
   deterministic within a TU, so it's fine. But we must ensure it doesn't leak across
   TU boundaries.

**Pipeline computation determinism**:

The points above cover *output* determinism (what bytes end up in the store). But
the memoization system also requires that *intermediate computations* within the
pipeline are deterministic — if a pipeline stage produces a different intermediate
result for the same inputs, downstream cache keys diverge and memoization breaks.
Every stage must be a pure function of its inputs:

7. **Overload resolution**: Given the same candidate set and argument types,
   overload resolution must always select the same winner. This is naturally
   deterministic in the C++ standard (the rules are fully specified), but the
   implementation must not depend on the order candidates are discovered. If
   candidates are collected into a set, that set must be ordered deterministically
   (e.g., by declaration source location or by a content hash) before ranking.

8. **Template instantiation order**: The order in which templates are instantiated
   must not affect the result of any individual instantiation. In standard C++,
   this is guaranteed (instantiations are independent), but implementation
   shortcuts — such as caching a partially-computed type and reusing it before
   it is complete — can introduce order-dependence. Each instantiation must see
   a fully consistent view of the type system.

9. **Name lookup and ADL**: Unqualified name lookup and argument-dependent lookup
   must produce the same result regardless of which TU triggered the lookup first.
   Since forgecc caches lookup results, the cache key must include all inputs that
   affect the result (the name, the argument types, the set of visible declarations
   in associated namespaces). Two lookups with the same key must return the same
   result.

10. **Implicit special member generation**: Whether a class's copy constructor,
    move constructor, destructor, etc. are defaulted, deleted, or trivial must be
    computed identically regardless of which TU first queries the property. The
    result depends only on the class definition and its members' properties — not
    on when or where the query happens.

11. **Constant evaluation**: `constexpr` and `consteval` evaluation must produce
    identical results for identical inputs. Since forgecc uses JIT for constant
    evaluation (§4.6a), the JIT'd code must be compiled deterministically (same
    machine code for the same ForgeIR), and the evaluation must not depend on
    host-specific state (e.g., stack addresses, allocation addresses). The
    sandboxed allocator for `constexpr new/delete` must assign deterministic
    addresses.

12. **SFINAE and concept satisfaction**: Substitution failure detection and concept
    constraint evaluation must be deterministic. Given the same template arguments
    and the same set of visible declarations, SFINAE must fail or succeed
    identically, and concept satisfaction must produce the same boolean result.
    This is naturally the case if name lookup and overload resolution are
    deterministic (points 7–9).

13. **Diagnostic emission**: Diagnostics (warnings, errors) must be emitted in a
    deterministic order. If two warnings are generated in parallel, they must be
    sorted before output. Diagnostic content must not include non-deterministic
    information (pointer addresses, thread IDs).

**Verification**: The daemon has a `--verify-determinism` mode that compiles
everything twice and asserts that all content hashes match — both final outputs
and intermediate artifacts at every pipeline stage. This catches determinism bugs
early, whether they originate in output formatting or in pipeline computations.

## 17. JIT Runtime Details

### 17.1 Import Library Parsing

Even in JIT mode, we need `.lib` parsing for **symbol validation** before execution.
The daemon must know what symbols are available from external DLLs (kernel32, d3d12,
ucrt, vcruntime, etc.) to validate that all references can be resolved.

**JIT flow**:
1. Parse import libraries (`.lib` files) at build start → build external symbol table
2. During compilation, validate that external references exist in the symbol table
3. At JIT execution time, `LoadLibrary` the actual DLLs into the target process
4. `GetProcAddress` to resolve each import to a concrete address
5. Patch call sites to point to the resolved addresses

`forge-link/imports.rs` is therefore needed for **both** JIT and release mode.

### 17.2 Static Initialization Order

The C++ standard says initialization order of globals across TUs is **unspecified**
(the "static initialization order fiasco"). Within a single TU, it's top-to-bottom
order of definition. Across TUs, it's implementation-defined.

In practice, MSVC/LLD orders them by the order `.obj` files appear on the link
command line, and within each `.obj` by the order of CRT initializer entries in
`.CRT$XCU` sections. Well-written C++ code (and UE5) should not depend on cross-TU
init order.

**forgecc**'s approach:
- Collect all `.CRT$XCU` initializer entries from compiled TUs
- Execute them before `main()`/`WinMain()`
- Order can be arbitrary, or match the order UBT sends compile actions (for compatibility)
- This is straightforward — just collect function pointers and call them

### 17.3 Thread-Local Storage

When JIT-compiling code that uses `__declspec(thread)`, the OS doesn't know about
the new TLS slots because the module wasn't loaded via `LoadLibrary`. This is the
same problem encountered in the
[in-process clang branch](https://github.com/aganea/llvm-project/tree/feat/clang_build_multiple_files_in_process).

The solution is to call the undocumented `ntdll!LdrpHandleTlsData` to register TLS
data for JIT-compiled modules. This approach has been proven to work in the in-process
clang work.

**Alternative approaches** (if `LdrpHandleTlsData` proves fragile across Windows versions):
- Convert `__declspec(thread)` to explicit `TlsAlloc`/`TlsGetValue`/`TlsSetValue`
  calls during IR lowering
- Use Fiber-Local Storage (FLS) API as a fallback

For the PoC, the `LdrpHandleTlsData` approach is the most pragmatic since it's
already proven.

### 17.4 Exception Handling

SEH and C++ exceptions require `.pdata`/`.xdata` sections and runtime support. In JIT
mode, we need to register unwind info with `RtlAddFunctionTable` or
`RtlInstallFunctionTableCallback` for each JIT-compiled function.

This is well-understood territory — LLVM's ORC JIT does the same thing. The key is
to call `RtlAddFunctionTable` after mapping code sections and before execution.

### 17.5 Virtual Function Tables

In JIT mode, vtables must be laid out correctly and patched when classes are recompiled.
If a class layout changes (new virtual function, reordered members), all vtable
references must be updated. This requires:
- Tracking which vtables reference which functions
- When a class is recompiled with a layout change, update the vtable and all objects
  that point to it
- For the PoC, layout changes could trigger a full restart of the target process
  (acceptable for a PoC; Live Coding has the same limitation)

---

## 18. Live PGO Injection (PoC Feature)

The daemon's JIT architecture enables a novel form of profile-guided optimization:
**live PGO injection** without the traditional profile-collect-rebuild cycle.

**Concept**: Since the daemon controls code execution and can re-JIT functions at any
time, it can collect profiling data from the running application and use it to
recompile hot functions with better optimization decisions — all while the application
is running.

**Implementation approach for the PoC**:

1. **Instrumentation**: The daemon instruments JIT-compiled code with lightweight
   counters (branch frequencies, call counts) during initial compilation. This adds
   minimal overhead (~2-5%) since counters are just memory increments.

2. **Profile collection**: Periodically (or on demand), the daemon reads the counters
   from the target process. Alternatively, use ETW/xperf to collect hardware
   performance counters externally without any code instrumentation.

3. **Re-JIT with profile data**: Feed the profile data back into the optimizer (even
   the minimal pass manager). For the PoC, this could be as simple as:
   - Reorder basic blocks in hot functions based on branch frequencies
   - Inline small, frequently-called functions
   - Improve register allocation hints for hot paths

4. **Hot-swap**: Replace the old function code with the optimized version, exactly
   like a source-code change would trigger a re-JIT.

**Why this is simpler than traditional PGO**: Everything stays in memory — no PGO
file format needed, no rebuild step, no profile merging. The daemon has the profile
data and the compiler in the same process. This is a natural extension of the JIT
architecture.

---

## 19. A VM for C++?

Yes, essentially. The **forgecc** daemon is a **C++ virtual machine** in the same
sense that the JVM is a Java VM or the CLR is a .NET VM:

- Source code goes in, the daemon compiles and executes it
- The application can only run while the daemon is running
- Code can be hot-patched at runtime
- The "VM" has full knowledge of types, symbols, and source locations (enabling the
  DAP debugger)
- Garbage collection is not needed (C++ manages its own memory)

The key difference: unlike JVM/CLR, **forgecc** compiles to **native x86-64 code**
(not bytecode), so there's no interpretation overhead. It's more like a "native JIT
runtime for C++" — similar to what Julia does for its language.

This framing helps explain the architecture: **forgecc** is to C++ what the JVM is
to Java — a runtime environment that compiles, loads, executes, and debugs code,
with the added benefit of persistent caching and distributed compilation. The
difference is that C++ code runs at full native speed with zero runtime overhead.

---

## 20. Future Work (Beyond the PoC)

These are out of scope for the proof-of-concept but are natural extensions of the
architecture. The design should not preclude them.

### 20.1 Console Development (PS5, Xbox Series X, Nintendo Switch)

Game studios target consoles alongside PC. The **forgecc** daemon architecture extends
naturally to console development via a **thin client on the devkit** that communicates
with the PC daemon over the network.

```
Developer PC                              Console Devkit (dev mode)
┌──────────────────────┐                  ┌──────────────────────┐
│   forgecc daemon     │                  │   forgecc-client     │
│                      │  HTTP+msgpack    │   (thin agent)       │
│  - Cross-compiles    │◄───────────────▶│                      │
│    to console arch   │  (same unified   │  - Receives sections │
│  - Content store     │   protocol §4.13)│  - Maps into process │
│  - Sema, IR, codegen │                  │  - Patches functions │
│  - DAP server        │                  │  - Debug primitives  │
│                      │                  │  - Sends back:       │
│  Targets:            │                  │    - crash dumps     │
│  - x86-64 (local PC) │                  │    - perf counters   │
│  - aarch64 (console) │                  │    - stdout/stderr   │
└──────────────────────┘                  └──────────────────────┘
```

**Key principles**:

- **All compilation stays on the PC.** The console client does NOT compile anything.
  Console devkits have limited CPU/RAM; the daemon already has all the infrastructure.
  The console client is purely a remote loader/patcher (~2,000–3,000 lines).

- **The console client is a remote `forge-daemon/loader.rs`.** It does what the local
  JIT loader does, but over the network: receives compiled machine code sections,
  maps them into the running game process, patches function stubs, registers exception
  tables, and manages static initializers.

- **The P2P protocol already supports this.** The console client is just another peer.
  It calls `GET /v1/artifact/{hash}` to fetch compiled sections. The only difference
  is that artifacts are compiled for a different target architecture. The content store
  key naturally includes the target triple, so `blake3(function_hash + "x86_64-pc-windows")`
  and `blake3(function_hash + "aarch64-ps5")` are different entries.

- **Lazy compilation still works.** When the game on the console calls an uncompiled
  function stub, the stub traps to the console client, which sends a compile request
  back to the PC daemon over the network. Latency is higher (~10–50ms round trip over
  LAN) but acceptable for development iteration.

**What changes from the PC-only design**:

- `forge-codegen` needs additional backends (aarch64 for PS5/Switch, x86-64 with
  different ABI for Xbox). ForgeIR is target-independent; only the lowering changes.
  Each backend is a separate module (~8,000–12,000 lines per target).
- `forge-daemon/loader.rs` splits into a local loader (current) and a remote loader
  protocol. The console client implements the remote side.
- The content-addressed store handles multi-target artifacts transparently.

**Debug info: DWARF vs. CodeView**:

Console targets (PS5, Switch) use **DWARF** debug info instead of CodeView/PDB.
However, this does **not** affect the DAP-first debugging strategy:

- In development mode, the daemon still acts as the DAP server. Debug info lives in
  Sema's internal format — it is never serialized to DWARF or CodeView during
  development. The daemon sends debug commands (set breakpoint, read memory, inspect
  variable) to the console client, which executes them via the console's debug API.
- DWARF vs. CodeView only matters for **release builds**, where `forge-link` would
  need to emit DWARF sections (`.debug_info`, `.debug_abbrev`, `.debug_line`, etc.)
  instead of CodeView/PDB. This is a separate code path in the linker, not in Sema
  or the DAP server.
- The codegen itself is unaffected — DWARF and CodeView are debug info formats, not
  code generation concerns. The same ForgeIR lowers to machine code regardless of
  which debug format will eventually be used.

**Existing devkit agents**:

Every console devkit already runs a debug agent that supports the operations
**forgecc**-client would need:

| Console | Debug Agent | Protocol | Key Capabilities |
|---|---|---|---|
| **PS5** | Target Manager + DECI5 | DECI5 over TCP | Memory read/write, breakpoints, module loading, process control, core dumps |
| **Xbox Series X** | XBDM / MSVSMon | TCP (ports 4020+) | Memory access, remote debugging, process attach, deployment |
| **Nintendo Switch** | GDB server (devkit) | GDB remote protocol | Breakpoints, memory inspection, step execution, variable inspection |

These agents already support **memory read/write** and **process control** — the
exact primitives **forgecc**-client needs for JIT patching. If the console maker
(Sony, Microsoft, Nintendo) wanted to support **forgecc** natively, they could extend
their existing debug agent with:

1. **Section mapping API**: Map arbitrary code sections into a running process's
   address space (similar to `VirtualAllocEx` on Windows)
2. **Function table registration**: Register exception handling / unwind info for
   JIT'd code (equivalent to `RtlAddFunctionTable`)
3. **TLS registration**: Register thread-local storage for dynamically loaded code

These are small extensions (~500–1,000 lines each) to agents that already handle
memory writes and process control. The alternative is for **forgecc**-client to
implement these on top of the existing memory read/write primitives, which is also
feasible but less efficient.

### 20.2 Artifact Extraction and Release Builds

During development, **forgecc** keeps everything in the content-addressed store and
the daemon's memory. For shipping builds, artifacts must be extracted into standard
formats:

**Object file extraction** (`.obj` / `.o`):

- The daemon can emit standard COFF `.obj` files (Windows) or ELF `.o` files
  (console/Linux) from its internal section records
- This enables interoperability with existing toolchains — e.g., link **forgecc**-compiled
  `.obj` files with MSVC's `link.exe` or console-specific linkers
- Useful for: CI/CD pipelines, build verification, mixing **forgecc** output with
  third-party prebuilt libraries

**Executable generation** (`.exe` / `.dll` / console executables):

- `forge-link` (Phase 8) handles PE/DLL generation for Windows
- Console targets would need additional linker output formats (ELF for PS5/Switch,
  XEX-like for Xbox)
- PDB generation (Windows) and DWARF emission (console/Linux) are only needed here

**The key insight**: release build generation is a **batch export** from the content
store, not a separate compilation. All the compiled artifacts already exist in the
store from development iteration. The release pipeline is:

1. Collect all section/symbol/relocation records from the store for the target
2. Apply release-mode optimizations (if any — beyond PoC scope)
3. Run the linker to produce the final executable + debug info
4. Sign/package for the target platform

This means a release build after a full development session could be **very fast** —
most of the compilation work is already done and cached.

### 20.3 Rust Front-End

A Rust front-end for forgecc is a natural long-term extension. The motivations are
complementary:

- **Bootstrapping**: forgecc is written in Rust. A Rust front-end lets forgecc
  compile itself, closing the bootstrap loop and providing the ultimate dogfooding
  scenario — every improvement to the compiler immediately benefits its own build.
- **Mixed C++/Rust codebases**: Game engines are increasingly adopting Rust for
  systems-level code (gameplay scripting, tooling, asset pipelines). A single daemon
  that compiles both C++ and Rust, sharing the same content-addressed store and
  memoization infrastructure, eliminates the overhead of coordinating two separate
  toolchains. Cross-language LTO and unified debug info become possible.
- **Faster Rust compilation**: Rust's compile times are a well-known pain point.
  forgecc's memoization, daemon architecture, and distributed caching apply equally
  to Rust — the front-end changes, but the entire backend (ForgeIR, codegen, store,
  JIT, DAP debugger) is reused.

**What a Rust front-end shares with the C++ front-end**:

| Component | Shared? | Notes |
|---|---|---|
| Content-addressed store | 100% | Language-agnostic |
| Memoizer | 100% | Same hash-in/hash-out model |
| ForgeIR | 100% | Both languages lower to the same IR |
| x86-64 codegen | 100% | Same backend |
| Runtime loader / JIT | 100% | Same patching mechanism |
| DAP debugger | 100% | Same debug info model |
| Linker | 100% | Same COFF/PE output |
| Daemon / HTTP server | 100% | New `/v1/compile-rs` endpoint |
| Distributed P2P | 100% | Same artifact format |
| Lexer | 0% | Completely different token set |
| Parser | ~10% | Different grammar; some AST node shapes overlap (functions, structs, enums) |
| Sema | ~20% | Type checking, trait solving, and borrow checking are unique to Rust; name resolution and overload-like dispatch (trait method selection) share concepts |

**What a Rust front-end requires that the C++ front-end does not**:

1. **Borrow checker** — Rust's defining feature. Implementing Polonius/NLL-style
   borrow checking is a multi-year effort. The borrow checker is a dataflow
   analysis over MIR (Mid-level IR), which maps to a ForgeIR analysis pass.
   Estimated: ~15,000–20,000 lines.

2. **Trait solver** — Rust's trait system (coherence, specialization, auto traits,
   negative impls, associated types with bounds) is comparable in complexity to
   C++ template instantiation + overload resolution combined. Estimated:
   ~8,000–12,000 lines.

3. **Proc macro execution** — Rust's procedural macros are arbitrary Rust code
   that runs at compile time. Two options: (a) run `rustc`-compiled proc macro
   `.so`/`.dll` files directly (pragmatic, requires ABI compatibility with
   `rustc`), or (b) compile proc macros with forgecc itself (requires the Rust
   front-end to be self-hosting first). Option (a) is the practical starting
   point.

4. **Pattern matching exhaustiveness** — Rust requires exhaustive pattern matching.
   This is a well-understood algorithm (Maranget's approach) but still ~2,000–3,000
   lines.

5. **Lifetime elision and inference** — Implicit lifetime rules that are unique to
   Rust. ~1,000–2,000 lines.

**Estimated scope**: ~30,000–40,000 lines for a Rust front-end (parser + sema +
borrow checker + trait solver), on top of the shared infrastructure. This roughly
doubles the front-end code but adds zero backend code.

**Phasing**: The Rust front-end is strictly post-PoC. The implementation order:

1. Rust lexer + parser (produces Rust AST)
2. Name resolution + type checking (without borrow checker — produces typed IR)
3. Lower to ForgeIR (at this point, Rust code compiles and runs but without
   borrow checking — similar to running with `unsafe` everywhere)
4. Borrow checker (the hard part — but the code already compiles and runs,
   so this is a verification pass, not a blocking dependency)
5. Proc macro support (load `rustc`-compiled proc macro crates)
6. Self-hosting (forgecc compiles itself)

**Precedent**: The `gccrs` project (Rust front-end for GCC) has been in development
since 2014 and is still not production-ready, illustrating the difficulty. However,
forgecc has a structural advantage: the entire backend is already built and tested
by the time the Rust front-end starts. `gccrs` had to integrate with GCC's backend
simultaneously. forgecc's Rust front-end is purely additive — it produces ForgeIR,
and everything downstream works unchanged.

---

## 21. N4950 Cross-Reference (C++23 Standard Draft)

This section documents the results of a systematic cross-reference between this design
document and N4950 (the C++23 working draft, which includes all C++20 features).
The cross-reference was performed chapter-by-chapter against the standard's core
language clauses (§5–§15). Items identified as gaps have been incorporated into the
relevant sections above.

### 21.1 Coverage Summary by Standard Chapter

| N4950 Chapter | Status | Notes |
|---|---|---|
| §5 Lexical conventions | ✅ Covered | Added: raw strings, UDL tokens, hex floats, all encoding prefixes, phases of translation (line splicing, BOM, string concat) |
| §6 Basics (ODR, scope, lookup, types) | ✅ Covered | ODR handled by content-addressed store; scope/lookup in Sema; added two-phase lookup |
| §7 Expressions | ✅ Covered | Added: noexcept operator, brace-init lists, UDL expression resolution |
| §8 Statements | ✅ Covered | if constexpr, if consteval, range-for, structured bindings |
| §9 Declarations | ✅ Covered | Added: constinit, explicit(bool), extern "C", static_assert, alignas, #pragma pack effects, bit-fields, NSDMI |
| §10 Modules | ⬜ Not needed | UE5 has zero C++20 module usage |
| §11 Classes | ✅ Covered | Added: bit-fields, defaulted comparisons, aggregate init, NSDMI |
| §12 Overloading | ✅ Covered | Added: UDL operator resolution, built-in operator candidates, address of overloaded function |
| §13 Templates | ✅ Covered | Added: CTAD/deduction guides, template template params, class-type NTTPs, two-phase lookup, variable template instantiation |
| §14 Exception handling | ✅ Covered | SEH + C++ exceptions; added: noexcept as part of function type |
| §15 Preprocessing | ✅ Covered | Added: #pragma pack/warning, #line, #error, #warning, __has_builtin, __has_feature, feature-test macros, predefined macros |
| §16–§33 Library | N/A | We link against MSVC STL; must parse STL headers but don't implement the library |

### 21.2 Intentionally Unsupported (with Justification)

| Feature | N4950 Section | Reason |
|---|---|---|
| C++20 Modules | §10 | Zero usage in UE5; memoization replaces the build-time benefit |
| Coroutines | §7.6.2.4, §17.12 | Zero usage in UE5 Engine/Source/Runtime |
| `char8_t` as distinct type | §6.8.2 | Recognized by lexer; full type system support deferred (UE5 uses `TCHAR`/`wchar_t`) |
| Extended float types (`std::float16_t`, etc.) | §6.8.3 | Conditionally-unsupported; lexer recognizes suffixes but rejects them |
| `std::source_location` compiler magic | §17.8 | Requires `__builtin_source_location()`; deferred to MSVC STL compat phase |

### 21.3 Key Complexity Areas (from Standard Cross-Reference)

These areas of the standard are disproportionately complex relative to their
specification length and deserve extra attention during implementation:

1. **Overload resolution (§12.2)** — 26 pages in the standard; the most intricate
   single algorithm in C++. Implicit conversion sequences, partial ordering of
   function templates, and concept-constrained candidates all interact.

2. **Template argument deduction (§13.10.3)** — 21 pages; deduction from function
   calls, partial ordering, CTAD, and SFINAE rules are deeply intertwined.

3. **Constant expressions (§7.7)** — 11 pages of rules defining what is and isn't
   allowed in constexpr context. The JIT approach (§4.6a) simplifies this significantly
   but UB detection still requires careful instrumentation.

4. **Initialization (§9.4)** — 17 pages covering direct-init, copy-init, list-init,
   aggregate-init, reference-init, and their interactions with constructors, conversion
   operators, and `std::initializer_list`. This is where many real-world compilation
   failures occur.

5. **Name lookup (§6.5)** — 13 pages; unqualified, qualified, ADL, and the interaction
   with `using` declarations/directives. Two-phase lookup in templates adds another
   layer.

---

## 22. Success Criteria

The proof-of-concept is successful if:

1. ✅ Game editor builds and runs via the **forgecc** daemon
2. ✅ Incremental rebuild after a single-line change completes in < 2 seconds
   (for a project of Game's scale)
3. ✅ Two machines share compilation work via the P2P network
4. ✅ Full rebuild time is within 3x of Clang `-O0` (acceptable for unoptimized PoC)
5. ✅ JIT mode: change a function, see the result in the running editor within seconds
6. ✅ No user-managed PCH, unity builds, or module maps — caching is fully automatic
7. ✅ No Sema source file exceeds 1,000 lines
8. ✅ All compilation stages support parallelism
9. ✅ Debugging works via DAP in VS Code (breakpoints, stepping, variable inspection)
10. ✅ Compilation is fully deterministic (`--verify-determinism` passes)
11. ✅ The application only runs through the daemon — no standalone .EXE needed for dev
12. ✅ All incremental target codebases (§8a) levels I–VI pass `forge-verify` (§8b)
13. ✅ `consteval` functions execute via in-process JIT, not an AST interpreter
