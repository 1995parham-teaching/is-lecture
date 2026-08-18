# Lecture 2 — Security Architecture

The management half of the course, and the only deck with no attack in it.
34 slides, 5 vertical stacks. Topics (TOC indices): `0` X.800 ·
`1` Enterprise Architecture · `2` Security Policy · `3` Risk Management ·
`4` Incidents & Continuity.

## The organising idea

Lecture 1 ends on _name the property, name the threat, then pick the
mechanism_. This deck is what an organization does with that sentence: X.800
turns the property into a **service**, policy writes it down, risk management
decides whether it is worth paying for, and incident response covers the case
where the answer was wrong.

The closing slide hands off to lecture 3 by observing that nearly every X.800
mechanism is cryptographic. That hand-off is why X.800 comes first in the deck
rather than last.

## Facts worth not re-deriving

- **X.800 is ITU-T, 1991, and ISO 7498-2 is the same architecture.** Both names
  are on the first slide on purpose — students meet both.
- The **five service categories** and the **eight specific / five pervasive
  mechanisms** are X.800's own lists and counts. If you add an item, it is no
  longer X.800.
- **Availability is deliberately not one of the five.** X.800 treats it as a
  property, and the slide says so — it is the standard's known gap, not an
  omission in the deck.
- The **service × mechanism table** is a reduced version of X.800's table 2.
  The teaching point is the bottom row: encipherment cannot give
  non-repudiation. Keep that column empty.
- The risk loop is **ISO 27005 / NIST SP 800-30**; the incident lifecycle is
  **NIST SP 800-61** and has four phases, the third being a loop.
- **NIST CSF has six functions since 2.0** — Govern was added. The slide lists
  six; do not "correct" it back to five.
- The ISO split is real and is a common exam question: **27001 is
  certifiable requirements, 27002 is guidance**.

## The ALE example

The two transcripts in the Risk Management stack are one real run of a script,
split across two slides. The numbers are chosen so the control **loses money**
— 42,000 of avoided loss against 60,000 of cost, a net of −18,000 per year.

That negative is the point of the pair. A worked example where the control pays
for itself teaches nothing, because the interesting move is what happens next:
the argument shifts to whether the asset is really worth 400,000. If you change
any input, re-run it and re-capture **both** blocks — the second slide's figures
are derived from the first's and will silently stop adding up.

## Cautions

- Six tables carry an explicit `font-size` between `0.78em` and `0.85em`. The
  service × mechanism matrix is the tightest at `0.78em`; it has six columns and
  will run off the slide at full size.
- The Standards You Will Meet slide is a list of names, not a recommendation.
  The one opinion on it — start with the CIS Controls — is in the closing line
  and is deliberate.
- Nothing in this deck was captured from a real organization, and it should stay
  that way. The examples are generic on purpose.
