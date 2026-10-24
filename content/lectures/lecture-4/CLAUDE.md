# Lecture 4 — Access Control

51 slides including the generated title (44 content, 6 TOC), 5 vertical
stacks, 8 hands-on slides, 6 diagrams, 7 real-world incident slides. Topics (TOC indices):
`0` The Four A's · `1` Access Control Models · `2` Passwords ·
`3` Second Factors · `4` Biometrics.

## The organising idea

One function, `allow(subject, action, object)`, stated on the first slide and
answered from four directions. The models section is about the
**authorization** half; the last three sections are about the
**authentication** half. The deck keeps them apart on purpose, because
conflating them is the bug the lecture is trying to prevent.

## The teaching path

The authentication half runs **know → have → are**, and **Three Kinds of
Evidence** (the first slide of Passwords) announces that order. Passwords come
first because the grade service uses one; **Attacks on Passwords** closes that
section on "the other three need a second factor — the next section", which is
the question Second Factors answers. The deck used to put TOTP before
passwords, under a topic named "Authentication"; do not move it back.

Each section ends on the question the next one answers:

| Section               | Ends on                                            |
| --------------------- | -------------------------------------------------- |
| The Four A's          | audit: record every decision, never the secret     |
| Access Control Models | every model trusts the subject's name              |
| Passwords             | three of the five attacks need a second factor     |
| Second Factors        | TOTP is phishable; WebAuthn exists because of this |
| Biometrics            | a biometric is a username, not a password          |

Terms are defined where they first appear: OWASP Top Ten (Identify, Then
Authenticate…), bearer token (Columns or Rows), offline attacker (Salt Is Not
Enough), KDF and memory-hard (What To Actually Use), EER (FAR, FRR and the
Threshold).

### Check Yourself

The second-to-last slide. Expected answers:

1. **Authorize** failed; the login worked. The service should send **403**.
   This is broken access control, and the reason the role check exists.
2. The **capability** (a row, held by the subject) answers "what may this user
   touch?" at once, and is the one that is hard to revoke. The ACL (a column)
   needs a walk over every object.
3. Each hash has its **own random salt**, so the inputs differ. A salt stops
   precomputation but not speed: an offline attacker still makes millions of
   SHA-256 guesses a second, which is why a slow KDF is needed.
4. A TOTP code is a number that the user can type anywhere, so a fake site
   forwards it within thirty seconds. A WebAuthn signature covers the **real
   origin**, so a signature made for a look-alike domain does not verify.

Keep the questions answerable from the slides alone.

## The grade service, continued

Lecture 1 built the grade service with identity taken from an `X-User` header
"filled in by the login layer in front", and promised that lecture 4 builds
that layer. This deck keeps the promise. Lecture 3's closing slide names the
header as the open problem; this deck's first slide picks it up. Lecture 6
continues from **this deck's final version**, so if the code or a captured
line changes here, re-capture there too.

Where it appears, each next to its concept rather than as an appendix:

- **The Four Steps, Drawn** (`four-as.svg`) — the request through identify →
  authenticate → authorize → audit, with the 401 and 403 exits.
- **You Already Wrote One** — lecture 1's `roles` map and `if`, named as RBAC.
  The code on the slide is copied from lecture 1's second version; keep them
  identical.
- **The Grade Service Gets a Login** · **Three Requests, Three Answers** ·
  **What the Clock and the Wire Say** — three `hands-on` slides after **What
  To Actually Use**. The first is an excerpt: the `users` map, `login`, and
  the block that replaces `r.Header.Get("X-User")`.
- **Where This Leaves Us** (formerly **Putting It Together**) says what the
  service still lacks (a second factor, and a password sent in the clear on
  every request) and hands that to lecture 6.

### The source

`main.go`, module `grades`, `go 1.27`, with `golang.org/x/crypto v0.58.0` in
`go.mod` (`go get golang.org/x/crypto/bcrypt`). Built with Go 1.27.2 on macOS,
`gofmt`, `go vet` and `go build` clean:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"strconv"

	"golang.org/x/crypto/bcrypt"
)

