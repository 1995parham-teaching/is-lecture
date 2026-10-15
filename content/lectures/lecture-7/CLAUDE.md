# Lecture 7 — Network & Transport Security

The closing deck, and the one that also closes the course. 42 slides,
5 vertical stacks, 4 hands-on sections, 7 diagrams. Topics (TOC indices):
`0` Network Threats · `1` Security Zoning · `2` Edge Security ·
`3` Secure Access & VPN · `4` VLAN & Segmentation.

## The organising idea

The opening slide states the premise the whole deck rests on: TCP/IP was built
for a network where every operator was known, so authentication was out of
scope rather than forgotten. Every control in the deck is then presented as a
patch over that decision — which is why the threats come first and the
mechanisms second.

The first slide opens by recalling where lecture 6 stopped (TLS protects the
channel; this lecture is the channel). Keep that sentence in step with
lecture 6's closing slide.

The final slide closes the **course**, not just the lecture, and repeats
lecture 1's sentence with a clause added: _and know what it does not do_. That
is the arc of all seven decks; do not replace it with a summary of VLANs.

## The grade service, zoned

The grade service is the course's one program, and this deck uses its **last**
form: lecture 6's **v4** — SQLite through `modernc.org/sqlite`, a
`__Host-session` cookie from `POST /login`, and `ListenAndServeTLS` on `:8080`.
`lecture-6/CLAUDE.md` has the full source; it is copied here unchanged.

**The Grade Service, Zoned** (`img/zones.svg`) puts it in the DMZ at
`192.0.2.10:8080`. Its database is a **SQLite file beside the binary**, so it
needs **no** inward flow — and the slide says the price of that: whoever takes
this host takes every grade. A database _server_ would sit in Restricted,
reached by one named flow. That is why the Classic Layout rule reads
"initiates nothing inward, except flows written down by name". Do not move
`grades.db` into the Restricted zone on the slide without changing the
program; SQLite is a file, not a network service.

## The hands-on slides, and why they are honest

Earlier versions had none, on purpose: the attacks that matter are attacks on
a shared network, and the instructor's own `arp -a` / `netstat -rn` output
contains the MACs and private addresses of a real LAN. Both reasons still
stand. The four hands-on slides avoid them by running **inside one disposable
Docker container started with `--network none` and `--privileged`**, with the
topology built from Linux network namespaces. Nothing leaves the container,
there is no shared network to attack, and every address is from the
documentation ranges (RFC 5737: `192.0.2.0/24`, `198.51.100.0/24`,
`203.0.113.0/24`). The MACs are set by hand to locally administered
`02:00:00:00:00:xx`. **Keep it that way**: never replace a capture with one
from a real interface.

Captured on 14 October 2026, Docker 29.8.2 on macOS (Linux VM, arm64), Debian
bookworm userland. The image:

```dockerfile
FROM debian:bookworm
RUN apt-get update && apt-get install -y --no-install-recommends iproute2 nftables iputils-arping iputils-ping netcat-openbsd curl socat openssl sqlite3 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY v4/grades /usr/local/bin/grades
```

`grades` is lecture 6's v4 source, unchanged, built with
`GOOS=linux GOARCH=arm64 CGO_ENABLED=0` (the driver is pure Go). It still
binds `localhost:8080`, so `socat` publishes it on the DMZ address as a plain
TCP relay — TLS passes through it end to end. Do not edit the Go source to
change the bind address; it is the same program as lecture 6's. The one thing
that differs is the **certificate**: lecture 6's is for `DNS:localhost`; this
one is generated in the container for `IP:192.0.2.10`, so `curl --cacert`
verifies it with no `-k` and no `--resolve`.

