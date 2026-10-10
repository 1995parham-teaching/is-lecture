# Lecture 2 — Security Architecture

The management half of the course, and the only deck with no attack in it.
43 slides including the generated title, 5 vertical stacks. Topics (TOC indices): `0` X.800 ·
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

## The grade service, continued

The grade service from lecture 1's `#/14` stack is the running example
(`lecture-1/CLAUDE.md` has the full source). Its four application slides now
sit beside their concepts: **The Same Code, Two Mechanisms** closes X.800;
**The Policy Is a Test** follows **Why Policies Fail**; **Is the Check Worth
It?** follows risk treatment; **The Day It Happens** closes Incidents &
Continuity as a reconstruction of the original incident before the fix.

The instructor's brief (6 October 2026) was _no concept under a different name
every time, more flow, more engineering and hands-on_. Keep the same code and
log, attaching ideas to lines that already exist. No "lecture 1 said / X.800
says" tables.

Enterprise Architecture starts with inventory, then the grade service's trust
boundaries, then SABSA's views of that same system. The policy opening connects
those boundaries to allowed actions and decision owners, and distinguishes an
internal mandatory standard from the external standards catalogue.

- **Two Mechanisms** annotates lecture 1's fix: the `if` is access control (a
  specific mechanism) and `log.Printf` is the security audit trail (a pervasive
  one). The check protects integrity; the log supports detection and
  investigation. The last bullet points the wifi reader at encipherment and
  lecture 3 so that confidentiality is not forgotten.
- **The Policy Is a Test** is the deck's own line from **Why Policies Fail**
  — the best policy is enforced by a mechanism — made literal. The test is
  `main_test.go` in the same package:

  ```go
  package main

  import (
  	"net/http"
  	"net/http/httptest"
  	"testing"
  )

  // The policy: only the instructor of record may enter or change a grade.
  func TestOnlyInstructorSetsGrade(t *testing.T) {
  	req := httptest.NewRequest("POST", "/grade?id=4001&grade=20", nil)
  	req.Header.Set("X-User", "4001") // a student
  	rec := httptest.NewRecorder()
  	setGrade(rec, req)
  	if rec.Code != http.StatusForbidden {
  		t.Fatalf("a student changed a grade: HTTP %d", rec.Code)
  	}
  	if grades["4001"] != 12 {
  		t.Fatalf("grade moved to %d", grades["4001"])
  	}
  }
  ```

  The slide shows it without the second `if`, for height. Run against
  lecture 1's first version it fails with `HTTP 200`; against the second it
  passes. **The `2026/10/06 01:01:24 grade 4001=20 by 4001` line inside the
  failing run is real**: the handler logs to stderr and `go test` shows it. Do
  not strip it. The second run uses `-count=1` so the output is not
  `(cached)`; the timing (`0.219s`) will differ on re-capture.

- **Is the Check Worth It?** is `risk.py`, in the same style as **The Control
  That Does Not Pay** and answering lecture 1's "which failure is worst?":

  ```python
  check_cost = 1_000          # one developer-day, paid once
  sle        = 5_000          # one hearing, one reissued transcript
  aro        = 2              # assumed successful changes per year without control

  ale = sle * aro
  print(f"ALE               = {sle:,} x {aro}  = {ale:>8,} per year")
  print(f"check pays back in  {check_cost / ale * 365:.0f} days")
  ```

  The numbers are invented and unitless, chosen so the check is bought
  (reduce) and the deadline-day outage is accepted. Keep that pair if you
  change them.

- **The Day It Happens** is `grep -v 'by parham' server.log` over lecture 1's
  first-version log, so the timestamp (`01:01:23`) is the same line the
  attack slide shows. It explicitly revisits the vulnerable version. Recovery
  requires verifying 12 against the instructor's grade record: the displayed
  log only records the new value, 20. The last bullet connects regression tests
  and dated incident evidence to a review of the risk assumptions.

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
The second slide states the assumed 75% frequency reduction (0.40 to 0.10
breaches/year), with loss per breach unchanged, before showing the result.

That negative is the point of the pair. A worked example where the control pays
for itself teaches nothing, because the interesting move is what happens next:
the argument shifts to whether the asset is really worth 400,000. If you change
any input, re-run it and re-capture **both** blocks — the second slide's figures
are derived from the first's and will silently stop adding up.

## Real-world photographs

Three photo slides follow Zones and Boundaries, Preparation Is the Whole Game,
and Backups, Honestly. Each carries a source/license credit, a teaching point,
and a question tied to the grade service. `img/SOURCES.md` records provenance.
The FEMA photo depicts a hurricane exercise, used explicitly as a preparation
analogy. The tape-library photo does not establish that its tapes are offline,
off-site, or used as backups; the slide makes that distinction explicit.
Images use a 300px contained frame without cropping to leave room for discussion.

## Cautions

- Six tables carry an explicit `font-size` between `0.78em` and `0.85em`. The
  service × mechanism matrix is the tightest at `0.78em`; it has six columns and
  will run off the slide at full size.
- The Standards You Will Meet slide is a list of names, not a recommendation.
  The one opinion on it — start with the CIS Controls — is in the closing line
  and is deliberate.
- The code and logs are generic teaching examples. Publicly sourced photographs
  illustrate concepts; do not add captures of an organization's private systems.
