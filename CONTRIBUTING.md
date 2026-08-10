## Design docs

Copy `docs/design/TEMPLATE.md` to `docs/design/<NNNN>-<slug>.md`,
using the next free number.

Target two pages. If it runs past four, split the problem.

Delete sections that don't apply. Prefer `n/a` with a short reason
over leaving one blank.

Status moves from Draft to Accepted via a PR reviewed by the other
workstream.

Edit docs in place. To replace one wholesale, write a new numbered doc
and mark the old one `Superseded`.

## Pinning the baseline

Add `upstream` as a fetch-only remote — the disabled push URL stops a
stray `git push upstream` from reaching the Grandine repository:

    git remote add upstream https://github.com/grandinetech/grandine.git
    git remote set-url --push upstream DISABLED

Take the SHAs from your checkouts:

    eip_sha                git -C EIPs rev-parse HEAD
    consensus_specs_sha    git -C consensus-specs rev-parse HEAD
    grandine_upstream_sha  git -C grandine merge-base HEAD upstream/develop

`grandine_upstream_sha` is the last upstream commit your work is based
on.

## Development updates

Weekly updates live in `updates/<username>/week-NN.md`, zero-padded to
match the cohort repo's table.
