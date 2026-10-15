# ADR-0000: Lab stack and host

- **Status:** Accepted
- **Date:** 2026-10-13
- **Phase:** 0 — scoping
- **Supersedes / superseded by:** none

## Context

This repository is a security-architecture lab for a *simulated* satellite
ground segment. The objective is to show, end to end, how an attacker can get an
unauthorised telecommand accepted by a spacecraft, and how a hardened
architecture prevents it.

Success criteria the stack must serve:

1. A telecommand replay/injection attack **succeeds** against the baseline.
2. The same attack **fails** against the hardened design.
3. Every control traces to a threat in a documented threat model, mapped to
   **SPARTA** (and **NIST IR 8401** where relevant).
4. The lab **rebuilds from the repo with minimal commands**.

Hard constraints:

- Author machine: Windows 11, 8 GB RAM, i7-1255U. Zero / near-zero hardware budget.
- Free, open-source tools only; versions and compatibility verified, not assumed.
- Lab only — no real RF transmission, no interaction with real satellites.
  Public data (e.g. SatNOGS) may be used for offline analysis only.

### Domain terms (first use)

- **TT&C** — Telemetry, Tracking & Command: the link that lets the ground
  command the spacecraft and receive its data.
- **Telecommand (TC)** — instruction sent *up* to the spacecraft (uplink).
- **Telemetry (TM)** — health/status data sent *down* to the ground (downlink).
- **CCSDS** — Consultative Committee for Space Data Systems; defines the space
  packet formats used here (CCSDS Space Packet).

## Decision

### 1. Flight software — NASA cFS (core Flight System)

Real NASA flight software, written in C. Its lab apps give an authentic attack
surface out of the box: `CI_LAB` ingests CCSDS telecommand packets over a
UDP/IP port with **no authentication and no anti-replay**, and `TO_LAB` emits
CCSDS telemetry over UDP/IP. That unprotected UDP command ingest *is* the
baseline vulnerability — it is not simulated or bolted on.

### 2. Ground software — self-built Python CCSDS client + Yamcs console

- **Core (built by hand):** a Python ground client and an on-path interposer
  that speak real CCSDS Space Packet format (primary header, APID, sequence
  count). This is the learn-by-building component and is authentic at the layer
  the security story lives on. The attacker tooling is the same code with the
  guardrails removed.
- **Operator console (real tool):** Yamcs, configured against the cFS packets
  (XTCE mission database, UDP TM/TC links). Provides a realistic mission-ops UI
  and a portfolio artifact. Optional — the lab functions without it.

OpenC3 COSMOS was **rejected** for this machine (see Alternatives).

### 3. Link — software-only UDP; RF deferred

The link is a UDP socket (what cFS already uses). GNU Radio / software-defined
RF is **deferred** to a later realism phase: command authentication and
anti-replay live at the packet/link layer, not the RF/PHY layer, so RF modelling
adds cost without changing the core security result.

### 4. Attacker model — on-path MITM on the link

The attacker is modelled as an **on-path (MITM) interposer** with access to the
link network: a container sitting between the ground and spacecraft containers
on the Docker network, able to intercept, replay, inject and (later) alter
traffic. No RF; network-level only.

### 5. Hardening direction — decided in Phase 4

Left open deliberately. Candidates: (a) an application-layer command-auth
gateway in front of cFS (HMAC + sequence/nonce anti-replay); (b) SDLS-style
data-link security closer to the CCSDS standard. Confidentiality (link
encryption) pairs with the MITM "alter in flight" story and the RF phase.

### 6. Orchestration — Docker Compose

cFS, the ground client, the interposer, and (optionally) Yamcs run as composed
containers on one Docker network.

### 7. Host / compute — GitHub Codespaces

The lab is developed and run in GitHub Codespaces via a devcontainer. It is
repo-native (directly serves criterion 4), x86 (no ARM-compatibility concerns),
and lives where the portfolio already is. Oracle Cloud Always Free was the
runner-up host (see Alternatives).

## Consequences

**Positive**

- Reproducible from the repo; a reviewer can launch the whole lab from a
  Codespace with minimal commands.
- Authentic at the layers that matter (real cFS, real CCSDS packets, real MITM
  position) while staying free and within the hardware budget.
- Clear baseline-vs-hardened demo built directly on `CI_LAB`'s open UDP ingest.

**Negative / risks**

- Codespaces free quota is limited (~120 core-hours/month ≈ 60 h on a 2-core
  machine); larger machines burn it faster. Not suitable for 24/7 running.
- Yamcs (and COSMOS, if ever added) are heavy; the Python-only path is the safe
  default if compute is tight.
- The MITM model's full payoff (eavesdropping, in-flight tampering shown as
  *defeated*) only lands once the hardened design adds link encryption — a
  later-phase deliverable. Injection and replay are demonstrable before then.

## Alternatives considered

- **OpenC3 COSMOS as ground software** — real, polished C2, but its guidance is
  4 GB RAM minimum to Docker / 16 GB recommended; alongside cFS on an 8 GB
  machine it risks swapping, and there is no out-of-the-box cFS integration.
  Rejected for this hardware; may be revisited if run on a larger cloud host.
- **Oracle Cloud Always Free (Ampere A1, ARM) VM** — a persistent box
  (currently 2 OCPU / 12 GB; a card is required for signup verification). Viable
  alternative host, but ARM architecture needs per-image checks, free ARM
  capacity is often unavailable, and idle instances can be reclaimed. Kept as a
  fallback, not the primary.
- **GNU Radio / SDR link now** — deferred; see Decision 3.

## References

- NASA cFS — https://github.com/nasa/cFS (`ci_lab`, `to_lab`, `cmdUtil`)
- SPARTA (Space Attack Research & Tactic Analysis) — https://sparta.aerospace.org
- NIST IR 8401, *Satellite Ground Segment: Applying the Cybersecurity Framework
  to Satellite Command and Control* — https://doi.org/10.6028/NIST.IR.8401
- Yamcs — https://yamcs.org
- GitHub Codespaces billing/quota —
  https://docs.github.com/en/billing/concepts/product-billing/github-codespaces
