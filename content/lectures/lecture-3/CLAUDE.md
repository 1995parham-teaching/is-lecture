# Lecture 3 — Cryptography

The toolbox lecture, and the one the rest of the course spends. 56 slides,
5 vertical stacks, 4 hands-on sections, 3 images, 13 diagrams. Topics (TOC indices):
`0` Foundations · `1` Symmetric Encryption · `2` Hashes & MACs ·
`3` Public-Key Cryptography · `4` Signatures & PKI.

## The organising idea

Every section ends by naming what it **cannot** do, and the next section is the
answer:

| Section                  | The gap it ends on                        |
| ------------------------ | ----------------------------------------- |
| Symmetric                | how do both sides get the key?            |
| Hashes                   | a bare hash does not survive an adversary |
| Authenticated encryption | who is the key shared with?               |
| Public key               | whose public key is it?                   |
| PKI                      | and who watches the CA?                   |

Do not reorder the stacks. Each one is the previous one's unanswered question.

## The captured material

Everything in a `hands-on` section is a real run on the instructor's machine.

- **The ECB images** (`img/ecb-plain.png`, `img/ecb-encrypted.png`,
  `img/cbc-encrypted.jpg`) were generated for this deck, not downloaded. The
  source image is drawn with PIL, dumped to a P6 PPM, the header split off, and
  the **raw pixel bytes** encrypted with
  `openssl enc -aes-128-ecb -nopad` and `-aes-128-cbc -nopad` under the same
  key, then the header is pasted back on. `-nopad` is what keeps the byte count
  identical so the header still describes the data.
  - Worth mentioning in class: the ECB PNG is **26 KB** and the CBC one is
    **432 KB** from identical input. Structure compresses; noise does not. That
    is the same fact the picture shows, in one number.
  - The CBC version is stored as JPEG on purpose — it is incompressible noise,
    and nothing about the slide depends on pixel-exactness.
- **The SHA-1 collision** is the real SHAttered pair from `shattered.io`,
  downloaded and hashed locally. Same SHA-1, different SHA-256, `cmp` reports
  they differ at byte 193. The PDFs are **not** committed — 422 KB each, and
  the transcript is the whole point.
- **AES-GCM** is `crypto/aes` + `crypto/cipher` in Go, `go vet` clean. The
  tampered `Open` returns `""` and an error, which is the slide after it.
- **Ed25519** is `openssl genpkey`/`pkeyutl`. The failing verification is a real
  failure against a message with one character appended.

If you re-capture any of these, re-capture the whole block. Half-updated
transcripts are the failure mode this deck is most exposed to.

## The diagrams

Thirteen hand-written SVGs in `img/`, each on its own slide (titled "…, Drawn"
or "…, Counted") straight after the slide it draws, so no existing slide got
more crowded. In deck order:

| Diagram            | Draws                                                 |
| ------------------ | ----------------------------------------------------- |
| `symmetric`        | Alice, Bob, one K, Eve on the wire                    |
| `pairwise-keys`    | 5 people, 10 keys; the n(n−1)/2 table                 |
| `modes`            | CBC and CTR                                           |
| `nonce-reuse`      | one keystream twice → P1 ⊕ P2                         |
| `hash-properties`  | preimage / second preimage / collision, and the costs |
| `avalanche`        | the 256 output bits that differ, from the hands-on    |
| `length-extension` | resuming SHA-256 from a published `H(K ‖ m)`          |
| `ctr-bitflip`      | `pay 100` → `pay 900` without the key                 |
| `hybrid`           | public key moves the key, AES-GCM moves the data      |
| `diffie-hellman`   | toy exchange, p = 23, g = 5                           |
| `forward-secrecy`  | static RSA vs ephemeral DH against one later leak     |
| `signature`        | sign and verify                                       |
| `chain`            | root → intermediate → leaf                            |

They are drawn for the league background (light strokes, transparent fill)
with the `$highlights` colours as literal hex, because an `<img>` SVG cannot see
the deck's CSS. Check one by rendering it (`rsvg-convert -b '#2b2b2b'`) before
and after an edit; text that overflows a box is not reported anywhere.

Widths: `modes.svg` is the tallest, which is why it is at `width="700"` — any
wider and its caption runs into the bottom of the slide. `length-extension`,
`ctr-bitflip` and `diffie-hellman` are at `760` for the same reason (their
captions are two or three lines); the rest are at `820`–`860`.

Four of the diagrams carry real data. Change the input and you must recompute,
not edit, the drawing:

- **`avalanche`** is the XOR of the two SHA-256 digests on the hands-on slide
  before it: **122 of 256** bits differ. If the hands-on is re-captured, redraw
  this from the new digests.
- **`ctr-bitflip`** is one run of a small Go program — AES-128-CTR, random key
  and IV from `crypto/rand`, then `ct[4] ^= '1' ^ '9'` and decrypt. The program
  is not committed; the hex is that run's output (`dd` → `d5`). A new run gives
  different hex, so re-capture all four rows together.
- **`diffie-hellman`**: 5⁶ mod 23 = 8, 5¹⁵ mod 23 = 19, and both sides get 2.
  Check with `pow()` if you change a number.
- **`pairwise-keys`**: 2 → 1, 5 → 10, 100 → 4,950, 1,000 → 499,500. The 499,500
  also appears on the Symmetric Encryption slide; keep them equal.

The Diffie–Hellman slide is also the deck's only explanation of _how_ DH works.
Lecture 6's handshake slide says "that is Diffie–Hellman from lecture 3", so do
not drop it. Its caption — DH alone does not say who is on the other end — is
what sets up "sign the exchange" on the Forward Secrecy slide after it.

## Facts worth not re-deriving

- AES is **FIPS 197**, selected 2001, Rijndael. DES is 56-bit and was broken by
  EFF hardware in 1998. 3DES is disallowed by NIST, not merely discouraged.
- The equivalent-strength table is **NIST SP 800-57**. 128-bit security is
  AES-128 / RSA-3072 / ECC-256 — note it is 3072, not 2048; 2048 is the
  112-bit row.
- The birthday bound is why an _n_-bit hash gives only _n_/2 bits of collision
  resistance. That is stated on the properties slide and is the reason SHA-1
  fell first at collisions.
- **Length extension** applies to SHA-2 (Merkle–Damgård) and is the reason HMAC
  nests. SHA-3 and BLAKE2 are not vulnerable to it — do not generalise the claim.
- DigiNotar, 2011, is a real incident with real Iranian victims. It is in the
  deck because it is local history, and it is what motivates Certificate
  Transparency on the same slide. Keep the two together.
- NIST published the post-quantum standards **ML-KEM and ML-DSA in 2024**.
- TLS 1.3 removed static-RSA key exchange, which is what made forward secrecy
  mandatory rather than a configuration choice.

## Cautions

- The three ECB/CBC image slides are **one argument in three steps** — before,
  after, and the fix. Splitting them across a stack boundary loses it.
- `⊕`, `‖`, `→` and `≈` are literal characters in the HTML, not entities. They
  render; leave them alone rather than "fixing" them to entities.
- Six tables carry an explicit `font-size`. The key-size table has four columns
  and is the one most likely to overflow if a row grows.
- The closing slide is the deck's take-away and is deliberately a list of rules
  rather than a summary of the maths. Students will not remember GCM's internals;
  they can remember "never reuse a nonce".
