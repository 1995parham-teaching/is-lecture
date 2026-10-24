# Lecture 6 — Web Security

55 `<section>` elements plus the generated title: 44 content slides, 6 TOC
slides, 5 vertical stacks, 9 hands-on slides, 8 diagrams. Topics (TOC indices):
`0` The Web Threat Model · `1` Server-Side Attacks · `2` Sessions & Cookies ·
`3` Client-Side Attacks · `4` TLS and HTTPS.

## Why this order

The sections follow the grade service's own order: SQL (v1, v2), then the
session cookie (v3), then HTTPS (v4). **Sessions come before the client-side
attacks on purpose.** XSS steals a cookie and CSRF rides one, and the CSRF and
SameSite drawings use the grade service's own session — a student must have
seen that cookie first. The order was client-side first until October 2026;
it was swapped in a learnability pass.

Each section ends on the question the next one answers:

| Section     | Ends on                                                             |
| ----------- | ------------------------------------------------------------------- |
| Threat      | the Golden Rule — keep data and code apart                          |
| Server-side | the password still goes with every request (HTTP Has No Memory)     |
| Sessions    | the browser sends the cookie even when another site starts it       |
| Client-side | every defence assumes nobody reads the cookie on the lab wifi       |
| TLS         | Check Yourself, then three take-aways and the hand-off to lecture 7 |

Within Sessions, **Attacks on Sessions** comes before the grade-service code,
so that "a new identifier at every login, which is the fixation fix" on
**Log In Once** points backwards. The Cookie Attributes table names XSS and
CSRF as "next section"; keep those two forward pointers if the order changes.

Terms are defined at first use: _origin_ and the _same-origin policy_ on
**Two Different Victims** (the policy stops reading, not sending — the setup
for CSRF), _JWT_ on **Logging Out Is a Server-Side Delete**. CORS was dropped
from that slide: it was named and never defined, and nothing later needs it.

## The organising idea

Same thesis as the web-security deck in `ie-lecture`: **never mix data with
code**. What is different here is that the rule arrives having already been
demonstrated — lecture 5 overflowed a buffer into a privilege flag, and the
Golden Rule slide names that as the same bug one layer down. Keep that
back-reference; it is what makes this deck a continuation rather than a
restart.

The second axis is the two-victims split on slide 2: server-side attacks take
your machine, client-side attacks take your users. Students consistently
under-weight the second, so the deck says outright that nothing on your server
is damaged and it is still your bug.

## Relationship to `ie-lecture`

`ie-lecture/content/lectures/lecture-11` covers the same ground for a different
audience. Do not assume a student here has seen it — this deck is
**self-contained**, and defines `HttpOnly`, `Secure` and `SameSite` rather than
citing them. Where the two decks state the same fact they should agree; if you
correct one, check the other.

## The grade service, continued

Lecture 4 left the grade service with HTTP Basic auth: a bcrypt-checked
password on every request, in the clear. Its closing slide hands that to this
deck, and the first slide here picks it up. This deck continues from
**lecture 4's final version** (`lecture-4/CLAUDE.md`) in four steps, each next
to its concept:

| Version | Change                                                   | Slides                                                                                                  |
| ------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| v1      | grades move into SQLite; new `GET /grades`, concatenated | **SQL Injection** · **One Student, Every Grade**                                                        |
| v2      | the same query with `?` placeholders                     | **Prepared Statements** · **The Same Request, Prepared**                                                |
| v3      | `POST /login` hands out a session cookie; `POST /logout` | **The Grade Service Gets a Session** · **Log In Once, Then Only the Cookie** · **Log Out, Then Replay** |
| v4      | `ListenAndServeTLS` with a self-signed `localhost` cert  | **The Grade Service, Over HTTPS**                                                                       |

The drawings use the same program: **CSRF on the Grade Service, Drawn** is the
instructor's browser posting `id=4001&grade=20` from `evil.example`, and
**SameSite, Drawn** ends on the grade service's own `Lax` cookie.

### The source

Module `grades`, `go 1.27`, Go 1.27.2 on macOS. `go.mod` requires
`golang.org/x/crypto v0.58.0` (lecture 4) and `modernc.org/sqlite v1.60.1`, a
pure-Go driver, so there is no cgo. Every version is `gofmt`, `go vet` and
`go build` clean. The final version, v4:

