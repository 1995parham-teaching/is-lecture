# Lecture 1 — Introduction

The opening session: who the instructor is, what the course is and is not, the
properties being defended, the vocabulary for threats and attacks, defence in
depth, and the policies. 38 slides, 4 vertical stacks.

Topics (TOC indices): `0` Instructor · `1` Course Overview · `2` Security
Properties · `3` Threats & Attacks · `4` Defence in Depth · `5` Course Policies.

## The organising idea

The deck ends on the line it is built to earn: **name the property, name the
threat, then pick the mechanism.** The three sections before it are those three
steps in order — CIA first, the attacker second, controls last. Slides that
jump straight to a mechanism break the spine.

## The instructor stack

Three slides — who is talking → how to reach me → why email — mirrored from
`ie-lecture` lecture 1, because the same person teaches both and the contact
policy is the same. Only the subject prefix differs: `[IS]` here, `[IE]` there.

**The two experience lines — 9 years backend, 4 of them on platform — are the
instructor's own figures.** They are copied, not derived. Do not extrapolate
them, do not increment them for a new term. If they need changing, that is a
fact only the instructor has, and it needs changing in both repositories.

**Deliberately absent: a reply-time promise**, for the same reason as in
`ie-lecture`.

## Facts worth not re-deriving

- The `shasum` transcript on **Integrity, in One Command** is a real capture.
  The two digests belong to `transfer 100 USD to alice` and
  `transfer 900 USD to alice`, each with a trailing newline. Re-capture both if
  you touch the sentence — changing one and keeping the other is the one way
  this slide can quietly become a lie.
- The slide after it is the point of the pair: a bare hash is integrity against
  accident, not against an adversary, because whoever rewrites the file rewrites
  the digest. Do not merge the two slides.
- FIPS 199 defines C, I and A. Authenticity and accountability come from NIST
  SP 800-33 — the attribution on **Three Is Not Always Enough** is deliberate.
- The eight design principles are Saltzer and Schroeder, 1975, and the names are
  theirs. `psychological acceptability` is the original term, not a paraphrase.
- Passive vs active, and the four active categories (masquerade, replay,
  modification, denial of service) are X.800's own taxonomy. Lecture 2 opens by
  naming the standard; this slide is the setup for that, so keep the wording in
  step with it.
- The grading table sums to 100 (35 midterm / 35 final / 30 homework) with a 40%
  minimum on each exam — the same policy as `ie-lecture`.

## Cautions

- **The Lab Rule slide is not decoration.** It is the one slide that makes the
  rest of the course teachable. Do not soften it, and do not move it later in
  the deck than the first session.
- Two tables carry `font-size: 0.8em` and one `0.85em`. At full size their rows
  wrap and the table runs past the 700px slide. Re-check the height before
  removing those styles.
- The syllabus list here and the README must agree — seven lectures, in this
  order. It is the same failure mode `ie-lecture` documents.
