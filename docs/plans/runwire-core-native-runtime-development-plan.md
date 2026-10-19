# Runwire Core — independent Rust native PHP runtime: full draft plan

**Status:** DRAFT / detailed implementation plan; all tasks open
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

## Detailed development specification and task-level tracker

The C0–C10 batch table above remains the progress summary. **The numbered tasks here are the authoritative execution checklist.** Every box is initially open. Proposed file paths are future deliverables because main currently contains only LICENSE; they are *not* claims of existing code, defects, benchmarks, or verified platform support.

### Decisions already approved, and items still gated

- **Approved:** standalone Rust native runtime installed as a PHP extension with PIE; independent from PHP Runwire and Runwire-Cells, with no required package dependency or runtime ABI bridge to either.
- **Approved:** own public PHP namespace, execution/transport implementations, releases and CI; PHP 8.4/8.5 and Linux x86_64/aarch64 are initial targets, subject to feasibility and runner availability.
- **Not yet approved:** exact user API signatures; callback scheduling model; pre-fork vs spawn model; server startup modes; stable binary ABI across PHP versions; supported HTTP/3 implementation; ZTS concurrency claims; Windows support; performance claims.
- **Non-negotiable:** callbacks and PHP values stay in a valid Zend/TSRM execution context; no arbitrary worker-thread invocation. Correctness, tenant/request isolation, bounded resources, and safe shutdown dominate raw RPS.
- **Architecture ownership:** Rust owns socket acceptance, protocol state and native scheduling, but PHP execution and PHP request lifecycle belong to the PHP VM and intentionally bound PHP executor context. Mixing these owners without a documented interface is forbidden.
- **Release precedence:** first prove PIE and PHP execution safety (C0), then freeze an implementation contract (C1), then construct engine (C2+). A failed feasibility proof may require design change; do not paper it over with unsafe FFI.

### Planned package layout and artifacts

- Cargo workspace with explicit crates for Zend binding/extension entry, engine/reactor, HTTP transport, process lifecycle, metrics and optional QUIC; split only where it reduces real ownership or test complexity.
- PIE php-ext package manifest and build integration suitable for clean source installation, including Cargo toolchain requirements, PHP headers and phpize; verify exact PIE/Composer schema and installed module name in C0. The typo-free packaging name proposed elsewhere is infocyph/runwire-core.
- PHP-facing stubs/arginfo/documentation consistent with actual exported Zend symbols; PHP-specific tests and PHPStan-facing reflection/introspection tests.
- Rust unit/property tests and fuzz targets for native parsers/state transitions, plus black-box real-socket tests driven by an independent client.
- Benchmarks for native-vs-PHP invocation overhead, end-to-end HTTP1/2/3, CPU/RSS, response correctness, connection and worker churn.
- An evidence manifest in docs/plans/evidence/ listing exact build hashes, PHP configure flags, extension version, CI jobs, commands and raw artifact links.

### Universal per-task and per-batch completion policy

1. Before coding, record the active Cn task, design decision, motivating evidence and exact files to touch. Do not make another project a dependency.
2. Implement the smallest correct slice; explicitly preserve native reactor / PHP executor / PHP request lifetime boundaries.
3. Verify positive, negative, overload, cancellation and shutdown behavior, with independent clients for every supported HTTP version.
4. Run affected Rust unit/fuzz/property tests, PHP binding/integration tests, real runtime acceptance, security checks, format/lints and performance comparison.
5. For hot paths report successful sustainable RPM (RPS × 60), p50/p95/p99, errors, CPU and peak/plateau RSS. Distinguish Rust-only microbenchmarks from full PHP callback service.
6. A performance regression, leaked zval, duplicate callback, unsafe thread crossing or unbounded queue blocks completion regardless of green small unit tests.
7. Update task boxes and the parent batch row, attach reproducible evidence and commit; only move to the next batch after the current applicable QA gates pass.
8. A skipped job remains unverified; a failed experiment may be closed as **rejected with evidence**, not misrepresented as a successful feature.
9. Do not merge, tag, publish, silently remove tasks or mark a batch done without complete evidence.

### C0 — prove PIE + Zend safety before designing an engine

**Likely new paths:** Cargo.toml, Cargo.lock, composer.json, config.m4 or verified equivalent, src/lib.rs, tests/php/smoke.php, scripts/build-smoke.sh, docs/architecture/feasibility.md.

