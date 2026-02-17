# **forgecc**: An Incremental, Distributed C++20 Compiler

**Status**: Proposal / Architecture Draft  
**Author**: Alexandre Ganea  
**Date**: 2026-02-15  

## 1. Executive Summary

**forgecc** is a proof-of-concept incremental C++20 compiler and linker, written in Rust,
running as a persistent daemon. Its primary goal is to dramatically reduce build times
for large C++ codebases (target: UE5 Editor / Game) by:

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
- No standalone compiler/linker tools — everything lives inside the daemon
- No Objective-C/OpenCL/CUDA
- No cross-compilation
- No LLVM IR compatibility — we define our own IR (potentially MLIR-based, see §4.6)
- No PCH — the memoization/caching system replaces PCH transparently
- No C++20 modules (UE5 does not use them; see §15 findings)

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
│             │              │                  │                       │
│  ┌──────────▼──┐  ┌───────▼────────┐  ┌──────▼───────────────┐      │
│  │  Build      │  │  P2P Network   │  │  Content-Addressed   │      │
│  │  Scheduler  │──│  (seeds,       │──│  Distributed Store   │      │
│  │             │  │   IPFS-like)   │  │  (local + remote)    │      │
│  └──────┬──────┘  └────────────────┘  └──────────┬───────────┘      │
│         │                                        │                   │
│         ┌────────────────────────────┐           │                   │
│         │   Compilation Pipeline     │           │                   │
│         │   (parallel throughout)    │           │                   │
│         │                            │           │                   │
│         │  ┌─────┐  ┌──────────┐    │           │                   │
│         │  │Lexer│─▶│Preproc-  │    │           │                   │
│         │  │     │  │essor     │    │           │                   │
│         │  └─────┘  └────┬─────┘    │           │                   │
│         │                │          │           │                   │
│         │         ┌──────▼──────┐   │           │                   │
│         │         │   Parser    │   │           │                   │
│         │         └──────┬──────┘   │           │                   │
│         │                │          │           │                   │
│         │    ┌───────────▼────────┐ │           │                   │
│         │    │  Sema (modular)    │ │  ┌──────┐ │                   │
│         │    │ ┌──────┐┌───────┐ │ │  │Memo- │ │                   │
│         │    │ │Lookup││Overld.│ │ │  │izer  │◀┘                   │
│         │    │ ├──────┤├───────┤ │─┼─▶│      │                     │
│         │    │ │Templ.││ Types │ │ │  └──────┘                     │
│         │    │ ├──────┤├───────┤ │ │                               │
│         │    │ │Layout││Mangle│ │ │                                │
│         │    │ └──────┘└───────┘ │ │                                │
│         │    └───────────┬───────┘ │                                │
│         │                │         │                                │
│         │         ┌──────▼──────┐  │                                │
│         │         │  IR Lower   │  │                                │
│         │         │  (ForgeIR)  │  │                                │
│         │         └──────┬──────┘  │                                │
│         │                │         │                                │
│         │         ┌──────▼──────┐  │                                │
│         │         │  x86-64     │  │                                │
│         │         │  CodeGen    │  │                                │
│         │         └──────┬──────┘  │                                │
│         │                │         │                                │
│         └────────────────┼─────────┘                                │
│                          │                                           │
│              ┌───────────┼───────────┐                              │
│              │           │           │                              │
│       ┌──────▼──────┐ ┌─▼────────┐ ┌▼──────────┐                  │
│       │   Linker    │ │ Runtime  │ │    DAP     │                  │
│       │ (release    │ │ Loader / │ │  Debugger  │◀── VS Code      │
│       │  mode only) │ │ JIT      │ │  Server    │                  │
│       └─────────────┘ └──────────┘ └────────────┘                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
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

### 4.3 Lexer & Preprocessor

The lexer and preprocessor are tightly coupled in C++ because the preprocessor operates
on tokens, and macro expansion can produce new tokens.

**Key design decisions**:

- **Lazy lexing**: Lex on demand as the preprocessor or parser requests tokens.
- **Include-once optimization**: Track `#pragma once` and include guards. If header
  content hash unchanged, skip re-lexing entirely.
- **Automatic header caching**: Any header's **fully parsed and type-checked result**
  is cached in the store — not just the preprocessed token stream, but the complete
  AST and type information. This is closer to an "automatic serialized AST per header"
  than traditional PCH. The key difference from PCH: the cache is per-header (not
  per-project), keyed by content + macro state, and requires zero user configuration.

