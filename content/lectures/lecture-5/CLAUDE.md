# Lecture 5 — System & Software Security

56 slides, 6 vertical stacks, 6 hands-on sections, 7 diagrams, 7 real-world incident slides. Topics (TOC indices):
`0` Operating System Security · `1` File System Security · `2` Malware ·
`3` Software Security · `4` Host Defences · `5` Virtualization.

## The organising idea

The overflow demonstration is the hinge of the deck, and it is deliberately
framed with lecture 6's sentence — _data crossed into control_. A buffer
overflowing into an adjacent flag is the same failure as SQL injection, one
layer down. The closing slide says so explicitly so that lecture 6 lands as a
continuation rather than a new topic.

## The learning path

The deck is read by a first-year student, so it keeps four habits:

- **It opens where lecture 4 stopped.** The first slide restates
  `allow(s, a, o)` with a process as the subject and the kernel as the
  decider. Keep that sentence if lecture 4's closing line changes.
- **Every section ends with the question the next one answers**, in one plain
  sentence under the last slide's content:

  | Section ends on           | Bridge sentence                                        |
  | ------------------------- | ------------------------------------------------------ |
  | Hardening (box)           | which account runs `./grades`? Mostly, files           |
  | Append-Only, and Who Can… | every check trusts the program that runs as you        |
  | Detecting It              | how does malware arrive? A bug in a trusted program    |
  | The Actual Fix            | bugs still ship, so the host must notice               |
  | In the Wild: Target's…    | detection will miss, so limit what falls with the host |

  If you reorder the sections, rewrite these sentences; they name the next
  topic.

- **Terms are defined where they are first used**, or credited to the deck that
  defined them: setuid and umask are lecture 4's, complete mediation and
  "who guards the digest?" are lecture 1's. FORTIFY and the canary are defined
  in one line on the hands-on that first uses them, before `mitigations.svg`
  draws the whole frame. Namespaces, cgroups, capabilities and seccomp each
  get a few-word gloss on **Containers Are Not Small VMs**.
- One name per concept: the HIDS slide's **misuse** and **anomaly** detection
  point back to the Malware section's **signature** and **behavioural**
  detection, and the deck says "false alarm" throughout, never "false
  positive".

### Check Yourself — answers

The retrieval slide before **Where This Leaves Us** has four questions. Answers
for the instructor:

1. `uappnd` is a user flag, so the owner — the service's own account — can
   lift it, as the last line of the hands-on shows. What protects the log is a
   different owner: `sappnd` set by root (`chattr +a` on Linux), or shipping
   each line to another host as it is written.
2. `name` is 16 bytes and `strcpy` does not check; the 17th–20th `A`s land in
   `is_admin`, and `0x41414141` is 1094795585. The canary's crash (exit 134)
   turns a silent privilege escalation into a denial of service, which is
   visible and far less harmful.
3. Attacks were rare: 100 in 1,000,000 events, so 0.1% false alarms on the
   other 999,900 gives ≈ 1,000 false against 99 true. Fixes: raise the base
   rate (watch a narrower population), correlate two signals, or drive the
   false-alarm rate down.
4. Containers: only the shared kernel (namespaces and cgroups) — one kernel
   bug crosses it. Type 1 VMs: the hypervisor, a much smaller TCB, with each
   guest's own kernel behind it.

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

## The grade service, on the host

Lecture 1's grade service (`lecture-1/CLAUDE.md` has the source) closes the
File System section in two `hands-on` slides: **The Service Can Rewrite Its
Own Log** and **Append-Only, and Who Can Lift It**. They are the deck's answer
to lecture 2's "The Day It Happens": that slide reads a log line, and these
show that the account that writes the line can also delete it.

Both are real runs on 14 October 2026, macOS, the **first** (vulnerable)
version of `main.go` started as `./grades 2>> server.log`, with
`curl -s` so no progress meter appears. They are two separate runs, which is
why the timestamps differ (`09:50:41`, `09:50:50`).

- `sed -i ''` deletes the line by writing a new file and renaming it over the
  old one. Under `uappnd` the rename fails, and the exact text is
  `sed: rename(): Operation not permitted`. Do not "correct" it to a write
  error.
- `chflags nouappnd` succeeding with no output **is the point** of the last
  line: `u` flags belong to the owner. The slide names the real wall —
  `sappnd` set by root, `chattr +a` on Linux, or shipping the log off the
  host — but **those were not captured** (no root on the capture machine), so
  they are stated, not shown. Do not add a transcript for them without a run.
- The closing sentence, _the process that writes the record must not own
  it_, is Hardening's "one service per account" made concrete.

## Containers, captured

**Two Distributions, One Kernel** is a real run on Docker Desktop 29.8.2:
`debian:bookworm` and `quay.io/centos/centos:stream9` both report
`Linux 7.0.14-linuxkit`, and the host reports `Darwin 27.0.0`. On a Mac the
containers live inside Docker Desktop's Linux VM — the slide says so, because
it is the drawing's hypervisor wall and kernel wall in one machine. `/dev` has
15 entries normally and 169 with `--privileged`. A different Docker Desktop
version gives a different kernel string and device count; re-capture the
whole block rather than edit one line.

## The diagrams

Seven hand-written SVGs in `img/`, styled like lecture 3's (league background,
`$highlights` as literal hex). Render with `rsvg-convert -b '#2b2b2b'` before
and after an edit.