var grades = map[string]int{"4001": 12, "4002": 17}

var roles = map[string]string{"parham": "instructor", "4001": "student"}

// bcrypt, cost 12: the salt and the cost are stored inside each hash
var users = map[string]string{
	"parham": "$2a$12$PED31kv6I3xrnjq.XvRgN.q5BIlLvFA6Ae2BklreAvhrgcLO/RZz6",
	"4001":   "$2a$12$fHdcH0q0mpgv4UNlutXXQuEmZDZ2eAqQ5tPaTvGd9DIH9XMtbSUmy",
}

func login(r *http.Request) (string, bool) {
	name, pass, ok := r.BasicAuth()
	if !ok {
		return name, false
	}
	err := bcrypt.CompareHashAndPassword([]byte(users[name]), []byte(pass))
	return name, err == nil
}

func setGrade(w http.ResponseWriter, r *http.Request) {
	user, ok := login(r)
	if !ok {
		log.Printf("FAILED login as %q", user) // the name, never the password
		http.Error(w, "login first", http.StatusUnauthorized)
		return
	}
	id := r.FormValue("id")
	grade, _ := strconv.Atoi(r.FormValue("grade"))
	if roles[user] != "instructor" {
		log.Printf("DENIED grade %s=%d by %s", id, grade, user)
		http.Error(w, "forbidden", http.StatusForbidden)
		return
	}
	grades[id] = grade
	log.Printf("grade %s=%d by %s", id, grade, user)
	fmt.Fprintf(w, "%s is now %d\n", id, grade)
}

func main() {
	http.HandleFunc("POST /grade", setGrade)
	log.Fatal(http.ListenAndServe("localhost:8080", nil))
}
```

The passwords are teaching values, published on the slides on purpose:
`parham` is `correct horse battery staple`, `4001` is `tulip-river-42`. The
hashes were made with `bcrypt.GenerateFromPassword(pw, 12)`; a new run gives a
new salt and so a new hash, which is fine — but then the slide must change too.

### How the captures were made

Captured on 14 October 2026. Each command was run exactly as it is printed on
the slide, including the `\` continuation lines, which are there for width:

```sh
go build -o grades . && ./grades 2> server.log &
curl -u parham:guess -d 'id=4002&grade=18' localhost:8080/grade
curl -u 4001:tulip-river-42 -d 'id=4001&grade=20' localhost:8080/grade
curl -u 'parham:correct horse battery staple' \
       -d 'id=4002&grade=18' localhost:8080/grade
cat server.log
curl -s -o /dev/null -w '%{http_code} in %{time_total}s\n' \
       -u parham:guess -d 'id=4002&grade=18' localhost:8080/grade
curl -s -o /dev/null -w '%{http_code} in %{time_total}s\n' \
       -u nobody:guess -d 'id=4002&grade=18' localhost:8080/grade
