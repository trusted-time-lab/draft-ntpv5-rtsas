# draft-qiu-ntpv5-root-time-source-alignment

Internet-Draft: **Root Time Source Alignment Signature and Transport Protocol for NTPv5 (RTSAS)**.

## Status

- **Revision**: -00 (submitted to the IETF Datatracker)
- **Datatracker**: https://datatracker.ietf.org/doc/draft-qiu-ntpv5-root-time-source-alignment/
- **Intended status**: Standards Track
- **Author**: ximing Qiu (Tianshu Shiyuan Technology Co., Ltd.)

## What it is

RTSAS is an NTPv5 extension that turns a time value's provenance from a *claim* into *evidence*.
A pre-trusted Root Time Source attests that a given time service instance's clock has been checked
for alignment against a physical source; that instance then signs the time value itself. A client
holding only the root public key verifies both signatures offline, and so can tell a time value that
carries verifiable provenance apart from one that merely claims it.

The extension deliberately does one thing and no more: it proves *where a signed time value came
from*. It does not adjust clocks, it does not rate sources, and it has no revocation. It is opt-in,
uses two extension fields, and touches nothing in the core protocol.

## Files

| File | Description |
|---|---|
| [`draft-qiu-ntpv5-root-time-source-alignment-00.md`](draft-qiu-ntpv5-root-time-source-alignment-00.md) | The Internet-Draft text (English) |
| [`draft-qiu-ntpv5-root-time-source-alignment-00_zh.md`](draft-qiu-ntpv5-root-time-source-alignment-00_zh.md) | Chinese translation (for reference) |

## Reference implementation

Running code for this draft lives in [`ntpv5-rtsas`](https://github.com/trusted-time-lab/ntpv5-rtsas):
a three-algorithm (Ed25519 / ECDSA P-256 / SM2) reference implementation that reproduces every test
vector in Section 11 of the draft.

## Feedback

Issues and pull requests are welcome here. For IETF discussion, use the NTP working group mailing
list (`ntp@ietf.org`).