| Diagram       | Slide                  | Draws                                                  |
| ------------- | ---------------------- | ------------------------------------------------------ |
| `syscall`     | The Boundary, Drawn    | user/kernel mode, the reference monitor, no side door  |
| `overflow`    | The Overflow, Drawn    | the 20 bytes of `struct session` for all three runs    |
| `mitigations` | Layers of Mitigation   | one frame, and which mitigation guards which part      |
| `base-rate`   | The Base Rate, Counted | the 1,000,000-event tree and the alarms to scale       |
| `isolation`   | Isolation, Drawn       | Type 2, Type 1, containers: what separates two tenants |
| `xz`          | The xz Backdoor, Drawn | maintainer → tarball → liblzma → libsystemd → sshd     |
| `heartbleed`  | Heartbleed, Drawn      | a 1-byte heartbeat claiming 65535, and the reply       |

- **`overflow` carries real data.** `parham` is `70 61 72 68 61 6d 00`; the
  struct is little-endian; `strcpy` writes a terminating `00`, which in the
  20-byte run lands one byte past the struct. The dots are uninitialised
  bytes (`name` is never cleared). If the inputs change, recompute the bytes.
- **`mitigations` replaced the five-row table** that used to be on that slide;
  every row (canary, NX, ASLR, FORTIFY, CFI) is a label on the drawing, and the
  two exit codes are the ones from the hands-on before it. It says
  "simplified" on purpose: real frame layout depends on compiler and CPU.
- **`base-rate`** uses the hands-on numbers. 0.1% of 999,900 is 999.9, which
  the table rounds to 1,000 — the drawing writes `≈ 1,000`. The green bar is
  72 of 800 px, i.e. 99/1,099. Change the inputs and you redraw this too.
- `namespaces · cgroups` is drawn as its own layer in `isolation` for
  legibility; they are part of the shared kernel, and the caption says the
  kernel bug crosses that wall.

## Real-world cases

Each `In the Wild` slide sits directly after the concept it proves, in the same
vertical stack. None is `hands-on`: they are cited, not captured. Every number
below was checked against the source on 24 October 2026; if you edit a slide,
re-check against the same source rather than memory.

| Slide (placed after)                              | Claim on the slide                                                                                                                                                     | Source                                                                                                                 |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| CrowdStrike Falcon, 2024 (Trusted Computing Base) | ~8.5 M Windows machines; kernel driver; 21 fields expected, 20 given; out-of-bounds read; fixes: bounds check + staged rollout                                         | CrowdStrike Channel File 291 RCA (PDF); Microsoft blog 20 July 2024 (8.5 M)                                            |
| The xz Backdoor, 2024 (+ Drawn)                   | CVE-2024-3094, 5.6.0/5.6.1; sshd → libsystemd → liblzma via distro patch; ≈ 0.8 s vs ≈ 0.3 s login                                                                     | Andres Freund, oss-security 2024/03/29/4; Russ Cox, research.swtch.com/xz-timeline (2021 first patch, 2022 maintainer) |
| Stuxnet, 2010 (Why the Worm Is Different)         | ~60% of infected hosts in Iran; LNK auto-execution over USB; four unpatched Windows bugs; two stolen signing certificates (Realtek, JMicron); Windows and PLC rootkits | Symantec W32.Stuxnet Dossier v1.4 (docs.broadcom.com)                                                                  |
| NotPetya and Maersk, 2017 (Ransomware…)           | M.E.Doc update; EternalBlue + Mimikatz; wiper; > $10 bn (White House); 4,000 servers, 45,000 PCs; ~150 DCs; Ghana DC offline from a blackout                           | Andy Greenberg, WIRED, Aug 2018                                                                                        |
| Heartbleed, 2014 (Layers of Mitigation, + Drawn)  | CVE-2014-0160; missing bounds check; up to 64 KB per heartbeat; no trace in logs; found by Neel Mehta and Codenomicon                                                  | OpenSSL advisory 7 Apr 2014; heartbleed.com                                                                            |
| Target's Alarms, 2013 (The Base-Rate Fallacy)     | 40 M cards, up to 70 M records; FireEye alerted on each install; auto-delete off; alert named `malware.binary`; report suggests staff saw it as a false alarm          | US Senate Commerce Committee staff report, 25 Mar 2014                                                                 |
| runc, 2019 (Containers Are Not Small VMs)         | CVE-2019-5736 overwrites host runc → root on host; needs root in the container; non-root, user namespaces, RHEL SELinux block it                                       | Aleksa Sarai, oss-security 2019/02/11/2; Kubernetes blog 11 Feb 2019                                                   |

- **Target** is only the alert angle. Lecture 7 owns the HVAC-vendor pivot into
  a flat network; do not add it here. The section's bridge sentence moved from
  **The Base-Rate Fallacy** onto this slide, because it is now the section's
  last.
- The Senate PDF on commerce.senate.gov returns 403 to curl and headless Chrome (Akamai bot block);
  the text was checked against an identical copy at covert.io. Microsoft's
  blog also 403s to curl and loads in a browser.
- **Heartbleed** sits before **The Actual Fix** on purpose: it is the case for
  "never trust a length that came with the data", and it shows that the
  mitigations on the slide before it only see writes. The CrowdStrike bug is
  the same class (an out-of-bounds read in the kernel); the two slides may be
  cross-referenced in class.
- **Stuxnet** is stated in the dossier's terms (targets, propagation,
  certificates). Keep attribution out of the slide; the source does not make it.

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
- One table (Malware, Sorted) carries `font-size: 0.8em`.
- Do not turn the overflow section into an exploitation tutorial. It stops at
  "the return address is the interesting neighbour" on purpose — the lab rule
  from lecture 1 applies to the deck as much as to the students.