```go
package main

import (
	"crypto/rand"
	"database/sql"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"sync"

	"golang.org/x/crypto/bcrypt"
	_ "modernc.org/sqlite"
)

var db *sql.DB

var roles = map[string]string{"parham": "instructor", "4001": "student"}

// bcrypt, cost 12: the salt and the cost are stored inside each hash
var users = map[string]string{
	"parham": "$2a$12$PED31kv6I3xrnjq.XvRgN.q5BIlLvFA6Ae2BklreAvhrgcLO/RZz6",
	"4001":   "$2a$12$fHdcH0q0mpgv4UNlutXXQuEmZDZ2eAqQ5tPaTvGd9DIH9XMtbSUmy",
}

var sessions sync.Map // session identifier -> user name

// POST /login checks the password once and hands out a session
func newSession(w http.ResponseWriter, r *http.Request) {
	name, pass := r.FormValue("name"), r.FormValue("password")
	err := bcrypt.CompareHashAndPassword([]byte(users[name]), []byte(pass))
	if err != nil {
		log.Printf("FAILED login as %q", name) // the name, never the password
		http.Error(w, "login first", http.StatusUnauthorized)
		return
	}
	id := rand.Text() // at least 128 bits from the OS, new at every login
	sessions.Store(id, name)
	http.SetCookie(w, &http.Cookie{Name: "__Host-session", Value: id, Path: "/",
		HttpOnly: true, Secure: true, SameSite: http.SameSiteLaxMode})
	fmt.Fprintf(w, "hello %s\n", name)
}

// POST /logout deletes the session on the server, not only in the browser
func endSession(w http.ResponseWriter, r *http.Request) {
	if c, err := r.Cookie("__Host-session"); err == nil {
		sessions.Delete(c.Value) // a stolen copy now refers to nothing
	}
	http.SetCookie(w, &http.Cookie{Name: "__Host-session", Path: "/", MaxAge: -1,
		HttpOnly: true, Secure: true, SameSite: http.SameSiteLaxMode})
}

// login now reads the cookie: no password travels after the first request
func login(r *http.Request) (string, bool) {
	c, err := r.Cookie("__Host-session")
	if err != nil {
		return "", false
	}
	name, ok := sessions.Load(c.Value)
	if !ok {
		return "", false
	}
	return name.(string), true
}

func setGrade(w http.ResponseWriter, r *http.Request) {
	user, ok := login(r)
	if !ok {
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
	_, err := db.Exec("UPDATE grades SET grade = ? WHERE student = ? AND course = 'IS'",
		grade, id)
	if err != nil {
		http.Error(w, "database error", http.StatusInternalServerError)
		return
	}
	log.Printf("grade %s=%d by %s", id, grade, user)
	fmt.Fprintf(w, "%s is now %d\n", id, grade)
}

// a student reads their own grade in one course
func myGrades(w http.ResponseWriter, r *http.Request) {
	user, ok := login(r)
	if !ok {
		http.Error(w, "login first", http.StatusUnauthorized)
		return
	}
	course := r.FormValue("course")
	rows, err := db.Query("SELECT student, course, grade FROM grades"+
		" WHERE student = ? AND course = ?", user, course)
	if err != nil {
		http.Error(w, "database error", http.StatusInternalServerError)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var s, c string
		var g int
		rows.Scan(&s, &c, &g)
		fmt.Fprintf(w, "%s %s %d\n", s, c, g)
	}
}

func main() {
	var err error
	if db, err = sql.Open("sqlite", "grades.db"); err != nil {
		log.Fatal(err)
	}
	http.HandleFunc("POST /grade", setGrade)
	http.HandleFunc("GET /grades", myGrades)
	http.HandleFunc("POST /login", newSession)
	http.HandleFunc("POST /logout", endSession)
	log.Fatal(http.ListenAndServeTLS("localhost:8080", "cert.pem", "key.pem", nil))
}
```

v1 and v2 are v4 without sessions or TLS: lecture 4's `login()` (Basic auth),
lecture 4's `setGrade` with the `FAILED login` line, and `ListenAndServe`.
v1's `myGrades` builds the query by concatenation:

```go
	query := "SELECT student, course, grade FROM grades" +
		" WHERE student = '" + user + "' AND course = '" + course + "'"
	rows, err := db.Query(query)
```

