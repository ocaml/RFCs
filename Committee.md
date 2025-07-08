# OCaml Language Committee

This documents the ways of working of the OCaml Language Committee. It is directly inspired by the [organizational](https://github.com/ghc-proposals/ghc-proposals/blob/master/README.rst) [documents](https://github.com/ghc-proposals/ghc-proposals/blob/master/committee.rst) of the [GHC Committee](https://github.com/ghc-proposals/ghc-proposals).

## Preamble

The goal of the committee is to provide a process by which the OCaml community can decide on the evolution of the OCaml language and its standard library (stdlib). Any member of the OCaml community can submit an idea for consideration by the committee. This should ordinarily be done only when general consensus is elusive; committee approval is *not* required to make changes to the language or stdlib. The committee will then deliberate and reach an internal consensus. It is hoped that the community will then proceed with the committee's recommendation, but exceptionally may disagree.

## Submitting an idea for committee consideration

### Purview: what the committee considers

The committee is well suited to consider changes to the user interface of OCaml, including both the language and its stdlib. For example, the following changes are definitely in scope:

* A change or addition to the syntax or semantics of the OCaml language

* A change or addition to commands in the toplevel

* A change or addition to the stdlib

Changes that are not necessarily within the scope of the committee include:

* Changes to compiler-libs

* Changes to the internal implementation details of the compiler (e.g. refactorings)

* Changes to compiler flags

It is possible that the OCaml maintainers seek the opinion of the
committee on these points, but it is generally not expected. (Compiler
flags are part of the interface to the compiler, but one used mostly
by other tools that are part of the OCaml ecosystem; those authors
will likely have a more nuanced opinion of this part of the compiler's
interface than members of the committee.)

### How to submit a question for consideration by the committee

In an Issue or PR on this RFCs repo or the [ocaml repo](https://github.com/ocaml/ocaml), tag the chair of this committee, currently @Octachron, asking for committee consideration. That's it!

### How the committee deals with requests

After an issue has been tagged for committee attention, the chair chooses a committee member as a _shepherd_. (This assignment ideally happens within days of the tag.) The chair then labels the issue with *Pending shepherd recommendation*.

The shepherd's first job is to summarize the question for the committee and make a recommendation of response. Responses need not be "yes" or "no"; sometimes the committee will be asked to choose among several different ideas for syntax, say. The shepherd corresponds with the author until the shepherd understands the issue well enough, and then formulates a summary of this communication and a recommendation. This summary and recommendation are posted to the rest of the committee via the committee's mailing list. The shepherd then sets the label *Under consideration*. This ideally happens within two weeks of the shepherd assignment.

The committee then debates, either via the mailing list or on the GitHub ticket.
(The mailing list, though technically public, is a good place for conversations
primarily intended to reach other committee members; the GitHub ticket will
attract more responses from the wider community.). To be as transparent as
possible, the committee chair lists the known conflicts of interest between the
committee members and the proponent of the language change under deliberation at
the start of the deliberation. Hopefully the committee reaches consensus; the
shepherd then posts the committee's response back to the author. If the
committee is unable to reach consensus, it votes; if there are many options to
consider, it may use a ranked voting algorithm, at the discretion of the
shepherd. (That is, the shepherd is broadly empowered to choose the most
effective approach toward making a decision for the particular issue at hand.)
If there is any vote, the shepherd posts the result of the vote, including
information about the strength of the win. If the shepherd might be in conflict,
they delegate the report on the committee deliberation to a second
non-conflicted member of the committee. Because the decision of the committee is
non-binding, reflecting the level of committee support in the final answer may
be of interest to the broader community in deciding how to proceed. Once the
shepherd posts the committee's answer, the issue is no longer under
consideration by the committee, and the *Under consideration* label is removed.
Ideally this step lasts no more than four weeks.

## Who is the committee?

You can reach the committee by email at [`ocaml-language-committee@inria.fr`](mailto:ocaml-language-committee@inria.fr). This is a mailing list with
[public archives](https://sympa.inria.fr/sympa/arc/ocaml-language-committee).

The current members, including their GitHub handle, when they joined first, when their term last renewed, when their term expires and their role, are:


|          | Name     | Handle   | Join Date | Renewal Date | Term End | Affiliation |
| -------- | -------- | -------- | --------- | ------------ | -------- |-------------|
| <img src="https://github.com/Octachron.png?size=80" /> | Florian Angeletti (**chair**)| [@Octachron](https://github.com/Octachron) | 2025/01 | 2025/01 | 2028/01 | Inria Paris
| <img src="https://github.com/nojb.png?size=80" /> | Nicol&aacute;s Ojeda B&auml;r | [@nojb](https://github.com/nojb) | 2025/01 | 2025/01 | 2028/01 | LexiFi
| <img src="https://github.com/let-def.png?size=80" /> | Fr&eacute;d&eacute;ric Bour | [@let-def](https://github.com/let-def) | 2025/01 | 2025/01 | 2028/01 |
| <img src="https://github.com/goldfirere.png?size=80" /> | Richard Eisenberg | [@goldfirere](https://github.com/goldfirere) | 2025/01 | 2025/01 | 2028/01 | Jane Street
| <img src="https://github.com/andrewjkennedy.png?size=80" /> | Andrew Kennedy | [@andrewjkennedy](https://github.com/andrewjkennedy) | 2025/01 | 2025/01 | 2028/01 | Meta
| <img src="https://github.com/fpottier.png?size=80" /> | Fran&ccedil;ois Pottier | [@fpottier](https://github.com/fpottier) | 2025/01 | 2025/01 | 2028/01 | Inria Paris
| <img src="https://github.com/gasche.png?size=80" /> | Gabriel Scherer | [@gasche](https://github.com/gasche) | 2025/01 | 2025/01 | 2028/01 | Inria Paris
| <img src="https://github.com/lpw25.png?size=80" /> | Leo White | [@lpw25](https://github.com/lpw25) | 2025/01 | 2025/01 | 2028/01 | Jane Street
| <img src="https://github.com/yallop.png?size=80" /> | Jeremy Yallop | [@yallop](https://github.com/yallop) | 2025/01 | 2025/01 | 2028/01 | University of Cambridge

<!--
We would also like to thank our former members:

| Richard Eisenberg | [@goldfirere](https://github.com/goldfirere) | 2025/01 - ??? |
-->

## Committee bylaws

### Committee composition

The committee is formed of roughly 9 members of the OCaml community who wish to volunteer for this service. It is our aim that this committee be diverse; by representing different viewpoints, we will make decisions that benefit larger segments of our community.

Specifically, we would like the committee to represent at least the following constituencies:

* OCaml developers
* Authors (whether in print or online)
* Educators
* Industrial users
* Researchers
* Tooling developers

We recognize that we may not always live up to this ideal, but when choosing new members, we aim to correct any imbalances.

### Officers

#### Chair

The committee has a chair. The chair is responsible for keeping the committee humming along, by dealing with requests and nominating members to consider language proposals. This role is one of service, and we are all grateful to the person who occupies it.
  
If the chair seems to be lax in their duties, it is the job of the rest of the committee to politely reach out to the chair and offer to let them step down.

#### Secretary

The secretary's role is to administer the process for electing new members to the committee, as described below. The secretary also serves as a backup chair, should the chair be away or unavailable to contribute for a fixed period of time.

#### Terms

Officers serve until the end of their term on the committee, or when they choose to step down. When an officer slot is vacant, any member of the committee can self-nominate for the role. In the event of a contested officer slot, the entire committee votes for the officers, as tabulated by the (possibly outgoing) secretary. Votes are sent by direct email to the secretary, in order to be private.

### Method of communication

Most official committee business takes place via the [`ocaml-language-committee@inria.fr` mailing list](https://sympa.inria.fr/sympa/info/ocaml-language-committee). The archives of this list are public, and any member of the community may subscribe. It is intended that the list be used by the committee, though community members might be asked to post there during relevant conversations.

Any business not on the public mailing list will be of a sensitive nature, most likely pertaining to individuals. In particular, discussions of selecting new members will not appear on the mailing list.

### Term limits and committee selection

Any community thrives best by continuing to refresh itself with new members. The main criteria for becoming a member of the committee are:

* A track record of constructive contribution to public discussion of OCaml language and stdlib proposals
* A track record of constructive contribution to one or more of the communities outlined above (users, tool authors, etc)
* An expressed willingness to respond to proposals in a timely manner.

These criteria are not exhaustive -- nominations and self-nominations are free to strengthen the case in whatever way they deem appropriate --- but a record of thoughtful contribution in a public space carries the most weight.

Membership on the committee comes with a three-year term, extended so that a term expires only at the end of a nomination process.

A nomination process is triggered when the number of members is about to drop below 9.


When a nomination process is triggered, the secretary of the committee will put out a public call for nominations for people to join the committee. Nominations include a brief bio of the nominee and reasons why they would be appropriate for the committee. Self-nominations are welcome and expected. Members whose term is due to expire are free to re-nominate themselves. When the secretary's term is about to expire, another (unexpiring) member of the committee fills in for the secretary to run the process for replacing the secretary and any other members, unless the secretary is not being re-nominated and wishes to run the process one last time.

Members of the committee, including those whose term is about to expire, vote on new membership via a ranked voting system, according to the Schulze algorithm. The ranking system includes a method for choosing multiple winners.

The voting process may result in a number of new members not equal to the number of outgoing members. This is fine; the size of the committee is not fixed.

The nomination and voting process is kept private, by using direct email to committee members, not the mailing list.

Any member of the committee is free to step down at any time; such a member may choose to leave the committee immediately or to wait until the end of a nominating process (which would be triggered only when the number of members is about to drop below 9).

There is no process for members of the public at large to directly add or remove committee members. (That is, there is no public vote.) Representative voting across the internet is fraught, and the drawbacks to such a system seem to outweigh any benefits. It is expected that a misbehaving committee (say, one that selects only its friends and ignores other nominations) loses legitimacy and is publicly called into question in an attempt to make changes for the better in its operation.

## Conflicts of interest

For the sake of transparency, the committee chair is expected to disclose any
known potential conflict of interest between committee members and proponents of
a specific language change at the start of the committee deliberation.

Taking in account the small world nature of the OCaml community, the committee
considers that transparency is a sufficient measure to avoid unconscious or
covert influences on the deliberation. Thus conflicted committee members are not
expected to recuse themselves from the deliberation nor the potential votes.

As a supplementary measure, if a shepherd is in a position of conflict of
interests, they should delegate the final report on the committee deliberation
to a non-conflicted committee member.

Currently, the committee classifies at least the following situation as being
unconditional source of conflict of interests:

A. being in the same institution as the proponents of a language change
B. on-going or recent contractual or financial ties to the proponent of the language change

Other sources of conflicts of interest should be reported to the committee chair
at the start of a deliberation at the personal discretion of committee members.
