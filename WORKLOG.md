## Worklog 

### 20251103
- [ ] Update doc and adapt repo for personal practice funsies purposes
- [ ] restore design for truncation
- [ ] revert this change - Remove -f; stream behaviour default is follow
- [ ] revert this change - Replace delete with run everywhere in semantics and design
- [ ] revert this change - No SIGTERM, use only SIGKILL to reduce scope; update protos to reflect

### 20251008
Peer review comment resolution https://github.com/nombiezinja/teleport-ti-zhang-challenge-1/pull/1
- [x] TOCTOU race elimination
- [x] clarify design on streaming
- [x] clarify design on FD usage

### 20251007
PR resolution notes for https://github.com/nombiezinja/teleport-ti-zhang-challenge-1/pull/1
- extracting changes needed from comments
  - [x] Remove -f; stream behaviour default is follow
  - [x] Replace delete with run everywhere in semantics and design
  - [x] No SIGTERM, use only SIGKILL to reduce scope; update protos to reflect
  - [x] Add note for UUID generation
  - [x] Improve stream design and add notes on multi-writer support (i.e. job registry design)
