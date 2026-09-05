---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: ethereum/EIPs
eip_sha: ec5fdb982e101ed0262f68742429d230ca960ca3
consensus_specs_sha: 7d6bd46a015a7dd316c5df855bd89e57c4aa6700
grandine_upstream_sha: eaf220e60699cd63d4223ad2481e42fd15f67802
grandine_stack_tip: 366fabc
grandine_stack:
  - feature/eip8025-progressive-byte-list (8de28ef)
  - feature/eip8025-proof-containers (4119302)
  - feature/eip8025-payload-binding (366fabc, tip — adopts simplify shape)
superseded_by:
---

# 0002 — Sign execution proofs under `DOMAIN_EXECUTION_PROOF`

## Context

EIP-8025 adds _execution proofs_: a prover (an active validator) signs an
`ExecutionProofEnvelope` message and gossips it as a
`SignedExecutionProofEnvelope`. Both spec sources agree on the signing recipe
(consensus-specs `prover.md`, EIP):

```python
domain = get_domain(state, DOMAIN_EXECUTION_PROOF, compute_epoch_at_slot(state.slot))
signing_root = compute_signing_root(proof_envelope, domain)
```

Grandine already has the exact primitive for this: `SignForSingleForkAtSlot<P>`
(`helper_functions/src/signing.rs:172`), whose default `signing_root` computes
the domain at `compute_epoch_at_slot(slot)`. This doc covers only the signing
primitive — containers, gossip validation plumbing, and the proof engine are
separate workstreams/docs.

**Upstream baseline (re-baselined, CL only).** The specs are WIP and contradict
each other on the consensus layer; the container stack settled the provisional
questions toward the simplify shape:

| Value                    | EIP text (CL only)                                              | consensus-specs `feat/simplify-eip8025`                                          | As implemented                 |
| ------------------------ | --------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------ |
| `DOMAIN_EXECUTION_PROOF` | `0x0D000000`                                                    | `0x0F000000`                                                                     | `0x0F000000` (consensus-specs) |
| `MAX_PROOF_SIZE`         | `409600` (400 KiB)                                              | `4194304` (4 MiB)                                                                | `4194304` (4 MiB)              |
| Proof data container     | `ByteList[MAX_PROOF_SIZE]`                                      | `ProofData = ProgressiveList[Byte]`                                              | `ProofData` (`ProgressiveByteList`) |
| `PublicInput`            | 3 fields (`root`, `bool`, `chain_config`)                       | 4-field (`root`, `bool`, `chain_id`, `schema_id`)                                | 4-field progressive            |
| Signed message           | `ExecutionProof` / `get_execution_proof_signature`              | `ExecutionProofEnvelope` / `get_execution_proof_envelope_signature`              | `ExecutionProofEnvelope`       |
| Gossip message           | `SignedExecutionProof`                                          | `SignedExecutionProofEnvelope`                                                   | `SignedExecutionProofEnvelope` |

EL (`chain_id`/`schema_id`/`0x1501`) is out of scope for this table.
`ChainConfig` appears only in the EIP CL (`PublicInput.chain_config`); the
simplify shape and this workstream use `chain_id` + `schema_id` instead.

`0x0F` is adopted. The EIP CL's `0x0D000000` (at `ec5fdb98`) is stale — it collides with the
spec's Gloas `DOMAIN_PROPOSER_PREFERENCES` assignment, which Grandine's own
`gloas/consts.rs` does not define at this pin. The simplification of
the spec (`tau-lepton/feat/simplify-eip8025`, adopted by the rewritten base)
confirms `0x0F` and defines the envelope as the signed message.

If the domain value moves, our cost is a one-line constant flip. If container
fields move, the signing impl is unaffected (it only requires
`ExecutionProofEnvelope: SszHash`); only hash-dependent tests need re-baselining.

## Goals and non-goals

**Goals**

- Provide `sign`, `verify`, and `signing_root` for `ExecutionProofEnvelope` matching
  the spec recipe exactly, via an existing Grandine signing trait.
- Add `SignatureKind::ExecutionProof` so signature failures are reported with a
  distinct, user-visible kind.

**Non-goals**

