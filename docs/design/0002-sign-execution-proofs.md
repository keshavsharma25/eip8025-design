---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: 321eca5b71049fcac6c63c2d956e5c5d7b60d689
grandine_upstream_sha: eaf220e60699cd63d4223ad2481e42fd15f67802
grandine_stack_tip: 368f36e
grandine_stack:
  - feature/eip8025-progressive-merkleization (e6694f3)
  - feature/eip8025-progressive-byte-list (c89b7d3)
  - feature/eip8025-proof-containers (eda4b27)
  - feature/eip8025-payload-binding (368f36e)
superseded_by:
---

# 0002 — Sign execution proofs under `DOMAIN_EXECUTION_PROOF`

## Context

EIP-8025 adds _execution proofs_: a prover (an active validator) signs an
`ExecutionProof` message and gossips it as a `SignedExecutionProof`. Both spec
sources agree on the signing recipe
(consensus-specs `prover.md`, EIP):

```python
domain = get_domain(state, DOMAIN_EXECUTION_PROOF, compute_epoch_at_slot(state.slot))
signing_root = compute_signing_root(proof, domain)
```

Grandine already has the exact primitive for this: `SignForSingleForkAtSlot<P>`
(`helper_functions/src/signing.rs:172`), whose default `signing_root` computes
the domain at `compute_epoch_at_slot(slot)`. This doc covers only the signing
primitive — containers, gossip validation plumbing, and the proof engine are
separate workstreams/docs.

**Upstream baseline (re-baselined).** The specs are WIP and used to contradict
each other; the container stack settled the provisional questions:

| Value                    | EIP text     | consensus-specs `_features/eip8025`  | As implemented                 |
| ------------------------ | ------------ | ------------------------------------ | ------------------------------ |
| `DOMAIN_EXECUTION_PROOF` | `0x0D000000` | `0x0F000000`                         | `0x0F000000` (consensus-specs) |
| `PublicInput`            | 3 fields     | 1 field (`new_payload_request_root`) | 1 field                        |
| Proof data container     | `ByteList`   | `ProgressiveByteList`                | `ProgressiveByteList`          |
| `MAX_PROOF_SIZE`         | 409600       | 4194304 ("not definitive")           | 4194304 (4 MiB)                |

`0x0F` is adopted. The EIP's `0x0D000000` is stale — it collides with the
spec's Gloas `DOMAIN_PROPOSER_PREFERENCES` assignment, which Grandine's own
`gloas/consts.rs` does not define at this pin. The in-works simplification of
the spec (`tau-lepton/feat/simplify-eip8025`) confirms `0x0F` and is cited as
tie-breaker only, not adopted as a baseline.

If the domain value moves, our cost is a one-line constant flip. If container
fields move, the signing impl is unaffected (it only requires
`ExecutionProof: SszHash`); only hash-dependent tests need re-baselining.

## Goals and non-goals

**Goals**

- Provide `sign`, `verify`, and `signing_root` for `ExecutionProof` matching
  the spec recipe exactly, via an existing Grandine signing trait.
- Add `SignatureKind::ExecutionProof` so signature failures are reported with a
  distinct, user-visible kind.

**Non-goals**

