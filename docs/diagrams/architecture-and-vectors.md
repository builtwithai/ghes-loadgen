# Runner Architecture and Workload Paths

This diagram shows how `ghcr.io/builtwithai/ghes-loadgen` separates workload generation, built-in runtime safeguards, single-node and multi-node execution paths, and offline HTML reporting.

```mermaid
flowchart TB
  classDef runner fill:#1b4721,stroke:#3fb950,stroke-width:2px,color:#ffffff
  classDef guard fill:#7a2e00,stroke:#f0883e,stroke-width:2px,color:#ffffff
  classDef single fill:#1f3a5f,stroke:#79c0ff,stroke-width:2px,color:#ffffff
  classDef cluster fill:#5a2ca0,stroke:#d2a8ff,stroke-width:2px,color:#ffffff
  classDef out fill:#0d419d,stroke:#58a6ff,stroke-width:2px,color:#ffffff

  subgraph LG["Containerized Runner (ghcr.io/builtwithai/ghes-loadgen)"]
    V["Workload Families<br/>REST · GraphQL · Git & LFS<br/>Web UI · Search · Projects v2<br/>Code Scanning · Actions"]:::runner
    C["Runtime Safeguards<br/>Explicit Network Opt-In<br/>Hard-Lease Cutoff & Abort"]:::guard
    V --> C
  end

  subgraph M1["Single-Node & Staging Runs"]
    DP["Connectivity Probe<br/>Operator Reachability &<br/>TLS Pre-Flight"]:::single
    SN["Non-Prod GHES Target<br/>Smoke, Upgrade Sanity<br/>& Workload Families"]:::single
    DP -.->|"Reachability & TLS"| SN
  end

  subgraph M2["Multi-Node Cluster Runs"]
    BL["Balanced Traffic Split<br/>Even Per-Node Fanout<br/>Across Cluster Nodes"]:::cluster
    CL["Non-Prod GHES Cluster<br/>Stepped Ladders &<br/>Ceiling Discovery"]:::cluster
    WD["External Health Gate<br/>Pre-Run Headroom Gate<br/>& Polled Live Abort"]:::guard
    BL --> CL
    WD -.->|"Monitor"| CL
    WD -.->|"Gate / abort"| C
  end

  R["Offline HTML Reports<br/>Latency Percentiles, RPS<br/>& Failure Breakdowns"]:::out

  C -.->|"Operator pre-flight"| DP
  C -->|"Direct mode"| SN
  C -->|"Cluster run path"| BL
  C -->|"Offline report"| R
```

## Built-In Safeguards

- **Explicit network opt-in:** Direct container modes (`direct-probe` and `direct`) refuse to run unless `GHES_LOADGEN_ALLOW_NETWORK=1` is set (exiting with status `64` otherwise). `direct-probe` is an operator-run HTTPS reachability and TLS pre-flight (`GHES_INSECURE_TLS=1` disables certificate verification for self-signed non-production appliances on both `direct-probe` and `direct` runs).
- **Zero baked credentials:** Test tokens, fixture inputs, and optional non-production SSH keys are mounted read-only at runtime — nothing sensitive lives inside the container image.
- **Hard-lease duration ceiling:** Every run enforces a polled outer workload cutoff (`GHES_HARD_LEASE_SECS`) so a disconnected terminal never leaves a forgotten load run hammering an appliance overnight.
- **Offline HTML reports:** Summary reports (throughput, latency percentiles, and failure breakdowns) are generated locally on a best-effort basis after a run, and missing reports can be built offline under `--network=none` via the `report` mode.
- **External health gates for clusters:** Multi-node runs pair even per-node traffic distribution with a pre-run headroom gate and a polled watchdog that aborts the container if target headroom drops below threshold.
