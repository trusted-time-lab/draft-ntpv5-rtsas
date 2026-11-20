# Root Time Source Alignment Signature and Transport Protocol for NTPv5

```
Network Working Group                                     ximing Qiu
Internet-Draft                                            Tianshu Shiyuan
Intended status: Standards Track                          20 November 2026
Expires: 24 May 2027


        Root Time Source Alignment Signature and Transport
                         Protocol for NTPv5
          draft-qiu-ntpv5-root-time-source-alignment-00
```

---

## Abstract

This document defines an NTPv5 extension that turns a time value's provenance from a *claim* into *evidence*. A pre-trusted Root Time Source attests that a given time service instance's clock has been checked for alignment against a physical source; that instance then signs the time value itself. A client holding only the root public key verifies both signatures offline, so it can tell a time value that carries verifiable provenance apart from one that merely claims it.

The extension deliberately does one thing and no more: it proves *where a signed time value came from*. It does not adjust clocks, it does not rate sources, and it has no revocation — a signed proof is unilateral, it exists once signed. It is opt-in, uses two extension fields, and touches nothing in the core protocol.

---

## Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current
   Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

   Internet-Drafts are draft documents valid for a maximum of six
   months and may be updated, replaced, or obsoleted by other documents
   at any time.  It is inappropriate to use Internet-Drafts as
   reference material or to cite them other than as "work in progress."

   This Internet-Draft will expire on 17 May 2027.

## Copyright Notice

   Copyright (c) 2026 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (https://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with
   respect to this document.  Code Components extracted from this
   document must include Revised BSD License text as described in
   Section 4.e of the Trust Legal Provisions and are provided without
   warranty as described in the Revised BSD License.

## Table of Contents

   1.  Introduction
     1.1.  Problem Statement
     1.2.  Design Philosophy
     1.3.  Trust Model and Boundary
     1.4.  Deliberate Non-Goals
     1.5.  Protocol Overview
   2.  Conventions and Terminology
     2.1.  Key Words
     2.2.  Terminology
   3.  Protocol Design
     3.1.  Roles and Keys
     3.2.  Two-Level Signature
     3.3.  Root Time Source Alignment Proof (RTSAP)
     3.4.  Request and Response Modes
   4.  Extension Field Definitions
     4.1.  Carrier and Layout
     4.2.  RTSAS-Request (Uplink)
     4.3.  RTSAS-Response (Downlink)
     4.4.  Algorithm Identifier
     4.5.  RTSAA (First-Level Signature) and Its Delivery
     4.6.  RTSIS (Second-Level Signature)
     4.7.  Client Signature and Credential
   5.  Client Verification
   6.  RTSAP Relay and Transit Chain
     6.1.  Transit Time Service Instance (TTSI)
     6.2.  Caching
     6.3.  Re-origination and Recursion
     6.4.  Tiered Authorization
   7.  Coexistence with Existing Mechanisms
   8.  Security Considerations
   9.  IANA Considerations
   10. References
   11. Test Vectors
   Appendix A.  Field and Signing-Payload Summary
   Authors' Addresses

---

## 1. Introduction

### 1.1. Problem Statement

NTP has always let a client *choose* among time sources, but never let it *verify* one. Stratum, the reference clock, Time Sources — all of it is self-reported: a server states its own upstream, and no one can check the claim. NTS (RFC 8915) authenticates the transport but says nothing about the anchor; Roughtime (RFC 10049) reports a self-declared error radius. A client that needs trustworthy time — judicial evidence, financial records, audit logs, AI collaboration that aligns on a shared timeline — can choose among self-described sources but verify none of them.

This document closes that gap. It turns a time value's provenance from a claim into evidence: a pre-trusted Root Time Source signs an attestation that a given time service instance's clock has been checked for alignment against a physical source, and that instance signs the time value itself. A client holding only the root public key verifies both, offline, and so can distinguish "a time value that carries verifiable provenance" from "one that merely claims it".

### 1.2. Design Philosophy

NTP is a closed-loop control system that converges toward alignment; but its convergence target — the physical anchor — is self-reported, hence unverifiable. This extension makes the target verifiable. Once the target is verifiable, the system's existing convergence machinery has something real to converge on; the protocol adds no new control mechanism.

The protocol deliberately stops there. That is enough because verifiability aligns every party's incentive: once verifiable provenance is what every party wants — the Root Time Source wants its anchor trusted, the instance wants to be selected, the client wants trustworthy time — their behaviors align on their own. The protocol supplies a verifiable reference point for convergence; the convergence itself remains NTP's closed-loop control and each party's self-interest.

The protocol rests on five judgments; everything else — the fields, the signatures, the modes — follows from them.

1. **Claim vs evidence.** NTP's clock-selection machinery assumes there are trustworthy and untrustworthy sources and lets the client *choose* among them, but it never gave the client a way to *verify* that a given server's time is actually anchored to an authoritative source. This document adds the missing half: it upgrades "I am anchored to a physical source" from an assertion to a verifiable signature.

2. **Anchor, not identify.** The protocol anchors the fact "this time value was signed by this instance, which this Root Time Source attested to". It does *not* identify "who this server is". Identification is the job of institutions above the protocol; putting it in the protocol would turn the protocol into a platform.

3. **Unilateral evidence.** A signed proof exists once signed and cannot be revoked. "Stop trusting" is the client's choice, never "the evidence was revoked". A Root Time Source signs or does not sign; it never unsigns.

4. **Trust starts from the physical anchor.** The Root Time Source operates a physical time source (an atomic clock). That is the one thing that needs no cryptographic proof — physical reality speaks for itself. The signature chain carries that verifiability down to the time value.

5. **Separate signature from transport.** There is exactly one signer (the Root-Anchored Time Service Instance) and arbitrarily many forwarders (Transit Time Service Instances). A proof is an invariant — it proves a fixed fact; forwarding is a variable — it moves the fact down a chain. The consequence: a downstream client verifies the original signer, never the forwarder.

### 1.3. Trust Model and Boundary

The cryptographic guarantees are **exactly three**:

1. The RTSAA is signed by the Root Time Source.
2. The RTSIS is signed by the RATSI whose public key is carried in that RTSAA.
3. The Proven Time (PT) was not modified after signing.

**Not guaranteed**, and stated as such: the RATSI's actual clock offset at signing time; that it reports time honestly within the validity window; that the root key was not compromised; that the physical source is correct; that the measurement was correct. These are trust assumptions, not cryptographic conclusions.

**Red line (MUST)**: an implementation or description of this protocol MUST say only that "the Proven Time was issued by an authenticated RATSI and not modified after signing". It MUST NOT say that "the time value is still anchored to a physical source at signing time", and MUST NOT say that "the current NTP time value delivered to the client was signed by the RATSI".

Three decouplings follow from this boundary:

1. **Proven Time ≠ packet time.** The Proven Time is the *signing instant* (a historical fact, covered by the RTSIS); the NTP header's transmit timestamp is the *current* time value (re-filled per hop, not covered by any signature). A client disciplines its clock with the packet time and verifies truth with the Proven Time — the two are decoupled.

2. **Proof of source ≠ proof of sync.** This protocol proves *who signed* (verifiable, non-repudiable), not *who the server synchronized from* (self-reported, may lie). A transit node swapping its upstream is a source-listing matter; it does not invalidate the source proof.

3. **Proof ≠ verdict.** The protocol supplies evidence (the RTSAP); it does not decide whether the client should trust it. Trusting, choosing an upstream, switching on divergence — these are the client's call. The protocol has no referee.

The boundary is the point of the design: the signature proves *who signed* and *that nothing was altered*; it does not prove *that the signer was honest*. Tampering from outside is exposed; dishonesty from inside is attributable but not detectable by the protocol. That asymmetry — external tampering always exposed, internal dishonesty located but not auto-detected — is deliberate.

**Guarantee matrix.** The following table states, per item, what this protocol can and cannot verify, and what remains self-reported or the client's own responsibility. It is the single source of truth for the protocol's boundary.

| Item | Verified by protocol? | Basis |
|---|---|---|
| RTSAA signed by the Root Time Source | Yes | root signature verification |
| RTSIS signed by the RATSI named in the RTSAA | Yes | RATSI-PKF association + RTSIS verification |
| Proven Time unmodified after signing | Yes | PT inside the RTSIS payload |
| RATSI's clock offset ≤ δ at signing time | No (trust assumption) | Root Time Source attestation |
| RATSI reports honestly within the validity window | No (trust assumption) | — |
| Root key not compromised | No (trust assumption) | — |
| Physical source correct, measurement correct | No (trust assumption) | — |
| Whether a TTSI actually recursed on Generate | No | TTSI deployment responsibility |
| Whether the proof is still current (not expired) | No | client's own clock, §5 |
| Which upstream a server synchronized from | No (self-reported) | Time Sources |

### 1.4. Deliberate Non-Goals

The protocol does one thing and no more. It explicitly does **not**:

- **Adjust clocks.** Time-value transfer (carrying the timestamp on the wire) is NTPv5's on-wire capability; computing an offset and disciplining the local clock is the deployment implementation's job (source selection, measurement filtering, and clock disciplining are out of scope of the NTPv5 wire protocol). This protocol only proves *where a signed time value came from*.
- **Rate sources.** Which upstream a server synchronizes from, and how "diluted" that makes it, is reported by Time Sources (self-declared). This protocol proves the anchor, not the chain.
- **Identify servers.** The fingerprint is a cryptographic anchor for verification, not an identifier for diagnosis.
- **Revoke.** A signed proof is unilateral; once signed, it exists. There is no in-protocol revocation.
- **Identify the requester.** The credential and signature control admission (who may enter), not identification (who the requester is). Mapping a key to a legal person is an institution's job above the protocol.
- **Align requirements.** Matching a requester's need to an assurance level is a matter above the protocol. The protocol provides admission; the ecosystem does the alignment.
- **Carry business policy.** Metering, billing, and QoS stay the operator's business.
- **Support multiple root domains.** This version targets single-root-domain deployment (see §4.5). Subscription/unsubscription is out of scope.

Each of these is NTP's job, an operator's job, or an institution's job — deliberately left out so the protocol stays a protocol.

### 1.5. Protocol Overview

```
Physical Time Source (PTS)
  |  time signal (time reference)
  v
Root Time Source (RTS)
  |  alignment check: measure offset δ; if within bound, sign RTSAA (first level)
  v
Root-Anchored Time Service Instance (RATSI)
  |  sign RTSAS-Response (RTSIS + RTSAA + Proven Time, second level)
  v
Client, or a Transit Time Service Instance that relays it
  |  verify RTSAA with root key -> obtain RATSI public key
  |  verify RTSIS with RATSI public key
  |  check Proven Time within the RTSAA validity window
  v
```

What a client gains, in four capabilities:

- **Offline verification at constant cost.** A client holding only the root public key verifies both signatures in the response, without walking the upstream graph or maintaining per-server state.
- **Tamper-evidence.** Any modification to the Proven Time, the attestation, or the signature is exposed on verification.
- **Relay without re-signing.** A proof travels unchanged down a chain of transit nodes; each downstream verifies the original signer, not the forwarder.
- **Invisible to ordinary users.** The mechanism is a protocol-layer default: client software enrolls automatically. Only applications that need strong assurance touch it explicitly.

The Proven Time is the signing instant, not a "current" time (see §1.3).

---

## 2. Conventions and Terminology

### 2.1. Key Words

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174] when, and only when, they appear in all capitals, as shown here.

