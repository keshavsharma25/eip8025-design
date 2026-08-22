---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: 321eca5b71049fcac6c63c2d956e5c5d7b60d689
grandine_upstream_sha: eaf220e60699cd63d4223ad2481e42fd15f67802
superseded_by:
---

# 0002 — Skeleton the execution-proof service: controller task pipeline and ProofEngine boundary

## Context

EIP-8025 lets a consensus client validate execution payloads by verifying
gossiped execution proofs instead of re-executing them. The proposal splits
the verifier side into two components: the **execution proof service**, the
consensus-layer orchestrator between gossip and proof verification, and the
**`ProofEngine`**, the implementation-dependent boundary that delegates
cryptographic verification to an external verifier
(proposal.md, "Scope and architecture").

This doc defines the initial interface skeleton for both — the week-8
deliverable of the verification workstream. It fixes shapes and ownership,
not bodies. Signing primitives are covered by
[0001](0001-sign-execution-proofs.md); the SSZ containers
(`ProofType`, `PublicInput`, `ExecutionProof`, `SignedExecutionProof`) are
the container workstream's deliverable (`types/src/eip_8025/containers.rs`).

One external component anchors the design as the integration target:

- **zkboost** — the external verifier this project targets. Its real verify
  contract is an SSZ POST of `(fork, new_payload_request_root,
chain_config, proof_type, proof bytes)` returning `VALID`/`INVALID`
  (`crates/types/src/lib.rs` `ProofVerificationBody`). The extra context
  is resolved engine-internally (see Design); the trait keeps the spec's
  bare `verify_execution_proof(ExecutionProof)` signature.

