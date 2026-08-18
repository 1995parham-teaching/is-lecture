# Lecture 3 — Cryptography

The toolbox lecture, and the one the rest of the course spends. 43 slides,
5 vertical stacks, 4 hands-on sections, 3 images. Topics (TOC indices):
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