v2 replaces those three lines with the two on **Prepared Statements**. v3 adds
`newSession`, `endSession` and the cookie-reading `login()`, and drops the
`FAILED login` line from `setGrade`, because a password is no longer checked
there. v4 changes only the `ListenAndServeTLS` line.

`grades.db` is made fresh before each run from `schema.sql`:

```sql
CREATE TABLE grades (student TEXT, course TEXT, grade INTEGER);
INSERT INTO grades VALUES ('4001', 'IS', 12), ('4002', 'IS', 17),
                          ('4001', 'IE', 15), ('4002', 'IE', 19);
```

The 12 and 17 are lecture 1's IS grades; the IE rows exist so that the
injection visibly leaks another course.

### How the captures were made

Captured on 14 October 2026. Each command ran exactly as printed, including
the `\` continuation lines. Each ran in a pseudo-terminal (`script -q
/dev/null`), so curl printed no progress meter — the same output as an
interactive shell. Before each run: `rm -f grades.db server.log`,
`sqlite3 grades.db < schema.sql`, `./grades 2> server.log &`.

```sh
# v1, then v2 — the same two commands against each
curl -u 4001:tulip-river-42 'localhost:8080/grades?course=IS'
curl -u 4001:tulip-river-42 -G localhost:8080/grades \
       --data-urlencode "course=x' OR '1'='1"

# v3 — one run, split across two slides at `cp`
curl -c cookies.txt -D - -d name=parham \
       -d 'password=correct horse battery staple' localhost:8080/login
curl -b cookies.txt -d 'id=4002&grade=18' localhost:8080/grade
cp cookies.txt stolen.txt
curl -b cookies.txt -c cookies.txt -X POST localhost:8080/logout
curl -b stolen.txt -d 'id=4002&grade=20' localhost:8080/grade
cat server.log

# v4 — the certificate is made before the server starts
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
       -nodes -keyout key.pem -out cert.pem -days 1 \
       -subj /CN=localhost -addext subjectAltName=DNS:localhost
curl -d name=parham -d 'password=correct horse battery staple' \
       http://localhost:8080/login
curl --cacert cert.pem -d name=parham \
       -d 'password=correct horse battery staple' https://localhost:8080/login