- Implementing `ExecutionProof`/`ExecutionProofEnvelope`/`SignedExecutionProofEnvelope`/`PublicInput`/`ProofType`
  containers (landed on the container workstream's stacked branches).
- Gossip validation path (BLS + `is_active_validator` checks in
  `process_execution_proof` belong on the controller task path; separate doc).
  EIP CL networking (`MAX_EXECUTION_PROOFS_PER_PAYLOAD`, `ExecutionProofStatus`,
  `ByRange`/`ByRoot`, `eproof` ENR, `proof_serve_range`) is likewise out of scope.
- `ProofEngine`/`ProofService` (separate doc).
- Prover client flow (subscribing to events, constructing `NewPayloadRequest`).

## Design

Types live in the `types` crate, and the signing impl lives in Grandine's
single signing registry — no new crate:

```
types/src/eip8025/
  consts.rs          # DOMAIN_EXECUTION_PROOF (this workstream); MAX_PROOF_SIZE (stack)
  primitives.rs      # ProofType (stack)
  containers.rs      # ExecutionProof, ExecutionProofEnvelope, SignedExecutionProofEnvelope, ProofData, PublicInput (stack)
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
impl<P: Preset> SignForSingleForkAtSlot<P> for ExecutionProofEnvelope {
    const DOMAIN_TYPE: DomainType = DOMAIN_EXECUTION_PROOF;
    const SIGNATURE_KIND: SignatureKind = SignatureKind::ExecutionProof;
}
```

No epoch field is needed on the container: the epoch derives from the state's
slot at sign/verify time, exactly as in `process_execution_proof`.

**Message wrapper.** `SignedExecutionProofEnvelope.message` is `Hc<ExecutionProofEnvelope>`.
`Hc<T>: SszHash` delegates to the inner value, so signing (and verifying) the
bare `ExecutionProofEnvelope` is equivalent to signing the wrapped message — no extra
impl is needed.

**`PublicInput` is proof-engine-facing only.** The prover validates
`chain_id == DEPOSIT_CHAIN_ID` / `schema_id == STATELESS_INPUT_SCHEMA_ID`
locally, then drops it (`get_signed_execution_proof_envelope`); the bare
`ExecutionProof` is reconstructed by `get_execution_proof` for engine
submission. It never travels and never enters the signing root.

**Verification split.** The trait supplies cryptographic verification
(`signed_proof.message.verify(config, &state, slot, signature, pubkey)`); the
`is_active_validator` check and Accept/Ignore/Reject decision live on the
controller task path (mirroring `ExecutionPayloadBidTask`,
`fork_choice_control/src/tasks.rs:504`). `proof_engine.verify_execution_proof`
is a third, separate stage.

```
prover:     ExecutionProofEnvelope --sign(state.slot)--> SignedExecutionProofEnvelope --> gossip
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
  Adopted; the simplify branch (now the base shape) confirms it.
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
`feature/eip8025-payload-binding` (`366fabc`, rewritten), in dependency order as four commits
plus one envelope re-target follow-up (R5 — pivot stays visible for PR review):

1. `feat: Add DOMAIN_EXECUTION_PROOF` — `types/src/eip8025/consts.rs`.
2. `feat: Add SignatureKind::ExecutionProof` — `helper_functions/src/error.rs`
   (additive; nothing exhaustively matches `SignatureKind`).
3. `feat: Implement signing for ExecutionProofEnvelope` — `helper_functions/src/signing.rs`
   registry entry.
4. `test: add execution proof signing tests` — suite below.
5. Follow-up: re-target impl + tests from bare `ExecutionProof` to
   `ExecutionProofEnvelope` per the rewritten base.

Test suite (child module `helper_functions/src/signing/tests.rs`, Minimal preset):

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
- **(e) `Hc` parity** — `Hc<ExecutionProofEnvelope>` and bare `ExecutionProofEnvelope` share an
  object root, so the signature over the envelope is exactly the one the
  signed container carries (mirrors `get_signed_execution_proof_envelope`).
- **(f) envelope-vs-bare regression** — envelope root differs from bare
  `ExecutionProof` root for identical proof bytes (bare input uses spec-valid
  `chain_id = deposit_chain_id`, `schema_id = STATELESS_INPUT_SCHEMA_ID`);
  encodes "we sign the envelope".

Gates: `cargo check`/`cargo test` for `helper_functions` and `types` with
`--features bls/blst` (the workspace `bls` dependency has no BLS backend unless
one is enabled), `cargo fmt --all -- --check`, and clippy `-D warnings` with
CLI allow-flags for pre-existing, upstream-owned failures (lint-group priority
in the root manifest, unfulfilled expectations in `bls-core`, unknown cfg in
`bls-blst`).

## Blockers

- **Containers** — resolved: the rewritten `feature/eip8025-payload-binding`
  (`366fabc`) adopts the simplify shape — `ExecutionProof`,
  `ExecutionProofEnvelope`, `SignedExecutionProofEnvelope`, 4-field progressive
  `PublicInput`, `ProofData`, `ProofType`, and `NewPayloadRequest` in
  `types/src/eip8025/`; pending upstream merge.
- **Domain value adopted pending upstream convergence.** `0x0F000000`
  (consensus-specs) is implemented; the EIP's `0x0D000000` is stale. Does not
  block implementation (one-line flip), but blocks Accepted status.
- **Pyspec cross-check left.** Test (b) pins are out-of-band (container suite +
  script), not pyspec-checked — no eip8025 pyspec vectors on
  `feat/simplify-eip8025` yet. Re-baseline pins when they land; no code change.

## Open questions

- Gossip validation task wiring (Accept/Ignore/Reject policy, cache timeouts)
  — deferred to the gossip workstream doc.
