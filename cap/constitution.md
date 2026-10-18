
# {Name} Capability Constitution

This constitution governs the design and engineering of a large, complex Go
library — a **capability** — distributed as one or more Go modules and
imported by other Go programs. Wherever this document says
"the library" or "the capability", substitute the name of the project that
adopts it.

A capability is a library large enough to have use-cases of its own: several
independent processes (e.g., write, read, compact, recover), several
interchangeable technologies underneath (storage backends, transports,
encryptions, etc), and several public front-ends on top. Such a library is architected
like an application — hexagonal and onion — with one difference stated
openly: the domain core and the ports are **public**, because the embedding
application programs against them, while the use-cases are **internal**,
because nothing outside the library may depend on how a process is carried
out.

This constitution treats [Effective Go](https://go.dev/doc/effective_go),
[Go Code Review Comments](https://go.dev/wiki/CodeReviewComments), and the
[Go Modules Reference](https://go.dev/ref/mod) as authoritative for idiom and
module mechanics. Where this document is silent, those govern; where this
document is explicit, it MUST be followed even if a contributor's prior Go
experience suggests otherwise.

The [Mandatory Libraries & Tooling](#mandatory-libraries--tooling) section
fixes the concrete toolchain — error annotation, assertion library, static
analysis, API compatibility gate, required CI workflows. These are explicit,
non-substitutable choices, not illustrative examples a project is free to
swap out.

## Core Principles

### I. Architecture Documentation & Decision Records

Architecture and major decisions MUST be documented, and Architecture
Decision Records are BINDING.

**Rules**:
- [ARCHITECTURE.md](ARCHITECTURE.md) is the single source of truth for the library's architecture
- When a feature changes or touches architecture, [ARCHITECTURE.md](ARCHITECTURE.md) MUST be updated in the same PR
- Non-functional requirements (throughput, latency, allocation budget, memory ceiling, concurrency guarantees, supported Go versions, compatibility guarantees) MUST be documented in [ARCHITECTURE.md](ARCHITECTURE.md)
- [ARCHITECTURE.md](ARCHITECTURE.md) MUST list every package of the library with its onion ring (ADR 001) and whether it is public API or internal
- Major architectural decisions MUST be recorded in [adrs/](adrs/) as Architecture Decision Records
- ADR files MUST follow the format `NNN-decision-title.md` (e.g., `004-segment-file-format.md`)
- ADRs MUST include: Context, Decision, Consequences, Status (Proposed/Accepted/Deprecated/Superseded)
- **ADRs with status "Accepted" are BINDING**: all code MUST follow the patterns documented in accepted ADRs
- Deviation from an accepted ADR is NOT permitted without a new superseding ADR
- When an ADR establishes a pattern (e.g., port shape, `Config` field naming, error taxonomy, on-disk format), that pattern MUST be followed consistently across every package
- **Before implementing, read every ADR referenced in the plan's Constitution Check section**: verify no plan decision contradicts an accepted ADR. When a conflict is found, the ADR takes precedence; the plan MUST be corrected before implementation begins
- **Before adding a new secondary port or adapter, check whether one for that capability already exists**: a second, divergent port for the same external concern is an architecture violation
- **Resolving cross-cutting design tension by ignoring an ADR is FORBIDDEN**: if a plan or spec appears to contradict an ADR, the conflict MUST be raised explicitly and resolved via a new ADR, not by quietly diverging

**Rationale**: Contributors and AI agents require rapid, accurate understanding of architectural constraints to contribute effectively and maintain consistency. Making ADRs binding prevents architectural drift in a code base whose public surface, once released, cannot be taken back.

### II. Domain-Driven Design & Glossary Management

Domain concepts MUST be explicitly modeled, documented, and maintained in the glossary. In a library, the domain model *is* the public vocabulary.

**Rules**:
- This project follows Domain-Driven Design (DDD): the domain core (ADR 001 ring 1) is a set of public types and their finite algebras, independent of any technology the library drives
- When the domain model is updated or new domain concepts are introduced, they MUST be added to [ARCHITECTURE.md](ARCHITECTURE.md)
- All domain terms MUST be added to the Glossary section in [ARCHITECTURE.md](ARCHITECTURE.md), including: term name, definition, relationships to other domain concepts, and the Go identifier that implements it
- Ubiquitous language MUST be used consistently in package names, exported identifiers, GoDoc, and documentation — an exported type `Record` documented as "an entry" and passed as a parameter named `item` is a glossary violation
- **Before defining a new type inside a use-case or adapter, check the domain core for an existing type**: introducing `type Key []byte` inside `internal/app/flush` when the core already defines `Key` duplicates the domain model. Reuse the existing type or elevate a genuinely shared type into the core
- Domain types MUST make illegal states unrepresentable where Go allows it (distinct named types, unexported fields with validated constructors, enumerations as typed constants) so that the compiler acts as a theorem prover for the library

**Rationale**: Consumers learn a library through its exported identifiers and GoDoc. One name per concept, everywhere, is what makes a large API learnable.

### III. Hexagonal Architecture & Clean Boundaries

The library MUST separate its domain, its use-cases, and the technologies it drives using hexagonal/onion architecture with dependencies pointing inward only. [ADR 001](../../adrs/001-system-architecture.md) defines mandatory and binding principles of system architecture.

**Rules**:
- The code base is organized in four rings (ADR 001): **(1) domain core** — public; **(2) domain libraries** — `internal/pkg/`; **(3) application** — one package per use-case under `internal/app/`; **(4) ports and adapters** — public primary ports, the composition root, public secondary ports, and adapters
- **Dependencies point inward only.** A package MUST NOT import a package of an outer ring. The domain core imports only the standard library and itself
- **Use-cases are strictly decoupled**: a package under `internal/app/<usecase>` MUST NOT import another use-case, not even its types or interfaces
- **Exactly one public package is the composition root** (ADR 001): it alone constructs use-cases and connects their ports to adapters. No other package instantiates a use-case
- **Primary adapters (alternative front-ends, encoders, query languages) drive the library through the public primary port only** — never by importing `internal/app/*`
- **Secondary ports are public interfaces over domain types only**: vendor SDK types MUST NOT appear in port signatures. A port exposes verbs like `Get`, `Put`, `Match` over domain types, never the underlying client's request/response structs
- **No internal type may appear in an exported signature**: consumers cannot name it, document it, or mock it
- Import rules SHOULD be enforced mechanically in CI (e.g., a `go list -deps` check per ring or a lint rule), not by review alone
- The package structure MUST be explained in [ARCHITECTURE.md](ARCHITECTURE.md); if it is unclear or misleading, it MUST be clarified with the user and the document updated

**Rationale**: Hexagonal architecture keeps every use-case testable against in-memory adapters, lets new technologies plug in without touching domain logic, and lets the public surface evolve without exposing how a process is implemented.

### IV. Functional Programming Style

Functional Programming Style is NON-NEGOTIABLE. All Go code MUST follow a functional-first design philosophy emphasizing clarity through structure rather than commentary.

**Rules**:
- **No Inline Comments**: Code MUST NOT include inline or block comments. The only exception is GoDoc comments, which are MANDATORY for every exported identifier and every package (Principle XII). Readability must come from the code itself
- **Self-Explanatory Naming**: All identifiers MUST be descriptive and unambiguous. Variable names clearly express purpose and contents; function names describe what they do, not how; avoid abbreviations unless universally understood
- **Small, Focused Functions**: Functions MUST be short (not more than 25 lines), focused on a single responsibility, and easy to understand in isolation. If a function requires explanation, refactor it into smaller composable pieces
- **Composition Over Complexity**: Favor composing small functions over large, monolithic ones. Functions MUST be easily chainable. Prefer passing data through transformations rather than mutating shared state
- **Immutability by Default**: Functions MUST avoid mutating inputs whenever possible. Prefer returning new values. Limit side effects and isolate them at the boundaries of the system (adapters), never in the domain core or domain libraries
- **Readability as a Requirement**: Code MUST be understandable without external explanation. If naming and structure are insufficient to convey intent, the code must be rewritten rather than annotated
- **Segregation**: Go code MUST accept interfaces, return structs
- Where a measured hot path (Principle XI) requires in-place mutation or buffer reuse, it MUST be confined to an unexported function whose exported caller remains free of observable side effects

**Rationale**: Pure functions are testable without mocks, composable into pipelines, and predictable under concurrency — properties that matter most in a library embedded into processes its authors do not control.

### V. Code Quality & Simplicity

All contributors MUST follow SOLID principles and implement only what the current feature requires (YAGNI).

**Rules**:
- **Single Responsibility**: Functions, types, methods, and packages MUST exhibit natural cohesion. A package's name MUST describe its single purpose; catch-all packages (`utils`, `common`, `helpers`, `types`, `models`) are forbidden
- **Open/Closed**: Compose simple types into more complex ones using embedding. Types are open for extension via embedding but closed for modification. Embedding in an exported struct exports the embedded type's methods — do it deliberately
- **Liskov Substitution**: Express dependencies between packages in terms of interfaces, not concrete types. Prefer small, single-method interfaces so implementations faithfully satisfy their contract (*require no more, promise no less*)
- **Interface Segregation**: Functions and methods MUST depend only on the behavior they actually need. Accept the narrowest interface that satisfies requirements; return concrete structs
- **Dependency Inversion**: High-level packages MUST NOT import low-level packages directly. Concrete dependencies are pushed up to the composition root; lower rings depend only on interfaces. The import graph MUST be acyclic and wide rather than deep
- **YAGNI for public API**: an identifier is exported only when a current spec scenario requires a consumer to use it. Unexporting later is a breaking change; exporting later is not

**Rationale**: SOLID principles structure the library into packages with natural cohesion. YAGNI is stricter for libraries than for applications because every exported identifier is a permanent promise.

### VI. Test-Driven Development & Automated Testing

Code quality and correctness MUST be ensured through Test-Driven Development (TDD) and automated tests, not manual validation.

**Rules**:
- **Tests MUST be written FIRST and MUST FAIL before implementation begins** (red-green-refactor cycle)
- **"Starting red" means tests MUST compile AND fail semantically (for the right reasons)**:
  - Tests MUST compile successfully with no compilation errors
  - Tests MUST fail because the functionality under test is missing or incomplete, not because the method doesn't exist
  - It is ACCEPTABLE to implement minimal structure to make tests compile (stub types, empty methods, interface definitions)
  - **Skipping or marking tests as pending is FORBIDDEN in the red phase**
  - **Tests MUST NOT contain comments marking them as "in the red phase"** (e.g., `// TODO: implement`) — tests are written once and turn green naturally
- Tests MUST drive design: implementation decisions emerge from test requirements, not vice versa
- Tests MUST change as little as possible during implementation; major test churn indicates poorly derived tests
- Automated tests MUST cover happy paths, error cases, and edge cases (nil/zero values, empty inputs, boundary sizes, canceled contexts, concurrent use)
- **`github.com/fogfish/it/v2` MUST be the sole assertion library** for unit and acceptance tests, used via its fluent `it.Then(t).Should(...)` style; packages MUST NOT mix in another assertion library (e.g., `testify`) alongside it
- Table-driven tests MUST be used for parameterized scenarios
- **The full test suite MUST pass under the race detector** (`go test -race ./...`) in CI
- **Every function that parses, decodes, or deserializes untrusted bytes MUST have a native Go fuzz test** (`func FuzzXxx(f *testing.F)`), with its seed corpus under `testdata/fuzz/`
- **Every operation listed as a hot path in [ARCHITECTURE.md](ARCHITECTURE.md) MUST have a benchmark** (`func BenchmarkXxx(b *testing.B)` reporting allocations); PRs touching a hot path MUST include a `benchstat` comparison against the base branch
- **Bash/shell scripts MUST NOT be used to validate code correctness**; they are reserved for infrastructure tasks (CI/CD, release, import-rule checks)
- Test coverage MUST be verifiable via `go test ./... -cover`
- Integration tests of secondary adapters MUST test real behavior where feasible (real file I/O in `t.TempDir()`, a local emulator); tests requiring live cloud credentials MUST be opt-in via a build tag or environment variable and MUST NOT gate PRs

**Rationale**: TDD ensures tests are independent verification of requirements, not retrofitted validation. The race detector, fuzzing, and benchmarks verify the three properties a consumer cannot verify from the outside: thread safety, robustness against hostile input, and performance budgets.

### VII. Ports, Adapters & Dependency Isolation

Every technology the library drives MUST sit behind a public secondary port, and every heavy technology dependency MUST be isolated from consumers who do not use it.

**Rules**:
- All external concerns (storage substrates, network transports, object stores, clocks, randomness, external indexes, the filesystem) MUST be accessed through a secondary port interface, with concrete implementations living in adapter packages
- **Secondary ports are public** (ADR 001): the embedding application MUST be able to supply its own implementation. Port interfaces MUST be small (one interface per capability) and expressed in domain and standard-library types only
- **Every secondary port MUST ship with an in-memory reference adapter in the root module**, depending on the standard library only. It is used by the library's own tests and is available to consumers for theirs
- **Every secondary port MUST ship with a conformance suite** — an exported package `<port>test` exposing a function such as `func TestPort(t *testing.T, factory func() port.X)` — that every adapter, in-repo or third-party, MUST pass. A port's behavioral contract is defined by its conformance suite, not by prose
- **Adapters with dependencies beyond the standard library and the libraries mandated here MUST live in their own nested Go module** (ADR 001), so that a consumer of the root module never compiles, downloads, or `go.sum`-pins a technology it does not use
- **Filesystem access MUST be expressed with stdlib `io/fs` (reads) and `io.Reader`/`io.Writer` supplied by the caller.** The `os` package's file/directory functions MUST NOT appear outside the one adapter that implements filesystem storage on the real OS
- **Time and randomness are ports**: domain and application code MUST NOT call `time.Now()` or global `math/rand` directly; they receive a clock/source so behavior is deterministic in tests
- **Before adding a new adapter, verify whether an adapter for that capability already exists**; duplicate adapters for the same technology are forbidden
- Network and long-running calls MUST accept and respect `context.Context` for cancellation; adapters MUST NOT impose hidden timeouts that the caller cannot override
- Retry, backoff, and caching policies MUST be explicit, documented in [ARCHITECTURE.md](ARCHITECTURE.md), and configurable by the consumer; an adapter MUST NOT retry a non-idempotent operation (ADR 001 port isolation rule 3)
- Adapters that handle credentials MUST accept them from the consumer (an SDK config or credential provider), MUST NOT read them from the environment themselves, and MUST NOT log them

**Rationale**: A library's dependency graph is imposed on every consumer. Public ports, conformance suites, and nested modules let consumers bring their own technology, prove it works, and pay only for what they import.

### VIII. Acceptance Testing & Spec Traceability

All features MUST have acceptance tests where each scenario in `spec.md` maps 1:1 to a test, exercised through the same public API a consumer actually calls.

**Rules**:
- **Every acceptance scenario in `specs/[NNN-feature-name]/spec.md` MUST have a corresponding acceptance test**
- Acceptance tests MUST be written BEFORE implementation begins (acceptance-test-driven development)
- **Acceptance tests MUST be black-box**: they live in the external test package (`package <pkg>_test`) of the composition root or the primary adapter they exercise, and use exported identifiers only. A scenario a test cannot express through the public API is a missing API, not a reason to test internals
- **Acceptance tests MUST be colocated with the package they exercise**, named after the feature (e.g., `facade/commit_test.go`) — not in a separate top-level test tree
- **A shared `sut` (system-under-test) helper MUST construct the library through the composition root**, wiring in-memory reference adapters (Principle VII), and register cleanup via `t.Cleanup`:
  ```go
  func sut(t *testing.T, cfg facade.Config) *facade.Facade {
      t.Helper()
      f, err := facade.New(memory.New(), cfg)
      it.Then(t).Should(it.Nil(err))
      t.Cleanup(func() { f.Close() })
      return f
  }
  ```
- Acceptance tests MUST NOT require live network access or live credentials to pass in CI
- Test fixtures MUST live in a `testdata/` directory colocated with the test file
- **Acceptance tests MUST compile AND fail semantically initially (red phase)**:
  - Assertions MUST use `github.com/fogfish/it/v2` (Principle VI) with concrete expected values, not placeholder always-fail assertions
  - **Skipping or marking tests as pending is FORBIDDEN**; `t.Skip()` calls MUST NOT appear in acceptance test files
  - **Tests MUST NOT contain "red phase" comments** (e.g., `// TODO: implement`)
  - It is ACCEPTABLE to implement minimal structure to make acceptance tests compile (an exported method returning a "not implemented" error)
- Each test function SHOULD include a comment referencing its spec scenario: `// Scenario X.Y from specs/[NNN-feature]/spec.md`
- **Every user story SHOULD additionally have a testable example** (`func ExampleFacade_Commit()` with an `// Output:` block) in an `example_test.go` file; examples are compiled and verified by `go test` and rendered on pkg.go.dev, so they are documentation that cannot drift
- Acceptance tests MUST change minimally during implementation; major changes indicate tests were derived from implementation, not from the spec
- Acceptance tests MUST be independent and isolated: no shared mutable package state, no dependency on execution order; they SHOULD call `t.Parallel()`

**Rationale**: Testing through exported identifiers proves the library behaves correctly for the consumer and keeps internals free to change. Black-box tests are also the first consumer of every new API; an awkward test is an early warning of an awkward API.

### IX. Public API Design

The exported surface of the library is its product. It MUST be minimal, idiomatic, consistent, and hard to misuse.

**Rules**:
- **Minimal surface**: everything that is not part of a primary port, a secondary port, the domain core, or the composition root lives under `internal/`
- **Naming**: package names are short, lower-case, singular nouns with no underscores; exported identifiers MUST NOT stutter with the package name (`facade.New`, not `facade.NewFacade`; `record.Key`, not `record.RecordKey`)
- **Construction**: types that need setup are created by `New(...)` (or `NewX` when a package holds several) returning a concrete type and an `error`; types that need no setup MUST have a useful zero value
- **Configuration is explicit and typed**:
  - **Required dependencies are positional parameters** of the constructor (e.g., `New(storage {port}, cfg Config) (*{Type}, error)`), so omitting one is a compile error
  - **Optional settings are fields of an exported `Config` struct** (`XConfig` when a package configures several types), passed by value; the constructor copies it, so later mutation by the caller has no effect
  - **The zero value of every field means its documented default**, and each field's GoDoc states that default; a zero `Config{}` MUST yield a working instance. When zero is a meaningful value distinct from the default, the field's GoDoc names an explicit sentinel (e.g., a negative duration disables a timeout); pointer fields used to signal "unset" are forbidden
  - The constructor **validates the whole `Config`** and returns an exported `faults` error (e.g., `ErrInvalidConfig`) naming the offending field; it MUST NOT silently clamp or ignore invalid values
  - Configuration is **immutable after construction**; reconfiguration requires a new instance or an explicit, documented method
  - `Config` MAY carry `json`/`yaml` struct tags so the consumer can load it from its own sources; the library MUST NOT load it itself (Principle X)
  - A **builder** MAY produce a `Config` when it has interdependent or conditionally required fields that a struct literal cannot express safely: each builder step returns a new value (Principle IV) and the terminal `Build() (Config, error)` validates. The constructor still accepts the `Config`; the builder is a convenience over it, never a replacement. The same builder shape applies to complex domain values (queries, requests, schemas)
  - **Functional options (`type Option func(*config)`, `WithX(...) Option`) are FORBIDDEN** in exported APIs, as are positional boolean parameters
- **`context.Context` is the first parameter** of every exported function that performs I/O, blocks, or may be canceled; contexts MUST NOT be stored in structs
- **Errors are values**: every exported function that can fail returns `error` as its last result. Exported functions MUST NOT panic on any input a consumer can construct; panics are reserved for programmer errors in unexported code and MUST NOT cross the public API
- **Interfaces are exported only for ports** — contracts that consumers implement or that have several implementations. A single concrete implementation is returned as a struct, not hidden behind an interface
- **Generics** are used when they remove duplication across types the consumer chooses; they are not used for a single instantiation, and type parameters MUST be constrained as narrowly as the algorithm allows
- **Sequences**: APIs yielding a potentially large or unbounded sequence SHOULD return an iterator (`iter.Seq`/`iter.Seq2`) or a cursor type, not a materialized slice
- **Ownership**: GoDoc MUST state whether the library retains slices/maps passed in or returned, and whether returned values may be mutated by the caller
- **Concurrency contract**: GoDoc of every exported type MUST state whether it is safe for concurrent use
- Exported identifiers MUST NOT expose types from `internal/` or from a third-party module that is not itself a deliberate part of the contract

**Rationale**: Consumers integrate against the API once and live with it for years. Idiomatic, minimal, explicit APIs are what make a library cheap to adopt and safe to upgrade.

### X. Library Citizenship & Side-Effect Hygiene

The library runs in a process it does not own. It MUST behave as a guest.

**Rules**:
- The library MUST NOT call `os.Exit`, `log.Fatal*`, or `log.Panic*`, and MUST NOT write to `os.Stdout` or `os.Stderr`
- The library MUST NOT read environment variables, command-line arguments, or flags, and MUST NOT register flags on `flag.CommandLine`; all configuration comes from the consumer through constructor parameters and `Config` (Principle IX)
- **No mutable package-level state**: package-level variables are limited to constants, sentinel errors, and values that are immutable after initialization. `init()` functions MUST NOT perform I/O, start goroutines, or register global side effects
- The library MUST NOT modify process-wide settings (`http.DefaultClient`, `http.DefaultTransport`, the default `slog` logger, `GOMAXPROCS`, signal handlers)
- **Logging is opt-in**: diagnostics are emitted only through a `*slog.Logger` supplied as a `Config` field; the zero value means a discarding logger. The library MUST NOT log at levels above `Debug` for conditions it also returns as errors
- **Metrics and tracing** are exposed through a narrow secondary port (e.g., an observer interface) or a nested-module adapter; the root module MUST NOT depend on a metrics or tracing SDK
- **The library MUST NOT collect telemetry or contact any network endpoint** that the consumer did not configure
- Secrets, credentials, and payload data MUST NEVER appear in log records or error messages
- Cgo and `unsafe` MUST NOT be used without an accepted ADR; the root module MUST build with `CGO_ENABLED=0` on every platform Go supports

**Rationale**: A library that exits, prints, reads the environment, or mutates globals makes its consumer's process unpredictable. Respecting those boundaries is the price of being embedded.

### XI. Concurrency, Resource Lifecycle & Performance

Concurrency, resources, and performance are part of the library's contract, not implementation details.

**Rules**:
- **Every goroutine started by the library has an owner** that stops it: a `Close`/`Shutdown` method or the cancellation of a context passed by the consumer. Goroutines MUST NOT outlive the value that started them
- Types that hold resources (goroutines, files, connections, buffers from a pool) MUST implement `io.Closer` (or `Close(ctx) error` where shutdown may block); `Close` MUST be idempotent and safe to call after a failed construction
- Tests of types that start goroutines MUST verify that no goroutine leaks after `Close`
- **Synchronous by default**: exported functions perform their work and return; asynchronous behavior is offered only when the domain requires it and is documented with its completion signal (channel, callback, or `Wait`)
- Channels in exported signatures MUST be directional, and GoDoc MUST state who closes them
- Shared state MUST be protected by `sync` primitives or confined to one goroutine; `sync/atomic` is used only for single-word state shown by benchmark to matter
- **Resource bounds are explicit**: buffers, queues, caches, worker pools, and reads of untrusted input (`io.LimitReader`) MUST have documented, consumer-configurable limits — no unbounded growth driven by input
- **Hot paths** are named in [ARCHITECTURE.md](ARCHITECTURE.md) with their latency/allocation budget; they MUST have benchmarks (Principle VI). Optimizations that compromise readability MUST be justified by a `benchstat` result in the PR
- `faults.Fast` MAY replace `faults.Type`/`faults.SafeN` on a documented hot path where a benchmark shows the error path matters (see [Mandatory Libraries & Tooling](#mandatory-libraries--tooling))

**Rationale**: Leaked goroutines, unbounded buffers, and undocumented thread-safety are the defects consumers of a library cannot fix themselves. Making them explicit turns them from surprises into contracts.

### XII. Documentation & Error Contract

The library MUST be self-documenting on pkg.go.dev, and its errors MUST form a stable, matchable taxonomy.

**Rules**:
- **Every package MUST have a package comment** (in `doc.go` for any non-trivial package) explaining its purpose, its place in the architecture, and a minimal usage snippet
- **Every exported identifier MUST have a GoDoc comment** starting with the identifier's name, stating behavior, errors returned, concurrency safety, and ownership of arguments (Principle IX)
- Testable examples (`Example*` functions) MUST exist for the composition root's primary entry points and SHOULD exist for every exported function a consumer calls directly (Principle VIII)
- A `README.md` at the repository root MUST document the purpose, installation (`go get`), a quick-start example, supported Go versions, the list of nested modules with their purpose, and a link to pkg.go.dev
- Documentation MUST be updated in the same PR as the API change it describes
- **Error context annotation MUST use `github.com/fogfish/faults`** (see [Mandatory Libraries & Tooling](#mandatory-libraries--tooling)), not ad hoc `fmt.Errorf("...: %w", err)` chains
- **Errors a consumer is expected to branch on are exported** constants (`const ErrNotFound = faults.Type("record not found")`) in the domain core or port package that defines the condition, documented in GoDoc, and part of the compatibility contract (Principle XIV); all other errors are unexported
- Consumers match errors with `errors.Is`/`errors.As` only; GoDoc MUST NOT invite matching on error strings
- Errors returned by adapters MUST be translated into the port's exported error taxonomy before crossing the port; vendor SDK error types MUST NOT reach the consumer through the primary port

**Rationale**: For a library, pkg.go.dev is the help system and the error taxonomy is part of the API. Both must be precise, stable, and impossible to drift from the code.

### XIII. Modules & Release Engineering

The library MUST be released as versioned, reproducible Go modules through an automated pipeline.

**Rules**:
- The module path MUST be the canonical import path of the repository (e.g., `github.com/{org}/{name}`); major versions ≥ 2 MUST use the `/vN` suffix (Principle XIV)
- **The `go` directive in `go.mod` declares the minimum supported Go version** and MUST NOT be raised without a MINOR release note; the library MUST support at least the two most recent Go releases
- **Released `go.mod` files MUST NOT contain `replace` directives**; `go.work` MAY be used for local multi-module development and MUST NOT be committed
- `go mod tidy` MUST leave `go.mod`/`go.sum` unchanged in CI; a `vendor/` directory MUST NOT be committed
- **Nested modules** (ADR 001) depend on the root module by released version, are tagged with their path prefix (`<path>/vX.Y.Z`, e.g., `portx/technologyy/v1.4.0`), and are released independently; the root module MUST NOT depend on any nested module
- **Every new direct dependency of the root module requires justification** in the PR (why the standard library is insufficient, its license, its maintenance status, its transitive footprint); dependencies with large transitive trees belong in nested modules
- Releases are Git tags with GitHub Releases generated in CI; there are no binaries and no manually published release artifacts
- Version tags MUST be created by CI from an explicit, documented increment rule (e.g., a marker in the merge commit message selecting major/minor/patch, defaulting to patch), never pushed manually
- The generated release notes MUST exclude non-consumer-facing commit categories (e.g., `^docs:`, `^test:`, `^ci:`)
- Runnable example programs MAY live under `examples/`; if they need dependencies not required by the library, `examples/` MUST be its own module

**Rationale**: Go modules are the distribution format of a library. Clean `go.mod` files, independent nested modules, and CI-driven tags make every release reproducible and every dependency deliberate.

### XIV. Versioning, Compatibility & Security

The library's version number MUST be a trustworthy statement about its API, and the library MUST be safe to feed untrusted input.

**Rules**:
- Versioning MUST follow **Semantic Versioning** applied to the exported Go API of each module, including exported error values, documented behavior, and documented concurrency guarantees
- **Import compatibility rule**: a breaking change MUST be released as a new major version with a new module path (`/v2`); in-place breaking changes under the same major version are forbidden
- **`Config` compatibility**: adding a field to an exported `Config` is MINOR; removing or renaming a field, changing its type, or changing the meaning of its zero value is MAJOR. `apidiff` does not detect a changed default, so reviewers MUST check it explicitly
- **`apidiff` (see [Mandatory Libraries & Tooling](#mandatory-libraries--tooling)) MUST run on every PR** against the latest release tag of each changed module; an incompatible change reported for a non-major release blocks merge
- `v0.x` modules MAY break between minor versions but MUST announce breaks in release notes; a capability SHOULD reach `v1.0.0` before it is recommended for production use
- **Deprecation precedes removal**: an identifier scheduled for removal MUST carry a `// Deprecated:` GoDoc paragraph naming the replacement for at least one minor release before the next major version
- A release published in error MUST be withdrawn with a `retract` directive in `go.mod`, never by deleting or moving a tag
- **`govulncheck` MUST scan every module in CI before every release**; known vulnerabilities reachable from the library's code MUST block release
- **Untrusted input is hostile**: decoders and parsers MUST validate lengths and counts before allocating, MUST bound recursion depth, and MUST be fuzzed (Principle VI)
- Cryptography, if any, MUST use the standard library or `golang.org/x/crypto`; hand-rolled cryptographic primitives are forbidden

**Rationale**: Consumers upgrade a library because its version number tells them it is safe to. An automated API-diff gate turns that promise from good intent into a checked property.

## Mandatory Libraries & Tooling

The following libraries and tools are not illustrative options — they are mandated by this constitution for every project that adopts it, so that test idioms, error handling, lint gates, and release mechanics are identical across every library under this governance. The root module's runtime dependencies are deliberately limited to this list plus dependencies justified per Principle XIII.

**Error Handling & Annotation**
- `github.com/fogfish/faults` MUST be used to annotate errors with context everywhere an error crosses a package boundary worth explaining (Principle XII). Ad hoc `fmt.Errorf("<context>: %w", err)` call sites for this purpose are forbidden.
- Every reusable, expected error condition MUST be declared once as a package-level constant using `faults.Type` (static message) or `faults.Safe1` through `faults.Safe5` (compile-time-typed dynamic arguments), never constructed inline at the call site: `const ErrSegmentCorrupted = faults.Type("segment checksum mismatch")`.
- Wrapping an underlying error MUST use the constant's `.With(err[, args...])` method: `return ErrSegmentCorrupted.With(err)`. Callers that need to branch on a specific failure MUST use `errors.Is(err, ErrSegmentCorrupted)` — never string matching or type assertion against the wrapped error.
- `faults.Fast` MAY replace `faults.Type`/`faults.SafeN` only on a documented hot path where a benchmark shows the error path is performance-sensitive (Principle XI); it MUST NOT be reached for by default.

**Logging**
- Standard library `log/slog` is the only logging API (Principle X). The library accepts a `*slog.Logger` as a `Config` field; a nil logger means `slog.New(slog.DiscardHandler)`. No third-party logging library is permitted in the root module.

**Filesystem**
- Standard library `io/fs`, `io.Reader`, and `io.Writer` are the only filesystem abstractions (Principle VII). Path conventions follow `fs.ValidPath`: relative, slash-separated, no leading or trailing slash, `"."` for the root. No third-party filesystem or object-storage library in the root module; object-storage backends are nested-module adapters.

**Testing**
- `github.com/fogfish/it/v2` MUST be the sole assertion library for unit and acceptance tests (Principles VI, VIII), used via its fluent `it.Then(t).Should(...)` style.
- Acceptance tests MUST follow the black-box, colocated, `sut()`-helper pattern defined in Principle VIII.
- `go test -race -coverprofile=profile.cov $(go list ./... | grep -v /examples/)` MUST be runnable from CI for every module; coverage MUST be published to a coverage-reporting service (e.g., Coveralls via `shogo82148/actions-goveralls`) so the trend is visible on every PR.
- Native Go fuzzing (`testing.F`) for every decoder/parser; `golang.org/x/perf/cmd/benchstat` for benchmark comparison on hot-path PRs.

**Static Analysis & Compatibility**
- `go vet` and `staticcheck` (`honnef.co/go/tools/cmd/staticcheck`) MUST run in CI on every pull request and MUST block merge on failure. `gofmt -l` MUST report no files.
- `apidiff` (`golang.org/x/exp/cmd/apidiff`, the API compatibility checker used by the Go project itself) MUST run on every pull request for each changed module in module mode with incompatible-only reporting (`apidiff -m -incompatible`), comparing the PR head against the module's latest release tag (Principle XIV). The `apidiff` version used in CI MUST be pinned.
- `govulncheck` (`golang.org/x/vuln/cmd/govulncheck`) MUST run on every module before every release (Principle XIV).

**Continuous Integration (GitHub Actions)**

A project adopting this constitution MUST implement at minimum these workflows under `.github/workflows/`:
- **`build`** — triggered on push to the default branch: `go build ./...` and `go test -race` with coverage for every module, an automatic SemVer tag increment per changed module (Principle XIII), then a GitHub Release of that tag
- **`check-code`** (or equivalent name) — triggered on pull request open/synchronize: `gofmt`, `go vet`, `staticcheck`, `go mod tidy` diff check, `apidiff`; MUST be a required check before merge
- **`check-test`** (or equivalent name) — triggered on pull request open/synchronize: `go build ./...` and `go test -race` with coverage for every module, run on a matrix of the minimum supported Go version and the latest stable Go version; MUST be a required check before merge
- The Go toolchain versions used in CI MUST be pinned explicitly (e.g., `actions/setup-go` with fixed `go-version` values), never left to float to "latest"

**Rationale**: Naming specific tools eliminates churn when a contributor moves between libraries under this constitution: test idioms, error idioms, the lint gate, and the compatibility gate are identical everywhere.

## Development Requirements

### Compliance Checklist

**PRECONDITIONS (must complete BEFORE implementation begins)**:

- [ ] Domain model designed: all entities, value objects, and their algebras identified and documented (Principle II)
- [ ] Domain concepts added to [ARCHITECTURE.md](ARCHITECTURE.md) Glossary section
- [ ] Public API contract designed: exported types, functions, `Config` fields with defaults, and error constants agreed before coding (Principle IX)
- [ ] New use-cases placed in `internal/app/<usecase>` with their private ports identified (Principle III, ADR 001)
- [ ] Secondary ports designed for any new technology, with in-memory adapter and conformance suite planned (Principle VII)
- [ ] Dependency impact assessed: new dependencies justified; heavy ones assigned to a nested module (Principles VII, XIII)
- [ ] **Acceptance tests written, black-box, colocated with the composition root or primary adapter, for all spec scenarios (Principle VIII)**
- [ ] **Acceptance tests compile successfully and fail semantically before implementation (Principle VIII)**

**Implementation Phase**:

- [ ] [ARCHITECTURE.md](ARCHITECTURE.md) reflects architectural changes (if any), including the package-to-ring map
- [ ] Major decisions recorded in [adrs/](adrs/) with correct numbering
- [ ] Code follows patterns established in accepted ADRs
- [ ] Dependencies point inward; no use-case imports another; only the composition root constructs use-cases (Principle III)
- [ ] No internal or vendor type appears in an exported signature (Principles III, IX)
- [ ] **Unit tests written FIRST, compile successfully, and fail semantically before implementation (Principle VI)**
- [ ] **Unit tests drive design and change minimally during implementation (Principle VI)**
- [ ] Tests pass under `-race`; decoders/parsers have fuzz tests; hot paths have benchmarks (Principle VI)
- [ ] Unit and acceptance tests use `github.com/fogfish/it/v2` exclusively (Principle VI)
- [ ] New secondary ports ship with an in-memory adapter and a conformance suite; every adapter passes it (Principle VII)
- [ ] Heavy technology dependencies live in nested modules; root `go.mod` has no new unjustified dependency (Principles VII, XIII)
- [ ] Filesystem access via `io/fs`/`io.Reader`/`io.Writer`; no `os` file functions outside the OS filesystem adapter (Principle VII)
- [ ] Public API follows naming, `Config`, context, error, and ownership rules; no functional options (Principle IX)
- [ ] No `os.Exit`, stdout/stderr writes, environment reads, mutable globals, or side-effecting `init()` (Principle X)
- [ ] Logging only via injected `*slog.Logger`; no secrets in logs or errors (Principle X)
- [ ] Every goroutine has an owner; resource-holding types implement idempotent `Close`; no goroutine leaks (Principle XI)
- [ ] GoDoc present for every package and exported identifier; examples for primary entry points (Principle XII)
- [ ] Error annotation uses `github.com/fogfish/faults`; branchable errors are exported constants matched via `errors.Is` (Principle XII)
- [ ] `go mod tidy` clean, no `replace`, `go` directive unchanged or release-noted (Principle XIII)
- [ ] `apidiff` reports no incompatible change for a non-major release; deprecations use `// Deprecated:` (Principle XIV)
- [ ] **Acceptance tests turn GREEN as implementation satisfies acceptance criteria and changed minimally (Principle VIII)**
- [ ] **All spec scenarios have passing acceptance tests (Principle VIII)**

### When Constraints Cannot Be Met

If **Principle I (Architecture Documentation & ADRs)** requires deviation:

1. STOP implementation immediately
2. Document why the accepted ADR pattern cannot be followed
3. Draft a new ADR proposing an alternative approach with rationale
4. Mark the new ADR as superseding the previous ADR
5. Do NOT implement the deviation without an accepted superseding ADR

If **Principle III (Hexagonal Architecture)** cannot be satisfied:

1. STOP implementation immediately
2. Document which ring or import rule cannot be respected and why
3. Propose an alternative with technical justification in a new ADR
4. Do NOT introduce an outward-pointing import or a use-case-to-use-case dependency without an accepted ADR

If **Principle VI (TDD & Automated Testing)** cannot be satisfied:

1. Document the specific reason why TDD or automated testing is not feasible
2. Propose an alternative validation approach
3. Escalate to project maintainers for exception approval
4. Do NOT skip test-first development without explicit justification

If **Principle VII (Ports, Adapters & Dependency Isolation)** cannot be satisfied:

1. STOP implementation immediately
2. Document why a port, an in-memory adapter, a conformance suite, or a nested module cannot be introduced
3. Propose an alternative approach with technical justification
4. Create an ADR documenting the exception and rationale
5. Do NOT add a heavy technology dependency to the root module without an accepted ADR

If **Principle VIII (Acceptance Testing & Spec Traceability)** cannot be satisfied:

1. STOP implementation immediately
2. Document why a scenario cannot be expressed as a black-box test through the public API
3. Treat the gap as a missing API first; propose the API or an alternative approach with justification
4. Escalate to project maintainers for exception approval
5. Do NOT ship features without acceptance tests mapped to spec scenarios

If **Principle IX (Public API Design)** cannot be satisfied:

1. STOP implementation immediately
2. Document precisely which rule cannot be followed and why (e.g., an interface mandated by a standard-library contract)
3. Escalate to project maintainers for sign-off before merging
4. Record the exception in an ADR, since a released API cannot be corrected without a major version

If **Principle XIII (Modules & Release Engineering)** cannot be satisfied:

1. STOP implementation immediately
2. Document why the module, tagging, or dependency rule cannot be applied
3. Escalate to project maintainers for exception approval
4. Do NOT publish a release with `replace` directives or a manually pushed tag without an accepted ADR

If **Principle XIV (Versioning, Compatibility & Security)** cannot be satisfied:

1. STOP implementation immediately
2. Document why the SemVer contract, the `apidiff` gate, the deprecation rule, or the `govulncheck` gate cannot be applied
3. Escalate to project maintainers for exception approval
4. Do NOT release an incompatible API change under the same major version

## Governance

### Amendment Procedure

1. Propose amendment via PR to [.specify/memory/constitution.md](.specify/memory/constitution.md)
2. Include rationale, impacted templates, and migration plan
3. Increment CONSTITUTION_VERSION per semantic versioning rules
4. Update the Sync Impact Report with changes
5. Propagate changes to dependent templates and documentation

### Versioning Policy

- **MAJOR**: Backward-incompatible governance/principle removals or redefinitions
- **MINOR**: New principle/section added or materially expanded guidance
- **PATCH**: Clarifications, wording, typo fixes, non-semantic refinements

### Compliance Review

- All PRs MUST verify compliance with this constitution
- Reviewers MUST challenge complexity and request justification when principles are violated
- Reviewers MUST verify adherence to accepted ADRs (Principle I)
- Reviewers MUST verify inward-only dependencies, use-case isolation, and the single composition root (Principle III)
- Reviewers MUST verify TDD was followed: tests written first, compiled, and failed semantically (Principle VI)
- Reviewers MUST verify tests pass under `-race`, decoders are fuzzed, and hot-path changes carry a `benchstat` comparison (Principle VI)
- Reviewers MUST verify new ports ship with an in-memory adapter and conformance suite, and heavy dependencies live in nested modules (Principle VII)
- Reviewers MUST verify acceptance tests exist for all spec scenarios, are black-box, and changed minimally during implementation (Principle VIII)
- Reviewers MUST verify every new exported identifier is required by a spec scenario and follows the API rules, including typed `Config` with documented zero-value defaults and no functional options (Principles V, IX)
- Reviewers MUST verify the library introduces no process-level side effects, globals, or unsolicited logging (Principle X)
- Reviewers MUST verify goroutine ownership, `Close` semantics, and resource bounds (Principle XI)
- Reviewers MUST verify GoDoc and examples are present and accurate, and errors use `github.com/fogfish/faults` with exported, matchable constants (Principle XII)
- Reviewers MUST verify `go.mod` hygiene and dependency justification (Principle XIII)
- Reviewers MUST verify `apidiff` output matches the intended version increment (no incompatible change unless the module's major version is bumped) (Principle XIV)
- Reviewers MUST verify mandated libraries and tooling (`fogfish/faults`, `fogfish/it/v2`, `log/slog`, `io/fs`, `staticcheck`, `apidiff`, `govulncheck`, required CI workflows) are used unchanged, per [Mandatory Libraries & Tooling](#mandatory-libraries--tooling)
- Template files in [.specify/templates/](.specify/templates/) provide execution workflows that enforce these principles

### Task List Requirements

Every feature's `tasks.md` file MUST include these mandatory sections from [tasks-template.md](.specify/templates/tasks-template.md):

**MANDATORY SECTIONS** (cannot be omitted):
1. **Phase 2: Design Preconditions** - Constitution PRECONDITIONS implementation
   - Phase 2a: Domain Model & Glossary (Principles II, V)
   - Phase 2b: Public API Contract Design (Principle IX)
   - Phase 2c: Use-Case, Port & Adapter Design (Principles III, VII)
   - Phase 2d: Acceptance Test Design (Principle VIII)
   - Phase 2e: Dependency & Compatibility Review (Principles XIII, XIV)

2. **Phase N: Constitution Compliance Verification** - Constitution Implementation Phase checklist
   - Design Phase Verification tasks
   - Implementation Phase Verification tasks grouped by principle
   - All verification tasks with explicit principle references

**OPTIONAL PHASES** (include when applicable — omit when not needed):
- **Phase 0: Pre-implementation Refactoring** — include when the feature requires significant changes
  to existing code (rename, restructure, extract interfaces, split packages). MUST be submitted as a
  separate PR from feature work. All existing tests MUST pass and `apidiff` MUST report no
  incompatible change after refactoring.
- **Phase 2.5: Use-Case Boilerplate** — include when the feature introduces new use-cases, ports, or
  adapters. Creates empty-but-compiling scaffolding (`internal/app/<usecase>` with `component.go`,
  private ports, composition-root wiring returning a "not implemented" error, an empty in-memory
  adapter) as a separate PR before business logic. Enables focused, incremental code review:
  structural scaffold PR → business logic PRs.

**CUSTOMIZABLE SECTIONS** (adapt to feature):
- Phase 1: Setup (project-specific initialization)
- Phase 3+: User Stories (based on spec.md user stories)
- Additional Polish (optional improvements)

**Enforcement**:
- AI agents generating tasks.md MUST retain Phase 2 and Phase N sections verbatim (adapting only task
  descriptions to the specific feature)
- Omitting mandatory sections violates this constitution and blocks feature completion

**Version**: 1.0.0 | **Ratified**: 2026-10-18 | **Last Amended**: 2026-10-18
