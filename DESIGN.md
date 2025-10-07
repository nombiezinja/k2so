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
  - Stream (byte offset, ReadAt, coalesced notify; binary-safe)
  - Lifecycle (TERM grace → KILL; reap; finalize status; GC TBD)

## Design Details
### 1 - Input Validation 
- Thin client, server-side validation. Client will use `flag` rather than `cobra`.
  - Can upgrade in future should need arise. 
- Additional input validation rules in [Security Considerations: Input Validation](#input-validation)

### 2 - Supported processes
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
- Principals can only `delete` jobs they created
- Principals must fullfill both action and resource check to perform request

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

5 - Output Streaming 
  - efficient discovery to avoid busy-waiting/polling: Coalescing notify channel (`chan struct{}`, buffer=1) prevents polling
  - historical replay: Late joiners start at offset=0, get full output from process start
  - bounded heap memory: Anonymous temp file prevents unbounded Go memory growth
  - no text assumptions: Raw byte streaming throughout, bina  -safe
  - concurrent client support: `os.File.ReadAt` allows multiple readers without contention

6 - Resource management:
    - heap memory: bounded by design (fixed buffers, no output accumulation)
    - disk usage: currently unbounded (TODO: add p    -job output size limits)
    - file cleanup: anonymous files deleted when last reader exits (TODO - decide on whether to put in scheduled cleanup)

### Non-functional Requirements
- Error handling:
  - Client-facing: clear gRPC status codes, short actionable error messages (no stack traces, redact sensitive info).
  - Server-side: error logs with pid/command and request id for failures/rejects; lifecycle logs (start/exit) only if --verbose is on.

### Error Model
Uses [canonical gRPC status codes](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)
- Metrics: None by default to keep scop small; if needed, could expose expvar counters (errors_total, jobs_running) behind a flag (TODO, not implemented).

### Build and Development 
- linting to satisfy style requirement and consistency 
- Makefile for reproducible builds (native to Linux), explicit targets, certificate generation
- stdlib + gRPC only, no third-party concurrency libraries
  - Majority of features required can be delegated to go and gRPC built-in abilities
- Built-in toolchain (`go test -race`, `go build -race`) for race detection (build this into testing and CI)
- Minimal dependencies: stdlib + gRPC only
- Hardcoded configurations with TODO comments for future extensibility

#### Testing
- Minimal happy/err path test coverage for critical paths: authn, authz
-	Team checkoff needed: 3rd party dependency "github.com/stretchr/testify/require" for readable tests

### CLI UX (kubectl-style, minimal)
- Use clear verbs and resource types: `k2so exec`, `k2so describe <job-id>`, `k2so stop <job-id>`, `k2so `, etc.
- TODO add some examples
- skipping `k2so login` because have mTLS; can have it in future for better UX and explicitness

## Proposed API
[See the gRPC API definition in k2so.proto](proto/k2so/v1/k2so.proto)

## Edge Cases
TODO link sections to these edge cases from other subsections
- Two clients try to execute same command at the same time
- Late joiners
- SLow/Long hanging clients 
- Client disconnection 

## Security Considerations

### Input Validation
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

### Process Isolation

Independent job instances: identical commands create separate processes to prevent:
- cross-client information leakage
- unexpected job termination affecting multiple clients
- complex ownership and authorization edge cases
- shared state debugging complexity

Resource implications: multiple identical jobs consume proportional resources, which is acceptable for the prototype scope.

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
- distributed scheduling, runner pools 
- security - authz with OPA, write REGO policies, SPIFFE, secrets management, cert rotation
- observability- OTEL traces, Prometheus metrics, structured logs, ebpf stuffs
- advanced isolation: seccomp/AppArmor/userns, rlimits, capabilities 
- storage - db-backed metadata, persistent output, retention + encryption at rest
- Multi-tenant: namespaces, per-tenant quotas
- supply chain and release integrity - signed binaries, SLSA, SBOMs
- platform maturity - versioned APIs, SLOs, chaos engineering, disaster recoveryy, rate limits, stream backpressure
- enterprise ready - short-lived certs generated for clients after SSO, compliance stuffs, policy-as-code