**Preprocessor features needed** (based on UE5/Game codebase analysis):

- `#include` / `#include_next`
- `#define` / `#undef` (object-like and function-like macros, variadic)
- `#if` / `#ifdef` / `#ifndef` / `#elif` / `#else` / `#endif`
- `#pragma once`, `#pragma push_macro`, `#pragma pop_macro`
- `#pragma comment(lib, ...)`, `#pragma comment(linker, ...)`
- `__has_include`, `__has_cpp_attribute`
- `_Pragma`
- `__VA_ARGS__`, `__VA_OPT__`
- `__FILE__`, `__LINE__`, `__COUNTER__`, `__FUNCTION__`, `__FUNCSIG__`
- MSVC-specific: `__declspec(...)`, `__forceinline`, `__pragma`
- UE-specific: `UCLASS()`, `UPROPERTY()`, `GENERATED_BODY()` — standard macros, but
  the compiler must handle UHT annotations without issues

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

**C++20 features needed** (based on UE5/Game codebase analysis):

- Classes, structs, unions, enums (including `enum class`)
- Templates (class, function, variable, alias templates)
- Template specialization (partial and full)
- **Concepts and requires-clauses** — UE5 uses these! Found in `Core/Public/Concepts/`
  (`CSameAs`, `CDerivedFrom`, `CConvertibleTo`, etc.), `MassEntity`, `CoreUObject`,
  `TypedElementFramework`, and many more. ~200+ files with concept/requires usage.
- Lambdas (including generic lambdas, init-capture)
- `constexpr` / `constinit`
- `consteval` — used in D3D12RHI, VerseVM, CoreUObject (~20 files)
- Structured bindings
- `if constexpr`
- Fold expressions
- Designated initializers
- Three-way comparison (`<=>`) — used in abseil, D3D12RHI
- `using enum`
- Attributes (`[[nodiscard]]`, `[[likely]]`, `[[no_unique_address]]`, etc.)
- MSVC extensions: `__declspec`, `__forceinline`, `__int64`, SEH (`__try/__except/__finally`)
- C language parsing mode (for .c files)

**NOT needed** (confirmed from codebase analysis):
- Coroutines (`co_await`, `co_yield`, `co_return`) — **zero** occurrences found in
  Engine/Source/Runtime
- C++20 modules (`export module`, `import`) — **zero** occurrences found

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
│   │   └── using.rs        # Using declarations/directives
│   │
│   ├── types/              # Type system
│   │   ├── mod.rs
│   │   ├── repr.rs         # Type representation (~500 lines max)
│   │   ├── equivalence.rs  # Type comparison, compatibility
│   │   ├── conversion.rs   # Implicit/explicit conversions
│   │   ├── deduction.rs    # Template argument deduction, auto
│   │   └── qualifiers.rs   # const, volatile, ref qualifiers
│   │
│   ├── overload/           # Overload resolution
│   │   ├── mod.rs
│   │   ├── candidates.rs   # Candidate set construction
│   │   ├── ranking.rs      # Implicit conversion ranking
│   │   └── resolution.rs   # Best viable function selection
│   │
│   ├── templates/          # Template instantiation
│   │   ├── mod.rs
│   │   ├── instantiate.rs  # On-demand instantiation with memoization
│   │   ├── specialize.rs   # Partial/full specialization matching
│   │   └── sfinae.rs       # Substitution failure handling
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
│   │   └── builtins.rs     # Compiler built-in constexpr functions (__builtin_*)
│   │
│   ├── decl/               # Declaration processing
│   │   ├── mod.rs
│   │   ├── variables.rs
│   │   ├── functions.rs
│   │   ├── classes.rs
│   │   ├── enums.rs
│   │   └── namespaces.rs
│   │
│   ├── expr/               # Expression type-checking
│   │   ├── mod.rs
│   │   ├── binary.rs
│   │   ├── unary.rs
│   │   ├── call.rs
│   │   ├── member.rs
│   │   ├── cast.rs
│   │   └── lambda.rs
│   │
│   ├── stmt/               # Statement checking
│   │   ├── mod.rs
│   │   └── control_flow.rs
│   │
│   ├── msvc/               # MSVC-specific
│   │   ├── mod.rs
│   │   ├── layout.rs       # MSVC class layout computation
│   │   ├── mangle.rs       # Microsoft name mangling
│   │   ├── seh.rs          # SEH __try/__except/__finally
│   │   └── abi.rs          # Calling conventions, vtable layout
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
> compilation and execution happens in **Phase 5a**, after the JIT infrastructure from
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

