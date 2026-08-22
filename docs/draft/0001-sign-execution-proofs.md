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

# 0001 — Sign execution proofs under `DOMAIN_EXECUTION_PROOF`

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

**Provisional upstream baseline.** The specs are WIP and contradict each other:

| Value                    | EIP text     | consensus-specs `_features/eip8025`  |
| ------------------------ | ------------ | ------------------------------------ |
| `DOMAIN_EXECUTION_PROOF` | `0x0D000000` | `0x0F000000`                         |
| `PublicInput`            | 3 fields     | 1 field (`new_payload_request_root`) |
| `MAX_PROOF_SIZE`         | 409600       | 4194304                              |

If the domain value moves, our cost is a one-line constant flip. If container
fields move, the signing impl is unaffected (it only requires
`ExecutionProof<P>: SszHash`); only hash-dependent tests need re-baselining.
Note `0x0D` collides with `DOMAIN_PROPOSER_PREFERENCES` reserved in Gloas
consensus-specs — a reason to prefer the `0x0F` baseline.

## Goals and non-goals

**Goals**

- Provide `sign`, `verify`, and `signing_root` for `ExecutionProof<P>` matching
  the spec recipe exactly, via an existing Grandine signing trait.
- Add `SignatureKind::ExecutionProof` so signature failures are reported with a
  distinct, user-visible kind.

**Non-goals**

- Implementing `ExecutionProof`/`SignedExecutionProof`/`PublicInput`/`ProofType`
  containers (container workstream).
- Gossip validation path (BLS + `is_active_validator` checks in
  `process_execution_proof` belong on the controller task path; separate doc).
- `ProofEngine`/`ProofService` (separate doc).
- Prover client flow (subscribing to events, constructing `NewPayloadRequest`).

## Design

Types live in the `types` crate, and the signing impl lives in Grandine's
single signing registry — no new crate:

```
types/src/eip_8025/
  consts.rs        # DOMAIN_EXECUTION_PROOF (container workstream — see [0003])
  primitives.rs    # ProofType, PublicInput (container workstream)
  containers.rs    # ExecutionProof, SignedExecutionProof (container workstream)
```

`types/src/lib.rs` declares the nested module mirroring `gloas`; the whole
module tree (`consts`/`containers`/`primitives`) lands with the container
workstream ([0003]). This doc only extends the import block inside
`helper_functions/src/signing.rs`.

Dependency direction is `helper_functions → types` (already the case; `types`
does not depend on `helper_functions`). The impl is a
`helper_functions`-local trait for a `types`-local type, so placing it in
`helper_functions/src/signing.rs` is orphan-rule valid with no cycle.

**Domain constant** — owned by the container workstream: [0003] defines and
lands `DOMAIN_EXECUTION_PROOF` in `types/src/eip_8025/consts.rs`, including the
`0x0F` (consensus-specs) vs `0x0D` (EIP) divergence note. The signing impl
consumes it via `DOMAIN_TYPE`; if the value moves upstream, the flip is still a
one-line constant change.

**Signing impl** (`helper_functions/src/signing.rs`) — one registry entry next
to the Gloas impls, with the `types::eip_8025::...` import block extended;
`sign`/`verify`/`signing_root` come free from the trait:

```rust
impl<P: Preset> SignForSingleForkAtSlot<P> for ExecutionProof<P> {
    const DOMAIN_TYPE: DomainType = DOMAIN_EXECUTION_PROOF;
    const SIGNATURE_KIND: SignatureKind = SignatureKind::ExecutionProof;
}
```

No epoch field is needed on the container: the epoch derives from the state's
slot at sign/verify time, exactly as in `process_execution_proof`.

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

- **Types in `types/` + impls in `helper_functions` vs a separate `eip_8025`
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
- **Wait for containers vs stub them.** Chose wait: land container-independent
  pieces first (SignatureKind); add the `signing.rs` impl when the real
  containers merge. Cost: the impl cannot compile or be tested until then.
  Rejected stubbing to avoid divergence with the container workstream.

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

Against `eip8025-grandine/grandine`, in dependency order:

1. `helper_functions`: add `SignatureKind::ExecutionProof`
   (`helper_functions/src/error.rs`, after `ExecutionPayloadEnvelope`).
   Independent of containers; gate `cargo check -p helper_functions`.
2. `helper_functions`: `SignForSingleForkAtSlot` impl for `ExecutionProof<P>`
   in `signing.rs` (+ `types::eip_8025` imports) + unit test. **Blocked** on
   the container workstream ([0003]: `types/src/eip_8025/containers.rs`).

Testing: unit test asserting `signing_root` equals
`compute_signing_root(hash_tree_root(proof), compute_domain(config,
DOMAIN_EXECUTION_PROOF, epoch_at_slot(slot)))`, plus a sign/verify round trip.

## Blockers

- **Containers do not exist in Grandine yet.** `ExecutionProof`,
  `SignedExecutionProof`, `PublicInput`, `ProofType` are owned by another
  workstream; the `signing.rs` impl cannot compile until they land (agreed
  location: `types/src/eip_8025/containers.rs`). Signing requires only
  `ExecutionProof<P>: SszHash`. Now drafted as
  [0003](0003-eip-8025-containers.md), which resolves the shape questions
  below (1-field `PublicInput`, classic `ByteList` alias, 4 MiB bound) —
  re-baseline hash-dependent test vectors only if those flip upstream.
- **Domain value unresolved upstream** (`0x0D` EIP vs `0x0F` consensus-specs).
  Does not block implementation (one-line flip), but blocks Accepted status.
- **Container shape in flux** (1-field vs 3-field `PublicInput`, `ByteList` vs
  `ProgressiveByteList` proof data, `MAX_PROOF_SIZE`). Does not block the
  signing impl; blocks stable signing-root test vectors.
- `NewPayloadRequest` does not exist in Grandine; not needed for signing
  itself, but the container workstream needs it for `PublicInput`'s root.

## Open questions

- Final `DOMAIN_EXECUTION_PROOF` value — re-pin this doc when specs converge.
- Confirm containers land in `types/src/eip_8025/containers.rs` as agreed
  between workstreams — recorded in [0003](0003-eip-8025-containers.md).
- Gossip validation task wiring (Accept/Ignore/Reject policy, cache timeouts)
  — deferred to the gossip workstream doc.
