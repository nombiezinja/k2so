# K2SO Technical Design Document
[See requirements in the README](README.md#requirements) condensed from(https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

## Naming Choice
- [K2SO is a blunt security bot who tells it like it is](https://starwars.fandom.com/wiki/K-2SO)
- K2SO is a good little bot and died too early; this control plane would die too early as well since everything is built with minimal scope and minimal prod considerations around reliability and resilience
- A more serious name is preferred for prod-ready projects (e.g. processctl,jobctl,etc). Boring is better for enterprise tooling.

## Architecture 
- CLI client (thin): minimal input validation, presents mTLS cert, constructs gRPC requests
- gRPC server: deep input validation, handles authn/authz, spawns jobs, manages streaming
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
- Identity extracted from SAN URI; CN ignored; EKU=ClientAuth required.
- Future: enterprise CA/PKI, cert rotation (see [Future Work](#future-work))

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
- Job Management: Principals can only `delete`, and `describe` jobs they own
- Cross-Principal Access: Denied - principals cannot access other principals' job logs or metadata
- Rationale: Prevents information leakage between different users/services sharing the same K2SO instance

#### Principal Attributes (from client cert SAN fields)
- User ID only: `email:nimbus@example.com` or `URI:urn:principal:nimbus`

#### Actions and Resources
- Actions: `run`, `delete`, `describe`, `logs`
- Resources: job instances, identified by job-id
- Future: OPA/Rego policies, cert revocation, resource tagging (see [Future Work](#future-work))

### 5 - Output Streaming 

#### Storage & Memory Management
- Anonymous temp files per job (unlink after open) to prevent unbounded heap growth
- Single writer (server goroutine), multiple concurrent readers via `os.File.ReadAt`
- Capture combined stdout/stderr to single file (no headers needed) (Not Prod Ready no per-stream tags, ordered per-write as delivered)
- Location: `/tmp/k2so/job-<uuid>.out`, e.g., `/tmp/k2so/job-a1b2c3d4.out` (Not Prod ready would prefer to put in `/run`)

#### Client Experience
- Multiple concurrent clients supported via independent streams
- `k2so <job-id> logs` snapshot 
- `k2so <job-id> logs -f` live updates
- Job completion closes all streams
- Server doesn't track per-client read positions, each gRPC stream is independent

#### Efficiency
- Buffered channel (cap=1) prevents blocking writers
- Notifications are hints, each reader does its own tracking

#### Future
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
- Not implemented: `k2so login`, `k2so stop`(no resuming jobs we are stupid)

```bash
# Execute commands
k2so run /usr/bin/ls -la /tmp
k2so run /usr/bin/ruby /path/to/script.rb
# View job outputs  
k2so logs <job-id>           
k2so logs -f <job-id>       
# Job management
k2so describe <job-id>       
k2so delete <job-id>       
```

## Trust Boundaries

### Assumptions 
Linux (cgroup v2), single host, dev CA with long-lived leaf certs

### In Scope
- AuthN: TLS 1.3 mTLS; SAN-based identity; EKU=ClientAuth (see [Authentication](#3---authentication))
- AuthZ: deny-by-default ABAC (subject × resource × action × context) (see [Authorization](#4---authorization))
- Input validation: absolute path, EvalSymlinks, allow-listed dirs, arg/env caps (see [Input Validation Details](#input-validation-details))
- Execution: direct execve (no shell/TTY), spawn in a new PGID (SysProcAttr{Setpgid:true}) and signal the PGID, env allow-list, umask 077
- Isolation: PGID signals (TERM 60s timeout -> SIGKILL); per-job output caps (L5: cgroup limits)
- Runtime storage: `/tmp/k2so/job-<uuid>.out`; unlink after open (see [Storage & Memory Management](#storage--memory-management))

### Out of Scope
- host OS hardening, network egress controls, multi-node/HA, persistent history, client environment (see [Future Work](#future-work))

### Controls by Boundary
- Network: TLS1.3 only, mutual auth, pinned CA (see [Authentication](#3---authentication))
- Identity: principal from SAN URI; ignore CN; long-lived certs (dev only) (see [Authentication](#3---authentication))
- Filesystem: absolute path within allow-listed roots; anonymous temp files (see [Input Validation Details](#input-validation-details))
- Process: execve + setpgid(); no PATH lookup; explicit cwd; O_CLOEXEC on all FDs
- Resource/DoS: arg/env/chunk/output caps; max concurrent jobs (see [Non-functional Requirements](#non-functional-requirements))

### Principles
deny-by-default; server-generated job IDs; no shell interpretation; binary-safe streaming (see [CLI UX](#cli-ux-kubectl-style-minimal), [Output Streaming](#5---output-streaming)).

### Input Validation Details
- Cap arg size at ~64 args, ~4kb per arg/env to prevent abuse
- Direct `exec.Command(binary, arg1, arg2, ...)` usage maps to Linux `execve()` without interpretation
- Future: whitelist of allowed binary directories (see [Future Work](#future-work))
- Validate file exists & is executable by the service user
- No shell is invoked; args are passed verbatim to execve (no global metachar bans)
- Future - for clients that attempt to execute a shell, can add shell detection and warning, and policy for principals allowed to execute shells

## Proposed API
[See the gRPC API definition in k2so.proto](proto/k2so/v1/k2so.proto)

## Edge Cases
- Two clients execute same command simultaneously: separate process instances, independent job IDs
- Multiple clients streaming same job: see [Client Experience](#client-experience)
- Principal attempts to access another principal's job: `PERMISSION_DENIED` (see [Authorization](#4---authorization))
- Late joiners to active job: snapshot mode gets full history (see [Storage & Memory Management](#storage--memory-management))
- Slow/hanging clients: gRPC flow control handles automatically (see [Efficiency](#efficiency))
- Client disconnection: gRPC context cancellation cleans up, temp files remain available
- Job becomes unresponsive: SIGTERM → SIGKILL lifecycle (60s timeout)
- Server graceful shutdown: SIGTERM to all jobs (60s drain), then SIGKILL cleanup, active streams get cancellation
- Server crash/kill -9: temp files and job registry lost, processes orphaned (L4 design limitation)
- Resource limits exceeded: return `RESOURCE_EXHAUSTED`, deny new job creation

## Milestones
See README.md 

## Future Work

### L5 Stretch Goals (Feasible for Challenge)
- process tree termination: ensure job's child processes are also terminated on stop no zombies
- cgroup v2 resource control- per-job cpu.max, memory.max, optional io.max
- process groups -proper signal propagation to all descendants  
- enhanced job lifecycle- graceful shutdown with configurable timeouts
- basic resource monitoring: track CPU/memory in describe
- per-job output cap and global limits; fail with RESOURCE_EXHAUSTED

### Production-Level Features (Beyond Challenge Scope)
- HA control plane w leader election, distributed runners, failover
- Using systemd (invoked via systemctl), apply cgroup limits, and ensure TERM KILL teardown.
- distributed scheduling, runner pools 
- security - authz with OPA, write REGO policies, SPIFFE for who is calling, secrets management, cert rotation
- observability- OTEL traces, Prometheus metrics, structured logs, ebpf stuffs
- advanced isolation: seccomp/AppArmor/userns, rlimits, capabilities 
- storage - db-backed metadata, persistent output, retention + encryption at rest
- Multi-tenant: namespaces, per-tenant quotas
- supply chain and release integrity - signed binaries, SLSA, SBOMs
- platform maturity - versioned APIs, SLOs, chaos engineering, disaster recoveryy, rate limits, stream backpressure
- enterprise ready - short-lived certs generated for clients after SSO, compliance stuffs, policy-as-code

