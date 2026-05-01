# Conway's Law and Team Topology

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." — Melvin Conway, 1968.

This is one of the few empirical regularities in software engineering. Treat it as physics, not advice.

## What the law actually predicts

If two teams own two halves of a system, the seam between those halves will be the most stable boundary in the design — even if it's the wrong boundary, technically. Conversely, parts of the system owned by one team will be tightly coupled even when they shouldn't be — because there's no organizational pressure to make a clean interface where no team boundary exists.

Specific consequences observed across many organizations:

- **Cross-team interfaces become rigid** (slow to change, formal, often over-engineered) regardless of whether the technical situation calls for it.
- **Within-team coupling grows** (modules call each other freely, share types, depend on internal details).
- **Components owned by no single team rot** — the broken-windows zone where everyone touches and nobody owns.
- **Team splits cause subsystem splits.** When a team is split in two, the system the team owned splits along that fault line within ~6–18 months.
- **Team merges cause subsystem fusion.** Components previously separate begin to interpenetrate.

## Implications for design review

When evaluating a proposed module boundary, ask:

- **Does this boundary match a team boundary?** If yes, it has organizational support and will be stable. If no, it will erode under cross-team friction; either change the boundary, or change the team structure, or accept the erosion as an explicit trade.
- **Does this boundary cross a team?** Then it must be a deliberate, documented interface with a clear ownership story for each side. Casual cross-team coupling becomes "team A keeps changing things on us with no warning" within months.
- **Is the proposed module owned?** Not "owned" in the GitHub-CODEOWNERS sense, but actually owned — a team that takes responsibility for its on-call, its evolution, its quality. Unowned modules become the broken-window zone.

## Inverse Conway maneuver

A specific organizational move: **change the team structure to produce the desired architecture.** Coined by Jonny LeRoy and Matt Simons (Cutter IT Journal, December 2010); popularized in *Team Topologies* (Skelton & Pais, 2019).

Example: you want to move from a monolith to a set of services. The monolith is owned by one team. Conway predicts that under one team, the system will trend back toward monolithic shape. The fix isn't more architectural rules — it's to split the team along the desired service boundaries. Each new team takes ownership of a service and is on-call for it. The architecture follows.

This is heavy machinery and only worth invoking for big shifts. But for big shifts, no amount of architectural diagrams will overcome team-structure pressure.

## Team Topologies (Skelton & Pais)

Useful vocabulary that has caught on:

- **Stream-aligned team.** Owns a slice of value end-to-end. Optimized for fast, independent flow.
- **Platform team.** Provides internal services (CI, observability, infrastructure) that stream-aligned teams consume. Should expose its services via a self-service API to avoid being a bottleneck.
- **Enabling team.** Helps stream-aligned teams gain new capabilities (e.g., adopting a new technology). Time-bounded engagement.
- **Complicated subsystem team.** Owns a piece that requires deep specialist knowledge (graphics engine, ML model serving, cryptography). Stream-aligned teams interact with it through an API.

The key idea: most of your teams should be **stream-aligned** (own a slice of value, ship independently). Other team types support that flow. Misalignment between team types and what teams are actually doing produces friction.

## When the design is right but the org is wrong

Sometimes a clean design is impossible because the org is structured wrong. Recognize this and say so explicitly:

- "Technically, the right boundary is between A and B. But team T owns both, so that boundary will erode within a year. Either we keep them together (and accept the technical compromise), or we split T (and accept the org change)."
- "This module sits between two teams with no clear owner. We need to assign owner before we ship it; otherwise it becomes the broken-windows zone."

These are honest engineering trade-offs, not whining. Design that ignores Conway's Law produces beautiful diagrams and ugly code.

## When to invoke Conway in a review

- A new module is proposed with no clear team ownership. Flag.
- A module is proposed that crosses a team boundary without an explicit interface contract. Flag.
- A team split is happening and someone is proposing not to split a system the old team owned. Predict the system will split along the new fault line; better to design the split than discover it.
- A team merge is happening and someone is proposing to keep two systems strictly separate. Predict the systems will fuse; better to plan for it.