### 2.2. Terminology

- **Physical Time Source (PTS)**: the physical device (atomic clock, optical clock, GNSS, radio, quartz, etc.) that produces the time reference. It holds no key and does not sign.
- **Root Time Source (RTS)**: the trust root — an entity that operates a Physical Time Source, holds a root private key, and signs RTSAAs (for example, a national time service center).
- **Root-Anchored Time Service Instance (RATSI)**: an NTPv5 server whose clock has been checked for alignment by a Root Time Source (it holds an RTSAA) and that signs RTSISs with its own private key. The only entity that can produce an RTSAP.
- **Transit Time Service Instance (TTSI)**: an intermediate node that holds no RTSAA, signs no RTSIS, and only relays an RTSAP downstream. Uplink it is a client; downlink it is a time-disseminating server.
- **Root Time Source Alignment Attestation (RTSAA)**: a statement signed by the Root Time Source attesting that the RATSI's clock offset is within a bound δ and within a validity window. First-level signature (168 octets, §4.5).
- **Root-Anchored Time Service Instance Signature (RTSIS)**: the RATSI's signature over the response mode, the RATSI-PKF, the RTSAA fingerprint, and the Proven Time (§4.6). Second-level signature.
- **Proven Time (PT)**: the self-contained time value the proof is about — the instant at which the RATSI signed. It is carried inside the proof, not referenced from the header.
- **Root Time Source Alignment Proof (RTSAP)**: the complete proof — response mode, RATSI-PKF, RTSIS, RTSAA, and Proven Time, 280 octets in total. An invariant, relayable unchanged.
- **RATSI-PKF**: the identifier-derived hash (§4.4) of the RATSI public key, used to associate an RTSIS with the RTSAA that carries that key.
- **RTSAA fingerprint**: the identifier-derived hash of the complete RTSAA, used to bind an RTSIS to the specific RTSAA carried in the same response.
- **API Credential**: an opaque, authority-issued, chain-verifiable authorization token (200 octets). Its internal format is defined by the issuing authority.
- **RTSAA Request / RTSAA Response**: the out-of-band delivery of the RATSI public key to the Root Time Source, and of the signed RTSAA back to the RATSI (§4.5).

A Root Time Source may attest many RATSIs, and a RATSI may be checked by more than one Root Time Source — a many-to-many relationship, not a fixed one-to-one binding. This version targets single-root-domain deployment: a RATSI follows one Root Time Source at a time (§4.5).

---

## 3. Protocol Design

### 3.1. Roles and Keys

| Role | Holds | Does |
|---|---|---|
| Physical Time Source (PTS) | a time signal (no key) | produces the time reference |
| Root Time Source (RTS) | root private key | checks alignment, signs RTSAA |
| Root-Anchored Time Service Instance (RATSI) | RATSI private key + public key; the RTSAA | signs RTSIS; produces the RTSAP |
| Transit Time Service Instance (TTSI) | a client private key; no RTSAA, no RTSIS | relays the RTSAP; disseminates time |
| NTP Client | root public key; a client private key | requests and verifies the RTSAP |

The root public key is pre-provisioned out-of-band. The RATSI's public key is carried in the RTSAA (signed by the Root Time Source), so the client obtains it dynamically from the response. Multiple RATSIs MAY share one RATSI private key (a shared-key group). When they do, the RATSI-PKF cryptographically locates the shared-key group, not the specific device within the group; attributing a specific device requires deployment logs or a separate device-identity mechanism.