```

Facts worth not re-deriving:

- **The injection keeps the ownership check and then removes it.**
  `student = '4001'` is in the query, as **Broken Access Control** asks;
  `AND` binds tighter than `OR`, so `... AND course = 'x' OR '1'='1'` matches
  every row. The v2 run prints **nothing** for the second command, which is
  the point; do not "fix" the empty output.
- **`user` comes from the login and is safe; `course` is the hole.** The SQL
  Injection slide says so, so that students do not think every concatenated
  value is equally exploitable.
- **`rand.Text()`** (Go 1.24+) returns 26 base32 characters, at least 128 bits
  from the OS. The session identifier on the slide is that run's; a new run
  gives a new one.
- **`__Host-` needs `Secure`, and `Secure` normally needs HTTPS.** curl treats
  `localhost` as a secure origin and sends the cookie over plain HTTP, which
  is what lets v3 run before the TLS section. That was **tested with curl
  8.7.1 only**; the slide claims it for curl and nothing else.
- **The replay is not logged.** v3's `login()` returns false for an unknown
  session without a log line. The slide raises that as lecture 4's audit
  question; if you add the log line, re-capture and rewrite that bullet.
- **v4 serves the same port.** `Client sent an HTTP request to an HTTPS
server.` is Go's real reply to plain HTTP on a TLS listener. The server log
  of that run records the matching `TLS handshake error`; it is not on the
  slide.
- `--cacert`, not `-k`: the previous slide calls `-k` a finding.

### The older captures

- **Guessing the Session** is a real capture from a small Go server that serves
  two endpoints, one using an `atomic.Uint64` counter and one using
  `crypto/rand`. Both `Set-Cookie` blocks are genuine output. The weak one
  deliberately carries no attributes and the strong one carries all three —
  that contrast is doing two jobs at once, and it is intentional.
- **The certificate slides** are one run of `openssl req` / `s_client` against a
  local `s_server` on port 8443. The self-signed certificate is generated for
  the demonstration; `verify error:num=18` is the real error code.
- The follow-up slide shows the **same connection succeeding** once verification
  is off — TLS 1.3, `TLS_AES_256_GCM_SHA384`, RSA-PSS. The pair is the argument:
  every primitive works, and the connection is still worthless. Do not separate
  them. **The Grade Service, Over HTTPS** follows them as the right way.

## The diagrams

Eight hand-written SVGs in `img/`, drawn for the league background with the
`$highlights` colours as literal hex. Render before and after an edit with
`rsvg-convert -b '#2b2b2b'`; text that overflows a box is not reported. Text
with significant spaces inside `<tspan>`s carries `xml:space="preserve"`
(`sql-boundary`, `tls13`); without it the spaces collapse.

| Diagram            | Slide                              | Draws                                                            |
| ------------------ | ---------------------------------- | ---------------------------------------------------------------- |
| `web-threat-model` | Two Victims, Drawn                 | attacker → your app → your machine, or → your users              |
| `sql-boundary`     | The Query the Database Sees        | glued, parsed, prepared — the grade query with the v1 payload    |
| `xss-kinds`        | Three Kinds                        | stored, reflected and DOM as three four-step flows               |
| `csrf`             | CSRF on the Grade Service, Drawn   | `evil.example` posts a grade change with the instructor's cookie |
| `samesite`         | SameSite, Drawn                    | when the cookie goes: same-site, then Strict / Lax / None        |
| `tls13`            | The Handshake, Briefly             | TLS 1.3, one round trip, key shares in the clear                 |
| `capital-one-ssrf` | In the Wild: Capital One, 2019     | WAF relays to the metadata service, role credentials, S3         |
| `ba-skimmer`       | In the Wild: British Airways, 2018 | edited JavaScript on BA's origin posts a card copy to BAways.com |

- `sql-boundary`, `xss-kinds` and `tls13` **replaced** text: the SQL code
  block, the three-kinds table and the handshake bullet list. Do not add that
  text back beside the drawings.
- `sql-boundary` carries the real v1 query and payload. If the handler or the
  payload changes, redraw it.
- `csrf` shows a cookie with **no SameSite**, on purpose: with the v3 cookie
  (`Lax`) the attack fails, which is what `samesite` shows next.
- `tls13` simplifies the server's encrypted flight to Certificate,
  CertificateVerify and Finished; EncryptedExtensions is left out for width.
  The "one round trip" bracket ends when the server's flight arrives — the
  client's Finished and its first request then go together.
- Widths: `xss-kinds` and `tls13` at `820`, `samesite` at `860`, `sql-boundary`
  and `csrf` at `840`, `web-threat-model` at `860`, `capital-one-ssrf` and
  `ba-skimmer` at `800`. Each slide was checked at
  960×700 in headless Chrome.

## Real-world cases

Eight "In the Wild" slides, each directly after the concept it illustrates and
inside that section's stack. Shape: `<h2>In the Wild: Name, Year</h2>`, three to
five bullets (what happened, which mechanism failed, what would have stopped
it), an optional one-line `box`, and a `0.55em` source line. They are not
hands-on slides. Every figure was checked against the source below in October
2026; do not round a number up.

| Slide                   | After                                   | Claim                                                                                        | Source                                                            |
| ----------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| TalkTalk, 2015          | The Same Request, Prepared              | SQLi into legacy Tiscali pages; 156,959 customers, 15,656 bank details; £400,000             | UK ICO, "how the ICO investigation unfolded"                      |
| Equifax, 2017           | The Same Bug, Other Interpreters        | Struts CVE-2017-5638 (`Content-Type` → OGNL); unpatched portal; 76 days; ~148 million        | US House Oversight report (Dec 2018); OGNL detail from NVD/S2-045 |
| Optus, 2022             | Broken Access Control                   | 2018 coding error left the API's access control ineffective; ~9.5 million; "trial and error" | CSO Online on ACMA's 2024 Federal Court filing — **allegations**  |
| Capital One, 2019       | Server-Side Request Forgery             | misconfigured WAF relayed to metadata; role read S3; ~100 M US, ~6 M Canada                  | KrebsOnSecurity; Capital One's facts2019 page                     |
| Firesheep, 2010         | The Cookie Attributes                   | login over HTTPS, session cookie over HTTP; one-click hijacking on open wifi                 | Eric Butler, codebutler.com                                       |
| Samy, MySpace, 2005     | Three Kinds                             | stored XSS worm; 1M+ users in <20 h; `java\nscript`, `eval`; read the anti-CSRF hash         | Samy Kamkar, samy.pl/myspace/tech.html                            |
| British Airways, 2018   | Stopping It (XSS)                       | contractor Citrix login, no MFA; edited JS; 15 days to BAways.com; 244,000 cards; £20 M      | UK ICO Penalty Notice, 16 Oct 2020 (Wayback copy)                 |
| Superfish, Lenovo, 2015 | The Same Handshake, Checking Turned Off | one root CA on every machine, private key recoverable; any site spoofable                    | US-CERT TA15-051A (seclists.org mirror)                           |

- **Capital One and SSRF.** Capital One and the court documents say only
  "configuration vulnerability" / misconfigured firewall. The slide says the
  WAF _relayed_ a request to the metadata service, which is Krebs's account
  and AWS's; it does not claim a named SSRF bug in Capital One's code.
- **Optus** is ACMA's allegation, and Optus disputes how sophisticated the
  attack was. The slide says "ACMA alleges". The slide does not claim
  enumerable IDs: no primary source states that.
- **BA's script was self-hosted**, so Subresource Integrity on a CDN tag is not
  the lesson; the slide sits after **Stopping It** because CSP is the layer
  that refuses the extra POST. That keeps **The Rest of the Client Side** as
  the section's last slide and its lab-wifi hand-off intact.
- **Superfish** sits after the `-k` pair, not between it: the pair is not
  separated.
- Links that refuse curl: `ico.org.uk` moved its penalty notices (the BA PDF
  404s; the Wayback copy is used), and `cisa.gov` returns 403 to every
  non-browser client, hence the seclists mirror of the same US-CERT alert.

## Check Yourself

The retrieval slide before **Where This Leaves Us**. Answers, for the
instructor:

1. `user` comes from the login, not from the request; `course` is typed by the
   student. The fix is a prepared statement: the value never reaches the
   parser, so there is nothing to escape (**The Same Request, Prepared**).
2. Yes. `HttpOnly` stops script _reading_ the cookie; the injected script
   runs on the grade service's origin, so its `POST /grade` carries the cookie
   anyway. Script on your origin is your user (**Three Kinds**).
3. `SameSite` — `Lax` or `Strict` keeps the cookie off a cross-site `POST`
   (**SameSite, Drawn**). And no `GET` may change state: `Lax` still sends the
   cookie on a clicked cross-site link, and a cookie with no attribute (Firefox,
   Safari) rides even an `<img>` (**Stopping CSRF**).
4. Nothing verified the certificate, so the encrypted channel can end at the
   attacker (**The Same Handshake, Checking Turned Off**).

## Facts worth not re-deriving

- **Broken access control is OWASP Top Ten #1 since 2021.** The fix on the slide
  is to put the ownership check in the `WHERE` clause, so that forgetting it
  returns nothing rather than returning someone else's row. That framing is the
  point of the slide.
- **Chromium treats a cookie with no `SameSite` as `Lax`. Firefox and Safari
  do not.** The deck once said Lax was the default in "current browsers",
  which was wrong; **Stopping CSRF** now says to set the attribute yourself.
  `ie-lecture` may carry the old claim — check it there too.
- **A JWT is signed, not encrypted.** The payload is readable base64url.
- **`HttpOnly` blocks script access, not the network.** This is a common
  misstatement and the slide corrects it explicitly.
- **XSS defeats CSRF tokens** — the injected script reads the token. That is why
  the XSS section comes before the CSRF section.
- The metadata address `169.254.169.254` is correct and is the SSRF example that
  matters in cloud environments.
- TLS 1.3: one round trip, legacy ciphers removed, forward secrecy mandatory.

## Cautions

- The XSS, CSRF and HTML samples contain markup and are escaped as `&lt;` /
  `&gt;`. Check that before editing — this is where the escaping bites.
- Session fixation's fix is one line and easy to lose in an edit: **issue a new
  identifier at login**. Keep it.
- One table — **The Cookie Attributes** — carries `font-size: 0.82em`. The
  three-kinds table was the second; it is now `xss-kinds.svg`.
- References for this deck are the OWASP Top Ten and the OWASP Cheat Sheet
  Series. Prefer them over blog posts.
