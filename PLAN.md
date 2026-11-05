# localns: link-local name resolution

Status: **candidate, not started.** No shell exists; section "Getting a shell" says what
creating one would involve. Tier 5 in `D:\code\PORTFOLIO.md`.

## The problem

Resolving a bare name on a local link means choosing between protocols that a different
library owns and a different trust model governs:

- unicast DNS, authenticated only if DNSSEC validates, cached by a resolver you did not
  choose;
- mDNS, which is conflict-detection based and answered by whoever is on the link;
- LLMNR, answered by whoever is listening, from any host, unauthenticated;
- NBNS (NetBIOS name service over UDP 137 / TCP 137), answered by whoever is listening,
  unauthenticated, still present in Windows environments.

Today an application that wants "give me the address of `printer.local` or `nas` or
`fileserver`, and tell me how much to believe the answer" has to pick a library per
protocol and lose the distinction between them. That distinction is the product: a
per-answer **trust label** stating which protocol replied and what it authenticated.
Nothing in Python models it, and it is exactly the thing that matters for discovery,
onboarding, migration audits and security review, because LLMNR and NBNS poisoning is a
known attack against precisely this ambiguity.

## Landscape, with dates

PyPI queried 2026-11-05 on both registry signals. `netbios`, `nbns`, `llmnr`, `nbstat`,
`nbtscan`, `netbios-ns`, `localns` and `linkname` are **absent**. `nbt` and `python-nbt`
exist but are Minecraft *Named Binary Tag* files, and `wins` is an economics package:
false friends, not competitors.

**Absent is not the same as safe.** `names.py --index` against the 931,439-name snapshot
finds **five live names one keystroke from `localns`**: `localdns`, `locals`, `locales`,
`localfs`, `localis`. `localdns` is the one that matters - a live Zeroconf-based DNS
registration library doing an adjacent job, so the confusion is semantic rather than a typo.
`derkit` was rejected from this portfolio for seven such neighbours; five is the same
failure shape, and it becomes expensive the moment a release exists rather than now, while
it costs one directory rename or a decision to fold this into `namespy` and inherit its name.

| Neighbour | State | Why it is not this |
|---|---|---|
| `zeroconf` | Live, maintained | mDNS/DNS-SD responder and browser for one protocol; no LLMNR, no NBNS, no trust labels |
| `impacket` | Live (0.13.1, 2026-05-19) | Monolithic synchronous offensive toolkit; not an importable resolver |
| `scapy` | Live | Packet layers you assemble yourself; no resolution semantics |

## Specification spine

Statuses read from the RFC Editor record on 2026-11-05.

| Document | Status | Published | Note |
|---|---|---|---|
| RFC 6762 (mDNS) | Proposed Standard | 2013-02 | no verified errata |
| RFC 6763 (DNS-SD) | Proposed Standard | 2013-02 | no verified errata |
| RFC 4795 (LLMNR) | **Informational** | 2007-01 | **six verified errata**; not a standard, so quote no MUST from it |
| RFC 1001 (NBNS concepts) | Internet Standard, STD 19 | 1987-03 | one verified erratum |
| RFC 1002 (NBNS detailed) | Internet Standard, STD 19 | 1987-03 | one erratum on file |

RFC 9250 (DNS over dedicated QUIC) is listed in `namespy`'s own gaps and is a candidate
obligation for whichever project takes transport diversity further.

## Obligations if this is built

| Id | Spec | Status | Where | Evidence |
|---|---|---|---|---|
| `ln.trust-label` | Model, all protocols | [ ] | `trust` | `test_answer_carries_its_protocol` |
| `ln.mdns-resolve` | RFC 6762 sections 5, 7 | [ ] | `mdns` | `test_query_on_link` |
| `ln.llmnr-resolve` | RFC 4795 sections 2.1.1, 3 | [ ] | `llmnr` | `test_llmnr_answer_is_labelled_unauthenticated` |
| `ln.nbns-resolve` | RFC 1002 sections 14.2, 4.2.1 | [ ] | `nbns` | `test_name_trailing_bytes_and_node_type` |
| `ln.ordering` | - | [ ] | `resolver` | `test_lookup_order_and_stop_conditions` |
| `ln.no-cross-cache` | RFC 4795 section 4 | [ ] | `cache` | `test_llmnr_and_dns_caches_are_separate` |
| `ln.timeout-ethics` | RFC 4795 section 3.2 | [ ] | `resolver` | `test_no_burst_on_the_link` |
| `ln.announce` | RFC 6762 section 8.3, RFC 1002 section 4.2.1 | [ ] | `announce` | `test_conflict_detection_before_claim` |

## Non-goals

- **No WINS server or client.** WINS is retired and nobody deploys it; NBNS covers the
  residual need and covers it as an audit surface rather than as infrastructure.
- **No responder reimplementation** of mDNS/DNS-SD: `zeroconf` does that and does it well.
- **No SMB, no NetBIOS session service.** Name resolution only; the session layer is
  `impacket`'s problem.

## Getting a shell

Three open decisions before that is worth doing, all recorded in `PORTFOLIO.md` section 9:
whether this is its own project or a module of `namespy`, whether `namespy`'s wire
codec is reused or imported as a dependency, and what it is called at all.

The reuse question decides whether `localns` is a thin resolver over a shared codec or a
second DNS implementation, and the second option is the one to avoid. The naming question is
separate and more urgent: the note above found five live one-keystroke neighbours, which is
the shape of problem that got `derkit` rejected, and a directory is cheap to rename while a
published name is not.
