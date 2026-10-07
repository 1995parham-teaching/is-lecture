# Lecture 1 — Introduction

The opening session: who the instructor is, what the course is and is not, the
properties being defended, the vocabulary for threats and attacks, defence in
depth, and the policies. 41 slides, 5 vertical stacks.

Topics (TOC indices): `0` Instructor · `1` Course Overview · `2` Security
Properties · `3` Threats & Attacks · `4` Defence in Depth · `5` Course Policies.

## The organising idea

The deck ends on the line it is built to earn: **name the property, name the
threat, then pick the mechanism.** The three sections before it are those three
steps in order — CIA first, the attacker second, controls last. Slides that
jump straight to a mechanism break the spine.

## The grade service

The grade database on **The Same System, Three Failures** is the deck's one
running example. The three-slide `hands-on` stack that closes the Defence in
Depth section — **The Grade Service** · **The Attack, in One Request** ·
**The Fix, and the Log** — turns it into a program, so that every term of the
lecture lands on a line of code instead of a table of names. The instructor's
brief (6 October 2026) was _no concept under a different name every time, more
flow, more engineering and hands-on_. Keep it that way: no vocabulary tables,
no renaming; a term appears once, next to the code or the transcript that
earns it.

Lecture 2 picks up the **same program** on its closing stack (`#/12`, "The
Same Code, Two Mechanisms" through "The Day It Happens"). If the code or a
captured line changes here, re-capture there too; the transcripts share the
same `server.log`.

### The source

Every transcript on the three slides is a real run, captured on 6 October 2026
on macOS with Go 1.27.1. The full first version, `main.go` (module `grades`,
`go 1.27` in `go.mod`):

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"strconv"
)

var grades = map[string]int{"4001": 12, "4002": 17}

func setGrade(w http.ResponseWriter, r *http.Request) {
	user := r.Header.Get("X-User") // filled in by the login layer in front
	if user == "" {
		http.Error(w, "login first", http.StatusUnauthorized)
		return
	}
	id := r.FormValue("id")
	grade, _ := strconv.Atoi(r.FormValue("grade"))
	grades[id] = grade
	log.Printf("grade %s=%d by %s", id, grade, user)
	fmt.Fprintf(w, "%s is now %d\n", id, grade)
}

func main() {
	http.HandleFunc("POST /grade", setGrade)
	log.Fatal(http.ListenAndServe("localhost:8080", nil))
}
```

The second version adds one map and one block, exactly as the fix slide shows:

```go
var roles = map[string]string{"parham": "instructor", "4001": "student"}
```

inserted after `grades`, and before `grades[id] = grade`:

```go
	if roles[user] != "instructor" {
		log.Printf("DENIED grade %s=%d by %s", id, grade, user)
		http.Error(w, "forbidden", http.StatusForbidden)
		return
	}
```

Both versions `gofmt`, `go build` and `go vet` clean. The `"POST /grade"`
pattern needs Go 1.22 or later.

### How the captures were made

```sh
go build -o grades . && ./grades 2> server.log &
curl -H 'X-User: parham' -d 'id=4002&grade=18' localhost:8080/grade
curl -H 'X-User: 4001'   -d 'id=4001&grade=20' localhost:8080/grade
tail -1 server.log                    # first version: the attack succeeds
head -1 server.log                    # second version: DENIED
```

Facts worth not re-deriving:

- **The `X-User` header is a deliberate simplification** and the first slide
  says so: identity comes from "the login layer in front", which lecture 4
  builds. Do not add sessions or passwords here; that is lecture 4's job, and
  the point of the stack is that the bug is the missing _role_ check, not the
  missing login.
- **The vulnerability is the `if`.** It checks that `user` is non-empty, i.e.
  logged in, and never checks what the user may do. The attack slide names
  that line; keep the check in the first version so the line exists to point
  at.
- **The log is in the first version on purpose.** It is what makes lecture 2's
  "The Day It Happens" possible: the attack was logged before anyone thought
  to check. Preparation existed; nobody read it.
- The timestamps (`01:01:23`, `01:01:25`) are the real capture. The server
  binds `localhost:8080` only.
- The marker on each slide is `class="hands-on"` on the `<section>`, per the
  repository rule.

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
- The hexdump on **Metadata, in One Command** is a real capture, taken with the
  command printed on the slide. It is the `server_name` extension of a
  ClientHello to `example.com`: `00 00` is the extension type, `00 0b` the
  length, then the hostname in ASCII. The offsets are OpenSSL's — a different
  client sends a different-sized hello and the name lands elsewhere, so
  re-capture rather than adjust the offsets by hand. Nothing here touches the
  instructor's own network: the only address involved is a public one.
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
