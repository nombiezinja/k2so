# K2SO Technical Design Document
[See requirements in the README](README.md#requirements) condensed from [original requirements](https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

## Naming Choice
- [K2SO is a blunt security bot who tells it like it is](https://starwars.fandom.com/wiki/K-2SO)
- K2SO is a good little bot
- K2SO died too early, this control plane would too since everything is built with minimal scope and minimal prod considerations around reliability and resilience
- A more serious name is preferred for prod-ready projects (e.g. processctl,jobctl,etc). Boring is better for enterprise tooling.

## Design Approach 
1 - Input Validation 
- Happens on server-side. To minimize scope, will have thin client, using `flag` rather than `cobra`.
  - Can upgrade in future should need arise. 
- Additional input validation rules in Security Considerations (TODO link this)

2 - Supported processes
- Executables on disk (e.g. /usr/bin/ls, /usr/bin/ruby)
- No raw syscalls: jobs can invoke syscalls, but control plane does not directly invoke kernal interfaces
- Default is absolute path only; support for $PATH look up in TODO
- TODO link security considerations

### Process Execution Assumptions 

### Authentication
- mTLS over TLS1.3 to satisfy requirement for strongest security
 - enforce this in Go by setting min/max version 
 - Go handles choice of cipher suite (TODO add the thingies Go use for default here)
   - https://go.dev/blog/tls-cipher-suites Go began doing this from 1.7, just ensure TLS13 is configured and forced with tls.Config
- 256-bit Ed25519 certs (more secure than 3072-bit RSA), faster signature operations, smaller cert size to reduce network overload, constant-time implementation to prevent timing attacks
- self-signed CA for development, generate with make-file 
- identity extracted from client certificates 
- Future: let's encrypt, enterprise CA/PKI

### Authorization 
- Hard-code permission model 
- ABAC instead of RBAC for extensibility and security
  - abilities: start, stop view, stream
  - resources: ( this can be tags of resources/processes; Future- allow-list of processes/binaries with tags)
- client ed25519 certs have attributes like user id, org
- stateless authz, rely on CA to issue/revoke certs. server only checks for whether identity extracted from client certs has authority to perform action on resourceß
- Future: use OPA and write Rego to replace hard-coded authz policies

### Output Streaming 
1 - efficient discovery: Coalescing notify channel (`chan struct{}`, buffer=1) prevents polling
2 - historical replay: Late joiners start at offset=0, get full output from process start
3 - bounded heap memory: Anonymous temp file prevents unbounded Go memory growth
4 - no text assumptions: Raw byte streaming throughout, binary-safe
5 - concurrent client support: `os.File.ReadAt` allows multiple readers without contention

### Resource management:
1 - heap memory: bounded by design (fixed buffers, no output accumulation)
2 - disk usage: currently unbounded (TODO: add per-job output size limits)
3 - file cleanup: anonymous files deleted when last reader exits (TODO - decide on whether to put in scheduled cleanup)

### Design Rationale
- Minimal, discoverable commands for core job lifecycle and output streaming
- All commands support mTLS authentication
- Output streaming supports late joiners and binary-safe data
- Consistent error handling and user feedback

### Non-functional Requirements
(Requirements: consistent error output & handling, no crashing)
- Error handling:
  - Client-facing: clear gRPC status codes, short actionable error messages (no stack traces, redact sensitive info).
  - Server-side: error logs with pid/command and request id for failures/rejects; lifecycle logs (start/exit) only if --verbose is on.
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
- Minimal happy/err path test coverage for critical paths: authn, authz,
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

Responsible for:
- Input validation on all gRPC requests
- Secure process execution via direct `execve()`
- mTLS authentication and authorization
- Process isolation and resource management

Outside of scope:
- User's local shell environment (login/non-login, aliases, history)
- Client-side command construction and quoting
- User's local filesystem permissions and access
- User's local environment variables (unless explicitly passed to jobs)

### Process Isolation

Independent job instances: identical commands create separate processes to prevent:
- cross-client information leakage
- unexpected job termination affecting multiple clients
- complex ownership and authorization edge cases
- shared state debugging complexity

Resource implications: multiple identical jobs consume proportional resources, which is acceptable for the prototype scope.

## Milestones
See README.md 
