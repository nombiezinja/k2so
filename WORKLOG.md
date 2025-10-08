## Worklog 

### 20251007
PR resolution notes for https://github.com/nombiezinja/teleport-ti-zhang-challenge-1/pull/1
- extracting changes needed from comments
  - [ ] Remove -f; stream behaviour default is follow
  - [ ] Replace delete with run everywhere in semantics and design
  - [x] No SIGTERM, use only SIGKILL to reduce scope; update protos to reflect
  - [ ] Add note for UUID generation
  - [ ] Improve stream design and add notes on multi-writer support (i.e. job registry design)