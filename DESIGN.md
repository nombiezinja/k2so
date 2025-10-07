# K2SO Technical Design Document
[See requirements in the README](README.md#requirements) condensed from [original requirements](https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

## Naming Choice
- [K2SO is a blunt security bot who tells it like it is](https://starwars.fandom.com/wiki/K-2SO)
- K2SO is a good little bot
- K2SO died too early, this control plane would too since everything is built with minimal scope and minimal prod considerations around reliability and resilience
- A more serious name is preferred for prod-ready projects (e.g. processctl,jobctl,etc). Boring is better for enterprise tooling.

## Architecture 
- CLI Client (thin): Presents mTLS cert, constructs gRPC requests
- gRPC Server: Handles authn/authz, delegates to Runner
- Job Runner: Spawns jobs, captures output to temp files
- Storage: Anonymous temp files for bounded-memory output streaming `/tmp/k2so/job-uuid1234.out`; clean up on server shut down

## Trust Boundaries

### Security Responsibilities
**K2SO Server is responsible for:**
- Input validation on all gRPC requests
- Secure process execution via direct `execve()`
- mTLS authentication and authorization
- Process isolation and resource management

**Outside of K2SO's security scope:**
- User's local shell environment (login/non-login, aliases, history)
- Client-side command construction and quoting
- User's local filesystem permissions and access
- User's local environment variables (unless explicitly passed to jobs)

### Trust Boundary Layers
- Network: mTLS (TLS 1.3) with mutual authentication
- Filesystem: Server validates exec targets; temp files isolated per job  
- Process: Direct execve() without shell interpretation

### Request Flow
client == mTLS/TLS1.3 ==> server
  - initial validation (arg/env caps, required fields)
  - authN (verify client cert; extract SAN identity + attrs)
  - authZ (hardcoded ABAC: principal × resource × action × context)
  - Deep validate (absolute path, EvalSymlinks, allow-listed dirs, exec perms)
  - Prepare job (job_id, anon tmp output file, new PGID, env)
  - Spawn (execve via exec.CommandContext; no shell/TTY)
  - Stream (ReadAt, notify channels; binary-safe)
  - Lifecycle (TERM grace → KILL; reap; finalize status; GC TBD)

## Design Details

### 1 - Input Validation 
- Thin client, server-side validation. Client will use `flag` rather than `cobra`.
  - Can upgrade in future should need arise. 
- Additional input validation rules in [Security Considerations: Input Validation](#input-validation)

### 2 - Supported Job Types
- Executables on disk (e.g. /usr/bin/ls, /usr/bin/ruby)
- No raw syscalls: jobs execute as normal processes via `execve()` by `exec.Command`; clients cannot specify syscalls directly (server may use syscalls internally for resource management)
- Absolute path only; support for $PATH look up a future TODO (see [Security Considerations: Input Validation](#input-validation))

### 3 - Authentication

#### PKI Structure
Development Root CA (self-signed, long-lived)
├── Server Certificate (CA-signed leaf)
├── Client Certificate - AAAAA (CA-signed leaf)  
└── Client Certificate - BBBBB (CA-signed leaf)

- mTLS over TLS1.3 to satisfy requirement for strongest transport encryption
- Enforce TLS1.3 in code(pin `MinVersion` and `MaxVersion` to `tls.VersionTLS13`); [Go handles choice of cipher suite after 1.17](https://go.dev/blog/tls-cipher-suites)
- Ed25519 certs (rather than RSA, to achieve comparable security with lower network load)
- Long-lived leaf certs generated via Makefile (DANGER - only doing this for development; should use short-lived once development complete)
- Identity extracted from client certificate SANs (never CommonName)
- Future: let's encrypt, enterprise CA/PKI

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
- Job Management: Principals can only `delete`, `describe`, and `delete` jobs they own
- Cross-Principal Access: Denied - principals cannot access other principals' job logs or metadata
- Rationale: Prevents information leakage between different users/services sharing the same K2SO instance

#### Principal Attributes (from client cert SAN fields)
- User ID only: `email:nimbus@example.com` or `URI:urn:principal:nimbus`

#### Actions and Resources
- Actions: `run`, `delete`, `describe`, `logs`
- Resources: job instances, identified by job-id
- Future: resource tagging (process types, binary allow-lists)

#### Future
- Implement cert revocation logic
- Replace hardcoded rules with OPA/Rego policies for policy-as-code
- Additional context validation

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
- Multiple readers can all receive the same notification

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
- No metrics by default to keep scope small; if needed, could expose expvar counters (errors_total, jobs_running) behind a flag (TODO, not implemented).
- Future improvements see [Production-Level Features](#production-level-features-beyond-challenge-scope)

### Build and Development 
- Minimal CI with linting to satisfy style requirement and consistency 
- Makefile for reproducible builds (native to Linux), explicit targets, certificate generation
- stdlib + gRPC only
  - Majority of features required can be delegated to go and gRPC built-in abilities
- Built-in toolchain (`go test -race`, `go build -race`) for race detection (build this into testing and CI)
- Hardcoded configurations with TODO comments for future extensibility

#### Testing
- Minimal happy/err path test coverage for critical paths: authn, authz, core functionalities
-	Team checkoff needed: 3rd party dependency "github.com/stretchr/testify/require" for readable tests

### CLI UX (kubectl-style, minimal)

```bash
# Execute commands
k2so run /usr/bin/ls -la /tmp
k2so run /usr/bin/ruby /path/to/lol.rb

# View job outputs  
k2so logs <job-id>           # Snapshot mode (current output)
k2so logs -f <job-id>        # Follow mode (live stream)

# Job management
# Job status, exit code, metadata
k2so describe <job-id>       

 # Terminate running job
k2so delete <job-id>      

# List recent jobs (basic info)
k2so list                    
```

#### Examples
```bash
# Run a command, get job ID back
$ k2so run /usr/bin/echo "hello world"
job-a1b2c3d4

# Stream the output
$ k2so logs job-a1b2c3d4
hello world

# Run long-running command and follow output
$ k2so run /usr/bin/ping -c 5 8.8.8.8
job-iamauuid
$ k2so logs -f job-iamauuid
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=12.345 ms
...

# Check job status
$ k2so describe job-iamalsoauuid
Job ID: job-iamalsoauuid
Command: /usr/bin/ping -c 5 8.8.8.8
Status: completed
Exit Code: 0
Started: 2025-01-15T10:30:45Z
Completed: 2025-01-15T10:30:50Z
```

#### Future 
- `k2po login` sso to login and generate client cert

### Security Considerations

#### Input Validation
- Cap arg size at ~64 args, ~4kb per arg/env to prevent abuse.
- Only support absolute path: prevents `PATH` manipulation attacks
- Use `filepath.EvalSymlinks()` to prevent directory traversal
- Validate file existence and permission
- Direct `exec.Command(binary, arg1, arg2, ...)` usage
  - directly maps onto Linux `execve()` syscall without interpretation layers
  - dangerous alternatives include `sh -c`, string concat approaches,or single string with spaces
- Shell metacharacters (`|&;<>(){}[]$`"'\\`) rejected in validation
NOTE FOR TODO IN DOC - make sure CLI design includes this consideration 
- Future TODO - whitelist of allowed binary directories

#### Process Isolation

- Independent job instances - identical commands create separate processes to prevent:
  - cross-client information leakage
  - unexpected job termination affecting multiple clients
  - complex ownership and authorization edge cases
  - shared state debugging complexity

- Resource implications: multiple identical jobs consume proportional resources, this is ok for the challenge

## Proposed API
[See the gRPC API definition in k2so.proto](proto/k2so/v1/k2so.proto)

## Edge Cases
- Two clients execute same command simultaneously: see [Process Isolation](#process-isolation)
- Multiple clients streaming same job: see [Client Experience](#client-experience)
- Principal attempts to access another principal's job:
  - Server returns `PERMISSION_DENIED` error for `logs`, `describe`, `delete` actions
  - Job ownership verified against client certificate identity (see [Authorization](#4authorization))
- Late joiners to active job:
  - Snapshot mode: get complete output history from start (see [Storage & Memory Management](#storage--memory-management))
  - (stretch)Follow mode: join at current EOF, receive only new output
- Slow/hanging clients: see [Efficiency](#efficiency)
  - gRPC handles per-client flow control automatically
- Client disconnection during streaming:
  - gRPC context cancellation cleans up streaming goroutines
  - Server doesn't track client state, so no cleanup needed
  - Anonymous temp files remain available for other clients
- Job becomes unresponsive
  - SIGTERM -> grace period -> SIGKILL lifecycle (see [Request Flow](#request-flow))
  - Process groups ensure child processes also terminated (L5 stretch goal)
  - Anonymous temp files auto-cleanup when server closes FDs
- Server restart/crash
  - Anonymous temp files are lost (by design for L4 scope)
  - In-memory job registry lost - no job ownership or metadata persists across restarts
  - Running jobs become orphaned (underlying processes not managed by K2SO after restart)
- mTLS certificates remain valid across restarts
- Output exceeds limits:
- Per-job caps terminate job and return RESOURCE_EXHAUSTED (see [Future](#future))
- Anonymous temp files prevent unbounded memory growth
- Notification channel remains responsive during cleanup
- Too many concurrent jobs:
  - No explicit limits in L4 - relies on OS process limits
  - Each job gets independent resources (see [Architecture](#architecture))
  - Future: implement global job limits and queuing 

## Milestones
See README.md 

## Future Work

### L5 Stretch Goals (Feasible for Challenge)
- process tree termination: ensure job's child processes are also terminated on stop no zombies
  - new PGID per job; SIGTERM (grace) SIGKILL to PGID; no zombies
- cgroup v2 resource control- per-job cpu.max, memory.max, optional io.max
- process groups -proper signal propagation to all descendants  
- enhanced job lifecycle- graceful shutdown with configurable timeouts
- basic resource monitoring: track CPU/memory in describe
- per-job output cap and global limits; fail with RESOURCE_EXHAUSTED

### Production-Level Features (Beyond Challenge Scope)
- HA control plane w leader election, distributed runners, failover
- Using systemd (invoked via systemctl), apply cgroup limits, and ensure TERM KILL teardown.
- distributed scheduling, runner pools 
- security - authz with OPA, write REGO policies, SPIFFE, secrets management, cert rotation
- observability- OTEL traces, Prometheus metrics, structured logs, ebpf stuffs
- advanced isolation: seccomp/AppArmor/userns, rlimits, capabilities 
- storage - db-backed metadata, persistent output, retention + encryption at rest
- Multi-tenant: namespaces, per-tenant quotas
- supply chain and release integrity - signed binaries, SLSA, SBOMs
- platform maturity - versioned APIs, SLOs, chaos engineering, disaster recoveryy, rate limits, stream backpressure
- enterprise ready - short-lived certs generated for clients after SSO, compliance stuffs, policy-as-code