Each capture is a fresh `docker run --rm --privileged --network none` that
runs the setup script and then the commands through a printer: `$ ` plus the
line as typed (a trailing `\` continues it on an indented line), then the
command under `script -qec … /dev/null` so curl sees a terminal and prints no
progress meter — the same output as an interactive shell. The ARP capture
predates that printer and used plain `sh -c`; nothing in it depends on a
terminal. `internet`, `dmz`, `lan`, `fw`, `victim` and `attacker` are wrappers
for `ip netns exec <name>`; the slide says so.

### ARP spoofing, in one broadcast

```sh
#!/bin/sh
set -e
ip netns add switch; ip -n switch link add br0 type bridge; ip -n switch link set br0 up
i=0
for h in victim:192.0.2.20:02:00:00:00:00:20 gateway:192.0.2.1:02:00:00:00:00:01 attacker:192.0.2.66:02:00:00:00:00:66; do
  n=${h%%:*}; rest=${h#*:}; a=${rest%%:*}; mac=${rest#*:}
  ip netns add $n; ip -n $n link set lo up
  ip link add p$n netns switch type veth peer name eth0 netns $n
  ip -n $n link set eth0 address $mac
  ip -n $n addr add $a/24 dev eth0; ip -n $n link set eth0 up
  ip -n switch link set p$n master br0; ip -n switch link set p$n up
  printf '#!/bin/sh\nexec ip netns exec %s "$@"\n' $n > /usr/local/bin/$n; chmod +x /usr/local/bin/$n
done
sleep 2
```

Then `ping`, `ip neigh`, `ip addr add 192.0.2.1/32` on the attacker and one
`arping -U`. The victim's entry flips from `…:01 REACHABLE` to `…:66 STALE`,
and the second ping still succeeds — the attacker now owns the address. The
`rtt` line is timing and will differ on re-capture.

### The firewall: rule base, five connections, one temporary rule

```sh
#!/bin/sh
set -e
for n in internet fw dmz lan; do ip netns add $n; ip -n $n link set lo up; done
# links: internet<->fw, dmz<->fw, lan<->fw
ip link add out netns fw type veth peer name eth0 netns internet
ip link add dmz netns fw type veth peer name eth0 netns dmz
ip link add lan netns fw type veth peer name eth0 netns lan
ip -n fw addr add 198.51.100.1/24 dev out
ip -n fw addr add 192.0.2.1/24 dev dmz
ip -n fw addr add 203.0.113.1/24 dev lan
ip -n internet addr add 198.51.100.20/24 dev eth0
ip -n dmz addr add 192.0.2.10/24 dev eth0
ip -n lan addr add 203.0.113.10/24 dev eth0
for d in out dmz lan; do ip -n fw link set $d up; done
for n in internet dmz lan; do ip -n $n link set eth0 up; done
ip -n internet route add default via 198.51.100.1
ip -n dmz route add default via 192.0.2.1
ip -n lan route add default via 203.0.113.1
ip netns exec fw sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
for n in internet dmz lan fw; do printf '#!/bin/sh\nexec ip netns exec %s "$@"\n' $n > /usr/local/bin/$n; chmod +x /usr/local/bin/$n; done
# the grade service (lecture 6, v4) in the DMZ: its SQLite file and a
# certificate for the DMZ address sit next to it; socat publishes it there
mkdir -p /srv && cd /srv && sqlite3 grades.db < /lab/schema.sql
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -nodes -keyout key.pem -out cert.pem -days 1 \
  -subj /CN=192.0.2.10 -addext subjectAltName=IP:192.0.2.10 2>/dev/null
cp cert.pem /lab/cert.pem
ip netns exec dmz sh -c 'cd /srv && grades 2>server.log &'
ip netns exec dmz sh -c 'socat TCP-LISTEN:8080,bind=192.0.2.10,fork,reuseaddr TCP:127.0.0.1:8080 &'
cd /lab
ip netns exec fw nft -f fw.nft
# a database in the internal zone, and an attacker's listener on the internet
ip netns exec lan sh -c 'while true; do nc -l 203.0.113.10 5432 </dev/null; done >/dev/null 2>&1 &'
ip netns exec internet sh -c 'while true; do nc -l 198.51.100.20 443 </dev/null; done >/dev/null 2>&1 &'
sleep 1
```

`schema.sql` is lecture 6's. `fw.nft` is loaded by the setup script, not in the
transcripts, and is shown verbatim on **The Rule Base, Written**:

```
table inet fw {
  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
    # the internet may reach the grade service, and nothing else
    iifname "out" ip daddr 192.0.2.10 tcp dport 8080 counter accept
    # the internal zone may reach the DMZ
    iifname "lan" oifname "dmz" counter accept
    # the DMZ initiates nothing: not inward, not outward
    iifname "dmz" counter drop
    counter drop
  }
}
```

**Five Connections, Five Answers** and **One Temporary Rule** are two separate
container runs, so the counters start from zero in each. Facts to keep in step:

- **Five Connections** is a student (`4001` / `tulip-river-42`, lecture 6's
  password) logging in and reading `4001 IS 12`, then three refused
  connections. The `out` rule counts **2** packets for two HTTPS requests:
  only each SYN; the rest ride `ct state established`.
- The DMZ drop rule counts **4** packets for two `nc -w 2` attempts — a SYN
  and one retransmission each. The slide states that arithmetic.
- In the temporary-rule run, `nft insert` puts the rule at the **top** of the
  chain, above `ct state`, so it matches every packet the DMZ host sends on that
  connection, not only the first (4 packets, 216 bytes). The deny rule shows
  **0** — the shadow, made visible. `img/firewall-rules.svg` draws exactly this
  rule list in this order; change one, change both.
- The listener on `203.0.113.10:5432` is `nc -l`, standing in for a database.
  The slide says so.

If you re-capture, re-capture the whole block, and re-check the
"4 packets" sentence against the new counters.

### What is still not demonstrated

- **DNS amplification** was attempted and abandoned honestly — the local
  resolver strips DNSSEC records, so every response came back around 120
  bytes and there was no amplification to show. The slide cites
  **US-CERT TA14-013A** for the published factors. Do not replace that
  citation with numbers from a run that did not happen.
- **VLAN double tagging** needs two 802.1Q switches with a native VLAN on a
  trunk; a Linux bridge in a namespace does not reproduce that faithfully. It
  is drawn (`img/vlan-hopping.svg`), not captured.

## The diagrams

Seven hand-written SVGs in `img/`, each on its own "…, Drawn" slide straight
after the slide it draws (except `zones.svg`, which is **The Grade Service,
Zoned**). Same conventions as lecture 3: league background, light strokes,
`$highlights` hex literals; render with `rsvg-convert -b '#2b2b2b'` before and
after any edit.

| Diagram          | Draws                                                    |
| ---------------- | -------------------------------------------------------- |
| `arp-spoofing`   | the victim's ARP cache before and after one forged reply |
| `zones`          | five zones, the grade service in the DMZ, allowed/denied |
| `east-west`      | flat network vs microsegmented, lateral movement         |
| `zero-trust`     | one check at the door vs PDP/PEP per request             |
| `firewall-rules` | first match wins; the temporary rule shadows rule 5      |
| `ipsec-modes`    | ESP transport vs tunnel: what is encrypted/authenticated |
| `vlan-hopping`   | double tagging across a native-VLAN trunk, one-way       |

`arp-spoofing` uses the hands-on's addresses and MAC suffixes, and
`firewall-rules` uses the hands-on's rule base; keep them in step.
`zero-trust` uses NIST SP 800-207's **decision point / enforcement point**
names, and the slide links them to lecture 4's `allow(s, a, o)` — if lecture 4
drops that function, drop the sentence.

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
- The grade service is **lecture 6's v4** (lecture 1's program, grown); if
  lecture 6 changes it, re-capture both firewall blocks. The zero-trust slide
  names **lecture 4's** `allow(s, a, o)`.

## The learning path

- **Edge Security runs concept → demo, twice.** What a Firewall Is · Three
  Generations · Egress Filtering · The Rule Base, Written · Five Connections
  (the egress counter) · Rules, and How They Go Wrong · First Match Wins, Drawn
  · One Temporary Rule (the shadow) · Where Detection Sits. The Rule Base says
  "the egress filter from the slide before", so Egress Filtering must stay
  directly before it.
- **Terms are glossed at first use**: ARP (on Active: Lying), SYN flood, WAF,
  IDS/IPS, NAC, IKE, and the native VLAN (on the VLAN slide, before VLAN
  Hopping uses it). Computer Networks is the prerequisite, so there is no
  refresher slide, only these one-line glosses.
- **Check Yourself** sits before Putting the Course Together. Answers:
  1. Nothing fails, so nothing alerts: the attacker is answering for the
     gateway and can read or change every packet while the victim sees a
     working network.
  2. Allowed: internet → DMZ on 8080 only, or internal → DMZ. Denied: DMZ →
     internet, DMZ → internal (the 4-packet counter). Taking the host gives
     every grade, because `grades.db` is a file on it.
  3. First match wins: the temporary accept sits above it, so the deny is
     never reached. Its counter stays at 0 — review the counters, not only
     the text.
  4. Confidentiality, integrity and peer authentication of the link. It does
     nothing about the endpoint: the malware is now inside the network.

## Cautions

- Four tables carry an explicit `font-size` of `0.8em`–`0.82em`.
- Diagram widths: `zones` 700, `ipsec-modes` 680, `zero-trust` 760, the rest 860. The three narrower ones carry a paragraph under the image; wider, and
  the paragraph runs into the navigation arrows.
- **Five Connections, Five Answers** is the tallest code block in the deck
  (16 lines, the longest at 84 characters). It fits at 960×700; adding a line
  to it will not.
- "A VLAN is segmentation, not encryption" is the take-away of the last stack.
  Keep it in the `box`.