echo NDAwMTp0dWxpcC1yaXZlci00Mg== | base64 -d
```

Facts worth not re-deriving:

- **Basic auth is a deliberate simplification**, like `X-User` was in
  lecture 1. The slide says so: the password crosses in base64 on every
  request, and sessions and TLS are lecture 6. Do not add sessions here.
- **401 versus 403 is the point of the second slide.** 401 is authentication
  failing, 403 is authorization failing — the deck's
  identify/authenticate/authorize split, as HTTP status codes.
- **The `FAILED login` line logs the name, never the password.** That is the
  "Do not log the secret" bullet of **Audit Is Not Optional**, in code.
- **The timing leak is real and is left in on purpose.** A known name costs one
  bcrypt comparison (0.18 s); an unknown name makes `CompareHashAndPassword`
  fail on an empty hash at once (0.0004 s), so the clock says which accounts
  exist. The slide sets it as an exercise. The usual fix is to compare against
  a fixed dummy hash when the name is unknown. If you fix it in the code,
  re-capture and rewrite that bullet.
- 0.18 s at cost 12 is the "few hundred milliseconds" of **What To Actually
  Use**, and about five online guesses a second per core.
- bcrypt, not Argon2id, because `x/crypto/bcrypt` is one call and stores salt
  and cost inside the string. The deck still names Argon2id as the first
  choice; do not change that slide to match the code.

## The diagrams

Six hand-written SVGs in `img/`, drawn for the league background with the
`$highlights` colours as literal hex. Render before and after an edit with
`rsvg-convert -b '#2b2b2b'`; text that overflows a box is not reported.

| Diagram          | Slide                      | Draws                                                 |
| ---------------- | -------------------------- | ----------------------------------------------------- |
| `four-as`        | The Four Steps, Drawn      | one grade request through the four steps, 401 and 403 |
| `acl-capability` | The Access Matrix          | the matrix; a column is an ACL, a row a capability    |
| `salt`           | The Salt, Drawn            | unsalted vs salted, the values from the hands-on run  |
| `far-frr`        | FAR, FRR and the Threshold | FAR and FRR against the threshold, the EER, two uses  |
| `twitter-2020`   | Twitter 2020, Drawn        | phone call → relayed VPN login and MFA push → tool    |
| `adobe-ecb`      | Adobe 2013, Drawn          | one-key 3DES-ECB plus hints vs a salted KDF           |

- `acl-capability` **replaced** the access-matrix table, and `far-frr`
  **replaced** the FAR/FRR bullet list. The same cells and the same claims are
  in the drawings; do not add the text back beside them.
- `salt` carries the real values from **Never Store It Twice the Same**
  (`f52fbd32…`, salts `MrTqQCcV…` and `9kSX7tN9…`). If that slide is
  re-captured, redraw this one.
- `far-frr` is a schematic, not data. The curves are mirror images so that they
  cross at x = 430, y ≈ 245, where the EER dot is.
- `four-as` shows `curl -u 4001:••••••••`: the student's request, which takes
  the 403 exit. The green `200` label is generic on purpose.

- `twitter-2020` uses only figures from the NYDFS report (130 accounts, 45
  tweeted, about $118,000, over 1,000 staff with the tool).
- `adobe-ecb` is a schematic: the names, hints and `$2a$…` values are
  invented, and the drawing says so. Do not replace them with real records.

## Real-world cases

Seven `In the Wild: Name, Year` slides, each placed directly after the concept
it illustrates, inside that section's stack. None is `hands-on`. Each carries
its source line; re-check a claim against it before editing.

| Slide                   | After                        | Source                                                                                                                                              |
| ----------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Twitter, 2020 (+ drawn) | You Already Wrote One (RBAC) | NYDFS, Report on Investigation of Twitter's July 15, 2020 Cybersecurity Incident — https://www.dfs.ny.gov/Twitter_Report                            |
| Colonial Pipeline, 2021 | Three Kinds of Evidence      | Reuters via Carrier Management — https://www.carriermanagement.com/news/2021/06/09/221844.htm                                                       |
| RockYou, 2009           | Passwords Are Still Here     | The Register, 16 Dec 2009 — https://www.theregister.com/2009/12/16/rockyou_password_snafu/ ; Kali wordlists page for `rockyou.txt`                  |
| LinkedIn, 2012          | The Salt, Drawn              | Krebs on Security, May 2016 — https://krebsonsecurity.com/2016/05/as-scope-of-2012-breach-expands-linkedin-to-again-reset-passwords-for-some-users/ |
| Adobe, 2013 (+ drawn)   | What To Actually Use         | CSO Online — https://www.csoonline.com/article/540070/network-security-adobe-confirms-stolen-passwords-were-encrypted-not-hashed.html               |
| Reddit, 2018            | Something You Have           | BleepingComputer — https://www.bleepingcomputer.com/news/security/reddit-announces-security-breach-after-hackers-bypassed-staffs-2fa/               |
| Touch ID, 2013          | Attacking and Defending It   | Chaos Computer Club — https://www.ccc.de/en/updates/2013/ccc-breaks-apple-touchid                                                                   |

Facts worth not re-deriving:

- **Twitter**: the MFA was an app push that some employees approved; the
  NYDFS report says hardware MFA "would have stopped the Hackers". The
  dfs.ny.gov site serves a Cloudflare challenge to `curl`; the report was read
  through its Wayback Machine snapshot.
- **Colonial**: the CEO told the Senate it was a "complicated password" on a
  legacy VPN account with single-factor authentication. How the password was
  obtained was not established at the hearing — do not add a claim about it.
- **Reddit**: the sources say "SMS intercept", not SIM swap. Keep the wording.
- **Adobe**: the leaked-file size differs by source (130 million in CSO, 150
  million in Krebs); the slide says "over 130 million" and Adobe's 38 million
  active users. The 1.9 million `123456` figure is Jeremi Gosney's analysis.
- **LinkedIn**: 6.5 million is the 2012 posting; 117 million is the 2016 sale.
- The Twitter and Colonial slides are inside sections whose closing slide must
  stay last (**Choosing**, **Attacks on Passwords**); none was placed after one.

## The "AAAA" question

The syllabus says AAAA, and the expansion is genuinely not standardised. The
deck teaches **authentication · authorization · accounting · auditing** and
then says, on the same slide, that RADIUS and Diameter ship the first three as
AAA and that some texts substitute identification. That hedge is deliberate —
students meet both lists and should not think one is a mistake.

## The captured material

- **DAC on This Machine** and **Two Bits Worth Knowing** are real `ls`/`chmod`
  output from macOS. `/usr/bin/su` really is `-rwsr-xr-x` and `/private/tmp`
  really is `drwxrwxrwt` — note the path is `/private/tmp`, not `/tmp`, because
  `/tmp` is a symlink there and `ls -ld` would show the link instead of the
  sticky bit.
- **TOTP** is a real Go implementation, `go vet` clean, and the five outputs are
  the **RFC 6238 test vectors reproduced exactly** (SHA-1, 8 digits, 30-second
  step, secret `12345678901234567890`). That is the strongest claim in the deck:
  if a change breaks those five numbers, the change is wrong.
- **The hashing pair** — `shasum` twice, then `openssl passwd -6` twice — is one
  run. The two `$6$` hashes are truncated on the slide for width; the salt is
  the part between the second and third `$`, and it differs, which is the whole
  point.
- **The KDF benchmark** is `hashlib` on one core of the instructor's laptop:
  3.68M SHA-256/sec against 17 PBKDF2/sec and 5 scrypt/sec. Quote it as "one
  laptop core", which is what the slide says — it is not a benchmark of anything
  else. The ratios are about 200,000× against PBKDF2 and 700,000× against
  scrypt; the slide once gave 200,000× for scrypt, which was wrong.

## Facts worth not re-deriving

- **Bell–LaPadula is confidentiality** (no read up, no write down); **Biba is
  integrity** and is its mirror (no read down, no write up). Getting these
  backwards is the classic slip.
- ACL = a **column** of the access matrix (per object); capability = a **row**
  (per subject). The bearer-token remark links this to lecture 6.
- **Broken access control has been OWASP Top Ten #1 since the 2021 edition.**
- NIST **SP 800-63B** is the source for the password guidance, including the
  removal of forced rotation and composition rules. This reverses advice most
  students were taught, so the slide is framed as "the rules changed".
- **WebAuthn/FIDO2 is phishing-resistant because the signature is over the
  origin.** TOTP is not, and the deck says so twice — once in the "have"
  section and once after the code. Keep both.
- Argon2id is the current first choice; bcrypt and scrypt remain acceptable.

## Cautions

- The TOTP code block escapes `&` as `&amp;` — it contains two bitwise ANDs.
  Check that before editing it.
- **"A biometric is a username, not a password"** is the take-away of the last
  stack. The two supporting facts — not secret, not revocable — must stay with
  it.
- The biometrics section mentions demographic bias in error rates. That is a
  measured finding, not an aside; do not cut it for space.
- Four tables carry an explicit `font-size` of `0.8em`–`0.85em`. The access
  matrix used to be a fifth; it is now `acl-capability.svg`.
