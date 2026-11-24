# Ground Rules and Access

## Before You Ask for Access

- **Non-production targets only.** Never point `ghcr.io/builtwithai/ghes-loadgen` at a production GitHub Enterprise Server (GHES) appliance. Write workloads create real issues, pull requests, branches, commits, deployments, Projects v2 items, SARIF uploads, and repositories, and load ramps are designed to find where CPU, memory, and disk I/O run out.
- **Not official GitHub Support.** This repository and the gated `ghcr.io/builtwithai/ghes-loadgen` container image are maintained independently for non-production validation and capacity testing. There is no GitHub Support SLA.
- **Keep environment details off public GitHub.** Do not post GHES hostnames, private IP addresses, node counts, cluster layouts, support bundles, or tokens in public issues, discussions, or pull requests.

## What Access Covers

- **Time-boxed registry pull access:** Named GitHub account `Read` access to `ghcr.io/builtwithai/ghes-loadgen` via a secret zero-repository-access team in `builtwithai` (with org base repository permissions locked to `No permission`), or a SHA256-verified `.tar.gz` image archive when direct `ghcr.io` egress is blocked or org membership is not permitted.
- **Runnable configuration bundle:** Target-appropriate workload definitions, fixture templates, and exact run commands for your non-production setup — shared privately once access is granted.
- **Multi-node coordination:** For clustered non-production appliances that need balanced per-node traffic distribution, a pre-run headroom gate, and a polled watchdog abort if target headroom drops, coordinate the run layout privately.

## How to Reach the Maintainer

- **Contact:** [`@megamanics`](https://github.com/megamanics)
- **What to send privately** (through your existing channel or account contact):
  1. Target non-production GHES version (for example, `3.23.0`)
  2. Single-node staging vs. multi-node cluster
  3. Desired testing window and the question you are trying to answer
- Detailed intake (named GitHub usernames, hardware layout, air-gap needs, and window dates) happens in a private thread using [`docs/private-intake-form.yml`](docs/private-intake-form.yml).
- **Closeout:** When your window ends, complete the local closeout steps first (delete PAT, `docker logout ghcr.io`, remove local image and workspace files, and clean up target test objects/org settings) in [`docs/access-lifecycle.md`](docs/access-lifecycle.md), then let `@megamanics` know so window-scoped package `Read` access can be revoked.
