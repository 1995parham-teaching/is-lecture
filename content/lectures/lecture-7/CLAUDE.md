# Lecture 7 — Network & Transport Security

The closing deck, and the one that also closes the course. 30 slides,
5 vertical stacks, no hands-on sections. Topics (TOC indices):
`0` Network Threats · `1` Security Zoning · `2` Edge Security ·
`3` Secure Access & VPN · `4` VLAN & Segmentation.

## The organising idea

The opening slide states the premise the whole deck rests on: TCP/IP was built
for a network where every operator was known, so authentication was out of
scope rather than forgotten. Every control in the deck is then presented as a
patch over that decision — which is why the threats come first and the
mechanisms second.

The final slide closes the **course**, not just the lecture, and repeats
lecture 1's sentence with a clause added: _and know what it does not do_. That
is the arc of all seven decks; do not replace it with a summary of VLANs.

## Why there are no hands-on slides here

Deliberate, and worth knowing before someone tries to add one.

- The demonstrations that would matter — ARP spoofing, VLAN hopping, a SYN
  flood — are **attacks on a shared network**. Lecture 1's lab rule forbids
  running them anywhere the instructor does not own, and a lecture hall network
  is precisely where not to.
- `arp -a` and `netstat -rn` on the instructor's machine were captured while
  writing this deck and **deliberately left out**: the real output contains the
  MAC addresses and private addresses of the instructor's own LAN and its
  neighbours, and this repository is public.
- A DNS amplification measurement was attempted and abandoned honestly — the
  local resolver strips DNSSEC records, so every response came back around 120
  bytes and there was no amplification to show. The slide cites
  **US-CERT TA14-013A** for the published factors instead of inventing a
  capture. Do not replace that citation with numbers from a run that did not
  happen.

If you do add a demonstration, build it from two VMs you own.

## Facts worth not re-deriving

- **802.1Q** adds a 4-byte tag with a **12-bit** VLAN ID, giving 4094 usable
  VLANs (0 and 4095 are reserved).
- **Double tagging is one-way.** The attacker can send into the target VLAN and
  receives nothing back. The slide says so, because "you get a shell on the
  other VLAN" is the common overstatement.
- Double tagging depends on the **native VLAN being carried untagged**. That is
  the mechanism, and the fix follows from it.
- **AH vs ESP**: AH gives integrity and authentication with no encryption, and
  is rarely deployed. ESP is the one in use.
- **Transport vs tunnel mode**: tunnel wraps the whole packet, hiding the inner
  addresses. That is the difference that matters, not the overhead.
- **NIST SP 800-207** is the zero-trust reference architecture.
- **802.1X** is supplicant → authenticator → authentication server, and the port
  passes nothing but the authentication exchange until it succeeds.
- WireGuard's non-negotiable cipher suite is a **security** property — no
  negotiation means no downgrade — not merely a simplification.

## Cross-lecture dependencies

- HSTS is defined in **lecture 6** and re-used here as the answer to TLS
  stripping. The two must stay consistent.
- The egress-filtering slide leans on **lecture 5's base-rate fallacy**: a rare
  event is a high-quality signal. That link is the reason the slide is phrased
  the way it is.
- Zoning is **lecture 2's** boundary drawing made concrete, and the deck says so.

## Cautions

- Four tables carry an explicit `font-size` of `0.8em`–`0.82em`.
- "A VLAN is segmentation, not encryption" is the take-away of the last stack.
  Keep it in the `box`.
