# ADR 001 - Principles of System Architecture (Capability Edition)

**Status**: Accepted
**Date**: 2026-10-18

## Context

We are aiming for a composable and maintainable architecture to support organic evolution of a large, complex Go library — a **capability** — imported by other Go programs as one or more modules. The following principles define the system architecture and structure of the solution, adapting the hexagonal/onion architecture cornerstones of an application to a library, as required by the project constitution (Principles III, VII, IX, XIII).

A capability differs from an application in the structural ways that matter for architecture:

1. **There is no `main` and no process of its own.** The library is driven by the embedding application, in that application's process. The library owns neither the entry point, nor the terminal, nor the environment, nor the exit code.
2. **The deliverable is the Go API.** The embedding application programs against the library's exported types. Part of the architecture is therefore necessarily public, and whatever is public is a compatibility promise (constitution Principle XIV).
3. **Dependencies are inherited.** Every module the library requires is compiled into, downloaded by, and `go.sum`-pinned by every consumer. The physical module layout is therefore an architectural concern, not a packaging detail.

The principles below account for all three.

## Decision

We apply three cornerstones for evolving system architecture:

1. [Screaming architecture](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html) philosophy that highlights use-cases over framework structure. The processes the library performs are the use-cases; reading the package tree names what the library does before it names how.
2. [Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/) supports a composable solution, equally drivable by the embedding application and by alternative public front-ends, evolved independently from any specific technology underneath (storage substrate, transport, object store, clock).
3. [Onion architecture](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/) organizes domain logic through composable abstraction of the hexagon, driving dependencies inward, toward the domain.

A fourth, library-specific constraint governs visibility:

4. **The domain core and the ports are public; the use-cases and the domain libraries are internal.** The embedding application programs against the vocabulary and the ports; nothing outside the library may depend on how a process is carried out. The layout states this rather than hides it.

The solution organically consists of:

1. **Domain** is the core building blocks — the public vocabulary of the problem domain, its finite algebras, types and protocols.
2. **Application Services** contain the logic to unfold the library's processes (use-cases). Use-cases are first-class citizens in the architecture (see screaming architecture). Each use-case is a cohesive internal package under `/internal/app`.
3. **Ports & Adapters**: **primary ports** are the public API the library is driven through; **primary adapters** drive it (public front-ends shipped with the library, and the embedding application itself). **Secondary ports** are the public interfaces the library drives the outside through; **secondary adapters** implement them (in-module, in nested modules, or supplied by the consumer).
4. **Composition Root** is one public package that constructs use-cases and connects their ports. It is the library's equivalent of an application's `cmd/`.

### Rings

Dependencies point inward through four rings, and only inward:

| Ring                  | Packages                                                                           | Visibility | Holds                                                                                        |
| --------------------- | ---------------------------------------------------------------------------------- | ---------- | -------------------------------------------------------------------------------------------- |
| 1. Domain core        | root package `{name}`, `coreX/`                                                    | public     | the vocabulary of the domain and its finite algebras: types, total orders, protocols         |
| 2. Domain libraries   | `internal/pkg/*`                                                                   | internal   | generic, pure logic over the core: encodings, formats, data structures — no I/O of their own |
| 3. Application        | `internal/bios`, `internal/app/*`                                                  | internal   | the use-cases, their narrow private ports, and the shared kernel they exchange               |
| 4. Ports and adapters | `facadeX/`, `portX/`, `portX/portXtest`, `portX/memory`, `frontX/`, nested modules | public     | the primary port and composition root, secondary ports, conformance suites, every adapter    |

**Import rules** (enforced mechanically in CI where feasible):

| Package kind                           | MAY import                                                         | MUST NOT import                  |
| -------------------------------------- | ------------------------------------------------------------------ | -------------------------------- |
| Domain core (ring 1)                   | stdlib, ring 1                                                     | anything else                    |
| Domain library (ring 2)                | stdlib, ring 1, other ring 2                                       | rings 3–4                        |
| Shared kernel `internal/bios` (ring 3) | stdlib, rings 1–2                                                  | `internal/app/*`, ring 4         |
| Use-case `internal/app/X` (ring 3)     | stdlib, rings 1–2, `internal/bios`                                 | other use-cases, ring 4          |
| Secondary port `portX` (ring 4)        | stdlib, ring 1                                                     | `internal/*`, adapters           |
| Secondary adapter (ring 4)             | stdlib, ring 1, its port, technology modules (nested modules only) | `internal/app/*`, other adapters |
| Composition root `facadeX` (ring 4)    | everything inward, secondary ports, in-module adapters             | nested modules, primary adapters |
| Primary adapter `frontX` (ring 4)      | stdlib, ring 1, the composition root's exported API                | `internal/*`                     |

