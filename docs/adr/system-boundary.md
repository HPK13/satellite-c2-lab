# System boundary

Phase 0 scoping artifact. Defines what the lab *is*, what it is *not*, and where
the trust boundaries sit. This feeds the data-flow diagram and threat model in
Phase 2 and the control mapping (SPARTA + NIST IR 8401).

## System under test (in scope)

- **Space segment — cFS**
  - `CI_LAB` — command ingest (CCSDS telecommands over UDP; no auth, no anti-replay in baseline)
  - `TO_LAB` — telemetry output (CCSDS telemetry over UDP)
  - cFE core + sample apps as needed to execute and acknowledge commands
- **Ground segment**
  - Python CCSDS ground client (command send + telemetry display)
  - Yamcs operator console (optional realism layer)
- **The link** — UDP TC uplink and TM downlink between ground and spacecraft
- **The shared Docker network** the components sit on
- **Security controls** — added in Phase 4 (authentication, anti-replay,
  segmentation, access control, logging)
- **Detection** — alerting on suspicious command activity (Phase 5)

## Actors

- **Operator** — legitimate user issuing telecommands from the ground software.
- **Attacker** — on-path (MITM) adversary with access to the link network;
  realised as an interposer container between ground and spacecraft. Capabilities:
  observe, replay, inject, and (later phases) modify traffic. No RF access.

## Trust boundaries

1. **Ground segment zone  ↔  Space segment zone.** The primary boundary. The
   TC/TM link crosses it; the spacecraft must not implicitly trust whatever
   arrives on the uplink.
2. **Attacker position.** The attacker sits *on the link*, i.e. astride boundary
   (1). This is the position the hardened design must neutralise.

(The baseline has effectively no enforced boundary — any host that can reach the
`CI_LAB` UDP port can command the spacecraft. Establishing and enforcing the
boundary is the hardening work.)

## Data flows

- **TC uplink:** ground software → `CI_LAB` (CCSDS telecommand, cleartext in baseline)
- **TM downlink:** `TO_LAB` → ground software (CCSDS telemetry)
- **Attack flows:** interposer → `CI_LAB` (replay / inject / tamper);
  interposer observes TM (eavesdrop)

## Assets — what we are protecting

- **Integrity and authenticity of telecommands** (primary) — only authorised,
  unmodified, non-replayed commands should execute.
- **Availability of command & control** — the operator must retain positive control.
- **Confidentiality of the link** (secondary; later phase, with encryption).

## Out of scope

- Real RF transmission; any interaction with real satellites or ground stations.
- The RF / physical layer (until the dedicated RF-realism phase).
- Orbital mechanics / flight dynamics.
- Physical, supply-chain, and host-OS-level attacks.
- Production key management and PKI (the lab demonstrates mechanisms, not an
  operational key infrastructure).
- SatNOGS or other public data used live — permitted for **offline analysis only**.

## Assumptions and constraints

- Free, open-source tooling only.
- The lab is self-contained and isolated; it never touches a real-world system.
- The environment rebuilds from the repo in GitHub Codespaces with minimal commands.
- Findings and controls are traced to SPARTA techniques and the NIST IR 8401 /
  CSF ground-segment profile.
