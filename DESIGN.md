# K3PO Technical Design Document
[See requirements in the README](README.md#requirements) condensed from [original requirements](https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

## Naming Choice
- [K3PO is a blunt security bot who tells it like it is](https://starwars.fandom.com/wiki/K-3PO)
- K3PO is a good little bot
- K3PO died too early, this control plane would too since everything is built with minimal scope and minimal prod considerations around reliability and resilience
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

### Authorization 

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

### CLI UX

### Non-functional Requirements

### Build and Development 
- linting to satisfy style requirement and consistency 
- Makefile for reproducible builds (native to Linux), explicit targets, certificate generation
- stdlib + gRPC only, no third-party concurrency libraries
- Built-in toolchain (`go test -race`, `go build -race`) for race detection (build this into testing and CI)
- Minimal dependencies: stdlib + gRPC only
- Hardcoded configurations with TODO comments for future extensibility

#### Testing
- Minimal happy/err path test coverage for critical paths: authn, authz,
-	Team checkoff needed: 3rd party dependency "github.com/stretchr/testify/require" for readable tests

## Proposed API
[See the gRPC API definition in k3po.proto](proto/k3po/v1/k3po.proto)

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

## Milestones