## Neglected

Standard package layouts do not provide additional clarity about fine-grained configuration:
- [Standard Go Project Layout](https://github.com/golang-standards/project-layout?tab=readme-ov-file)
- [Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)

We neglected a **flat, single-package library** (all types in the root package). It is the right choice for small libraries (see the companion *library* constitution), but it does not scale once a library has several independent processes and several interchangeable technologies: every use-case sees every other use-case's internals, and every technology dependency lands in the one module consumers import.

We neglected a **`pkg/` directory** for public packages. In a library every non-`internal` package is public already; `pkg/` adds a path segment to every import without adding information.

We neglected **one Go module per package**. Module boundaries carry a release and tagging cost; they are introduced only where they isolate a dependency tree (nested adapter modules), not as a default.

Our approach is a combination of multiple architecture patterns; none dominates in our solution:
- [Screaming architecture](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)
- [Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/), [Hexagonal Architecture in Go](https://medium.com/@matiasvarela/hexagonal-architecture-in-go-cfd4e436faa3)
- [Onion architecture](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/)
- [Go Modules Reference](https://go.dev/ref/mod) — multi-module repositories

## To Achieve

Architectural simplicity and maintainability of a library whose processes and technologies grow independently, with a public surface that stays small and stable while internals evolve, and a dependency footprint that each consumer pays only for what it imports.

## Accepting

The downside of the pattern is extra complexity in the definition of interfaces, structs, and conversion of domain models (aka boilerplate) — and, specific to a library, two kinds of duplication: a use-case's private port repeats a subset of a public secondary port, and nested modules carry their own `go.mod` and release tags. We accept this cost because it keeps every use-case testable against in-memory adapters with no network and no credentials, keeps the public API independent of internal process structure, and keeps heavy technology dependencies out of consumers that do not use them.

## Implementation notes

### Domain

For us, domain is **the core** building blocks. It represents real-world concepts (entities and types) in the problem domain. The core domain defines the common vocabulary of data structures interpretable by humans and machines in the context of the library and of every application embedding it. Ideally, the compiler acts as a theorem prover for the library.

The domain core is **public**: it is the vocabulary consumers program with. The root package of the module (named after the module) holds the central types; further core packages (`coreX/`) hold vocabularies that are large enough to warrant their own namespace (e.g., a query algebra, a cursor algebra). Core packages import only the standard library and each other.

We do not see the domain as a monolithic core; screaming architecture is also applicable there. The generic logic over core types — encodings, on-disk formats, data structures, interning — is organized into domain libraries (aka domain services), each maintaining the finite algebra for a given type or problem. These are **internal** and perform no I/O of their own; they operate on byte slices, `io.Reader`/`io.Writer`, and iterators supplied by their callers.

The evolution of domain logic passes through a few phases of development:
1. `/internal/pkg`: initially, generic domain logic is internal to the library — free to change in every release.
2. root or `coreX/`: domain logic that consumers must reason about (e.g., an ordering they rely on, a codec they need to read data independently) is promoted to the public core, accepting the compatibility promise that comes with it.
3. `github.com`: the pinnacle of evolution is promotion of a domain library to a reusable open-source module with its own lifecycle — this is how a data structure or format proven inside one capability becomes a general-purpose Go library.

### Application Services

The library **only implements** a process because a consumer needs it. Application services contain the logic to unfold these processes (aka use-cases): e.g., `commit`, `read`, `flush`, `compact`, `recover`. Each use-case is one package under `/internal/app`, named after the process it performs.

Use-cases are strictly decoupled. A use-case has no direct knowledge of, or dependency on, any other use-case. It has no reference to any fine-grained code unit from another use-case — not even structs or interfaces.

We use Dependency Injection (no "foreign" services are instantiated inside the use-case) and Dependency Inversion (a use-case depends on abstractions, not concrete implementations) to decouple use-cases. Go interfaces give us effective Dependency Injection/Inversion within use-cases, so the architecture can evolve — for example, swapping a storage substrate for an in-memory adapter in tests, or adding a second substrate — without touching the use-case itself.

Only Go structs violate this principle — it is impossible to maintain the "no reference" rule unless the structure is defined via an interface or shape abstraction. We solve this with restrictions:
1. Only domain types are allowed within use-case interfaces.
2. Alternatively, a shared kernel as application-core functionality decouples dependencies through definition of basic input/output specification objects and global ports (`/internal/bios`). The shared kernel is where use-cases exchange state (e.g., an in-memory table written by `commit` and drained by `flush`) without importing each other.

A use-case can be generalized as a process:
- use a port to retrieve one or several instances of domain types (from a substrate, a cache, a stream);
- tell those instances to do some domain logic;
- use a port to persist or emit the result.

A use-case returns domain values and errors from the port's exported taxonomy. It never logs on its own initiative, never reads configuration from the environment, and never starts a goroutine it does not own (constitution Principles X, XI). Configuration reaches it as plain values from the composition root.

Each use-case package is structured as:
- `component.go` — the use-case's entry point: its constructor (taking its private ports and configuration as plain values) and the operations the composition root calls;
- `port/` — the narrow secondary ports this use-case needs, private to it;
- `service/` — the implementation of the use-case logic, when it outgrows `component.go`;
- `kernel/` — domain objects and logic specific to this use-case only, when needed.

### Ports & Adapters

In the hexagonal architecture, when any driver wants to use the functionality at a port, it sends a request that is converted by an adapter, for the specific technology of the driver, into a usable procedure call, which passes further inward. The library is blissfully ignorant of the driver's technology. When the library has something to send out, it sends it out through a port to an adapter, which produces the appropriate signal for the receiving technology.

**Primary side.** The primary port is the public API of the composition root (`facadeX/`): the interfaces and concrete types the embedding application calls (e.g., `Store`, `Snapshot`, `Reader`). Primary adapters are public packages shipped with the library that offer an alternative front-end over the same primary port — an encoding, a query language, a protocol binding (`frontX/`). A primary adapter calls the primary port only; it never imports a use-case. The embedding application is itself a primary adapter.

**Secondary side.** Secondary ports are public packages (`portX/`) declaring the interfaces the library drives the outside through: storage substrate, external index, placement, clock. They are public because the embedding application MUST be able to implement them. Each secondary port package ships with:
1. an in-memory reference adapter (`portX/memory`), standard library only;
2. a conformance suite (`portX/portXtest`), an exported test helper every adapter MUST pass;
3. the exported error constants that form the port's error taxonomy.

In Go, an interface belongs in the package that uses values of that type. We honor this at two levels: the library as a whole is the consumer of a public secondary port, so the port is declared by the library; and each use-case is the consumer of its private port, so the use-case declares the narrow subset it needs in its own `port/` package. The public port's adapters satisfy the private ports structurally — the composition root passes them through without conversion code.

**Port isolation rules** (to prevent cross-use-case coupling):

1. **A use-case's `port/` package is private to that use-case.** Other use-cases MUST NOT import it. If use-case B needs a behaviour that use-case A's infrastructure provides, use-case B defines its own narrow port interface (Interface Segregation Principle) and the composition root connects the two.

2. **Ports must be as narrow as the use-case's actual need.** If a use-case only calls `Get`, its port declares only `Get` — not `Put`, `Delete`, `Match`. Copying a broader interface from the public port, or from a vendor SDK's client interface, creates an implicit coupling and violates ISP (see constitution Principle VII).

3. **Ownership of a side-effecting, non-idempotent operation follows the use-case that performs it.** The use-case that calls a creating, deleting, or otherwise state-mutating operation through a port is the sole owner of that call. Callers MUST NOT repeat it speculatively "to be safe". If another use-case needs to know whether the operation already happened, the owning use-case exposes a query for that through the shared kernel, not an invitation to call the mutating operation again.

The evolution of adapters passes through a few phases of development:
1. `internal/app/X/port` + test fake: initially, a technology is a need of one use-case, served by a fake in that use-case's tests.
2. `portX/` + `portX/memory`: once the technology is a pluggable concern of the library, the port is made public with its in-memory reference adapter and conformance suite.
3. `portX/technologyY/` (nested module): adapters that depend on anything beyond the standard library and the mandated libraries live in their own Go module, so consumers of the root module never inherit their dependency tree.
4. `github.com`: the pinnacle of evolution is promotion of an adapter to a separate open-source repository, maintained against the port's conformance suite.

### Composition Root

Exactly one public package — the composition root (`facadeX/`) — constructs use-cases and connects their ports. It is the library's equivalent of an application's `cmd/`:
- its constructor `New(<required secondary adapters>..., cfg Config) (*FacadeX, error)` takes required secondary adapters as positional parameters, and every optional setting — `*slog.Logger`, clock, limits, optional adapters — as fields of an exported `Config` struct whose zero values mean documented defaults (constitution Principle IX); functional options are not used;
- it validates `Config` once and passes each use-case only the plain values it needs — `Config` itself never crosses into `internal/`, so internal settings never become public API;
- it instantiates the shared kernel and every use-case, passing each the adapters that satisfy its private ports;
- it implements the primary port by delegating to use-cases, and owns the lifecycle (`Close`) of every goroutine and resource the use-cases hold;
- it translates nothing: domain types flow through unchanged, and errors are already in the exported taxonomy.

No other package — not a primary adapter, not a use-case, not a test outside the composition root's own package — instantiates a use-case.

### Modules & Release

A capability is a multi-module repository:
- **The root module** (`github.com/{org}/{name}`) contains the domain core, the internal rings, the composition root, the public secondary ports with their in-memory adapters and conformance suites, and primary adapters that depend on the standard library only. Its `go.mod` requires only the mandated libraries and justified, lightweight dependencies.
- **Nested modules** (`portX/technologyY/go.mod`) contain secondary adapters with heavy technology dependencies (cloud SDKs, database drivers). They require the root module by released version, import its conformance suite in their tests, and are tagged independently as `portX/technologyY/vX.Y.Z`. The root module never requires a nested module.
- **`examples/`**, if present, is its own module when the examples need dependencies the library does not.

Releases are tags created by CI per changed module, with GitHub Releases generated from commit history. There are no binaries. Breaking changes of any module follow the import compatibility rule (`/v2`) per constitution Principle XIV.

### Project Layout

```
root/                                 # module github.com/{org}/{name}
├── doc.go                            # package {name}: overview, architecture map, quick start
├── {name}.go                         # ring 1: central domain types of the vocabulary
├── {name}_test.go
├── coreX/                            # ring 1: further public domain vocabulary (e.g. query, cursor)
│   └── ...
│
├── facadeX/                          # ring 4: composition root + primary port (public API)
│   ├── doc.go
│   ├── facadeX.go                    # New(opts...), primary port methods, Close
│   ├── config.go                     # Config struct, documented zero-value defaults, validation
│   ├── errors.go                     # exported error constants (faults.Type)
│   ├── featureX_test.go              # black-box acceptance tests, package facadeX_test
│   └── example_test.go               # testable examples rendered on pkg.go.dev
│
├── portX/                            # ring 4: public secondary port
│   ├── portX.go                      # small interfaces over domain types
│   ├── errors.go                     # the port's exported error taxonomy
│   ├── memory/                       # in-memory reference adapter, stdlib only
│   │   ├── memory.go
│   │   └── memory_test.go            # runs portXtest against the in-memory adapter
│   ├── portXtest/                    # conformance suite every adapter MUST pass
│   │   └── portXtest.go
│   └── technologyY/                  # secondary adapter with heavy deps: NESTED MODULE
│       ├── go.mod                    # requires github.com/{org}/{name} by version
│       ├── go.sum
│       ├── technologyY.go
│       └── technologyY_test.go       # runs portXtest; live tests behind a build tag
│
├── frontX/                           # ring 4: primary adapter, drives facadeX's public API
│   └── ...
│
├── internal/
│   ├── pkg/                          # ring 2: generic domain libraries, pure, no I/O
│   │   ├── libX/
│   │   │   ├── libX.go
│   │   │   ├── libX_test.go
│   │   │   ├── libX_fuzz_test.go     # mandatory for decoders/parsers
│   │   │   └── libX_bench_test.go    # mandatory for hot paths
│   │   └── ...
│   ├── bios/                         # ring 3: shared kernel exchanged between use-cases
│   │   └── ...
│   └── app/                          # ring 3: use-cases, one package per process
│       ├── usecaseX/
│       │   ├── component.go          # use-case constructor and operations
│       │   ├── port/                 # private, narrow secondary ports of this use-case
│       │   ├── service/              # use-case logic, when it outgrows component.go
│       │   ├── kernel/               # use-case-specific domain objects, when needed
│       │   └── README.md             # documentation of this process
│       └── ...
│
├── examples/                         # runnable example programs (own module if it needs deps)
├── testdata/                         # shared fixtures; fuzz corpora live next to fuzz tests
├── adrs/                             # architecture decision records
├── ARCHITECTURE.md                   # package-to-ring map, glossary, NFRs, hot paths
├── README.md
├── LICENSE
├── .github/workflows/                # build, check-code, check-test
├── go.mod
└── go.sum
```