- [ ] **C0.01** Confirm current PIE php-ext package contract, supported installation modes and Rust/Cargo integration with a clean PHP 8.4 installation; document actual build commands, logs and dependency packages.
- [ ] **C0.02** Add minimal Rust extension registering module version, a harmless function and one PHP class; smoke-test extension_loaded, reflection and PHP value conversion.
- [ ] **C0.03** Validate extension lifecycle module init/shutdown and request init/shutdown across CLI repeated requests or supported persistent PHP contexts; reject access after teardown.
- [ ] **C0.04** Independently test clean PIE installation and unloading on PHP 8.4 and PHP 8.5, identifying PHP ABI/Zend API requirements and library load-path behavior.
- [ ] **C0.05** Test NTS and ZTS builds separately where available; document what PHP callback concurrency is actually permitted. Do not equate Rust thread safety with PHP VM thread safety.
- [ ] **C0.06** Build safe native-to-PHP completion proof using bounded queue; invoke the PHP callback exclusively from a verified PHP executor context and fail on wrong thread/generation.
- [ ] **C0.07** Validate String/zval ownership, conversion, exception propagation, panic containment, reference counting and deallocation with repeated and large values.
- [ ] **C0.08** Test cancellation and shutdown when native operations finish after the PHP request has exited; completion must be discarded or safely transferred, never dereference dead PHP state.
- [ ] **C0.09** Perform separate proof for PHP worker fork before/after native Tokio/reactor/thread initialization; prohibit inheriting thread pools or mutex state unsafely.
- [ ] **C0.10** Measure PHP→Rust invocation and Rust→PHP completion round-trip latency/allocation versus equivalent PHP-only callback dispatch under realistic repetition.
- [ ] **C0.11** Establish Rustfmt, Clippy, dependency/security audit, PHP lint/functional smoke and basic CI on the exact prototype SHA.
- [ ] **C0.12** Publish feasibility ADR: passing platforms, Zend executor model, selected PIE packaging mechanism, fork strategy, unsupported assumptions and explicit go/no-go.

**QA gate:** Real extension built and loaded through PIE, memory ownership proven, no invalid cross-thread PHP callback, and module/request shutdown tests pass on supported builds. If fail, architecture revisited before C1.

### C1 — specification, API and native/PHP ownership contracts

