# forgecc

An incremental, distributed C++20 compiler and linker, written in Rust, running as a persistent daemon.

## What is this?

**forgecc** is a proof-of-concept C++ compiler designed to dramatically reduce build times for large codebases (target: Unreal Engine 5). Instead of the traditional compile-then-link cycle that produces object files on disk, forgecc runs as a long-lived daemon that compiles, links, loads, and debugs C++ code entirely in-memory.

It is informed by LLVM/Clang's architecture but is a clean implementation, not a fork. It targets **C++20, x86-64, Windows** initially.

## Key ideas

- **Persistent daemon** — a single long-running process replaces the traditional spawn-per-file compilation model. Parsed headers, type information, and compiled code stay in memory across builds.

- **Content-addressed store** — every compilation artifact (preprocessed headers, AST fragments, IR, machine code) is stored in a local database keyed by BLAKE3 hash. Nothing is recomputed if the inputs haven't changed.

- **Fine-grained memoization** — caching happens at every stage (header, function, type level), not just at the translation-unit level. Changing one function recompiles only that function.

- **JIT execution** — the daemon acts as an OS loader, building the target application in-memory progressively and compiling functions on demand. No `.obj` or `.exe` files needed during development.

- **Runtime patching** — when source code changes, only the affected functions are recompiled and patched into the running process. Sub-second iteration times.

- **On-demand debug info** — no PDB or DWARF files during development. The daemon retains all compilation metadata and serves debug information live through a built-in [DAP](https://microsoft.github.io/debug-adapter-protocol/) server (VS Code compatible).

- **Peer-to-peer distribution** — compilation knowledge is shared across machines via a distributed hash table. If a colleague already compiled a function with the same inputs, the result is fetched over the network instead of recompiled locally.

- **Build system integration** — communicates with build systems (UBT, Ninja) over HTTP/2 using MessagePack (with JSON for debugging). Writes shallow `.obj` files to satisfy timestamp-based dependency tracking while keeping real artifacts in the store.

## Architecture at a glance

```
  Build system (UBT / Ninja)          VS Code (DAP)          Peer daemons
              │                            │                       │
              └────────── HTTP/2 (msgpack/json) ───────────────────┘
                                    │
                            forgecc daemon
                    ┌───────────────┼───────────────┐
                    │               │               │
              Build Scheduler   P2P Network   Content-Addressed
                    │          (seed-based)     Distributed Store
                    │                               │
              Compilation Pipeline ─────────────────┘
              Lexer → Preprocessor → Parser → Sema → IR → CodeGen
                    │
              Runtime Loader / JIT ──→ Target Process
                    │
              DAP Debug Server ──→ VS Code
```

## Status

**Proposal / Architecture Draft.** No code yet — the full design document is available at:

> **[docs/IncrementalCppCompiler.md](docs/IncrementalCppCompiler.md)**

The proposal covers the complete architecture, compilation pipeline, runtime loader, distributed protocol, phased implementation plan (~98 weeks / ~33–49 weeks with AI assistance), and validation strategy against a ladder of real-world C/C++ projects from single-file libraries up to UE5.

## Prior art

| Project | Relationship to forgecc |
|---|---|
| [CERN Cling](https://root.cern/cling/) | C++ interpreter/JIT on LLVM — proves C++ JIT is viable at scale |
| [Zapcc](https://github.com/yrnkrn/zapcc) | Caching compiler daemon — closest architectural precedent |
| [Live++](https://liveplusplus.tech/) | Binary hot-reload — proves runtime patching is production-viable |
| [SN-Systems prepo](https://github.com/SNSystems/llvm-project-prepo/wiki) | Content-addressed compilation database — proves DB-backed builds work at scale |
| [Circle](https://www.circle-lang.org/) | Single-developer C++ compiler — proves a small team can build a working C++ compiler |

## License

[Business Source License 1.1](LICENSE) (BSL 1.1)

- **Free for internal use** — you can use forgecc to compile, link, and debug your own software
- **No competing service** — you may not offer forgecc as a hosted/managed compilation service without a commercial license
- **Converts to open source** — on 2030-02-17 (or 4 years after each version's release, whichever is first), the license automatically converts to **Apache 2.0 with LLVM Exceptions**
