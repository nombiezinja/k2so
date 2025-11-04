# K2SO Technical Design Document
[See requirements in the README](README.md#requirements) condensed from(https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

## Naming Choice
- [K2SO is a blunt security bot who tells it like it is](https://starwars.fandom.com/wiki/K-2SO)
- K2SO is a good little bot and died too early; this control plane would die too early as well since everything is built with minimal scope and minimal prod considerations around reliability and resilience
- A more serious name is preferred for prod-ready projects (e.g. processctl,jobctl,etc). Boring is better for enterprise tooling.

## Architecture 
- CLI client (thin): minimal input validation, presents mTLS cert, constructs gRPC requests
- gRPC server: handles authn/authz, spawns jobs, manages streaming
- Job registry: in-memory job metadata and ownership tracking, sync.Mutex for concurrent access (TODO: upgrade to RWMutex if read-heavy workloads emerge)
- Storage: anonymous temp files per job for bounded-memory output streaming

## Design Details

### 1 - Input Validation 
- Thin client, server-side validation. Client will use `flag` rather than `cobra`
  - Can upgrade in future should need arise
- Additional input validation rules in [Input Validation Details](#input-validation-details)

### 2 - Supported Job Types
- Executables on disk (e.g. /usr/bin/ls, /usr/bin/ruby)
- No raw syscalls: jobs execute as normal processes via `execve()` by `exec.Command`; clients cannot specify syscalls directly (server may use syscalls internally for resource management)
- Absolute path only; $PATH lookup and binary allow-lists in future (see [Future Work](#future-work))

### 3 - Authentication
- mTLS over TLS1.3 to satisfy requirement for strongest transport encryption
- TLS: Enforce TLS 1.3 only (`tls.Config{MinVersion: tls.VersionTLS13}`);[Go handles choice of cipher suite after 1.17](https://go.dev/blog/tls-cipher-suites)
- Ed25519 certs (rather than RSA, to achieve comparable security with lower network load)
- Long-lived root cert, short lived leaf certs generated via Makefile; avoid revocation complexity
- Identity extracted from SAN URI; CN ignored; EKU=ClientAuth required
- Future: enterprise CA/PKI, cert rotation (see [Future Work](#future-work));SPIFFE 

### 4 - Authorization 

#### ABAC Model
- Attribute-Based Access Control (ABAC) instead of RBAC for extensibility and security
- Authorization decision: `allow = f(principal_attributes, action, resource, context)`
  - context will be minimal for this challenge; future could include IP restrictions, rate limiting, etc. 
- Stateless evaluation: server only checks client cert attributes against hardcoded policies
- Principals must fulfill both action and resource check to perform request

#### Job Ownership and Access Control
- Job Creation: Any authenticated principal can create jobs (`run` action)
- Job Ownership: Jobs are owned by the principal that created them
- Log Access: Principals can only access logs (`logs` action) for jobs they own
- Job Management: Principals can only `stop`, and `describe` jobs they own
- Cross-Principal Access: Denied - principals cannot access other principals' job logs or metadata
- Rationale: Prevents information leakage between different users/services sharing the same K2SO instance

#### Principal Attributes (from client cert SAN fields)
- User ID only: `email:nimbus@example.com` or `URI:urn:principal:nimbus`

#### Actions and Resources
- Actions: `run`, `stop`, `describe`, `logs`
- Resources: job instances, identified by job-id
- Future: OPA/Rego policies, cert revocation, SPIFFE (see [Future Work](#future-work))

### 5 - Output Streaming 

#### Storage & Memory Management
- Anonymous temp files per job (unlink after open) to prevent unbounded heap growth
- Single writer (server goroutine capturing process stdout/stderr), multiple concurrent readers via shared `*os.File`
- Concurrent `ReadAt()` operations using `pread()` - no shared offset coordination needed
- Capture combined stdout/stderr to single file (no headers needed) (Not Prod Ready no per-stream tags, ordered per-write as delivered)
- Location: `/tmp/k2so/job-<uuid>.out`, e.g., `/tmp/k2so/job-a1b2c3d4.out` (Not Prod ready would prefer to put in `/run`)

#### Job Registry Management
- Thread-safe job map: `map[string]*Job` with `sync.Mutex` for simplicity (upgrade to RWMutex if read contention becomes an issue)
- Job ID Generation: UUIDv4 using `crypto/rand` package; 16-byte slice with `rand.Read()`, then bit manipulation to ensure v4 compliance. No external UUID dependency (e.g., google/uuid) to reduce build size and dependencies; only need v4 UUIDs without additional wrapper functionality.
- Writer goroutine lifecycle: context cancellation for cleanup when process exits or job is stopped by client instruction
- Reader cleanup: automatic via gRPC context cancellation when clients disconnect
- Error propagation: writer goroutine failures propagated to readers via `job.done.Store(true)` + final notification
- Late joiner coordination: new readers start from `readCursor=0` and catch up using existing atomic size tracking
- Job persistence: jobs remain in registry until server death; stopped jobs retain their status, metadata, and output files for continued access
- Output availability: temp files and content remain accessible for log streaming even after job termination (completion, failure, or stop)
- File descriptor management: job registry holds single FD per job to unlinked temp files until server shutdown; minimal FD usage with shared `*os.File` access

#### Client Experience
- Multiple concurrent clients supported via independent streams
- Principal can have multiple clients (i.e. multiple clients can read from same job if principal owns job)
- `k2so <job-id> logs` streams live output 
- Job completion - finish reading to end of output and close streams.
- Server doesn't track per-client read positions, each gRPC stream is independent

#### Efficiency & Notification Mechanism
Writer path (per-job):
- Input: pipes (stdout/stderr) -> mux -> single writer goroutine per job for centralized I/O
- Cursor update: append to temp file -> on successful append `atomic.StoreUint64(&job.bytesWritten, newSize)` -> non-blocking notify
- Notification: coalesced `notifyCh := make(chan struct{}, 1)` per job. Writer does `select { case job.notifyCh <- struct{}{}: default: }` (non-blocking)
  - Multiple reader coordination: single notification channel acceptable because notifications are efficiency hints only, channel close on finalize broadcasts to all waiters, and atomic size checks provide real coordination
  - Scalable design: avoids mutex contention from sync.Cond, optimized for high concurrent reader scenarios
- Reader File Access: All readers share the job's `*os.File` via an `io.ReaderAt` wrapper; concurrent `ReadAt()` uses `pread()` (no shared offset). `dup()` per reader is optional (FD isolation) but increases FD pressure.

Reader path (per-job):
- Cursor state: each gRPC stream keeps independent in-memory `readCursor` for the specific job
- Read loop: `toRead := min(64<<10, int(atomic.LoadUint64(&job.bytesWritten)-readCursor))` with partial read handling
- Blocking: race-free state-check loop where the terminal state (`job.done.Load()`) is checked both before blocking and immediately after reading all available data:
  ```go
  for {
    r.readAvailableData(job)  // drains until cursor == atomic.LoadUint64(&job.bytesWritten)
    if job.done.Load() {
      r.readAvailableData(job)  // Final read after terminal state
      return
    }
    select {
    case <-job.notifyCh:
    case <-ctx.Done():
      return
    }
  }
  ```
  This ensures readers never miss data via atomic coordination, with notifications providing efficiency hints for minimal latency.
  - `readAvailableData()`: Drains all available bytes until `cursor == atomic.LoadUint64(&job.bytesWritten)`, handling partial `ReadAt()` returns with retry loops.
- Flow control: gRPC backpressure isolates slow readers, preventing them from blocking writer or other readers

Safety & lifecycle (per-job):
- Secure file access: `os.OpenFile` with job-id filename and 0600 perms, use `O_CREATE|O_EXCL` to prevent race conditions (optional for minimal scope, job-id is uuid so collision negligible)
- Call `os.Remove()` immediately to unlink file and ensure anonymity and keep FD open
  - Unlinking (reduces inode link count to 0) while keeping writer FD open; inode persists anonymously until all FDs closed
- File Access Strategy: shared `*os.File` passed to all readers for concurrent `ReadAt()` operations. Anonymous file (post-unlink) remains accessible via existing FD. Readers use independent in-memory cursors with `pread()` syscall avoiding shared offset coordination
- Finalize sequence: `job.done.Store(true)` -> `close(job.notifyCh)` -> readers detect terminal state and exit after final read
- Panic-proof shutdown: entire finalization sequence guarded by `sync.Once` for exactly-once cleanup per job
- Writer exit guard: writer I/O loop checks `job.done.Load()` before processing new data to stop before FD cleanup

#### Future
-sync.Cond() implementation would bottleneck under high concurrent environment due to Mutex contension, should update implementation
-Use tmpfs (/run/k2so) so data never hits disk.
-O_TMPFILE (Linux): create nameless files from the start (no brief window with a name)
-memfd_create: pure RAM FD, cannot be linked; add seals to prevent writes/shrinks (needs x/sys/unix/CGO).
- Lock down process access: run under a dedicated service user, umask 077, per-job dir 0700; consider procfs hidepid=2 and disallow ptrace (Yama) to reduce /proc snooping.
- Always set O_CLOEXEC to prevent FD inheritance; don’t log FD paths (/proc/.../fd/...) or job IDs in places others can read.
- Output caps (128 MiB/job), return gRPC status code RESOURCE_EXHAUSTED (code 8) 
- Enforce per-job and global size caps; fail gracefully on exhaustion

### Non-functional Requirements
- Error handling:
  - Client-facing: clear gRPC status codes, short actionable error messages (no stack traces, redact sensitive info)
  - Server-side: persist error logs with pid/command and request id for failures/rejects; log all else to stdout
  - Uses [canonical gRPC status codes](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)
- Resource limits: Max 100 concurrent jobs globally, max 10 per principal (configurable)
- File descriptor safety: O_CLOEXEC on all opens, cleanup on client disconnect
- No metrics by default to keep scope small; if needed, could expose expvar counters (errors_total, jobs_running) behind a flag (TODO, not implemented).
- Future: observability, metrics (see [Future Work](#future-work))

### Build and Development 
- Makefile for builds, cert generation; minimal CI with linting
- stdlib + gRPC only; built-in race detection (`go test -race`, `go build -race`)
- Minimal test coverage for authn, authz, core paths; uses testify/require
- Hardcoded configs with TODO comments for future extensibility

### CLI UX (kubectl-style, minimal)
- Not implemented: `k2so login` 

```bash
# Execute commands
k2so run /usr/bin/ls -la /tmp
k2so run /usr/bin/ruby /path/to/script.rb
# View job outputs, behaves like -f 
k2so logs <job-id>           
# Job management
k2so describe <job-id>       
k2so stop <job-id>
```

## Trust Boundaries

### Assumptions 
Linux (cgroup v2), single host, dev CA with long-lived root certs and short-lived leaf certs

### Principles
deny-by-default; server-generated job IDs; no shell interpretation; binary-safe streaming (see [CLI UX](#cli-ux-kubectl-style-minimal), [Output Streaming](#5---output-streaming)).

### In Scope
- AuthN: TLS 1.3 mTLS; SAN-based identity; EKU=ClientAuth (see [Authentication](#3---authentication))
- AuthZ: deny-by-default ABAC (subject × resource × action × context) (see [Authorization](#4---authorization))
- Input validation: absolute path, EvalSymlinks, arg caps (see [Input Validation Details](#input-validation-details))
- Execution: direct execve (no shell/TTY), default env only, umask 077
- Isolation: PID-only signals (SIGKILL-only for min scope); deliberate choice over PGID for L4 simplicity, aware of child process orphaning risk; per-job output caps (L5: cgroup limits)
- Runtime storage: `/tmp/k2so/job-<uuid>.out`; unlink after open (see [Storage & Memory Management](#storage--memory-management))

### Out of Scope
- host OS hardening, network egress controls, multi-node/HA, persistent history, client environment (see [Future Work](#future-work))

### Controls by Boundary
- Network: TLS1.3 only, mutual auth, pinned CA (see [Authentication](#3---authentication))
- Identity: principal from SAN URI; ignore CN; long-lived certs (dev only) (see [Authentication](#3---authentication))
- Filesystem: absolute path within allow-listed roots; anonymous temp files (see [Input Validation Details](#input-validation-details))
- Process: execve only; no PATH lookup; explicit cwd; O_CLOEXEC on all FDs
- Resource/DoS: arg/chunk/output caps; max concurrent jobs (see [Non-functional Requirements](#non-functional-requirements))

### Input Validation Details
- Cap arg size at ~64 args, ~4kb per arg to prevent abuse
- Basic validation only
- Direct `exec.Command(binary, arg1, arg2, ...)` usage maps to Linux `execve()` without interpretation
- Validate file exists & is executable by the service user
- No shell is invoked by server; args are passed verbatim to execve (no global metachar bans)
- Future: 
  - whitelist of allowed binary directories (see [Future Work](#future-work))
  - shell detection and policy controls - shells provide interactive access,scripting capabilities,shell-built-ins, which may not be appropriate for all principals in a multi-tenant environment
  - No custom environment variables - jobs run with server's default environment only (minimal scope)

## Proposed API
[See the gRPC API definition in k2so.proto](proto/k2so/v1/k2so.proto)

## Edge Cases
- Two clients execute same command simultaneously: separate process instances, independent job IDs
- Multiple clients streaming same job: see [Client Experience](#client-experience)
- Principal attempts to access another principal's job: `PERMISSION_DENIED` (see [Authorization](#4---authorization))
- Late joiners to active job: snapshot mode gets full history (see [Storage & Memory Management](#storage--memory-management))
- Slow/hanging clients: gRPC flow control handles automatically (see [Efficiency](#efficiency))
- Client disconnection: gRPC context cancellation cleans up, temp files remain available
- Server graceful shutdown: SIGTERM to all jobs (60s drain), then SIGKILL cleanup, active streams get cancellation
- Job becomes unresponsive: SIGKILL-only for simpler L4 implementation
  - Future: SIGTERM -> SIGKILL lifecycle (60s timeout)
- Server crash/kill -9: temp files and job registry lost, processes orphaned (L4 design limitation)

## Milestones
[Project milestones](README.md#milestones)

## Future Work

### L5 Stretch Goals (For Future Poking-around)
- DoS prevention: mandatory hard file size limit (100 MiB) per job with truncation/overwrite policy when hit
- process tree termination: upgrade from PID-only to PGID signals (SysProcAttr{Setpgid:true}) to ensure job's child processes are terminated and prevent orphaned processes
- cgroup v2 resource control- per-job cpu.max, memory.max, optional io.max 
- process groups - proper signal propagation to all descendants  
- enhanced job lifecycle- graceful shutdown with configurable timeouts
- basic resource monitoring: track CPU/memory in describe
- per-job output cap and global limits; fail with RESOURCE_EXHAUSTED

### Production-Level Features (Beyond Challenge Scope)
- HA control plane w leader election, distributed runners, failover
- Using systemd (invoked via systemctl), apply cgroup limits, and ensure TERM KILL teardown
- distributed scheduling, runner pools 
- security - authz with OPA, write REGO policies, SPIFFE for who is calling, secrets management, cert rotation
- observability- OTEL traces, Prometheus metrics, structured logs, ebpf stuffs
- advanced isolation: seccomp/AppArmor/userns, rlimits, capabilities 
- storage - db-backed metadata, persistent output, retention + encryption at rest
- Multi-tenant: namespaces, per-tenant quotas
- supply chain and release integrity - signed binaries, SLSA, SBOMs
- platform maturity - versioned APIs, SLOs, chaos engineering, disaster recovery, rate limits, stream backpressure
- enterprise ready - short-lived certs generated for clients after SSO, compliance stuffs, policy-as-code

