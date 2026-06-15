# <LANGUAGE>.rules.md

Version: 1.0
Status: Normative

---

# 1. Purpose

These rules define mandatory engineering constraints for production systems.

Goals:

- Deterministic behavior
- Predictable latency
- Strong correctness guarantees
- High maintainability
- High observability
- Explicit ownership
- Reproducible builds
- Security by default

---

# 2. Rule Language

The following keywords are normative:

- MUST
- MUST NOT
- SHOULD
- SHOULD NOT
- MAY

---

# 3. Core Invariants

These invariants override all other rules.

## Determinism

Systems MUST produce identical observable behavior when provided:

- identical inputs
- identical initial state
- identical configuration
- identical runtime environment

## Single Writer

Mutable state MUST have exactly one logical writer.

Concurrent readers MAY exist.

## Bounded Work

Every execution path MUST have a provable upper bound.

Unbounded loops, recursion, retries, or retries-until-success are forbidden.

## Explicit Ownership

Ownership and lifetime MUST be obvious from code.

Hidden ownership transfer is forbidden.

## No Hidden Control Flow

State changes, side effects, and execution transitions MUST be explicit.

Framework magic, implicit callbacks, and hidden orchestration SHOULD be avoided.

---

# 4. Execution Model

## Runtime Phases

Systems MUST separate:

### Initialization

Allowed:

- configuration
- dependency construction
- resource acquisition
- allocation
- discovery

### Runtime

Allowed:

- bounded work
- deterministic computation
- validated state transitions

Runtime code MUST NOT:

- block indefinitely
- allocate unexpectedly
- depend on external mutable state

---

# 5. State Management

## State Ownership

State MUST:

- have a clear owner
- have documented lifetime
- have documented mutation rules

## State Mutation

Mutation MUST occur through explicit operations.

Shared mutable global state is forbidden.

---

# 6. Memory Rules

## Allocation

Allocation policy MUST be documented.

Projects MUST define:

- where allocation is allowed
- where allocation is forbidden
- exhaustion behavior

## Lifetimes

Object lifetimes MUST be explicit.

Dangling references are release-blocking defects.

## Resource Management

Resources MUST have deterministic acquisition and release.

---

# 7. Concurrency Rules

## Shared State

Shared writable state MUST be minimized.

## Synchronization

Synchronization contracts MUST be documented.

## Data Races

Data races are release-blocking defects.

## Threading

Thread creation and destruction MUST occur only at approved lifecycle boundaries.

---

# 8. Error Handling

## Expected Failures

Expected failures MUST be represented explicitly.

Examples:

- result types
- status values
- error objects

## Fatal Failures

Fatal failures MUST have deterministic behavior.

Examples:

- fail-stop
- fault state
- process termination

## Error Propagation

Errors MUST propagate explicitly.

Hidden global error channels are forbidden.

---

# 9. API Design

## API Classification

Every API MUST be classified as:

- Runtime Safe
- Initialization Only
- Boundary

## Contracts

APIs MUST document:

- inputs
- outputs
- ownership
- failure modes

## Units

Units MUST be explicit.

Examples:

- Duration
- Bytes
- Counts

---

# 10. Data Structures

## Selection Criteria

Data structures MUST be chosen based on:

- access patterns
- cache behavior
- bounded complexity
- memory constraints

## Hot Paths

Hot paths SHOULD favor:

- contiguous storage
- fixed capacity structures
- predictable access patterns

---

# 11. Observability

## Logging

Logging MUST:

- be bounded
- be non-blocking
- have defined overflow behavior

## Telemetry

Telemetry MUST NOT alter functional behavior.

## Diagnostics

Diagnostics MUST be removable without affecting correctness.

---

# 12. Platform Boundaries

## External Systems

All interaction with:

- filesystems
- networks
- operating systems
- hardware
- clocks

MUST occur through explicit boundary layers.

## Dependency Isolation

Platform-specific behavior MUST be isolated.

---

# 13. Security

## Input Validation

All untrusted input MUST be validated before use.

## Secrets

Secrets MUST NOT be stored in source control.

## Dependency Management

Dependencies MUST be versioned and pinned.

---

# 14. Build Rules

## Toolchains

Toolchains MUST be reproducible.

Compiler versions MUST be pinned.

## Warnings

Warnings MUST be treated as errors.

## Reproducibility

Build outputs SHOULD be reproducible.

---

# 15. Testing Requirements

Projects MUST include:

- unit tests
- integration tests
- deterministic replay tests
- fault injection tests

Where applicable:

- sanitizer runs
- race detection
- fuzz testing
- performance regression testing

---

# 16. Performance Rules

Performance claims MUST be measured.

Optimization decisions MUST be justified with profiling data.

Micro-optimizations without evidence are forbidden.

---

# 17. Prohibited Practices

The following are forbidden unless explicitly documented and approved:

- hidden allocations
- hidden ownership transfer
- hidden control flow
- unbounded retries
- unbounded queues
- undocumented synchronization
- undocumented global state
- undefined behavior
- data races
- silent failure handling

---

# 18. Exception Process

Rule violations MUST:

1. be documented
2. have an owner
3. have tests covering risk
4. have a removal plan

Temporary exceptions MUST expire.