### 3.2. Two-Level Signature

The protocol name — "Root Time Source Alignment Signature and Transport" — has two orthogonal halves:

- **Signature** is the processing: how the two-level signature produces the RTSAP.
- **Transport** is the flow: how the RTSAP travels along a chain of transit nodes.

The two levels mirror the trust hierarchy: the Root Time Source (first level) attests the RATSI's clock alignment; the RATSI (second level) signs the time value. A client holding only the root key verifies both — it never trusts an intermediate to vouch for itself.

### 3.3. Root Time Source Alignment Proof (RTSAP)

The RTSAP = `Response Mode(8) ∥ RATSI-PKF(32) ∥ RTSIS(64) ∥ RTSAA(168) ∥ Proven Time(8)` = 280 octets. It is an **invariant**: it proves a fixed fact — "a Root Time Source checked this RATSI's clock against a physical source, and the RATSI signed this time value at the Proven Time" — which does not change no matter how many transit nodes later relay it.

### 3.4. Request and Response Modes

Both extension fields begin with an 8-octet mode field. The request mode declares what the client wants the server to do; the response mode declares what content the server returned. They are **orthogonal**: the response mode does not echo the request mode.

Request modes (octet 0):

| Value | Mode | Meaning |
|---|---|---|
| 0x01 | Extract | return the latest cached proof (single-hop; no recursion, no alignment check). |
| 0x02 | Generate | request a fresh proof (recurses to the RATSI, triggering re-alignment). |
| 0x03 | Query | discover — keys, root information, instance addresses. |

Response modes (octet 0):

| Value | Mode | Meaning |
|---|---|---|
| 0x01 | Proof | a signed RTSAP (verifiable, non-repudiable). Enters the RTSIS payload. |
| 0x02 | Notice | unsigned status/discovery information (self-reported, may lie). |
| 0x03 | Fault | an unsigned failure report (fault code + description). |

The query type is carried in octet 1 of the request mode when octet 0 = 0x03. The query types are:

| Query type (octet 1) | Meaning |
|---|---|
| 0x01 | latest root public key |
| 0x02 | Root Time Source information |
| 0x03 | recently cached time service instance names and IPs |
| 0x04 | Root-Anchored Time Service Instance (RATSI) names and IPs |

Values 0x00 and 0x05–0xFF are undefined and MUST be rejected (0x05–0xFF reserved for future extension). When octet 0 ≠ 0x03, octet 1 MUST be zero.

The fault code is carried in the first octet after the response mode when response mode = 0x03:

| Fault code (octet 8, byte 0) | Meaning |
|---|---|
| 0x01 | Change: this node or its upstream has a persistent problem; switch path. |
| 0x02 | Fix: the client's own problem; fix itself. |
| 0x03 | Retry: transient; retry later. |
| 0x04 | Abandon: temporarily unreliable; do not use for now (not a permanent ban — an unsigned hint MUST NOT drive irreversible action). |

Values 0x05–0xFF are reserved. A client receiving an undefined fault code SHOULD treat it as "Abandon" (read only the description text) and MUST NOT map it to "Change/Fix/Retry".

**Request-response legality matrix (MUST).** Request and response modes are orthogonal, but only certain combinations are legal. A client MUST reject unexpected combinations (this prevents "response-mode downgrade" — an attacker relabeling a proof 0x01 as a notice 0x02 to push the client down the unsigned path):

| Request mode | Allowed response mode | Meaning |
|---|---|---|
| 0x01 Extract | 0x01 Proof | cache hit |
| 0x01 Extract | 0x02 Notice | only "NO_CACHE" (miss) |
| 0x01 Extract | 0x03 Fault | extract failed |
| 0x02 Generate | 0x01 Proof | generate succeeded |
| 0x02 Generate | 0x03 Fault | generate failed |
| 0x03 Query | 0x02 Notice | query succeeded |
| 0x03 Query | 0x03 Fault | query failed |

All other combinations MUST be rejected. In particular: Generate receiving Notice MUST be rejected; Query receiving Proof MUST be rejected; Extract receiving a NO_CACHE notice is only an unsigned hint (to trigger Generate), not a trusted fact. The matrix limits response content types and prevents treating an unexpected response as a successful proof; it does NOT prevent an attacker from forging a NO_CACHE notice for an Extract request (inducing the client to send Generate — an availability attack). Deployments requiring response-content integrity MUST enable transport integrity protection.

**Generate is a fresh-proof trigger, not a privileged operation.** The protocol provides the mechanism but not its usage: who triggers a fresh proof — the RATSI on its own or a client requesting 0x02 — is deployment policy, and the protocol neither requires nor forbids either.

**Admission is access control, not abuse control.** The credential and client signature control who may enter the system: the credential is an authority-issued authorization token, and the signature proves the request comes from the holder of the credential's key. Admission is universal, yet invisible to ordinary users.

**The same credential and client signature are also the anchors for tiering — not the tiering itself.** An operator may use them to tier the six capabilities (Extract / Generate / Query upward; Proof / Notice / Fault downward) — who may do what, and what each tier receives. But *how many tiers there are, which tier may do what, what content each tier receives, and whether any of it is metered* — those are deployment policy, permanently outside the protocol. The protocol anchors *who made the request* (verifiable); it does not anchor *what tier of service they get* (a business decision). See §6.4 for the precise distinction between a RATSI and a TTSI here.

---

## 4. Extension Field Definitions

### 4.1. Carrier and Layout

This specification is an extension on top of NTPv5 ([draft-ietf-ntp-ntpv5]), carried in Extension Fields (16-bit Type + 16-bit Length, 4-octet aligned), with NTPv5's request/response symmetry rule.

The RTSAP carries its own time-value field (the Proven Time); the extension field carries the signature, the credential, the attestation, and the Proven Time. The NTPv5 header's Client Cookie is reused for request-response association and anti-replay.

Both the RTSAS-Request Data and the RTSAS-Response Data are 280 octets, so no Padding is required for symmetry. A server that supports this extension MUST return an RTSAS-Response to an RTSAS-Request; a server that does not support the extension compensates the length with Padding.

### 4.2. RTSAS-Request (Uplink, client → server)

| Field | Length | Meaning |
|---|---|---|
| Request Mode | 8 octets | octet 0 = 0x01/0x02/0x03; octet 1 = query type (when octet 0 = 0x03, MUST be 0x01–0x04); octets 2–7 reserved zero |
| Client Signature Algorithm | 8 octets | octet 0 = 0x01/0x02/0x03; octets 1–7 reserved zero |
| API Credential | 200 octets | opaque, authority-issued authorization token |
| Client Signature | 64 octets | signature over `"RTSAS-Client-v1" ∥ Request Mode ∥ Client Cookie` |

Data layout: `Request Mode(8) ∥ Client Signature Algorithm(8) ∥ API Credential(200) ∥ Client Signature(64)` = **280 octets**.

The client signature algorithm and the client signature MUST be non-zero; the API credential MUST be non-zero. The client signature algorithm octets 1–7 MUST be zero; non-zero reserved octets MUST be rejected. The algorithm flag and the client signature MUST be consistent (one non-zero and the other all-zero is inconsistent and MUST be rejected).

### 4.3. RTSAS-Response (Downlink, server → client)

