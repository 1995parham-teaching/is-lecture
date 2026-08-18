# Lecture 5 — System & Software Security

39 slides, 6 vertical stacks, 3 hands-on sections. Topics (TOC indices):
`0` Operating System Security · `1` File System Security · `2` Malware ·
`3` Software Security · `4` Host Defences · `5` Virtualization.

## The organising idea

The overflow demonstration is the hinge of the deck, and it is deliberately
framed with lecture 6's sentence — _data crossed into control_. A buffer
overflowing into an adjacent flag is the same failure as SQL injection, one
layer down. The closing slide says so explicitly so that lecture 6 lands as a
continuation rather than a new topic.

## The overflow capture

`login.c` is a **struct**, not two locals on the stack, and that is not an
accident. The obvious version — `char name[16]; int admin;` as locals — does
**not** reproduce on arm64 macOS, because the compiler reorders locals to put
arrays above scalars precisely to make this harder. A struct's member order is
guaranteed by the C standard, so the demonstration is deterministic on every
target.

Three runs, all real:

| Input             | Result                |
| ----------------- | --------------------- |
| `parham`          | `is_admin=0`          |
| 20 × `A`          | `is_admin=1094795585` |
| 16 × `A` + `\x01` | `is_admin=1`          |

`1094795585` is `0x41414141`. The second run is the accident; the third is the
attack, and the pair is the point — do not drop either.

The mitigation slide is the **same source file** compiled three ways. The exit
codes are real: `133` is SIGTRAP from `__strcpy_chk` (Apple's `_FORTIFY_SOURCE`
is on by default, which is why the unmitigated build needs
`-D_FORTIFY_SOURCE=0 -fno-stack-protector`), and `134` is SIGABRT from the stack
protector. There is no `*** stack smashing detected ***` message on macOS —
that string is glibc's. Do not add it to the transcript.

## Other captured material

- The **base-rate table** is a real computation, not a quoted example.
  1,000,000 events, 100 attacks, 99% detection, 0.1% false alarm → 9.01%
  precision. If you change any input, re-run it; the three derived rows will not
  survive editing by hand.

## Facts worth not re-deriving

- **Morris 1988 · Slammer 2003 · WannaCry 2017.** Slammer's doubling time was
  about 8.5 seconds and it fitted in a single UDP packet. WannaCry's SMB patch
  (MS17-010) had been out for about two months.
- The **~70% memory-safety** figure is reported independently by Microsoft (MSRC)
  and Google (Chromium). Attributing it to both is what makes it credible —
  keep both names.
- **NX is what produced return-oriented programming.** The mitigation slide says
  each defence raises cost rather than fixing anything, and ROP is the evidence.
- Type 1 vs Type 2 hypervisor: the TCB argument is the reason to care, not
  performance.
- **Containers share one kernel.** The deck states this as the difference that
  matters and lists the hardening flags; `--privileged` is called out because it
  discards all of them at once.
- Ransomware targeting backups is what makes the lecture 2 "offline or
  immutable" rule a technical control rather than an aspiration. The two decks
  cross-reference here on purpose.

## Cautions

- The C snippets contain no `<` or `>`; if you add any, they need escaping.
- The `printf '\x01'` line is inside a `lang-text` block and shows a shell
  substitution. It is real input, not pseudo-code.
- Four tables carry `font-size: 0.8em`.
- Do not turn the overflow section into an exploitation tutorial. It stops at
  "the return address is the interesting neighbour" on purpose — the lab rule
  from lecture 1 applies to the deck as much as to the students.
