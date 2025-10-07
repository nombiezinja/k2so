# Teleport Challenge-1 L4

This repository contains Ti Zhang's solution to [Teleport's candidate assessment challenge](https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md). 

Below are concise checklists consolidated from the original doc, to ensure all requirements are met, and as a convenient reference during verification during development and review. 

## TODOs 
- [x] Parse requirements
- [x] Write design doc
- [ ] Initial protos
- [ ] Design doc communications & approval
- [ ] Implementation 
  - [ ] Milestone 1
  - [ ] Milestone 2
  - [ ] Milestone 3 
- [ ] Verification 

## Development Milestones 
M1: protos and gRPC service
M2: start/stop job
M3: output streaming support
M4: mTLS authn
M5: hard-coded abac authz
M6: CLI client
M7: testing/hardening/CI
M8: docs cleanup

## Requirements Checklist 

### Overall Principles
- [ ] Minimal code/scope; hard code where needed; cut corners and indicate intention
- [ ] No 3rd party dependencies
- [ ] Make tradeoffs and explain why
- [ ] high performance,availability, &scaleability not expected; but explain how would add in future
- [ ] No custom hand-rolled security/auth 
- [ ] No global state unless justified

### Checklist 
- [ ]Works on 64-bit linux machines
- [ ]Server does not rely on shell scripts, external binaries or use containers to execute jobs.
- [ ]Follow
  [Go Coding Style](https://github.com/golang/go/wiki/CodeReviewComments) 
- [ ]Key components happy path& error case tests; no need for 100% coverage
- [ ]Reproducible builds
- [ ]Consistent err handling & reporting; no crashing
- [ ]Avoid concurrency and networking errors.
  - [ ] Check for data races
  - [ ] Check ofr networking error handling
  - [ ] Check for goroutine leaks 
- [ ]Security 
  - [ ] strongest posible transport encryption; tested
  - [ ] mTLS authn with strong cipher suite
  - [ ] Simple hard-coded authorization scheme

#### Components Checklist
- [ ]Library
  -[ ]start/stop/query status of a job.
  -[ ]stream the output of a running job.
  -[ ]support multiple concurrent clients 
  -[ ]Discovering new output should be efficient, avoid busy-waiting or polling.
  -[ ]Output should be from start of process execution.
        Multiple concurrent clients should be supported.
        Do not make any assumptions about the process's output - it may be text or raw binary data.
- [ ]API
  -[ ]GRPC API to start/stop/get status/stream output of a running process.
  -[ ]Use mTLS authentication and verify client certificate. Set up strong set of cipher suites for TLS and good crypto setup for certificates. Do not use any other authentication protocols on top of mTLS.
  -[ ]Use a simple authorization scheme.
-[ ]Client
  -[ ]CLI should be able to connect to worker service and start, stop, get status, and stream output of a job.