- Implementing `ExecutionProof`/`SignedExecutionProof`/`PublicInput`/`ProofType`
  containers (landed on the container workstream's stacked branches).
- Gossip validation path (BLS + `is_active_validator` checks in
  `process_execution_proof` belong on the controller task path; separate doc).
- `ProofEngine`/`ProofService` (separate doc).
- Prover client flow (subscribing to events, constructing `NewPayloadRequest`).

## Design

Types live in the `types` crate, and the signing impl lives in Grandine's
single signing registry — no new crate:

```
types/src/eip8025/
  consts.rs          # DOMAIN_EXECUTION_PROOF (this workstream); MAX_PROOF_SIZE (stack)
  primitives.rs      # ProofType (stack)
  containers.rs      # ExecutionProof, SignedExecutionProof, ProofData, PublicInput (stack)
  container_impls.rs # new_payload_request_root (stack)
  error.rs           # PayloadBindingError (stack)
```

`types/src/lib.rs` declares the nested module mirroring `gloas`. The container
surface landed on the container workstream's stacked branches
(`feature/eip8025-proof-containers`, `feature/eip8025-payload-binding`); this
workstream adds `consts::DOMAIN_EXECUTION_PROOF` and the signing surface in
`helper_functions/src/signing.rs`.

Dependency direction is `helper_functions → types` (already the case; `types`
does not depend on `helper_functions`). The impl is a
`helper_functions`-local trait for a `types`-local type, so placing it in
`helper_functions/src/signing.rs` is orphan-rule valid with no cycle.

**Domain constant** — owned by this workstream: `DOMAIN_EXECUTION_PROOF =
0x0F000000` is defined in `types/src/eip8025/consts.rs`, following
consensus-specs (the `0x0D` divergence is recorded in the baseline table
above). The signing impl consumes it via `DOMAIN_TYPE`; if the value moves
upstream, the flip is still a one-line constant change.

**Signing impl** (`helper_functions/src/signing.rs`) — one registry entry next
to the Gloas impls, with the `types::eip8025` import block extended;
`sign`/`verify`/`signing_root` come free from the trait:

```rust
impl<P: Preset> SignForSingleForkAtSlot<P> for ExecutionProof {
    const DOMAIN_TYPE: DomainType = DOMAIN_EXECUTION_PROOF;
    const SIGNATURE_KIND: SignatureKind = SignatureKind::ExecutionProof;
}
```

No epoch field is needed on the container: the epoch derives from the state's
slot at sign/verify time, exactly as in `process_execution_proof`.

**Message wrapper.** `SignedExecutionProof.message` is `Hc<ExecutionProof>`.
`Hc<T>: SszHash` delegates to the inner value, so signing (and verifying) the
bare `ExecutionProof` is equivalent to signing the wrapped message — no extra
impl is needed.

**Verification split.** The trait supplies cryptographic verification
(`signed_proof.message.verify(config, &state, slot, signature, pubkey)`); the
`is_active_validator` check and Accept/Ignore/Reject decision live on the
controller task path (mirroring `ExecutionPayloadBidTask`,
`fork_choice_control/src/tasks.rs:504`). `proof_engine.verify_execution_proof`
is a third, separate stage.

```
prover:     ExecutionProof --sign(state.slot)--> SignedExecutionProof --> gossip
verifier:   gossip --> [task: BLS verify + is_active_validator]
                     --> [proof engine: verify_execution_proof]
```

## Trade-offs and alternatives

- **Types in `types/` + impls in `helper_functions` vs a separate `eip8025`
  crate.** Chose types-in-`types`: every signing impl in Grandine lives in
  `helper_functions/src/signing.rs` (a single registry, including the Gloas
  impls), and fork features live in the `types` crate — a new crate would
  fragment the registry and split the WIP surface from the rest of the fork
  types. `builder_api` is the exception that proves the rule: an external
  protocol integration, not a consensus fork feature. Cost of the chosen route:
  EIP-8025 WIP types share the large `types` crate, and the signing registry
  file gains a fork-specific entry.
- **Baseline `0x0F` vs `0x0D`.** Chose `0x0F` (consensus-specs): newer,
  client-facing, and `0x0D` collides with `DOMAIN_PROPOSER_PREFERENCES`.
  Adopted; the simplify branch confirms it as tie-breaker only.
- **Wait for containers vs stub them.** Chose wait: land container-independent
  pieces first (SignatureKind); add the `signing.rs` impl once the real
  containers exist. Cost paid: the impl could not compile or be tested until
  the container stack landed. Rejected stubbing to avoid divergence with the
  container workstream.

## Security and compatibility

- Domain separation prevents cross-domain replay: an execution-proof signature
  cannot be substituted for any other signature kind, and
  `SignForSingleForkAtSlot` binds the domain to the fork at the slot's epoch.
- Invalid signatures reject with `Error::SignatureInvalid(
SignatureKind::ExecutionProof)`; `SignatureKind` is only consumed via
  `Display`, so the new variant is additive and safe.
- A valid signature does **not** imply a valid proof — the cryptographic check
  and `verify_execution_proof` are independent stages; both must pass.
- Nodes that don't opt in are unaffected: signing only occurs for configured
  prover validators; verification code is inert without `execution_proof`
  gossip.

## Implementation and testing

Landed against the fork (`eip8025-grandine/grandine`), branch
`feature/sign-execution-proofs`, stacked on
`feature/eip8025-payload-binding`, in dependency order as four commits:

1. `feat: Add DOMAIN_EXECUTION_PROOF` — `types/src/eip8025/consts.rs`.
2. `feat: Add SignatureKind::ExecutionProof` — `helper_functions/src/error.rs`
   (additive; nothing exhaustively matches `SignatureKind`).
3. `feat: Implement signing for ExecutionProof` — `helper_functions/src/signing.rs`
   registry entry (non-generic `ExecutionProof`).
4. `test: add execution proof signing tests` — suite below.

Test suite (unit tests in `helper_functions/src/signing.rs`, Minimal preset):

- **(a) formula** — `signing_root` equals an independent transcription of the
  spec recipe in both fork-selection cases (current fork; and a slot from an
  epoch before `fork.epoch` → `previous_version` path, made observable by a
  distinct `current_version`), with the domain additionally pinned to an
  out-of-band literal, mirroring `test_compute_domain`.
- **(b) pinned vector** — full signing root for a length-100 proof pinned with
  provenance. The object root is the container workstream's independently
  computed reference; the signing root was computed out-of-band. Same caveat as
  the container roots: not pyspec-cross-checked.
- **(c) round trip** — `sign` with a real BLS secret key; `verify` accepts.
- **(d) negative** — tampered message rejects with
  `Error::SignatureInvalid(SignatureKind::ExecutionProof)`, displayed as
  "execution proof signature".
- **(e) `Hc` parity** — `Hc<ExecutionProof>` and bare `ExecutionProof` share an
  object root, so the signature over the bare proof is exactly the one the
  signed container carries.

Gates: `cargo check`/`cargo test` for `helper_functions` and `types` with
`--features bls/blst` (the workspace `bls` dependency has no BLS backend unless
one is enabled), `cargo fmt --all -- --check`, and clippy `-D warnings` with
CLI allow-flags for pre-existing, upstream-owned failures (lint-group priority
in the root manifest, unfulfilled expectations in `bls-core`, unknown cfg in
`bls-blst`).

## Blockers

- **Containers** — resolved: the container stack
  (`feature/eip8025-progressive-merkleization` →
  `feature/eip8025-payload-binding`) landed `ExecutionProof`,
  `SignedExecutionProof`, `ProofData`, `PublicInput`, `ProofType`, and
  `NewPayloadRequest` in `types/src/eip8025/`; pending upstream merge.
- **Domain value adopted pending upstream convergence.** `0x0F000000`
  (consensus-specs) is implemented; the EIP's `0x0D000000` is stale. Does not
  block implementation (one-line flip), but blocks Accepted status.
- **In-works simplify branch moves the signed message.**
  `tau-lepton/feat/simplify-eip8025` renames the signed message to
  `ExecutionProofEnvelope` (4-field `PublicInput`, progressive
  `NewPayloadRequest`). Not adopted; if it lands, the impl retargets with a
  one-line change and only hash-dependent test vectors re-baseline.

## Open questions

- Gossip validation task wiring (Accept/Ignore/Reject policy, cache timeouts)
  — deferred to the gossip workstream doc.
