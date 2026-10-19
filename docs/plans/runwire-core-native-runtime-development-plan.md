# Runwire Core — independent Rust native PHP runtime: full draft plan

**Status:** DRAFT / design and planning only
**Updated:** 2026-10-19
**Base:** `main` (license-only starter repository) @ `994cd874bbb6aa17cdafe5d25cde60185946038a`
**Branch:** `plan/runwire-independent-ecosystem-roadmaps`
**Implementation:** NOT STARTED
**Publication:** No release, tag, merge or extension publication authorized.

## Vision

Build a standalone, high-performance, Rust-implemented PHP-native network and execution runtime, distributed as an installable PIE PHP extension. **This is a separate product, not an add-on, accelerator, compatibility layer, or backend of `infocyph/runwire`**. It neither requires nor detects Runwire or Runwire-Cells. It has independent public PHP classes, native runtime ownership, tests, releases, and benchmarks.

Target: predictable low p95/p99 latency, high sustained throughput, minimal memory per connection, safe PHP callback dispatch, bounded admission/backpressure and operationally meaningful observability. Native efficiency must be measured against comparable PHP runtime servers; no unmeasured "fastest" claims.

## Product boundary and success definition

Own: standalone PHP API, Rust reactor(s), network sockets/transports, HTTP/1.1 and HTTP/2 support, optional HTTP/3, native buffer/timer scheduling, native worker lifecycle, PHP-facing callbacks/response, startup and shutdown, metrics and security limits.

Do NOT own: PHP application framework, dependency injection, routing, ORM, message broker, distributed cells/actors/replication, `infocyph/runwire` API compatibility, arbitrary user-created Rust-thread PHP object execution.

**Phase-1 criteria:** on PHP 8.4 and 8.5, a PIE-built extension launches a native HTTP/1.1 server, executes PHP callbacks on a valid PHP VM context, enforces backpressure/cancellation, gracefully stops, passes soak/security/QA, and beats or matches the selected baseline on explicitly identified workloads with no correctness or p99/RSS regression.

## Installation and packaging target

- Repository: `infocyph/Runwire-Core`; Rust crate/workspace and PHP extension code within this repository alone.
- Proposed PIE package: `infocyph/runwire-core`; PHP module: `runwire_core` (names provisional until registry check).
- Initial platforms: Linux x86_64 and aarch64; macOS arm64 if feasible; PHP 8.4/8.5 NTS and ZTS verified independently.
- Windows, musl, custom PHP builds and additional PHP versions: explicit future/experimental scope until validated.
- Build integration: investigate `ext-php-rs` and the PHP/Zend API; verify real PIE `php-ext` packaging with `phpize/config.m4` and Cargo; do **not** assume `cargo build` by itself makes the extension PIE-installable.
- Package manifest, extension metadata, repeatable source build, dependency audit, artifact provenance, Rust/toolchain lock/pinning, extension load smoke (`php -m`, `php --ri`), CLI startup and shutdown.
- Investigate PIE-supported binaries, integrity verification and source fallback; do not claim precompiled downloads before the publishing path is verified.

## Critical feasibility issues and architectural invariants

### Zend / PHP VM ownership

- Never call Zend or execute user PHP callbacks from arbitrary Tokio worker threads.
- PHP VM/TSRM state, zvals, reflection, exceptions and FFI resource lifetime remain owned by the correct PHP execution thread and request.
- Native I/O may execute in Rust threads; completions must be marshalled into bounded queues and dispatched by valid PHP execution contexts.
- Choose and benchmark one of: dedicated PHP callback executor bound to an approved Zend context; one PHP process per native worker; or host-event-loop dispatch where ownership is unambiguous. Document NTS vs ZTS behavior.
- Define callback concurrency guarantees: serialized per PHP VM when required; no implied parallelism for user PHP code.
- Never let Rust retain borrowed PHP strings/values beyond their Zend lifetime; copy or own data across async boundaries.
- No background access to Zend state after RSHUTDOWN/MSHUTDOWN.

### Fork/process safety

- Rust Tokio runtimes, native mutexes, sockets and thread pools must be initialized **after** a worker fork or be discarded and reconstructed correctly.
- Avoid speculative prefork strategy: first prove single-process safety, then multi-process ownership.
- Native shutdown, worker restarts, graceful draining and signal delivery must be bounded and have idempotent teardown.

### Performance boundary

- Quantify cost of each PHP↔Rust transition.
- Keep expensive socket/protocol loops native; batch completions where safe.
- Prevent callbacks from blocking native reactor execution; expose application-blocking metrics.
- Backpressure must work all the way through ingress, dispatch queue, callback, response and outbound socket.

## Proposed code organization (subject to R0 proof)

```
Cargo.toml / Cargo.lock
composer.json              # type php-ext
config.m4                  # PIE/PECL/phpize integration
src/
  lib.rs                  # extension registration and PHP API
  api/                    # Zend mapping, exceptions, value boundaries
  engine/                 # runtime ownership, commands, completions
  net/                    # socket I/O, TLS, buffers, admission
  http/                   # H1/H2, later H3
  process/                # worker startup/restart/shutdown
  metrics/                # bounded snapshots, optional tracing
tests/
  php/                    # black-box PHP behavior and lifecycle
  rust/                   # low-level property/unit/fuzz tests
  integration/            # real HTTP clients, process/crash faults
benchmarks/
  protocols/              # matched end-to-end
  ffi/                    # boundary-cost calibration
docs/plans/
```

## API sketch — proposal only