Non-normative prior art (other clients' WIP branches) was studied during
drafting and deliberately not relied upon below; every rule here traces to
the EIP, the consensus-specs feature files, or Grandine's own patterns.

**Provisional upstream baseline.** The recursive `ExecutionProof` layer
(checkpoint-anchored beacon-chain proofs) has no settled container or
validation rules upstream; no document exists that is certain to describe
how it will pan out. This skeleton treats it at interface level only.

## Goals and non-goals

**Goals**

- A new `proof_engine` crate exposing the spec-shaped `ProofEngine` trait as
  an `ExecutionEngine` twin (trait + forwarding impls + `NullProofEngine` +
  `MockProofEngine`), bodies stubbed.
- A `ProcessExecutionProofTask` skeleton in `fork_choice_control` plus the
  `ProofOutcome { Accept, Reject, Ignore }` message type, mirroring
  `ExecutionPayloadBidTask`.
- Lock the boundary: the service performs all consensus-layer validation
  (bounds, dedup, validator eligibility, BLS signature) before routing to
  the engine; the engine only performs delegated cryptographic verification.

**Non-goals**

- Signing primitive (doc 0001) and containers (doc 0003).
- Gossip topic subscription and p2p routing (weeks 12–16).
- k-of-n aggregation body; the threshold value itself is deferred.
- Recursive anchor verification, retention/pruning, restart re-derivation.
- The zkboost client body (external-verifier integration, weeks 9–11) and
  anything prover-side (`request_proofs` generation flow).

## Design

The service is not a loop or thread; in Grandine everything is a task
struct on the controller's thread pool. The lifecycle clones three existing
patterns:

```
eth2_libp2p (gossip) → p2p router → Controller
                                      │ spawn_execution_proof_task()
                                      ▼
                      ProcessExecutionProofTask::run()
                        bounds → dedup → eligibility → BLS
                        → payload-context resolution
                        → proof_engine.verify_execution_proof()
                                      │
                                      ▼
                MutatorMessage::ExecutionProof { result }
                                      │
                    mutator arm → outcome → p2p republish
```

The concurrency consequence of this shape: validation (including the
potentially slow engine call) runs concurrently on the thread pool, off
the critical network path, while state changes apply serially in the
controller's mutator loop — so a flood of 4 MiB proofs queues as tickets
instead of blocking block processing or spawning unbounded work. In
standard terms this is producer–consumer work distribution where each
ticket is a command object (`Run`), closed out by a single-writer result
path; nothing here is new machinery.

**Engine trait** (`proof_engine/src/engine.rs`) — mirrors
`execution_engine/src/execution_engine.rs:22`:

```rust
pub trait ProofEngine<P: Preset> {
    const IS_NULL: bool;

    fn verify_execution_proof(&self, execution_proof: ExecutionProof<P>) -> bool;

    fn notify_new_payload(&self, new_payload_request: NewPayloadRequest);

    fn notify_forkchoice_updated(
        &self,
        head_block_hash: Hash32,
        safe_block_hash: Hash32,
        finalized_block_hash: Hash32,
    );

    fn stop(&self);
}
```

with forwarding impls for `&E` / `Arc<E>` / `Mutex<E>`,
`NullProofEngine` (`IS_NULL = true`; notifies no-op, verify returns
`false` — fail-closed: an engine that verifies nothing never asserts
validity), and `MockProofEngine` (constructor takes
`execution_proofs_valid: bool`, mirroring `MockExecutionEngine`).

**Context resolution.** The signature stays spec-faithful — a bare
`ExecutionProof`. The context the external verifier needs (fork,
`new_payload_request_root`, chain config) is bound engine-internally from
`notify_new_payload` history, which is the same reason the spec gives the
engine notify methods; the future zkboost client assembles its wire body
from that state. Binding context never crosses the trait.

Crate layout:

```
proof_engine/src/
  lib.rs
  engine.rs        # ProofEngine trait + forwarding impls
  null_engine.rs   # NullProofEngine
  mock_engine.rs   # MockProofEngine
```

The zkboost client (`client.rs`) lands with external-verifier integration
(weeks 9–11); the trait bodies will route to it.

**Service task** (`fork_choice_control`) — mirrors
`tasks.rs:504` `ExecutionPayloadBidTask`:

```rust
pub struct ProcessExecutionProofTask<P: Preset, W, E: ProofEngine<P>> {
    pub store_snapshot: Arc<Store<P, Storage<P>>>,
    pub proof_engine: Arc<E>,
    pub mutator_tx: Sender<MutatorMessage<P, W>>,
    pub signed_proof: Arc<SignedExecutionProof<P>>,
}
```

- Entry point `Controller::spawn_execution_proof_task` (clones
  `controller.rs:798`); nothing calls it until gossip wiring lands.
- `run()` opens with the `E::IS_NULL` short-circuit: a Null-engine build
  emits `Ignore` and returns before any pipeline work, so the engine's
  fail-closed `false` (and its Reject side effect) is unreachable in
  practice.
- Queued via a `LowPriorityTask::ExecutionProof` variant (proofs are up to
  4 MiB and involve BLS plus potentially slow external verification).
- `run()` implements the pipeline cheapest-check-first — this order is the
  DoS contract, taken from the p2p-interface IGNORE/REJECT matrix:
  1. message bounds (`proof_data` non-empty, ≤ `MAX_PROOF_SIZE`)
  2. dedup against a service-owned bounded LRU cache keyed
     `(new_payload_request_root, proof_type, validator_index)` — entry
     count fixed by a named const, tunable later — marked seen _before_
     validity so duplicates cannot be replayed
  3. `is_active_validator` at the current epoch
  4. BLS signature via the doc-0001 primitive
     (`DOMAIN_EXECUTION_PROOF`, epoch of slot)
  5. payload-context resolution: bind
     `public_input.new_payload_request_root` to an accepted payload
     supplied by fork-choice; unknown root → `Ignore`
  6. `proof_engine.verify_execution_proof(...)`; `false` → `Reject`
- Outcome emitted as `MutatorMessage::ExecutionProof { result }` carrying
  `ProofOutcome` only; identifiers (`new_payload_request_root`,
  `proof_type`, `validator_index`) are deliberately deferred until a
  consumer exists for them (k-of-n counters, fork-choice gating, weeks
  15+), at which point the variant grows. The arm clones the `PayloadBid`
  handling at `mutator.rs:318` and hands outcomes to p2p so accepted
  proofs are re-gossiped.

**Ownership summary.** Service owns: dedup cache, all consensus-layer
checks, payload-context map (fed later by fork-choice notifications),
checkpoint/verified-head anchor tracking (recursive layer, interface only).
Engine owns: delegated cryptographic verification and, later, retained
proof state consumed via the notify methods.

## Trade-offs and alternatives

- **New `proof_engine` crate vs living in `fork_choice_control`.** Chose the
  crate for parity with `execution_engine` — both are boundaries to an
  external system.
- **Spec-shaped trait vs thin verifier client.** The notify methods exist so
  the engine tracks payload context itself; the thin-client pattern survives
  as the future `client.rs` behind the trait.
- **Dedup in the service vs querying engine state.** Service-local — dedup
  must gate ingress before any expensive work.
- **Task struct vs dedicated service actor.** Task — reuses the controller's
  thread pool and mutator return path with no new concurrency primitives.

## Security and compatibility

- Any peer can flood `execution_proof` subnets with junk up to 4 MiB;
  cheapest-first ordering plus pre-validity dedup caps repeat senders at a
  cache lookup.
- BLS + eligibility checks run before any cryptographic proof work; invalid
  signatures `Reject` and feed peer scoring via the outcome.
- A failed proof never invalidates a payload accepted through the Engine API —
  proofs stay auxiliary to existing payload validation.
- Opt-out nodes inject `NullProofEngine` and subscribe to nothing; opted-in
  nodes only add a parallel validity signal to the standard Engine API flow.

## Implementation and testing

Against `eip8025-grandine/grandine`, in dependency order:

1. `proof_engine` crate scaffold: trait + forwarding impls +
   `NullProofEngine` + `MockProofEngine`, workspace registration. Gate
   `cargo check -p proof_engine`. **Blocked** on containers (the trait
   names `ExecutionProof<P>`).
2. `fork_choice_control`: `ProofOutcome`, `MutatorMessage::ExecutionProof`
   variant, `ProcessExecutionProofTask` with stubbed `run()`,
   `spawn_execution_proof_task`, `LowPriorityTask` variant. **Blocked** on
   the scaffold above and containers (`SignedExecutionProof<P>`).
3. Tests: Null/Mock engine unit tests (notify no-ops; mock verify returns
   configured result); task smoke test running the stub pipeline against
   `MockProofEngine` and asserting the emitted outcome message.

Later weeks replace stubs in place: consensus checks (weeks 9–11), gossip
wiring (12–14), outcome→p2p routing (15–16), recursive anchors (17–19).
Integration testing uses zkboost's in-process mock backend
(`mock_proving_time`, `mock_proof_size`, `mock_failure`) once the client
lands.

## Blockers

- **Containers** (`types/src/eip_8025/containers.rs`, container
  workstream): every interface here names `ExecutionProof<P>` or
  `SignedExecutionProof<P>` and cannot compile until they land. Interfaces
  are drafted now, compile-gated on that merge — the same posture doc 0001
  takes toward its signing impl.

## Open questions

- Where `MAX_PROOF_SIZE` lives and its exact value: resolved by
  [0003](0003-eip-8025-containers.md) — it enters the `Preset` trait as
  `P::MaxProofSize` (4194304, mirroring `P::MaxExtraDataBytes`), and
  `MAX_SIGNED_EXECUTION_PROOF_SIZE` joins `eip_8025/consts.rs`; the bound
  is enforced at SSZ decode time, ahead of pipeline stage 1.
- k-of-n threshold values (k and the bounded n), deferred; aggregation is
  a client-side counter over distinct `ProofType`s either way.
- Recursive `ExecutionProof` container and checkpoint-anchor rules —
  upstream WIP; re-pin this doc when specs converge.

Resolved during drafting: trait signature stays spec-faithful with
engine-internal context resolution; `NullProofEngine` verify returns
`false` fail-closed with an `E::IS_NULL` short-circuit in the task;
`request_proofs` omitted until the generation stretch goal; low-priority
queue confirmed; dedup cache is a bounded LRU with const size; task is
generic over `E: ProofEngine<P>` so `IS_NULL` compiles away.
