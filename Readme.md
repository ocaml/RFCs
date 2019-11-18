# OCaml RFCs

This repository is for proposals to change the OCaml language
or the internals of its compiler.

**It is for proposals by people who actually intend to implement the
proposed changes. Feature requests from users of the language should
instead by made as issues on [ocaml/ocaml](https://github.com/ocaml/ocaml/issues)**.

## Making an RFC

RFCs are made by creating a pull request that adds a file to the
`rfcs` folder. The `rfcs` folder contains accepted proposals for
changes to the language. The pull request will only be merged
once there is consensus to accept the change in principle.

We'll adjust and adapt the process as we go, but as a starting point
RFCs should provide:

 - A high-level summary of the change
 - Motivation for the change
 - Technical details of the change
 - Drawbacks of the change and alternatives to the change
 - Unresolved questions

## Discussing an RFC

RFCs will be discussed in the comments of the pull request that
proposes them. Authors should try to respond to queries and integrate
feedback into the RFC document. Commenters should try to avoid
unnecessary bike-shedding.

The OCaml development team will moderate these discussions. We may
delete comments or close pull requests that we feel are not
productive.

## Merging an RFC

Once those involved in discussing an RFC think that it is ready to be
considered for approval they can nominate it for consideration at the
next developers meeting. At that meeting a decision will be made to
either:

- Accept the proposal and merge the RFC

- Reject the proposal and close the RFC

- Request further changes/discussion on the RFC before reconsidering
  at another meeting.

Once an RFC has been accepted into the repository authors can begin
implementing the proposal and be reasonably confident that a suitable
implementation of the feature will be accepted upstream into the
compiler.