**Likely paths:** docs/architecture/runtime-ownership.md, docs/architecture/php-api.md, docs/adr/, php/stubs/, src/api/*.

- [ ] **C1.01** Select the PHP execution architecture (bound single interpreter/worker, managed PHP process, or independently proven ZTS variant) and document callback serialization/parallelism.
- [ ] **C1.02** Define module, extension and namespace names; freeze first API surface for Server, Listener, Request, Response, Options, Cancellation, Context and Metrics based on PHP feasibility.
- [ ] **C1.03** Specify Server state machine: configured → bound → starting → ready → draining → stopped/failed; invalid transitions fail deterministically.
- [ ] **C1.04** Specify request lifetime: accept, decode, queue, dispatch, respond, flush, complete, cancel; define exactly-once callback and terminal response semantics.
- [ ] **C1.05** Define bounded ingress, connection, response and callback queues plus overload/rejection behavior with no hidden unlimited buffers.
- [ ] **C1.06** Specify configuration validation, defaults, errors, exception codes, timestamp/deadline model and immutable per-server/worker options.
- [ ] **C1.07** Document Zend ownership policy for PHP closures/objects, borrowed strings, native handles, callbacks, exceptions, nested invocations and request shutdown.
- [ ] **C1.08** Document multi-worker resource initialization, restart, signals, crash boundaries and native context generation; distinguish process ownership from thread ownership.
- [ ] **C1.09** Define API scope for v1: HTTP1/native sockets first; H2, H3, outbound client and optional WebSocket have separately gated milestones, not automatically promised in MVP.
- [ ] **C1.10** Publish testable API conformance fixtures and ADR approvals, including rejected designs and examples of valid/invalid consumer usage.

**QA gate:** Peer-reviewed spec and executable conformance skeleton that prevents accidental ownership changes later.

### C2 — Rust reactor, execution queues and lifecycle engine

**Likely paths:** src/engine/*, src/reactor/*, src/net/*, src/api/*, tests/rust/engine_*.

- [ ] **C2.01** Select and pin Rust asynchronous I/O backend (Tokio/mio or measured equivalent), define feature flags and ownership; no second hidden reactor per request.
- [ ] **C2.02** Implement independently owned engine creation/start/stop with idempotent lifecycle, stable diagnostics and explicit failure states.
- [ ] **C2.03** Implement bounded accept queue, bounded callback-dispatch queue and bounded socket-completion queue; overload must reject rather than grow indefinitely.
- [ ] **C2.04** Implement timer/deadline wheel or existing library mechanism with monotonic timing and safe timer cancellation.
- [ ] **C2.05** Implement generation-safe opaque native handles: close/reuse cannot accidentally target a newly allocated connection or stale worker.
- [ ] **C2.06** Implement safe completion marshaling to PHP owning executor and starvation-resistant dispatch batching; never execute PHP on arbitrary Tokio threads.
- [ ] **C2.07** Implement cancellation ownership and propagation: user cancel, connection abort, response completion, request timeout, worker drain and engine stop.
- [ ] **C2.08** Define Rust panic handling, task failure logging and shutdown policy; panic in one request must not cause undefined PHP behavior.
- [ ] **C2.09** Add concurrency/model tests for enqueue-vs-close, completion-vs-cancel, timer-vs-shutdown, double stop and handle reuse.
- [ ] **C2.10** Benchmark engine idle CPU, timer error, queue throughput, per-connection memory and PHP callback scheduling overhead; confirm no busy-spin or unbounded state.

**QA gate:** Synthetic engine remains safe under heavy concurrency, stale completions and repeated shutdown, with no PHP lifetime violation.

### C3 — HTTP/1.1 server and PHP request/response execution

**Likely paths:** src/http/h1/*, src/net/listener.rs, src/api/server.rs, src/api/request.rs, src/api/response.rs, tests/php/http1_*, tests/integration/h1_*.

- [ ] **C3.01** Build TCP bind/listen/accept and clear ownership of listening socket, accepted connections and PHP callback executor.
- [ ] **C3.02** Use established Rust HTTP/1 stack where compatible; document parser/library selection, version pinning and patch policy instead of writing another custom parser.
- [ ] **C3.03** Normalize request method, target, headers, body stream and connection metadata into bounded PHP API values without eager unbounded buffering.
- [ ] **C3.04** Support correct status/header/body semantics, HEAD/no-body statuses, transfer framing, chunked streaming and terminal write-once behavior.
- [ ] **C3.05** Implement bounded buffered writes and backpressure to PHP callbacks; partial writes, stalled client and closed peer must resolve explicitly.
- [ ] **C3.06** Enforce request line, header count/bytes, body limits, connection limits, idle/total/request timeouts and pending PHP callback ceilings.
- [ ] **C3.07** Validate keep-alive and connection reuse; decide pipelining behavior explicitly and prohibit request state overlap if PHP executor is serialized.
- [ ] **C3.08** Validate parsing failures, malicious body-length conflicts, invalid transfer encoding, slowloris and half-open clients with independent client harness.
- [ ] **C3.09** Handle PHP thrown exceptions, PHP fatal/bailout boundary, Rust handler failure and transport shutdown; no double response or use-after-request.
- [ ] **C3.10** Publish basic PHP server example and measure full PHP handler successful RPM, latency, RSS and correctness against a matched standalone native runtime baseline.

**QA gate:** Real independent HTTP client interoperability, correct PHP callbacks, malicious request rejection, bounded memory and safe stop.

### C4 — workers, supervision, reload and graceful recycling

**Likely paths:** src/process/*, src/engine/worker.rs, src/api/runtime.rs, tests/integration/worker_*.

- [ ] **C4.01** Choose process architecture after C0 (separate PHP child workers vs other proven model) and prove no unsafe initialized-Rust runtime survives a fork.
- [ ] **C4.02** Implement readiness handshake with bounded startup/warmup timeout and clear error propagation to supervisor.
- [ ] **C4.03** Support configured worker count, CPU/cgroup awareness where practical, listener ownership and fair traffic distribution.
- [ ] **C4.04** Implement child crash detection, bounded restart/backoff and intentional stop distinction; prevent restart storms.
- [ ] **C4.05** Implement graceful drain boundaries: stop admitting new work, settle accepted requests within deadline, then close with auditable completion/error semantics.
- [ ] **C4.06** Implement rolling reload/recycling with fresh PHP/Zend and native reactor resources; invalidate stale connection/request/generation handles.
- [ ] **C4.07** Implement signal, Ctrl-C, container stop and unexpected supervisor death behavior for all supported OSes; no orphan child processes.
- [ ] **C4.08** Protect from resource exhaustion via admission, FD/connection capacity, limits and OS/container hard boundaries; distinguish PHP allocator from native RSS.
- [ ] **C4.09** Test worker kill -9, failed warmup, repeated reload, partial responses, stalled PHP callbacks and restart under request pressure.
- [ ] **C4.10** Document deployment topology and verify real multi-worker end-to-end PHP callback correctness and stable resource consumption.

**QA gate:** Worker/process ownership demonstrably correct; reload, abrupt failure and graceful drain have bounded, proven outcomes.

### C5 — TLS, ALPN and HTTP/2

**Likely paths:** src/net/tls.rs, src/http/h2/*, src/api/tls.rs, tests/integration/h2_*.

- [ ] **C5.01** Evaluate Rust TLS stacks, PHP trust/cert configuration and build reproducibility; define supported cipher/protocol policies.
- [ ] **C5.02** Implement listener TLS with certificate/key validation, hostname/SNI handling where exposed, explicit configuration and fail-closed startup.
- [ ] **C5.03** Integrate ALPN selection for h2 and HTTP/1.1; prohibit unintended plaintext or protocol downgrade.
- [ ] **C5.04** Reuse established H2 protocol framing/HPACK implementation, pin version and limit HPACK/headers/frames/settings independently.
- [ ] **C5.05** Map multiplexed streams to PHP callbacks and responses without confusing stream cancellation with whole connection cancellation.
- [ ] **C5.06** Implement correct window updates, bounded stream concurrency, fair scheduling, GOAWAY/drain and response backpressure.
- [ ] **C5.07** Exercise independent H2 clients, invalid frames, header bombs, reset storms, slow receivers and simultaneous worker recycling.
- [ ] **C5.08** Benchmark TLS handshakes, resumed/keepalive requests, H2 multiplexing, CPU/memory cost and p99 under saturation.
- [ ] **C5.09** Provide TLS/H2 operator documentation and secure config templates; no hidden insecure development default in production.

**QA gate:** Independent H2 protocol and security tests pass; request cancellation, flow control and shutdown are bounded.

### C6 — optional native QUIC / HTTP/3

**Likely paths:** src/http/h3/*, src/quic/*, tests/integration/h3_*.

- [ ] **C6.01** Reassess release scope after C5; record explicit release/defer decision based on PHP callback safety and H1/H2 maturity.
- [ ] **C6.02** Select pinned QUIC+HTTP/3 library stack and TLS integration; define supported targets and build dependencies.
- [ ] **C6.03** Implement UDP listener/QUIC connection ownership, safe handshake, no 0-RTT PHP application dispatch unless independently security-reviewed.
- [ ] **C6.04** Enforce QPACK, stream, header block, connection and control-frame bounds; protect against untrusted encoded sizes.
- [ ] **C6.05** Integrate per-stream PHP dispatch, flow control, backpressure, resets, deadline and cancellation semantics.
- [ ] **C6.06** Implement graceful drain/GOAWAY, accepted-but-incomplete headers boundary, connection migration behavior and timeout rules.
- [ ] **C6.07** Test independent HTTP/3 clients including fragmentation, loss/reordering, invalid frames and adversarial stream churn.
- [ ] **C6.08** Benchmark H3 under matched TLS/network/warmup conditions with CPU/RSS, errors and p99; do not compare against H1 plaintext numbers.
- [ ] **C6.09** Document supported platforms and feature flag; if deferred, explicitly mark unsupported instead of claiming silently disabled support.

**QA gate:** If enabled, H3 interop/fault and memory bounds pass; otherwise record a justified scope deferral, not a finished H3 implementation.

### C7 — outbound networking and PHP async ergonomics

**Likely paths:** src/api/client.rs, src/net/dns.rs, src/net/connection.rs, tests/php/outbound_*.

- [ ] **C7.01** Define optional outbound TCP/TLS/socket APIs with documented PHP callback context and deterministic lifetime.
- [ ] **C7.02** Implement asynchronous connect and stream readable/writable operations with cancel/deadline semantics and no blocking native reactor.
- [ ] **C7.03** Define DNS resolution and TLS handshake execution model; guard timeout, failure, reconnect and address changes.
- [ ] **C7.04** Expose bounded read/write buffers, nonblocking send completion and peer close semantics to PHP.
- [ ] **C7.05** Investigate outbound HTTP client as optional package/feature; require value beyond existing PHP clients before exposing new API.
- [ ] **C7.06** Test simultaneous incoming handlers waiting on outbound I/O, duplicate completion, cancellation during DNS/handshake and slow peer.
- [ ] **C7.07** Ensure per-request context cannot be held alive indefinitely by native pending operations; graceful shutdown drains/cancels reliably.
- [ ] **C7.08** Benchmark overlapped I/O vs equivalent synchronous requests, including queue pressure, threads, CPU and response latency.
- [ ] **C7.09** Document precisely which operations are nonblocking; no claim that arbitrary PHP PDO, curl or filesystem work is auto-async.

**QA gate:** Proven application-level concurrency benefit without PHP VM misuse, stalled completions or unbounded queues.

### C8 — observability, security, resource budgets and incident resilience

**Likely paths:** src/metrics/*, src/engine/limits.rs, src/api/diagnostics.rs, docs/security.md, tests/fuzz/*, tests/integration/fault_*.

- [ ] **C8.01** Define low-overhead counters/histograms for admission, successful responses, failed callbacks, connection closure, queue depth, slow callback and cancellation.
- [ ] **C8.02** Expose stable bounded PHP metrics snapshots, enabled diagnostic profiles and version information without native state leaks.
- [ ] **C8.03** Distinguish PHP-managed memory, Rust heap/native buffers, process RSS, cgroup limit, FD count and thread count.
- [ ] **C8.04** Implement fail-closed defaults for headers, body, queue, connections, idle/handler duration, max PHP callback concurrency and outbound bytes.
- [ ] **C8.05** Fuzz/proptest request parsing, header canonicalization, protocol state, native-handle generation and encode/decode boundaries.
- [ ] **C8.06** Test PHP callback exception/bailout, Rust panic, native OOM-adjacent pressure, malformed TLS/protocol and client disconnect at every response stage.
- [ ] **C8.07** Verify server cannot become a privilege-escalation or remote arbitrary command surface through config, filesystem or metrics endpoints.
- [ ] **C8.08** Test graceful stop/crash recovery while serving requests and during connection churn; quantify lost in-flight work honestly.
- [ ] **C8.09** Run security/dependency audits including supply-chain integrity, build scripts, TLS updates and extension/module load permissions.
- [ ] **C8.10** Publish operational troubleshooting: latency attribution, blocked PHP callbacks, queue saturation, memory limits, backoff and misconfigured TLS.

**QA gate:** Fuzz/abuse/fault suite passes; instrumentation overhead and remaining resource risks are documented.

### C9 — matched performance, profiling and optimization

**Likely paths:** benchmarks/*, tests/performance/*, docs/benchmarks.md.

- [ ] **C9.01** Build repeatable independent client test runners with exact OS/CPU/cores, PHP version, extension build, network, PHP workers and OPcache provenance.
- [ ] **C9.02** Compare to existing standalone Runwire, FrankenPHP, RoadRunner and Swoole as *independent competitors*; never make any a runtime dependency.
- [ ] **C9.03** Match workloads: Hello plaintext, small JSON, streaming 64KiB/1MiB, H1 keepalive, H2 multiplexing, H3 if available, mixed CPU+I/O and overload.
- [ ] **C9.04** Record startup, warm first call, warm steady state, native↔PHP crossing overhead, Rust-only protocol cost and whole application successful RPM separately.
- [ ] **C9.05** Run concurrency sweep through throughput plateau and collapse; validate response contents, timeouts, error rate and resource use on every trial.
- [ ] **C9.06** Record median successful RPM and RPS, variance/CV, p50/p95/p99, CPU, RSS plateau, FD count and backpressure/queue growth.
- [ ] **C9.07** Profile hottest native and Zend-bound paths; prioritize end-to-end wins over microbenchmark vanity metrics.
- [ ] **C9.08** Optimize without unsafe aliasing, thread transfer, unbounded caching or excessive duplication; rerun tests and matched comparison after each candidate.
- [ ] **C9.09** Measure diagnostics-enabled overhead and NTS/ZTS overhead; document any target where performance is worse or feature missing.
- [ ] **C9.10** Publish reproducible evidence/graphs and failure cases; permit a no-gain outcome rather than falsely asserting superiority.

**QA gate:** Correct end-to-end performance evidence, representative sustained runs, no required unverified headline and no safety-regression tradeoff.

### C10 — independent PIE packaging, compatibility and release handoff

**Likely paths:** composer.json, config.m4/Makefile fragments, Cargo.lock, php/stubs/*, .github/workflows/*, README.md, docs/install.md, docs/release.md.

- [ ] **C10.01** Validate package name, module name, PHP namespace, PIE manifest type/install metadata and Composer registry collision policy.
- [ ] **C10.02** Verify clean PIE source installs for Linux x86_64 and aarch64 with PHP 8.4 and 8.5; document toolchain/root/build dependencies.
- [ ] **C10.03** Separately smoke-test NTS/ZTS supported configurations, CLI and intended persistent worker execution; unsupported combinations must fail informatively.
- [ ] **C10.04** Verify binary distribution only if PIE supports the chosen mechanism, with artifact integrity/provenance, per-ABI selection and a source fallback.
- [ ] **C10.05** Build test matrix for ext load/unload, reflection, class stubs, request lifecycle, full host integration and process restart; no leaked handles after module shutdown.
- [ ] **C10.06** Run format, Clippy, Rust tests, fuzz smoke, dependency audit, PHP tests, independent real clients, soak, security and matched performance.
- [ ] **C10.07** Document install/uninstall, production service setup, worker ownership, TLS, network ports, defaults, limits and diagnosis of common extension errors.
- [ ] **C10.08** Publish accurate API docs and failure/timeout semantics, with independent runnable PHP examples and no references to PHP Runwire packages as prerequisites.
- [ ] **C10.09** Re-run exact candidate release-certification and audit all skipped/cancelled jobs; record known unsupported platforms/security constraints.
- [ ] **C10.10** Prepare tagged-release checklist, provenance, changelog, license/dependency notices, reproducible binaries/source-build instructions and signed-off go/no-go.
- [ ] **C10.11** Leave plan branch unmerged, untagged and unpublished; publishing remains a separate maintainer action.

**QA gate:** Clean install, safe PHP lifecycle, complete supported target matrix and independently verifiable runtime behavior. No implied published package until maintainer release.

### Supported capability and fault matrix (expand with actual C0 results)

| Axis | Initial checks | Failure/negative cases |
|---|---|---|
| PHP | 8.4 and 8.5; NTS and ZTS where runner exists | invalid ABI, wrong PHP minor/build flags, missing phpize |
| Platform | Linux x86_64, aarch64; later optional macOS | missing runtime libraries, unsupported CPU/musl |
| Zend lifecycle | MINIT/MSHUTDOWN, RINIT/RSHUTDOWN or API equivalents | callback after request close, double destroy, panic during cleanup |
| Process | single process then multiworker, SIGTERM, rolling reload | stale callbacks after fork, orphan child, worker kill -9 |
| HTTP | H1, TLS/H2, optionally H3 | malicious body length, HPACK/QPACK abuse, stream cancellation |
| Networking | TCP, TLS, optional UDP/QUIC and outbound I/O | slow peer, half-close, partial write, timeout, queue overflow |
| Backpressure | accept, native completion, PHP callback, response queues | saturation, blocked callback, fairness and memory plateau |
| Security | TLS, config validation, bounded logging and metrics | wrong certificate, unsafe privilege, malformed inputs |
| Performance | real independent clients + PHP handlers | false success from invalid responses, short warmup, client bottleneck |

### Artifact and status log schema

For each task/batch log **Task ID**, decision/reproduction, source SHA, files, command and toolchain versions, PHP/extension/OS mode, positive and negative test counts, CI run URLs, raw performance artifacts, successful RPM, CPU/RSS/tail, any skip, defect references, reviewer decision and next task. Update the C0–C10 summary row only after every required task has evidence and the batch's relevant QA gate passed.

**No implementation, extension build, benchmark result, new release or compatibility claim is implied by this detailed draft.**


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