### 4.6b ForgeIR — Intermediate Representation (MLIR Consideration)

**Should we use MLIR?**

MLIR (Multi-Level IR) is worth serious consideration. Pros and cons:

| | Custom ForgeIR | MLIR-based |
|---|---|---|
| **Simplicity** | Simpler to start | More infrastructure upfront |
| **Extensibility** | Must build everything | Dialects, passes, transformations for free |
| **Parallelism** | Must implement | MLIR has parallel pass infrastructure |
| **Optimization** | Must build pass manager | Can leverage existing MLIR passes |
| **Debugging** | Must build printer/verifier | Built-in verification, printing, testing |
| **Code size** | Smaller | Larger (MLIR dependency) |
| **Rust interop** | Native | FFI to C++ MLIR libraries (or Rust MLIR bindings) |

**Recommendation**: Start with a **simple custom ForgeIR** for the PoC. The IR is
SSA-based, typed, with explicit control flow. If we later need more sophisticated
transformations, we can lower to MLIR or adopt MLIR dialects. The key reason: MLIR
is a C++ library, and adding a C++ dependency to a Rust project adds significant
build complexity. For the PoC, keeping everything in pure Rust is more pragmatic.

However, we should **design ForgeIR to be MLIR-compatible** — use similar concepts
(operations, regions, blocks) so that migration is straightforward if needed.

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
│                 Target Process                   │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ main()   │  │ Foo::bar │  │ stub: Baz()  │  │
│  │ compiled │  │ compiled │  │ → trap to    │  │
│  │          │──│          │──│   daemon     │  │
│  └──────────┘  └──────────┘  └──────┬───────┘  │
│                                      │          │
└──────────────────────────────────────┼──────────┘
                                       │
                    ┌──────────────────▼──────────┐
                    │        forgecc Daemon        │
                    │                              │
                    │  1. Compile Baz()            │
                    │  2. Map into target process  │
                    │  3. Patch stub → real code   │
                    │  4. Resume execution         │
                    └──────────────────────────────┘
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

### 4.10 UBT Integration (Command-Passing Protocol)

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

UBT modifications needed: A new `ForgeccToolChain.cs` that constructs `ForgeccCompileAction`
objects and sends them to the daemon instead of spawning `cl.exe` or `clang-cl.exe`.

### 4.11 Daemon Architecture

The daemon is a long-running process that owns the local store and compilation pipeline.

