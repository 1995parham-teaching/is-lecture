# Lecture 6 — Web Security

37 slides, 5 vertical stacks, 3 hands-on sections. Topics (TOC indices):
`0` The Web Threat Model · `1` Server-Side Attacks · `2` Client-Side Attacks ·
`3` Sessions & Cookies · `4` TLS and HTTPS.

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

## The captured material

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
  them.

## Facts worth not re-deriving

- **Broken access control is OWASP Top Ten #1 since 2021.** The fix on the slide
  is to put the ownership check in the `WHERE` clause, so that forgetting it
  returns nothing rather than returning someone else's row. That framing is the
  point of the slide.
- **`SameSite=Lax` is the default** in current Chrome, Firefox and Edge.
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
- Two tables carry `font-size: 0.82em`.
- References for this deck are the OWASP Top Ten and the OWASP Cheat Sheet
  Series. Prefer them over blog posts.
