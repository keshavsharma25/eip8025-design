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

# 0004 — EIP-8025 SSZ containers in Grandine types

## Context

EIP-8025 defines five wire types — `ProofType`, `ProofData`, `PublicInput`,
`ExecutionProof`, `SignedExecutionProof` — that everything else in the
roadmap manipulates. Both existing drafts compile against them:
[0002](0002-sign-execution-proofs.md) needs `ExecutionProof<P>: SszHash`
for its signing primitive, and [0003](0003-execution-proof-service.md)
names these types throughout its trait and task signatures. This doc lands
the types first so the signing and service docs unblock.

Two normative sources define them, and they disagree:

| Item             | EIP text                          | consensus-specs `_features/eip8025`   |
| ---------------- | --------------------------------- | ------------------------------------- |
| `PublicInput`    | 3 fields (`+successful_validation`, `+chain_config`) | 1 field (`new_payload_request_root`) |
| `proof_data`     | `ByteList[MAX_PROOF_SIZE]`        | `ProgressiveByteList`                 |
| `MAX_PROOF_SIZE` | 409600 (400 KiB)                  | 4194304 (4 MiB)                       |

`SignedExecutionProof` (`message`, `validator_index`, `signature`) and the
`ExecutionProof` field set agree everywhere.

**Provisional upstream baseline.** There is no blanket tiebreaker: each
conflict is decided case-by-case on normative function, with the argument
recorded per axis under Trade-offs. Non-normative prior art (other
clients' WIP branches) was studied during drafting and deliberately not
relied upon anywhere below. Two Grandine facts anchor the choices:

- **Grandine already skipped progressive lists once.** Gloas specs moved
  `Transaction`/`BlockAccessList` to `ProgressiveByteList`; Grandine's Gloas
  still uses the classic bellatrix `Transaction`
  (`types/src/deneb/containers.rs:139`), and the `ssz` crate has no
  progressive machinery at all.
- **The exact bound pattern exists**: `extra_data:
  Arc<ByteList<P::MaxExtraDataBytes>>` (`deneb/containers.rs:134`),
  declared as `type MaxExtraDataBytes: MerkleElements<u8> + ...`
  (`preset.rs:156`), valued in the `Mainnet` impl, delegated for `Minimal`.

## Goals and non-goals

**Goals**

- Land the five types in `types/src/eip_8025/{primitives,containers}.rs`,
  extending the layout agreed in 0002.
- Add `P::MaxProofSize` to the `Preset` trait (all three presets) and
  `DOMAIN_EXECUTION_PROOF` plus `MAX_SIGNED_EXECUTION_PROOF_SIZE` to
  `eip_8025/consts.rs`.
- `cargo check -p types` green with unit tests; no downstream dependency
  left blocked on anything but this merge.

**Non-goals**

- Req/resp and gossip list containers (`ProofByRootIdentifier`,
  `ProofTypes`, `SignedExecutionProofs`, `ProofByRootIdentifiers`) — gossip
  workstream, weeks 12–16.
- An SSZ `NewPayloadRequest`: nothing computes `new_payload_request_root`
  until payload-context binding lands (weeks 9–11), and its shape is the
  least stable part of the EIP.
- Progressive merkleization (EIP-7916) in the `ssz` crate.
- Signing primitive (0002), service/engine skeleton (0003), gossip
  validation plumbing.

## Design

File layout — extends the sketch agreed between workstreams in 0002:

```
types/src/eip_8025/
  consts.rs        # DOMAIN_EXECUTION_PROOF, MAX_SIGNED_EXECUTION_PROOF_SIZE (this doc)
  primitives.rs    # ProofType, ProofData, PublicInput (this doc)
  containers.rs    # ExecutionProof, SignedExecutionProof (this doc)
types/src/preset.rs  # type MaxProofSize (this doc)
```

`types/src/lib.rs` gains `pub mod primitives;` and `pub mod containers;`
inside the existing `pub mod eip_8025` block, mirroring the fork modules.
No `container_impls.rs` yet — the signing impl lives in `helper_functions`
(0002), and no other trait impls exist until later weeks.

**Primitives** (`primitives.rs`):

```rust
/// The identifier of the proof system that produced an execution proof.
pub type ProofType = u8;

/// The opaque proof bytes of an execution proof, bounded by
/// `P::MaxProofSize`. Alias over classic `ByteList` — see Trade-offs.
pub type ProofData<P> = ByteList<<P as Preset>::MaxProofSize>;

#[derive(Clone, Copy, PartialEq, Eq, Debug, Default, Deserialize, Serialize, Ssz)]
#[serde(deny_unknown_fields)]
pub struct PublicInput {
    /// SSZ hash-tree-root of the new payload request this proof certifies.
    pub new_payload_request_root: H256,
}
```

`H256` is Grandine's `Root`; `u8` type aliases match house style
(`gloas/primitives.rs`). One field only, per the baseline decision.

**Containers** (`containers.rs`):

```rust
#[derive(Clone, PartialEq, Eq, Debug, Default, Deserialize, Serialize, Ssz)]
#[serde(bound = "", deny_unknown_fields)]
pub struct ExecutionProof<P: Preset> {
    pub proof_data: ProofData<P>,
    pub proof_type: ProofType,
    pub public_input: PublicInput,
}

#[derive(Clone, PartialEq, Eq, Debug, Default, Deserialize, Serialize, Ssz)]
#[serde(bound = "", deny_unknown_fields)]
pub struct SignedExecutionProof<P: Preset> {
    pub message: ExecutionProof<P>,
    #[serde(with = "serde_utils::string_or_native")]
    pub validator_index: ValidatorIndex,
    pub signature: SignatureBytes,
}
```

Serde conventions copied from `gloas/containers.rs`: numerics via
`string_or_native`, `deny_unknown_fields` throughout, empty serde bounds on
generic structs. `ValidatorIndex` comes from `phase0::primitives`,
`SignatureBytes` from the `bls` crate. The `Ssz` derive supplies
serialization *and* `hash_tree_root`, satisfying 0002's single requirement
(`ExecutionProof<P>: SszHash`) with no extra code. No inner `Arc` around
`proof_data`: consumers share whole proofs via outer handles (0003 holds
`Arc<SignedExecutionProof<P>>`), unlike payload internals where
`Arc<ByteList>` pays for frequent cheap clones.

**Preset** (`preset.rs`) — four touches, mirroring `MaxExtraDataBytes`:

```rust
// trait declaration, next to the other byte bounds:
type MaxProofSize: MerkleElements<u8> + Eq + Debug + Send + Sync;

// impl Preset for Mainnet:
type MaxProofSize = U4194304;   // 4 MiB, consensus-specs baseline

// impl Preset for Minimal and Medalla (delegate lists):
type MaxProofSize;
```

Making the bound a preset associated type is not decoration: an unused
`<P>` on `ExecutionProof` fails compilation (E0392), and this gives `P`
real work while following the established byte-bound idiom. It also keeps
the door open if `MAX_PROOF_SIZE` ever forks per-preset.

**Constants** (`consts.rs`) — the domain constant moves into this doc's scope
from [0002](0002-sign-execution-proofs.md) so the module lands self-contained:

```rust
pub const DOMAIN_EXECUTION_PROOF: DomainType = H32(hex!("0F000000"));

pub const MAX_SIGNED_EXECUTION_PROOF_SIZE: u64 = 4194449; // MAX_PROOF_SIZE + encoding overhead (p2p-interface)
```

The 4 MiB bound is not an arbitrary pick between sources: the
p2p-interface's derived constant only coheres against it —
`4194449 = 4194304 + 145` bytes of SSZ container encoding overhead — so
the pair is internally consistent as a set only in consensus-specs. The
constant is included now because it is one line and gossip validation will
need it; the remaining p2p constants arrive with the gossip workstream.

## Trade-offs and alternatives

- **1-field vs 3-field `PublicInput`.** Chose 1-field, on normative
  function: neither source's validation rules ever read
  `successful_validation` or `chain_config` off the wire — the specs'
  `process_execution_proof` and the p2p-interface matrix bind proofs via
  `new_payload_request_root` alone, and the extra fields describe
  engine-side semantics, which 0003 resolves to keep engine-internal
  anyway. Every extra field is also hash-tree-rooted into the signing
  root, widening the signed surface for zero consensus-layer use, and
  `ChainConfig` lives in the EIP's Execution-Layer section, not the CL
  containers. Cost: if the EIP text holds, re-adding fields changes
  `hash_tree_root` and invalidates signing vectors — recorded as an open
  question, not silently assumed away.
- **`ByteList` alias vs implementing `ProgressiveByteList`.** The sources
  split — the EIP itself specifies classic `ByteList[MAX_PROOF_SIZE]`,
  consensus-specs progressive — so this is a case-by-case pick of classic:
  serialization is byte-identical under both rules
  (`ssz/simple-serialize.md` serializes lists and progressive lists in one
  offset scheme), so gossip bytes are unaffected either way; only
  hash-tree-root differs. Grandine has no progressive machinery, and its
  Gloas implementation already kept classic transactions when the specs
  moved them. Implementing EIP-7916 merkleization first would gate both
  downstream docs on a sub-project. Cost: if progressive wins out, the
  swap is one alias line plus re-baselined hash vectors.
- **Preset-typenum bound vs hardcoded `U4194304` + `PhantomData`.** Chose
  the preset route: it follows `MaxExtraDataBytes` exactly, avoids the
  unidiomatic `PhantomData` needed solely to satisfy E0392, and makes the
  bound tunable. Cost: `MAX_PROOF_SIZE` becomes a preset item although the
  specs file lists it under Constants — a small taxonomy deviation.
- **Core-types-only scope vs landing p2p/`NewPayloadRequest` too.** Chose
  minimal: every excluded type is either unused for weeks (req/resp lists,
  weeks 12–16) or the least stable shape in play (`SszNewPayloadRequest`);
  landing them now maximizes flux exposure for zero unblocking value.

## Security and compatibility

- Serialization is identical under classic and progressive list rules
  (`ssz/simple-serialize.md`), so the `proof_data` choice cannot split the
  gossip wire format; the residual exposure is hash-tree-root/signing-root
  agreement across clients, tracked as an open question until reference
  vectors exist.
- Bounded `ByteList` enforces `MAX_PROOF_SIZE` at decode time — the DoS
  bound 0003's pipeline relies on (stage 1) is enforced before any task
  logic runs.
- Inert data types: no behavior change anywhere until 0002/0003 land their
  consumers; opt-out nodes never construct these objects.
- Strict decoding (`deny_unknown_fields`, bounded lists) matches the
  house posture applied to every other wire container.

## Implementation and testing

Against the Grandine fork — **unblocked**, in dependency order:

1. `preset.rs`: trait declaration + `Mainnet` value + `Minimal`/`Medalla`
   delegate entries for `MaxProofSize`.
2. `types/src/eip_8025/primitives.rs` (new) + `containers.rs` (new),
   shapes above.
3. `consts.rs`: `DOMAIN_EXECUTION_PROOF` + `MAX_SIGNED_EXECUTION_PROOF_SIZE`.
4. `lib.rs`: module registration.

Unit tests (in-crate, `#[cfg(test)]`):

- SSZ serialize/deserialize round trip: empty `proof_data`,
  small proof, and a `MaxProofSize`-sized proof.
- Serde round trip including `string_or_native` `validator_index` and
  rejection of unknown fields.
- Hash-tree-root pins — expected roots derived independently from the SSZ
  merkleization rules (outside the implementation) and asserted as hex
  literals against default fixtures, in the manner of
  `helper_functions` `test_compute_domain`; provisional until reference
  vectors exist for `_features/eip8025`.
- `Default` sanity for both generic containers.

Gate: `cargo check -p types --features bls/blst && cargo test -p types
--features bls/blst eip_8025` (the bls crate has no default backend).

Landing order afterwards: [0002](0002-sign-execution-proofs.md) and
[0003](0003-execution-proof-service.md)'s `proof_engine` scaffold proceed in
parallel; 0003's service task follows its scaffold.

## Blockers

None hard — this doc is the blocker for the rest, not the reverse.

Upstream flux to re-pin on convergence: final `DOMAIN_EXECUTION_PROOF`
value (`0x0D` EIP vs `0x0F` specs; tracked in 0002), the progressive-list
direction, and any `PublicInput` field additions.

## Open questions

- Migration trigger for `ProgressiveByteList`: watch consensus-specs for
  progressive SSZ reference vectors and client-facing tooling; swap cost
  is the alias line plus vector re-baselining, so we move when the
  normative tooling demands it, not speculatively.
- Front-matter `workstream` label: `verification` used here since the
  containers exist to unblock the verification docs; whether shared
  foundation work should use `converged` instead is a CONTRIBUTING-level
  question.