| Field | Length | Meaning |
|---|---|---|
| Response Mode | 8 octets | octet 0 = 0x01/0x02/0x03; octets 1–7 reserved zero |
| RATSI-PKF | 32 octets | identifier-derived hash of the RATSI public key |
| RTSIS | 64 octets | second-level signature (§4.6) |
| RTSAA | 168 octets | first-level attestation (§4.5) |
| Proven Time (PT) | 8 octets | the self-contained time value the proof is about |

Data layout: `Response Mode(8) ∥ RATSI-PKF(32) ∥ RTSIS(64) ∥ RTSAA(168) ∥ Proven Time(8)` = **280 octets**. Response mode octets 1–7 MUST be zero.

**Notice response (response mode 0x02, unsigned).** When response mode is 0x02, the remaining 272 octets (offset 8–279) are reused as the *notice result* — a UTF-8 free-text string, NOT signature material. It is unsigned, self-reported, and may lie; the client treats it as reference, not as verified truth.

The notice result is returned by query type:

| Query type (octet 1) | Notice result format |
|---|---|
| 0x01 (latest root public key) | `"<algorithm>:<raw public key hex>"` — algorithm MUST be in {Ed25519, ECDSA-P256, SM2} (ASCII, case-sensitive); e.g. `"Ed25519:<64 hex chars>"`, `"ECDSA-P256:<128 hex chars>"`, `"SM2:<128 hex chars>"` (x‖y, no 0x04 prefix) |
| 0x02 (Root Time Source info) | free text (physical clock type / location / precision) |
| 0x03 (cached instances) | free text (names and IPs) |
| 0x04 (RATSIs) | free text (names and IPs) |

Empty or unknown results for query 0x03/0x04 return the fixed text `"NONE"`. When an Extract request (0x01) misses the cache, the notice result is the fixed text `"NO_CACHE"`.

Padding and termination (MUST): the sender MUST keep at least one 0x00 as a terminator (valid text ≤ 271 octets); the text ends at the first 0x00; octets short of 272 are zero-filled. A client MUST treat only the exact fixed text "NO_CACHE" (ASCII, case-sensitive, no localization) as a cache miss. The notice result is human-readable and NOT machine-parseable; a machine-parseable format is future work.

Trust-root boundary (MUST NOT): a client MUST NOT add a root public key learned from an unsigned notice to its trusted-root set, or replace a pre-provisioned root key from it. Root-key trust establishment MUST go through a separate out-of-band authentication process; Query is discovery reference only.

**Fault response (response mode 0x03, unsigned).** When response mode is 0x03, the remaining 272 octets are reused as the *fault description*, split into two groups:

- **Machine-readable group (fault code)**: offset 8–15 (8 octets), octet 0 = fault code (0x01 Change / 0x02 Fix / 0x03 Retry / 0x04 Abandon), octets 1–7 reserved zero.
- **Human-readable group (description text)**: offset 16–279 (264 octets), UTF-8 free text, e.g. "upstream A has no cache", "root key SM2 verification failed", "API credential expired". The sender MUST keep at least one 0x00 terminator (valid text ≤ 263 octets); octets short of 264 are zero-filled.

Notice results, description texts, and fault codes are all unsigned, self-reported, and may lie — the client treats them as reference, not as verified truth.

### 4.4. Algorithm Identifier

An 8-octet field: octet 0 carries the algorithm value, octet 1 the RTSAA Era, octets 2–7 reserved zero.

| Value | Algorithm family | Signature hash | Identifier hash | Raw public key length |
|---|---|---|---|---|
| 0x00 | no signature (placeholder) | — | — | — |
| 0x01 | Ed25519 | SHA-512 (built-in) | SHA-256 | 32 B |
| 0x02 | ECDSA P-256 / SHA-256 | SHA-256 | SHA-256 | 64 B (x ‖ y) |
| 0x03 | SM2 / SM3 | SM3 (with Za) | SM3 | 64 B (x ‖ y) |

Values 0x04–0xFF are reserved (including post-quantum). Receiving an undefined algorithm value MUST be rejected — it MUST NOT be treated as "no signature" or as a default algorithm.

The public-key field in the RTSAA is fixed at 64 octets (Ed25519 raw 32 B left-aligned, trailing 32 octets MUST be zero — a receiver MUST reject a non-zero trailing 32 octets). Two hashes, two purposes: the *signature hash* is what the signature algorithm internally hashes; the *identifier-derived hash* produces the 32-octet RATSI-PKF and RTSAA fingerprint.

The identifier-derived hash of a public key MUST be computed over the raw public-key bytes: for SM2/ECDSA the 64 B (x‖y, no 0x04 prefix); for Ed25519 the 32 B raw public key (NOT the zero-padded 64-octet field). The RTSAA fingerprint is the identifier-derived hash of the complete 168-octet RTSAA.

SM2 uses the empty-string user_id (ENTL = 0x0000); Za = SM3(ENTL ‖ a ‖ b ‖ xG ‖ yG ‖ xA ‖ yA), and the pre-digest is e = SM3(Za ‖ M). Signatures are raw `r ‖ s`, not DER.

The algorithm identifier is explicit in both directions: downlink, the RATSI algorithm is declared by the RTSAA's algorithm identifier; uplink, the client signature algorithm is declared by the request's client-signature-algorithm field. There is no negotiation — the algorithm information is obtained dynamically from the RTSAA (downlink), pre-provisioned out-of-band (root algorithm), or discovered via Query (0x01 returns the root public key with its algorithm name). **Algorithm mismatch**: if the RTSAA declares a defined algorithm the client has not implemented (e.g., the client supports only Ed25519 and the RATSI uses SM2), the client MUST reject that RTSAA and switch upstream or report per its deployment policy; the protocol provides no negotiation.

### 4.5. RTSAA (First-Level Signature) and Its Delivery

The Root Time Source's attestation that the RATSI's clock offset is within δ and within a validity window. Structure: `Algorithm Identifier(8) ∥ RATSI Public Key(64) ∥ Offset Bound δ(8) ∥ Issued At(8) ∥ valid_from(8) ∥ valid_until(8) ∥ Root Signature(64)` = 168 octets.

Signing payload: `"RTSAS-Calib-v1"(14B) ∥ Algorithm Identifier(8) ∥ RATSI Public Key(64) ∥ Offset Bound δ(8) ∥ Issued At(8) ∥ valid_from(8) ∥ valid_until(8)` = 118 octets.

