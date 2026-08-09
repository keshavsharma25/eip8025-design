## Design docs

Copy `docs/design/TEMPLATE.md` to `docs/design/NNNN-slug.md`, using
the next free number.

Target two pages. If it runs past four, split the problem.

Delete sections that don't apply, except *Security and compatibility*.
Prefer `n/a` with a short reason over leaving a section blank.

Status moves from Draft to Accepted via a PR reviewed by the other
workstream.

Edit docs in place. For a wholesale replacement, write a new numbered
doc and set `status: Superseded` and `superseded_by` on the old doc.