```php
<?php
$server = \RunwireCore\Server::http('127.0.0.1:8080');
$server->onRequest(static function ($request, $response): void {
    $response->end('hello');
});
$server->run();
```

Names and signatures are undecided; validate ergonomics, exception propagation, body streaming, cancellation and shutdown semantics before freezing a public API. Do not copy `Infocyph\Runwire` PHP classes as a dependency.

## Roadmap / implementation tracker

| Batch | Deliverable | Gate | Status |
|---|---|---|---|
| C0 | Feasibility: PIE, Zend, NTS/ZTS, callback lifetime, fork strategy | Repeatable isolated extension load/call/unload, no unsafe cross-thread callbacks | [ ] Not started |
| C1 | Architecture/ABI/API contracts | Reviewed ownership/exception/lifetime and platform matrix | [ ] Not started |
| C2 | Native engine, reactor and timer/command queues | Rust correctness/fuzz tests, bounded queues, clean shutdown | [ ] Not started |
| C3 | Standalone HTTP/1.1 with PHP callbacks | Real-client interop, slowloris/body/stream/backpressure coverage | [ ] Not started |
| C4 | Workers, lifecycle and reload | Fork-safe initialization, restart/recycle, drain/race fault tests | [ ] Not started |
| C5 | TLS and HTTP/2 | ALPN/flow control/header limits/multiplexing/interoperability | [ ] Not started |
| C6 | Optional HTTP/3 / QUIC | Independent QUIC clients, QPACK/rate/stream limits, no unsafe 0-RTT | [ ] Not started |
| C7 | Outbound async API and scheduling | Cancellation, deadline, wakeup fairness, no blocking reactor | [ ] Not started |
| C8 | Metrics, security and fault injection | Resource ceilings, crash/restart, cgroup RSS, observability off-path | [ ] Not started |
| C9 | Matched benchmark and performance iteration | p50/p95/p99, CPU/RSS/RPS; representative competitors/workloads | [ ] Not started |
| C10 | PIE release matrix + documentation | Linux x64/ARM64 PHP 8.4/8.5 NTS/ZTS, reproducible signed artifacts where supported | [ ] Not started |

### C0 proof-of-concept tasks (must come first)

1. Create minimal Rust extension with one PHP callable and a class exposed by Zend.
2. Package/install via PIE on a clean Linux PHP 8.4 CLI; repeat PHP 8.5 and other target variants.
3. Prove safe callbacks, string/value copying, failure/exception translation, and shutdown/reload.
4. Simulate native task completion while a PHP callback is scheduled; show callback always runs in valid PHP context.
5. Test NTS and ZTS, multiple requests in process, memory ownership, sanitizers where available.
6. Measure PHP call-to-native overhead, callback overhead, allocations and CPU under concurrency.
7. If Zend safety or native/PIE assumptions fail, redesign before implementing protocols.

### C2–C7 runtime scope

- Native engine should use Rust libraries for the parts they are good at (Tokio, Hyper/h2, rustls where appropriate, QUIC/h3 after validation) rather than implementing all protocol codecs anew.
- Define listener, connection, request/response, per-worker control, graceful drain and metrics lifetimes.
- Prevent cross-thread PHP resource handles; response and request stream APIs need bounded ownership and lifetime checks.
- Backpressure: listener admission -> in-flight request bound -> PHP dispatch queue -> response write -> socket flush; rejected work must not grow memory indefinitely.
- HTTP/1: keepalive, pipelining policy, chunked and streaming bodies, timeouts and malformed request rejection.
- HTTP/2: proper ALPN, cancellation, flow control, GOAWAY and fairness.
- HTTP/3 only once native thread/process and PHP callback semantics have proven stable.

## Benchmark and certification contract

- Compare **independent** Core with the PHP Runwire library, FrankenPHP, RoadRunner, Swoole/OpenSwoole using matched transports, worker count, PHP version, payload, TLS, OPcache, duration and independent clients.
- Workloads: hello/plaintext, JSON, streaming 1MiB, H2 multiplexing, WebSocket where supported, outbound I/O wait, mixed CPU+I/O, saturation and shutdown under load.
- Publish raw measurements and command lines, including latency distribution, errors, dropped requests, CPU and peak RSS; also measure callback dispatch and FFI boundary separately.
- Performance promotion criteria are proposed, not guaranteed: zero correctness failures; no meaningful p99/RSS regression; sustained RPS improvement above repeated measurement noise for prioritized workloads.
- Run fault injection: PHP callback exception, malicious request, slow consumer, Rust panic containment, abrupt process kill, worker replacement, service stop during streaming.
- Rust unit/property/fuzz tests, PHP integration suite, cargo fmt/clippy/audit and artifact-load smoke on every supported build.
- Release gate requires completed non-skipped certification on exact candidate tag SHA.

## Open design decisions

1. Is a pure PIE extension enough for standalone server ownership, or should CLI worker management include an optional companion native launcher?
2. PHP executor architecture under ZTS/NTS; document selected threading/process model with proof.
3. Exact public namespace, registration and PHP callback context; select only after C0.
4. Whether to support persistent worker user state across request boundaries and how PHP request lifecycle maps to native worker lifecycle.
5. Which transport functions are materially cheaper in Rust after including FFI cost?
6. Extension build strategy and supported platforms, including reproducible trusted binaries.
7. Default TLS implementation and dependency/upgrade cadence.

**All tasks are open. This plan intentionally does not create runtime implementation or claim performance benchmarks.**
