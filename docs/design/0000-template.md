---
status: Draft            # Draft | Accepted | Superseded
authors: [@handle-a]
workstream: gossip       # gossip | verification | converged
eip_repo: frisitano/EIPs
eip_sha: <sha>
consensus_specs_sha: <sha>
grandine_upstream_sha: <sha>
superseded_by:           # doc number, if any
---

# 0000 — <Imperative phrase: "Verify execution proofs before attesting">

## Context
What we're building and why, plus the part of Grandine and the part of the spec
you need to know to read the rest. Objective facts only; link rather than
re-explain. Note here anything in the spec baseline that is still provisional
upstream and what breaks for us if it moves.

## Goals and non-goals
**Goals** — include numbers where the design turns on them (latency budgets,
memory bounds, proof counts).
- ...

**Non-goals** — things that could reasonably have been goals and are
deliberately excluded. Not "it shouldn't crash."
- ...

## Design
Overview, then detail. Include a diagram when the change crosses component
boundaries. Sketch the interfaces and the retained state; skip pseudo-code
unless the algorithm is novel, and link prototypes instead.

## Trade-offs and alternatives
What this design costs us, and which alternatives were rejected and on which
trade-off. Be brief about rejected options — but don't omit the one the
reviewer is going to ask about.

## Security and compatibility
Required. Adversarial model, DoS and resource-exhaustion surface, what an
invalid or withheld proof can cause — and the behaviour for nodes that don't
opt in, since the feature must stay safely default-off.

## Implementation and testing
How this lands as a reviewable PR series against `grandinetech/grandine`,
which parts stand alone without EIP-8025, and how it gets tested
(unit, spec conformance, devnet).

## Open questions
What must be settled before this moves to Accepted, and what is deliberately
deferred.
