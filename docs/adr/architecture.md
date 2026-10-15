# Baseline architecture

Phase 0 sketch of the **baseline (deliberately insecure)** system. The hardened
architecture is introduced in Phase 4; this is the "before" picture the attack
in Phase 3 runs against.

```mermaid
flowchart LR
    subgraph GROUND["Ground segment — trust zone"]
        MC["Mission control<br/>Python CCSDS client / Yamcs"]
    end

    subgraph SPACE["Space segment (cFS) — trust zone"]
        CI["CI_LAB<br/>command ingest over UDP<br/>no auth, no anti-replay"]
        TO["TO_LAB<br/>telemetry output over UDP"]
    end

    MC -->|"telecommand — CCSDS, cleartext"| CI
    TO -->|"telemetry — CCSDS"| MC

    ATK(["Attacker<br/>on-path MITM on the link"])
    ATK -.->|"intercept / replay / inject TC"| CI
    ATK -.->|"eavesdrop TM"| TO

    classDef danger fill:#FCEBEB,stroke:#A32D2D,color:#501313;
    classDef space fill:#E6F1FB,stroke:#185FA5,color:#042C53;
    class ATK danger
    class CI,TO space
```

## How to read it

- **Two trust zones** (ground, space) with the **link crossing between them**.
  In the baseline nothing enforces that boundary.
- **The vulnerable surface** is `CI_LAB`: it accepts CCSDS telecommand packets
  over UDP with no authentication and no anti-replay, so any host that can reach
  the port can command the spacecraft.
- **The attacker** is on-path on the link (a MITM interposer container). Solid
  arrows are the legitimate TC/TM flows; dashed arrows are the attack.

## What changes later

- **Phase 3** — reproduce an unauthorised / replayed telecommand across this link.
- **Phase 4** — introduce the hardened architecture (segmentation, command
  authentication, anti-replay, access control, logging); each control recorded
  as its own ADR and mapped to the threat it defeats.
- **Later realism** — software-defined RF link and link encryption, which is
  what makes the MITM "alter / eavesdrop" paths visibly defeated.