```
forgecc-daemon
    │
    ├── HTTP Server (unified protocol — serves UBT, peers, and DAP)
    │   ├── /v1/compile      — build system submits compile actions
    │   ├── /v1/link         — build system submits link actions
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
with the determinism requirements in §17). This means the build index is portable if
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
mandatory, UDP may be blocked by corporate firewalls). HTTP/3 could be revisited later
if WAN-distributed builds become a use case.

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
UBT / ninja ──── 1 HTTP/2 connection ────▶ local daemon (:9473)
                 (hundreds of concurrent       │
                  compile streams)              │
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
| `/v1/compile` | POST | Submit compile action(s) | Build system (UBT, ninja) |
| `/v1/link` | POST | Submit link action (.obj paths resolved via build index) | Build system (UBT, ninja) |
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
its primary DAP transport — a thin `forgecc-dap` wrapper process connects to the
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
│   │   │   ├── lookup/           # Name lookup (5 files, ~2000 lines total)
│   │   │   ├── types/            # Type system (5 files, ~3000 lines total)
│   │   │   ├── overload/         # Overload resolution (3 files, ~2000 lines)
│   │   │   ├── templates/        # Template instantiation (3 files, ~3000 lines)
│   │   │   ├── concepts/         # C++20 concepts (2 files, ~1000 lines)
│   │   │   ├── consteval/        # Constant evaluator (2 files, ~2000 lines)
│   │   │   ├── decl/             # Declaration processing (5 files, ~3000 lines)
│   │   │   ├── expr/             # Expression checking (6 files, ~4000 lines)
│   │   │   ├── stmt/             # Statement checking (2 files, ~500 lines)
│   │   │   ├── msvc/             # MSVC-specific (4 files, ~3000 lines)
│   │   │   └── access.rs         # Access control (~500 lines)
│   │   └── Cargo.toml
│   │   # Total: ~40 files, ~25,000 lines, no file > 1000 lines
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
│   └── forge-verify/             # Verification CLI tool (see §8b)
│       ├── src/
│       │   ├── main.rs           # CLI entry point, argument parsing
│       │   ├── daemon_client.rs  # HTTP client to connect to forgecc daemon
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
│ has X123 │◄── get(X123) ──────────│ needs    │
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

---

## 7. Build System Integration

### 7.1 UBT Integration (Primary)

A new `ForgeccToolChain.cs` in UBT that:
1. Constructs `ForgeccCompileAction` objects from `CppCompileEnvironment`
2. Sends them to the **forgecc** daemon via HTTP (`POST /v1/compile`)
3. Receives compilation results (success/failure + diagnostics)
4. Reports results back to UBT's action graph

Based on the existing `VCCompileAction` structure, the data we need to send:
- Source file path
- Include paths (user + system + VC include paths)
- Preprocessor definitions
- Force-include files
- Language standard (C++20)
- Architecture (x64)
- PCH settings → **ignored by forgecc** (automatic caching replaces PCH)

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
> **Adjusted total**: The ~91 manual weeks could realistically compress to
> **~30–45 weeks** for one person working full-time with AI assistance. The largest
> gains come from the lexer/preprocessor, codegen, and linker phases; the smallest
> from Sema and JIT runtime correctness work.

### Phase 0: Infrastructure (Weeks 1–4)
- [ ] Cargo workspace setup
- [ ] `forge-store`: Content-addressed local store on redb
- [ ] `forge-common`: Diagnostics, source map, string interning, hashing
- [ ] `forge-daemon`: Basic daemon lifecycle, HTTP server (axum)
- [ ] `forge-net`: Seed-based peer discovery, basic DHT
- [ ] Determinism infrastructure: deterministic hasher, `--verify-determinism` mode

**Milestone**: Daemon starts, peers discover each other, store get/put works locally
and across network. All data structures use deterministic ordering.

### Phase 1: Frontend — Lexer & Preprocessor (Weeks 5–10)
- [ ] `forge-lex`: Full C/C++20 lexer
- [ ] `forge-pp`: Preprocessor with all required directives
- [ ] Automatic header caching (replaces PCH)
- [ ] Memoization of preprocessed output in the store

**Milestone**: Can preprocess UE5 header files correctly. Headers are automatically
cached. No PCH configuration needed.

### Phase 2: Parser (Weeks 11–18)
- [ ] `forge-parse`: Recursive descent parser for C and C++20
- [ ] AST definition covering UE5's usage patterns
- [ ] Concepts/requires-clause parsing
- [ ] Error recovery

**Milestone**: Can parse UE5/Game source files into an AST.

### Phase 3: Semantic Analysis (Weeks 19–34)
- [ ] `forge-sema`: Modular Sema (see §4.5 for structure)
- [ ] Name lookup, type checking (parallel where possible)
- [ ] Template instantiation with memoization
- [ ] Overload resolution
- [ ] Concept satisfaction checking
- [ ] MSVC class layout and name mangling
- [ ] `constexpr` basic evaluation (literal folding, simple expressions)
- [ ] `consteval` context detection — Sema records deferred `consteval` calls for
  later JIT evaluation (requires IR + codegen + JIT from Phases 4–5; see §4.6a)
- [ ] SEH support (`__try/__except/__finally`)

**Milestone**: Can type-check UE5/Game source files. No Sema file > 1000 lines.
`consteval` contexts are detected and recorded; full evaluation deferred to Phase 5a.

### Phase 4: IR & Code Generation (Weeks 35–46)
- [ ] `forge-ir`: SSA-based IR with SIMD intrinsic support
- [ ] Minimal passes (DCE, constant folding)
- [ ] `forge-codegen`: x86-64 lowering, register allocation, emission
- [ ] SSE/AVX intrinsic lowering
- [ ] COFF section generation
- [ ] Debug metadata generation (source maps, type info, frame layouts — stored in
  internal format for DAP; CodeView serialization deferred to Phase 6)
- [ ] Basic `__asm` block support

**Milestone**: Can compile simple C/C++ programs to working x86-64 code.

### Phase 5: Runtime Loader / JIT (Weeks 47–54)
- [ ] `forge-daemon/loader.rs`: Process creation, section mapping
- [ ] Lazy compilation stubs
- [ ] Incremental patching (function-level)
- [ ] Import library parsing (.lib) for Windows SDK / third-party DLLs
- [ ] Static initializer ordering and execution
- [ ] Exception handling registration (`RtlAddFunctionTable`)

**Milestone**: Can JIT-compile and run a simple C++ program. Changes are patched live.

### Phase 5a: `consteval`/`constexpr` via JIT (Weeks 55–57)
- [ ] `forge-sema/consteval`: Wire deferred `consteval` calls to the JIT pipeline
- [ ] Compile `consteval` functions to native code via forge-ir → forge-codegen
- [ ] Map compiled code into daemon's address space (`VirtualAlloc` + `PAGE_EXECUTE`)
- [ ] Execute and capture results; feed back into Sema as compile-time constants
- [ ] Lightweight sanitizer instrumentation (overflow, null deref, bounds checks)
- [ ] Sandboxed allocator for `constexpr` `new`/`delete` (C++20)
- [ ] Memoization of `consteval` results: `blake3(function_hash + args) → result`

**Milestone**: `consteval` functions compile and execute in-process. Results are
cached and reused across TUs. No AST interpreter needed.

### Phase 5b: DAP Debugger (Weeks 58–61)
- [ ] `forge-debug`: DAP protocol server (JSON-over-stdio)
- [ ] Breakpoint management (int3 patching)
- [ ] Single-stepping (hardware debug registers)
- [ ] Variable inspection (ReadProcessMemory + Sema type info)
- [ ] Stack unwinding
- [ ] VS Code launch.json integration

**Milestone**: Can debug JIT-compiled programs from VS Code. No PDB needed.

### Phase 6: Linker — Release Mode (Weeks 62–67)
- [ ] `forge-link`: Symbol resolution, section layout
- [ ] PE executable generation
- [ ] DLL generation (for UE5 module structure)
- [ ] Import library reading
- [ ] Incremental linking
- [ ] PDB generation (release mode only — deferred priority)

**Milestone**: Can link multi-file C++ programs into working PE executables and DLLs.

### Phase 7: UE5/Game Compatibility (Weeks 68–83)
- [ ] UBT integration (`ForgeccToolChain.cs`)
- [ ] Handle UE5-specific patterns (UHT-generated code, `*_API` DLL exports, etc.)
- [ ] Windows SDK header compatibility
- [ ] MSVC STL header compatibility (link against vcruntime.lib, ucrt.lib)
- [ ] Deterministic compilation verification on UE5 codebase
- [ ] Fix edge cases found while building Game
- [ ] Build Game editor successfully

**Milestone**: Game editor builds and runs via the daemon. Debugging works via DAP.

### Phase 8: Distributed Features (Weeks 84–91)
- [ ] Full DHT implementation with replication
- [ ] Cross-machine artifact sharing
- [ ] Benchmark: two machines sharing compilation work
- [ ] File watching and pre-compilation

**Milestone**: Two machines share compilation work via P2P network.

---

### 8a. Incremental Target Codebases

Rather than jumping from "Hello World" to UE5, **forgecc** is validated against a ladder
of real-world open-source C/C++ projects of increasing complexity. Each level serves
as a milestone gate: the compiler must produce correct, runnable output for level N
before moving to level N+1.

| Level | Project | ~LoC | Why | Key features exercised |
|---|---|---|---|---|
| 0 | Hand-crafted test suite | <1k | Controlled tests for each language feature | All, incrementally |
| 1 | [**stb libraries**](https://github.com/nothings/stb) (stb_image.h, etc.) | ~7k per lib | Pure C, single-file, no dependencies, widely used | C99, macros, function pointers |
| 2 | [**LMDB**](https://github.com/LMDB/lmdb) | ~12k | Pure C, real-world database, COFF/PE linking exercise | C99, Win32 API, threads |
| 3 | [**nlohmann/json**](https://github.com/nlohmann/json) (single-header) | ~25k | Header-only C++11/17, heavy templates, tests memoization | Templates, SFINAE, exceptions, STL |
| 4 | [**Dear ImGui**](https://github.com/ocornut/imgui) | ~70k | C++11, moderate complexity, Win32/DX backends, good for JIT testing | Classes, vtables, function pointers, Win32 |
| 5 | [**Google Test**](https://github.com/google/googletest) | ~30k | C++14, templates, macros, used by many projects | Templates, macros, exceptions, RTTI |
| 6 | [**LLVM lit test suite**](https://github.com/llvm/llvm-test-suite) (selected C/C++ tests) | varies | Known-good reference tests with expected outputs | Wide coverage |
| 7 | **UE5 Lyra demo** / Game editor | ~millions | Full target | Everything |

**How levels map to phases**:
- Levels 0–1 are achievable after Phase 4 (codegen) — pure C, simple linking
- Level 2 exercises the JIT loader (Phase 5) — threads, Win32 API
- Levels 3–5 require full Sema (Phase 3) + consteval (Phase 5a) — heavy C++ templates
- Level 6 provides regression coverage across all phases
- Level 7 is the final target (Phase 7)

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
| `forge-store` | 3,500 | Local store + distributed layer + dependency signature tracking (+500) |
| `forge-net` | 5,000 | HTTP server (axum), MessagePack/JSON content negotiation, P2P networking, DHT, gossip |
| `forge-lex` | 4,000 | Lexer + literal parsing (C + C++) |
| `forge-pp` | 6,000 | Preprocessor + macro expansion |
| `forge-parse` | 15,000 | Parser + AST definitions (C + C++) |
| `forge-sema` | 22,000 | Modular Sema (~40 files, none > 1000 lines); consteval via JIT saves ~3k lines vs interpreter |
| `forge-ir` | 5,000 | IR + minimal passes |
| `forge-codegen` | 14,000 | x86-64 lowering + encoding + SIMD |
| `forge-debug` | 3,000 | DAP server + breakpoints + variable inspection |
| `forge-link` | 6,000 | Linker + PE + DLL (PDB deferred to release mode) |
| `forge-daemon` | 5,500 | Daemon + scheduler + loader/JIT + determinism verification (HTTP server moved to forge-net) |
| `forge-verify` | 1,500 | Verification CLI: daemon client, reference compiler diff, execution diff (see §8b) |
| **Total** | **~92,500** | |

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

**Not implementing C++20 modules for the PoC.** Rationale:

- UE5/Game has **zero** usage of C++20 modules (`export module`, `import`)
- UE5's "modules" are its own build system concept (`.Build.cs`, `Private`/`Public`
  directories), not C++20 modules
- The purpose of C++20 modules is primarily build time improvement — which **forgecc**
  achieves through its memoization/caching system instead
- C++20 modules add significant compiler complexity (module dependency scanning,
  BMI generation, build ordering constraints)

If modules become needed later, the architecture supports them — the content-addressed
store naturally handles BMI caching.

---

## 15. UE5/Game Codebase Analysis Findings

### 15.1 C Code
- **4,335 .c files** in Engine/Source (all third-party: zlib, libpng, FreeType,
  Bink Audio, Rad Audio, ICU, etc.)
- Engine code itself is C++ only
- **forgecc** must support C compilation for these third-party libraries

### 15.2 Inline Assembly (`__asm`)
- **~23 files** in Engine/Source/Runtime use `__asm`
- Mostly in: audio codecs (Bink, Rad), platform detection, floating point state,
  AutoRTFM LongJump
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
- **22 files** in Engine/Source/Runtime
- Concentrated in: crash handling (`WindowsPlatformCrashContext.cpp` — 20 matches),
  D3D11/D3D12 device creation, thread management, stack walking
- **Important** for Windows stability, but not in hot paths

### 15.5 C++20 Features
- **Concepts/requires**: ~200+ files use concepts. UE5 defines its own concepts in
  `Core/Public/Concepts/` (CSameAs, CDerivedFrom, CConvertibleTo, CFloatingPoint,
  CIntegral, CPointer, etc.). Also used in MassEntity, CoreUObject, TypedElementFramework.
  **Must support.**
- **consteval**: ~20 files (D3D12RHI, VerseVM, CoreUObject). **Must support.**
- **Three-way comparison (`<=>`)**: Used in abseil (third-party) and D3D12RHI. **Should support.**
- **Coroutines**: **Zero** usage. Not needed.
- **C++20 modules**: **Zero** usage. Not needed.

### 15.6 MS Extensions
- `__declspec(dllexport/dllimport)` — pervasive via `*_API` macros (CORE_API, ENGINE_API, etc.)
- `__forceinline` — pervasive via `FORCEINLINE` macro
- `__declspec(align(...))` — used for SIMD alignment
- `__pragma` — used in various places
- `__int64`, `__cdecl`, `__stdcall` — Windows API interop
- **All critical, must support**

### 15.7 DLL Structure
- Game editor builds **~190 game modules** as DLLs (see `GameEditor.Target.cs`)
- Engine adds hundreds more (Core, Engine, RHI, Slate, etc.)
- Each module uses `*_API` macros for dllexport/dllimport
- `FModuleManager` handles dynamic loading
- Live Coding (Live++) patches running DLLs — limited to function body changes
- **Circular dependencies**: UE5 uses a two-step lib creation process for modules
  with circular dependencies — first create an import `.lib` (with exports only),
  then build the real DLL. **forgecc** must handle this in release mode; in JIT mode,
  circular dependencies are resolved naturally since all symbols live in one address space

### 15.8 UBT Compiler Invocation
- `VCToolChain.cs` (~3,979 lines) handles both MSVC and clang-cl modes
- Game already uses clang-cl (multiple `HVN_START(CLOUD-1590, Enable clang-cl for Game)` markers)
- `VCCompileAction.cs` structures compilation as action objects with all necessary metadata
- **Clang-cl mode is the better integration point** for **forgecc**

---

## 16. Resolved Questions

1. **UHT (Unreal Header Tool)**: UHT now runs in-process within UBT (previously had
   an out-of-process mode, no longer used). UHT generates `.generated.h` and
   `.gen.cpp` files from UCLASS/UPROPERTY/UFUNCTION macros. **forgecc** consumes these
   generated files as regular source — no special UHT handling needed. UHT runs as
   part of UBT before **forgecc** receives compile actions.

2. **Shader compilation**: Out of scope. HLSL shaders are compiled by
   ShaderCompileWorker, which is a completely separate pipeline. Nothing to do for **forgecc**.

3. **Debugger strategy**: DAP-first, no PDB for development builds (see §4.8). The
   daemon IS the debugger. PDB generation is deferred to release mode only.

4. **Memory budget**: Deferred. This is a PoC; memory optimization comes later.

## 17. Deterministic Compilation (Required)

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

**Verification**: The daemon should have a `--verify-determinism` mode that compiles
everything twice and asserts that all content hashes match. This catches determinism
bugs early.

## 18. JIT Runtime Details

### 18.1 Import Library Parsing

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

### 18.2 Static Initialization Order

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

### 18.3 Thread-Local Storage

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

### 18.4 Exception Handling

SEH and C++ exceptions require `.pdata`/`.xdata` sections and runtime support. In JIT
mode, we need to register unwind info with `RtlAddFunctionTable` or
`RtlInstallFunctionTableCallback` for each JIT-compiled function.

This is well-understood territory — LLVM's ORC JIT does the same thing. The key is
to call `RtlAddFunctionTable` after mapping code sections and before execution.

### 18.5 Virtual Function Tables

In JIT mode, vtables must be laid out correctly and patched when classes are recompiled.
If a class layout changes (new virtual function, reordered members), all vtable
references must be updated. This requires:
- Tracking which vtables reference which functions
- When a class is recompiled with a layout change, update the vtable and all objects
  that point to it
- For the PoC, layout changes could trigger a full restart of the target process
  (acceptable for a PoC; Live Coding has the same limitation)

---

## 19. Live PGO Injection (PoC Feature)

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

## 20. A VM for C++?

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

## 21. Future Work (Beyond the PoC)

These are out of scope for the proof-of-concept but are natural extensions of the
architecture. The design should not preclude them.

### 21.1 Console Development (PS5, Xbox Series X, Nintendo Switch)

Game studios target consoles alongside PC. The **forgecc** daemon architecture extends
naturally to console development via a **thin client on the devkit** that communicates
with the PC daemon over the network.

```
Developer PC                              Console Devkit (dev mode)
┌──────────────────────┐                  ┌──────────────────────┐
│   forgecc daemon     │                  │   forgecc-client     │
│                      │  HTTP+msgpack    │   (thin agent)       │
│  - Cross-compiles    │◄───────────────▶│                      │
│    to console arch   │  (same unified  │  - Receives sections │
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

### 21.2 Artifact Extraction and Release Builds

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

- `forge-link` (Phase 6) handles PE/DLL generation for Windows
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
12. ✅ All incremental target codebases (§8a) levels 0–5 pass `forge-verify` (§8b)
13. ✅ `consteval` functions execute via in-process JIT, not an AST interpreter