Issued At is audit trail only; valid_from is when the attestation takes effect. The three time fields MUST be UTC; the validity window MUST NOT cross an NTP Era boundary (the Era is carried in the algorithm identifier's octet 1, the RTSAA Era, which MUST be 0 in this version).

The algorithm identifier in the RTSAA declares the RATSI's algorithm (octet 0); the Root Time Source's own algorithm is determined out-of-band and is not carried in the RTSAA.

**RTSAA delivery (out-of-band).** The RTSAA is produced and delivered in two out-of-band steps, before it ever appears in an RTSAS-Response:

1. **RTSAA Request (RATSI → Root Time Source).** The RATSI reports its public key to the Root Time Source over a secure channel. The core fields (MUST, uniform across Root Time Sources, enter the RTSAA and are signed) are:

   ```
   RTSAA Request (core) = Algorithm Identifier(8) ∥ RATSI Public Key(64)
   ```

   The algorithm identifier declares the RATSI public key's algorithm family (octet 0 = 0x01/0x02/0x03, octets 1–7 reserved zero). The RATSI public key is 64 octets (SM2/ECDSA x‖y; Ed25519 raw 32 B left-aligned + 32 B zero). Deployment-defined extension fields (RATSI name, serial number, authentication credential, last-alignment time, alignment count, etc.) MAY follow the core fields; they do NOT enter the RTSAA, are not signed by the Root Time Source, and do not affect client verification.

   The secure channel MUST prevent substitution of the RATSI public key by a man-in-the-middle (the public key is not secret, so confidentiality is not required; authenticity is). Examples: same-process delivery, physically isolated management network, mutual-TLS management API, secure file transfer with RATSI-PKF cross-check, or manual out-of-band RATSI-PKF verification.

2. **RTSAA Response (Root Time Source → RATSI).** The Root Time Source measures the RATSI's clock offset; if it is within its admission threshold, it signs the RTSAA and returns it over the same secure channel:

   ```
   RTSAA Response (core) = RTSAA(168B)
   ```

   The RTSAA carries the root signature, so it needs only protection against deletion (DoS), not against substitution — substitution fails verification. Deployment-defined extension fields (delivery metadata, next-alignment recommendation, etc.) MAY follow.

The admission threshold (whether to sign at all) and δ (the offset bound committed once signed) are distinct quantities: δ MUST include the measured offset |Δ| and the measurement error budget (§6), so "offset > δ" cannot hold; the admission threshold is the Root Time Source's endorsement decision.

### 4.6. RTSIS (Second-Level Signature)

The RATSI's signature over the response mode, the RATSI-PKF, the RTSAA fingerprint, and the Proven Time.

Signing payload: `"RTSAS-Trust-v1"(14B) ∥ Response Mode(8) ∥ RATSI-PKF(32) ∥ RTSAA fingerprint(32) ∥ Proven Time(8)` = 94 octets. All fields are carried inside the response — nothing is referenced from the NTPv5 header.

The response mode entering the payload prevents a proof from being relabeled as a notice or fault. The RTSAA fingerprint binds the signature to the exact RTSAA carried. The Proven Time entering the payload is what makes the proof non-repudiable: the signature attests *this* time value.

### 4.7. Client Signature and Credential

The client signs the request mode and the Client Cookie, proving the request came from the holder of the registered key — admission control, alongside the credential.

Signing payload: `"RTSAS-Client-v1"(15B) ∥ Request Mode(8) ∥ Client Cookie(8)` = 31 octets.

**Credential–public-key association.** The API credential does NOT carry the client public key. The server (RATSI) maintains the "credential → registered client public key" mapping out-of-band; the protocol does not define the mapping's internal encoding. The client public key is registered out-of-band.

**Verification tiering (MUST).** Credential verification is tiered by request mode: Generate (0x02) MUST verify the credential; Extract (0x01) and Query (0x03) SHOULD verify it. The client signature, however, MUST be verified for ALL requests (including Query) by a RATSI — see §6.4 for the RATSI/TTSI distinction.

---

## 5. Client Verification

The client verifies in order; any failing step means the response MUST NOT be accepted. The steps below apply to all four client variants (terminal clients and transit uplinks).

**Step 0 — request-response legality.** First check the "request mode × response mode" combination against the legality matrix (§3.4); an illegal combination MUST be rejected.

**Step 1 — split and field checks.**

Read the response mode (offset 0, octet 0). Octet 0 MUST be 0x01/0x02/0x03; octets 1–7 MUST be zero.

- If 0x02 (notice) or 0x03 (fault): skip the rest of this step and steps 2–4; read the notice result or the fault code + description (§4.3). Still perform Step 5 (Client Cookie match) — it applies to ALL responses.
- If 0x01 (proof): split the 280-octet Data by fixed offset:

  ```
  offset 0–7    response mode (8B)
  offset 8–39   RATSI-PKF (32B)
  offset 40–103 RTSIS (64B)
  offset 104–271 RTSAA (168B)
  offset 272–279 Proven Time (8B)
  ```

  Then three checks:

  - **Algorithm identifier check**: octet 0 MUST ∈ {0x01 Ed25519, 0x02 ECDSA, 0x03 SM2}; octets 2–7 MUST be zero. 0x00 is NOT accepted here (an RTSAA always carries a root signature).
  - **RTSAA Era check**: octet 1 (RTSAA Era) MUST be 0 in this version; non-zero is rejected.
  - **RATSI-PKF association check**: compute the identifier-derived hash of the RATSI public key inside the RTSAA (offset 8–71 of the RTSAA) — SM3 if octet 0 = 0x03, SHA-256 if 0x01/0x02; for Ed25519 the hash input is the 32 B raw key (offset 8–39 of the RTSAA), not the 64-octet field — and verify it equals the RATSI-PKF field (offset 8–39 of the response) byte-for-byte.

**Step 2 — verify the RTSAA root signature.** Rebuild the 118-octet signing payload (§4.5) and verify the root signature (RTSAA offset 104–167) with the pre-provisioned root public key, using the root algorithm (determined out-of-band). Failure → reject.

**Step 3 — verify the RTSIS, Flags and Stratum checks.** Rebuild the 94-octet signing payload (§4.6): response mode (offset 0–7), RATSI-PKF (offset 8–39), RTSAA fingerprint (identifier-derived hash of the complete 168-octet RTSAA), and Proven Time (offset 272–279). Verify the RTSIS (offset 40–103) with the RATSI public key extracted from the RTSAA, using the algorithm declared by the algorithm identifier.

Strict verification rules (MUST): the RATSI public key MUST be a valid curve point (SM2/ECDSA) or an acceptable encoding per RFC 8032 strict verification (Ed25519); SM2/ECDSA signature r, s MUST ∈ [1, n−1]; Ed25519 MUST reject small-order points and non-canonical encodings.

Flags (not in the RTSIS payload): Interleaved Mode (0x0002) and Authentication NAK (0x0004) MUST be rejected; Unsynchronized (0x0001) MUST be rejected by a synchronizing consumer and MAY be relaxed by an evidence-only consumer. Stratum: 0 or > 15 MUST be rejected by a synchronizing consumer and MAY be relaxed by an evidence-only consumer. Signature validity ≠ synchronization validity.

**Step 4 — zero-value and window checks.** Reject if any of Issued At, valid_from, valid_until, or Proven Time is zero. Then require `valid_from ≤ Proven Time ≤ valid_until`. This proves the signing instant fell within the attested window — it does NOT prove the proof is still current. The Proven Time, valid_from, and valid_until share the same Era (the RTSAA Era). A client that requires current time MUST have an independent trusted current-time source to compare against valid_until; the protocol provides none.

**Step 5 — Client Cookie match (MUST for ALL responses).** For proof 0x01, notice 0x02, and fault 0x03, and regardless of direct RATSI or transit, the response's Client Cookie MUST equal the request's Client Cookie byte-for-byte. This is NTPv5's native response-validity rule (§9), not an RTSAS-specific mechanism. Cookie match proves "this response is associated with this request", not that an unsigned notice/fault is true.

A client that uses the response for clock adjustment additionally applies NTP's own checks (Flags, Stratum) per NTPv5; these are NTP's native rules, not part of the RTSAP's cryptographic guarantees.

---

## 6. RTSAP Relay and Transit Chain

### 6.1. Transit Time Service Instance (TTSI)

A **Transit Time Service Instance (TTSI)** acts as a client uplink (requesting and verifying an RTSAP) and as a time-disseminating server downlink. It holds the pre-provisioned root public key (to verify the upstream RTSAP) and a client private key (to sign its requests); it holds no RTSAA and signs no RTSIS. It is a source-listable node, not a root-attested one.

A chain of transit nodes relays the RTSAP unchanged: **generate** recurses up to the RATSI (triggering a fresh alignment check and a fresh proof); **extract** is single-hop (answering from the transit node's own cache). A downstream client verifies the *signer* (the RATSI), not the forwarder — if a transit node tampers with the proof, verification fails and the tampering is exposed. A relayed proof is a historical fact, cacheable and relayable one-to-many.

### 6.2. Caching

Every NTP service instance — the RATSI and every TTSI — maintains a cache (possibly empty at startup). The cache holds the most recently written RTSAS-Response Data (280 octets: response mode ∥ RATSI-PKF ∥ RTSIS ∥ RTSAA ∥ Proven Time), NOT the outer NTP header (which is re-built per hop).

Only proof responses (0x01) are cached. The cache is **per-RATSI, one slot per RATSI**: a new proof from the same RATSI overwrites that RATSI's slot; proofs from different RATSIs occupy separate slots and do not overwrite each other. Notice and fault responses are unsigned and are not cached.

- The RATSI caches its own latest signed proof (it is the source; it has no upstream).
- A TTSI caches the latest proof received from upstream.

Caching is passive: keep the latest per RATSI; do NOT proactively fetch to fill the cache.

On an Extract request, the server only answers from cache: hit → proof 0x01; miss → "NO_CACHE" notice 0x02. The server does NOT judge whether the cached proof's valid_until is still sufficient — the client does that (§5 Step 4). "Most recently written" means "written most recently", NOT "currently valid" and NOT "best source": the cache does not judge valid_until, and selecting the best upstream is NTP's source-selection job, not the cache's.

This per-RATSI-slot mechanism assumes a single root domain: a TTSI serves one root-trust domain (all its upstreams' RTSAAs are signed by the same Root Time Source). Multi-root-domain cache organization is deployment-defined and out of scope.

### 6.3. Re-origination and Recursion

**Extract is single-hop.** The terminal client asks only its direct upstream; the upstream answers from its cache or returns "NO_CACHE", and does NOT recurse further. The proof is a historical fact and can be cached for a long time (valid within the RTSAA window).

**Generate recurses to the RATSI.** On "NO_CACHE" (or when a fresh proof is needed), the client sends Generate; the request recurses up the chain to the RATSI, which re-aligns to the Root Time Source (which re-signs the RTSAA) and signs a fresh proof; the time value then flows down through NTP. The downlink RTSAA is always the RATSI's own (TTSIs do not hold or sign RTSAAs).

**Re-origination.** When a TTSI receives a downstream Generate request, it does NOT forward the downstream request verbatim. It verifies the downstream credential (per its tiering policy), then re-originates: it issues a new request upstream with its OWN credential and its OWN client signature and Client Cookie. This is the re-origination mechanism. The de-duplication key for a request is the hash of the complete 280-octet Request Data.

**Generate freshness is not verifiable downstream.** A TTSI receiving Generate MUST re-originate and recurse to the RATSI (it MUST NOT answer Generate from cache) — but this MUST behavior cannot be verified cryptographically downstream: the RTSIS does not cover the request mode or Client Cookie, so a downstream cannot tell "a proof freshly signed after re-alignment" from "a cached proof signed earlier". A client requiring verifiable fresh generation MUST connect to the RATSI directly; Generate via a TTSI guarantees only that the proof is authentic, not that it was freshly re-aligned.

**Recursion bounds.** (a) The server is not responsible for freshness judgment (the client is); it MAY evict cache entries if it has a trusted current time, but eviction is not a substitute for freshness judgment. (b) To prevent an alignment-storm, implementations SHOULD use "single-flight" for the alignment task (root measurement + RTSAA re-signing) — multiple Generate requests within a window share one root measurement and RTSAA re-signing — but each Generate request still independently signs its own Proven Time + RTSIS. (c) Each TTSI MUST maintain local recursion state and MUST NOT loop back to the same upstream (local loop prevention); recursion depth is bounded by local policy (SHOULD set a limit, e.g. ≤ 8). This version has NO hop/depth field in the request — cross-node loop detection is a deployment concern (a TTSI only points at configured upstreams), and a hop/depth field is future work.

### 6.4. Tiered Authorization

The credential and client signature are the anchors for tiering, not the tiering itself (§3.4). But the RATSI and the TTSI differ in what they can actually verify, and this MUST be stated precisely so implementations agree:

- **RATSI**: verifies the credential (tiered: Generate MUST, Extract/Query SHOULD) AND verifies the client signature (MUST for ALL requests), locating the client public key via the "credential → registered public key" mapping. It can therefore associate a request with a registered key holder.
- **TTSI**: verifies the credential (tiered: Generate MUST, Extract/Query SHOULD; bearer authorization). It does NOT have to verify the downstream client signature or locate the downstream client public key — the protocol does not require this. A TTSI MAY additionally locate the client public key and verify the client signature if its deployment wants to associate a request with a key holder (for finer-grained tiering). By default — without that additional verification — a TTSI can only judge whether the credential matches its authorization policy.

Consequences:

- The client-signature anti-sniffing guarantee (§8) applies only where the signature is actually verified — i.e., the RATSI always, and a TTSI only if its deployment additionally verifies the client signature. A TTSI that does not verify the client signature is bearer authorization: an attacker who sniffs a valid credential can construct a new request with an arbitrary non-zero signature and pass the TTSI. A TTSI's credential confidentiality therefore requires transport protection (TLS/DTLS) or protocol-level credential encryption.
- Public-key location and credential-validity verification are two independent operations. Public-key location (and the client-signature verification it enables) is a MUST for ALL requests at a RATSI; credential-validity verification is tiered (Generate MUST, Extract/Query SHOULD). An implementation MUST NOT skip client-signature verification or public-key location merely because Extract/Query uses SHOULD-level credential verification.

---

## 7. Coexistence with Existing Mechanisms

| Mechanism | What it establishes | Layer |
|---|---|---|
| NTS (RFC 8915) | the packet is authenticated and unmodified in transit | transport |
| Time Sources (NTPv5 §7.5) | which upstream sources a server *claims* to use | monitoring / debugging |
| PTP (IEEE 1588) | precise clock synchronization with hardware timestamping (LAN-oriented) | clock synchronization |
| Roughtime (RFC 10049) | a signed time response with a self-reported error radius | time authentication |
| **RTSAS (this document)** | **a Root Time Source's attestation anchoring the time to a physical source** | **trust (time anchor)** |

- **NTS**: does not replace NTS; NTS authenticates transport, RTSAS carries the anchor. Both may be used together.
- **Time Sources**: Time Sources *lists* the sources a server claims; RTSAS *proves* the anchor. Complementary, not overlapping.
- **MAC / NTS placement**: per NTPv5 §7.3, the MAC extension field [RFC 8573] MUST be last. RTSAS-Request/RTSAS-Response MUST be placed before MAC, and before the NTS Authenticator when NTS is used. If both are used, the order is RTSAS → NTS Authenticator → MAC.

---

## 8. Security Considerations

The cryptographic guarantees are limited to the three in §1.3. The RATSI's actual clock offset at signing time, its honesty within the validity window, the root key's security, and the correctness of the physical source and measurement are **not** guaranteed.

| Threat | Mitigation | Residual risk |
|---|---|---|
| Tamper with RTSAP internal fields (PT, RTSIS, RTSAA, RATSI-PKF) | RTSIS/RTSAA verification fails → reject | none (under key-security and algorithm assumptions) |
| Tamper with outer NTP header fields (transmit timestamp, Stratum, Root Delay) | NTS/MAC, when enabled and covering those fields, provides transport integrity | if no transport protection, outer fields can be tampered; RTSAP does not detect this |
| Sniff credential and replay | RATSI: client signature binds non-reused Client Cookie; actual verification rejects a credential-only replay. TTSI: bearer authorization — credential confidentiality requires TLS/DTLS or protocol-level encryption | RATSI: credential alone cannot bypass signature verification; credential + client private key both leaked → spoofable. TTSI: credential leak → spoofable until expiry/revocation |
| Forge a notice response (spoof RATSI address) | unsigned; a client MUST NOT trust a notice alone; a subsequent RTSAP request to that address only succeeds if it presents a valid RTSAA verified by the pre-provisioned root | DoS / privacy leak, not authenticity break |
| Replay a full request | per-hop de-duplication window (§6.3); client signature binds Client Cookie | a full-request replay that also replays the exact Data is covered; a modified-Cookie replay needs transport integrity |
| Root private key leak | clients replace the root public key out-of-band | attributable but not auto-detected |
| RATSI private key leak | Root Time Source stops attesting; validity window expires | attributable but not auto-detected |
| Signer dishonesty (internal) | nothing auto-detects a signer that lies about its clock | attributable after the fact, not detectable by the protocol |

- **Downgrade**: a client requiring strong verification MUST treat the absence of RTSAS-Response as a verification failure. A fault response is an explicit failure, not a silent downgrade.
- **Algorithm agility**: the algorithm is carried explicitly in the RTSAA's Algorithm Identifier. SM2/SM3 is included as a China-deployment option; the algorithm identifier is explicit and self-contained, so interoperation does not depend on any default.
- **Trust-anchor management**: this document assumes the client pre-provisions the root public key out-of-band; deployments MUST establish their own root-key governance, including rotation and leak response.
- **Deployment overhead**: RTSAS is opt-in; the fields appear only when a client explicitly requests strong verification. Only the Root Time Source and the RATSI sign; transit nodes do not.

---

## 9. IANA Considerations

This document requests two values in the NTP Extension Field Types registry:

| Field Type | Meaning | Reference |
|---|---|---|
| TBD1 | Root Time Source Alignment Signature and Transport Protocol Request (RTSAS-Request) | §4.2 |
| TBD2 | Root Time Source Alignment Signature and Transport Protocol Response (RTSAS-Response) | §4.3 |

The values 0xF50C / 0xF50D are used in this draft only as Private or Experimental Use code points (RFC 9748 reserves 0xF000–0xFFFF for experimentation); they do NOT constitute a formal IANA registration. The permanent code points are [[TBD]] and MUST be allocated by IANA from the 0x0000–0xEFFF range per the applicable NTP Extension Field Types registration policy when this document progresses. Draft implementations MAY continue to use the experimental code points until then.

The algorithm-identifier namespace (algorithm_id octet 0) is protocol-internal and is defined by this document's version evolution; it does not require a separate IANA registry.

---

## 10. References

**Normative**:
- [RFC 2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119.
- [RFC 8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174.
- [RFC 8032] Josefsson, S. and I. Liusvaara, "Edwards-Curve Digital Signature Algorithm (EdDSA)", RFC 8032.
- [RFC 8915] Franke, D., Sibold, D., Teichel, K., Dansarie, M., and R. Sundblad, "Network Time Security for the Network Time Protocol", RFC 8915.
- [draft-ietf-ntp-ntpv5] "Network Time Protocol Version 5", draft-ietf-ntp-ntpv5-10 (work in progress).

**Informative**:
- [RFC 9748] Salz, R., "Updating the NTP Registries", RFC 9748, February 2025.
- [RFC 10049] Ladd, W. and M. Dansarie, "Roughtime: A Protocol for Rough Time Synchronization", RFC 10049, October 2026.
- [RFC 8573] Malhotra, A. and S. Goldberg, "Message Authentication Code for the Network Time Protocol", RFC 8573, June 2019.

---

## 11. Test Vectors

This section provides test vectors so an implementer can verify its signature-payload construction, hash inputs, and offset splitting. Fixed private keys and random k values are public test values, for vector reproduction only — they MUST NOT be used in production.

Fixed keys and nonces:

```
sk_root   = 0x0000000000000000000000000000000000000000000000000000000000000001
sk_ts     = 0x0000000000000000000000000000000000000000000000000000000000000002
sk_client = 0x0000000000000000000000000000000000000000000000000000000000000003

k_calib   = 0x000000000000000000000000000000000000000000000000000000000000000A
k_trust   = 0x000000000000000000000000000000000000000000000000000000000000000B
k_client  = 0x000000000000000000000000000000000000000000000000000000000000000E
```

SM2 public keys (x‖y, no 0x04 prefix):

```
pk_root   = 0x32c4ae2c1f1981195f9904466a39c9948fe30bbff2660be1715a4589334c74c7bc3736a2f4f6779c59bdcee36b692153d0a9877cc62a474002df32e52139f0a0
pk_ts     = 0x56cefd60d7c87c000d58ef57fa73ba4d9c0dfa08c08a7331495c2e1da3f2bd5231b7e7e6cc8189f668535ce0f8eaf1bd6de84c182f6c8e716f780d3a970a23c3
pk_client = 0xa97f7cd4b3c993b4be2daa8cdb41e24ca13f6bd945302244e26918f1d0509ebf530b5dd88c688ef5ccc5cec08a72150f7c400ee5cd045292aaacdd037458f6e6
```

SM2 timestamp64 values (UTC, NTPv5 fixed-point format):

```
EBE6168000000000 = 2025-06-01 00:00:00 UTC
EBE6249000000000 = 2025-06-01 01:00:00 UTC (+3600 s)
EBE618D880000000 = 2025-06-01 00:10:00.5 UTC
```

### 11.1. RTSAA (SM2, algorithm_id = 0x03)

Inputs: algorithm_id = `0x03 00 00 00 00 00 00 00`; RATSI public key = pk_ts; δ = 100 ns; Issued At = valid_from = `0xEBE6168000000000`; valid_until = `0xEBE6249000000000`.

```
signing_payload =
"RTSAS-Calib-v1"(14B) ∥
0x0300000000000000 ∥
pk_ts(64B) ∥
0x0000000000000064 ∥           // δ = 100
0xEBE6168000000000 ∥           // Issued At
0xEBE6168000000000 ∥           // valid_from
0xEBE6249000000000             // valid_until
```

Root signature (SM2, Za user_id = empty string, k_calib = 0xA):

```
e = SM3(Za_root ∥ signing_payload) = 0x441ded1c6bb61d5fd22e8df89b9463b81b5857c734fc23b59eddd18c6cb4564d
root_signature = 0x1817357fbd4c3f20f394ee5a91f0a1eb5c2548c1e057bcbccfc225980e313c80f3f4653f2159e06f863588d2b707af09c3f13b0a319a26ccebdae13d32bca2e8
```

Complete 168-octet RTSAA (algorithm_id ∥ RATSI pubkey ∥ δ ∥ Issued At ∥ valid_from ∥ valid_until ∥ root_signature):

```
0x0300000000000000
  56cefd60d7c87c000d58ef57fa73ba4d9c0dfa08c08a7331495c2e1da3f2bd5231b7e7e6cc8189f668535ce0f8eaf1bd6de84c182f6c8e716f780d3a970a23c3
  0000000000000064
  EBE6168000000000
  EBE6168000000000
  EBE6249000000000
  1817357fbd4c3f20f394ee5a91f0a1eb5c2548c1e057bcbccfc225980e313c80f3f4653f2159e06f863588d2b707af09c3f13b0a319a26ccebdae13d32bca2e8
```

RTSAA fingerprint = SM3(complete 168B) = `0xce03d94a7447053549fe6a2e44854214fa63e34b800f8236a2143bf0f17f57ca`.

### 11.2. RTSIS (SM2)

Inputs: response mode = `0x0100000000000000`; RATSI-PKF = SM3(pk_ts) = `0xdad451ec36b06f8a16efb4867def22db0ae251fa143931b0067192998c48517e`; RTSAA fingerprint = `0xce03d94a...` (above); Proven Time = `0xEBE618D880000000`.

```
signing_payload =
"RTSAS-Trust-v1"(14B) ∥
0x0100000000000000 ∥
RATSI-PKF(32B) ∥
RTSAA fingerprint(32B) ∥
0xEBE618D880000000
```

RTSIS (SM2, Za user_id = empty string with pk_ts, k_trust = 0xB):

```
e = SM3(Za_ts ∥ signing_payload) = 0x570d32429acf8cdd2d0bbed22076c218cc50151eb2a8bec190f91ca9ac6838de
RTSIS = 0x5bc0fd53649665bfa9266a492fdeb75bdeade2f43c6bbe43f7859181394a43ac6d7f571d124666d58f3bb9248ac0db17b78ea84f4391849a3d79970555b1fe53
```

### 11.3. RTSAA (Ed25519, algorithm_id = 0x01)

Inputs: seed_root = 1 (Ed25519); RATSI seed_ts = 2; RATSI raw public key pk_ts = `0x7422b9887598068e32c4448a949adb290d0f4e35b9e01b0ee5f1a1e600fe2674`; RATSI-PKF = SHA-256(pk_ts raw 32B) = `0x2c5a92ed92c0b7999f215be93c8f0433f58072bdba21a8b277faa495b57bf7f3`; δ = 100 ns; same time fields as §11.1.

```
signing_payload =
"RTSAS-Calib-v1"(14B) ∥
0x0100000000000000 ∥
pk_ts(32B raw) ∥ 0x0000000000000000000000000000000000000000000000000000000000000000 ∥  // 32B zero fill
0x0000000000000064 ∥
0xEBE6168000000000 ∥
0xEBE6168000000000 ∥
0xEBE6249000000000
```

Root signature (Ed25519, deterministic):

```
root_signature = 0xa90a83130f05e6f03b0fb7ad09ccd8f0136936dbebc060c4041ab67553a5f2e804aff421dea9dd7012450d5ad258cd36c3ec1915725539fadf4fedd30fa4df02
```

RTSAA fingerprint = SHA-256(complete 168B) = `0xaee21b30c2f9c90ea17bdf3949ccaeec2290859f759a2c948197ea503f1f40ea`.

### 11.4. Negative Tests (rejection semantics)

| # | Input | Expected behavior |
|---|---|---|
| N1 | algorithm_id octet 0 = 0x04 (undefined) | reject — MUST NOT treat as "no signature" or a default |
| N2 | algorithm_id octet 0 = 0x00 in an RTSAA | reject — an RTSAA always carries a root signature |
| N3 | Ed25519 public-key field trailing 32 octets non-zero | reject — MUST be 32 B left-aligned + 32 B zero |
| N4 | client signature algorithm octet 0 = 0x00 | reject — signature is mandatory |
| N5 | client signature algorithm non-zero but client signature all-zero | reject — flag and signature MUST be consistent |
| N6 | request mode octet 0 = 0x05 (undefined) | reject — request mode MUST ∈ {0x01/0x02/0x03} |
| N7 | query type octet 1 = 0x00 or 0x05–0xFF | reject — query type MUST ∈ {0x01–0x04} |
| N13 | client signature algorithm octets 1–7 non-zero | reject — reserved octets MUST be zero |
| N14 | response mode octet 0 = 0x04 (undefined) | reject — response mode MUST ∈ {0x01/0x02/0x03} |
| N15 | SM2/ECDSA public key not a valid curve point | reject — MUST be on the curve |
| N16 | SM2/ECDSA signature r = 0 or s = 0 | reject — r, s MUST ∈ [1, n−1] |
| N17 | Ed25519 small-order point / non-canonical encoding | reject — RFC 8032 strict verification |

---

## Appendix A. Field and Signing-Payload Summary

RTSAS-Request Data (280 octets):

```
Request Mode(8) ∥ Client Signature Algorithm(8) ∥ API Credential(200) ∥ Client Signature(64)
```

RTSAS-Response Data (280 octets):

```
Response Mode(8) ∥ RATSI-PKF(32) ∥ RTSIS(64) ∥ RTSAA(168) ∥ Proven Time(8)
```

RTSAA (168 octets):

```
Algorithm Identifier(8) ∥ RATSI Public Key(64) ∥ Offset Bound δ(8) ∥ Issued At(8) ∥ valid_from(8) ∥ valid_until(8) ∥ Root Signature(64)
```

RTSAA Request core (out-of-band, RATSI → Root Time Source):

```
Algorithm Identifier(8) ∥ RATSI Public Key(64)
```

Signing payloads:

```
RTSAA    = "RTSAS-Calib-v1"(14B) ∥ Algorithm Identifier(8) ∥ RATSI Public Key(64) ∥ Offset Bound δ(8) ∥ Issued At(8) ∥ valid_from(8) ∥ valid_until(8)   = 118 octets
RTSIS    = "RTSAS-Trust-v1"(14B) ∥ Response Mode(8) ∥ RATSI-PKF(32) ∥ RTSAA fingerprint(32) ∥ Proven Time(8)                                 = 94 octets
Client   = "RTSAS-Client-v1"(15B) ∥ Request Mode(8) ∥ Client Cookie(8)                                                                          = 31 octets
```

---

## Acknowledgements

The authors thank Danny Mayer for the discussion that sharpened this design's boundary, Hal Murray for implementer-side feedback, and Xing Wang of the National Time Service Center, CAS for the physical-time-source perspective. The chain-of-trust discussion by Mike Ellis, Miroslav Lichvar, and Ulrich Windl helped clarify what this document is and is not.

## Implementation Status

This section records the status of known implementations of the protocol defined by this specification at the time of posting of this Internet-Draft, and is based on a proposal described in RFC 7942. The description of implementations in this section is intended to assist the IETF in its decision processes in progressing drafts to RFCs. Note that the listing of any implementation here does not imply endorsement by the IETF.

A reference implementation of the two-level signature (Ed25519, ECDSA P-256, SM2) and the field encoding exists, reproducing the test vectors in Section 11 (positive and negative cases). It is available at https://github.com/trusted-time-lab/ntpv5-rtsas.

[RFC Editor: please remove this section before publication.]

## Authors' Addresses

   ximing Qiu
   Tianshu Shiyuan Technology Co., Ltd.
   Hangzhou 310000, China
   Email: zt2882@gmail.com
