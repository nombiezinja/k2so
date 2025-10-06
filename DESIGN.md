# Technical Design Document
[See requirements in the README](README.md#requirements) condensed from [original requirements](https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

## Design Approach 

### Tooling 
1 - Input Validation 
- Happens on server-side. To minimize scope, will have thin client, using `flag` rather than `cobra`.
  - Can upgrade in future should need arise. 
- Additional input validation rules in ### Security Considerations

2 - Supported processes
- Executables on disk (e.g. /usr/bin/ls, /usr/bin/ruby)
- No raw syscalls: jobs can invoke syscalls, but control plane does not directly invoke kernal interfaces
- Default is absolute path only; support for $PATH look up in TODO


### Process Execution Assumptions 

### Output Streaming 

### Build and Development 

### CLI UX

### Non-functional Requirements


## Proposed API

[See the gRPC API definition in k3po.proto](proto/k3po/v1/k3po.proto)

## Testing

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
