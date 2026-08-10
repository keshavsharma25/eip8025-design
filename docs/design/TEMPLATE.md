---
status: Draft                # Draft | Accepted | Superseded
authors: [<username>]
workstream: verification     # gossip | verification | converged
eip_repo: frisitano/EIPs
eip_sha: <sha>               # These three SHAs pin the baseline this
consensus_specs_sha: <sha>   # doc was written against; re-pin when
grandine_upstream_sha: <sha> # revising. 
superseded_by:               # Doc number, if any
---

# <NNNN> — <Imperative phrase: "Verify execution proofs before attesting">

## Context
What we're building and why, plus the part of Grandine and the part of
the spec you need to know to read the rest. Link rather than
re-explain. Note here anything in the spec baseline that is still
provisional upstream and what breaks for us if it moves.

## Goals and non-goals
**Goals** — include numbers where the design turns on them (latency
budgets, memory bounds, proof counts).
- ...

**Non-goals** — things that could reasonably have been goals and are
deliberately excluded.
- ...

## Design
Overview, then detail. Include a diagram when the change crosses
component boundaries. Describe the interfaces and the retained state;
skip pseudo-code and link prototypes instead.

## Trade-offs and alternatives
What this design costs us, and which alternatives were rejected and on
which trade-off. Be brief, but don't omit the one the reviewer is
going to ask about.

## Security and compatibility
Required. Adversarial model, DoS and resource-exhaustion surface, what
an invalid or withheld proof can cause, and behavior for nodes that
don't opt in.

## Implementation and testing
How this lands as a reviewable PR series against
`grandinetech/grandine`, which parts stand alone without EIP-8025, and
how it gets tested (unit, spec conformance, devnet).

## Open questions
What must be settled before this moves to Accepted, and what is
deliberately deferred.
