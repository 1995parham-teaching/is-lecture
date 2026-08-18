# Lecture 4 — Access Control

37 slides, 5 vertical stacks, 4 hands-on sections. Topics (TOC indices):
`0` The Four A's · `1` Access Control Models · `2` Authentication ·
`3` Passwords · `4` Biometrics.

## The organising idea

One function, `allow(subject, action, object)`, stated on the first slide and
answered from four directions. The models section is about the
**authorization** half; the last three sections are about the
**authentication** half. The deck keeps them apart on purpose, because
conflating them is the bug the lecture is trying to prevent.

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
  else.

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
- Five tables carry an explicit `font-size` of `0.8em`–`0.85em`.
