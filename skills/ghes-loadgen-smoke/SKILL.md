---
name: ghes-loadgen-smoke
description: >-
  Set up, preflight, execute, verify, and cleanly tear down a 44-vector non-production
  GitHub Enterprise Server (GHES) smoke test using ghcr.io/builtwithai/ghes-loadgen
  without trial and error. Use this skill whenever the user wants to load test a GHES
  cluster or staging appliance, run a ghes-loadgen smoke test, set up ghes-loadgen
  from scratch, validate GHES non-prod capacity or upgrade readiness, troubleshoot
  direct-probe or cluster harness failures, build offline load-test HTML reports, or
  run engagement closeout and PAT revocation.
compatibility: Dedicated Linux x86_64 host with Docker Engine (non-root user), curl, sha256sum, gunzip, base64, ssh, ssh-keygen, ssh-keyscan, and access to a non-production GHES target.
---

# GHES Load Generator: From-Scratch Smoke Test Runbook

> **Non-production targets only.** Never point `ghcr.io/builtwithai/ghes-loadgen` at production GHES. The 44-vector smoke profile executes real write operations (issues, pull requests, branches, commits, repositories, deployments, Projects v2 items, SARIF uploads, and Actions dispatches).

Follow Steps 0 through 6 in order. Front-loading every host and target check before pulling images or staging workspaces avoids the trial-and-error failures that normally derail first-time smoke runs.

For the full symptom-to-fix table, exit-code matrix, 44-vector coverage breakdown, and copy-pasteable preflight/closeout scripts, read [`references/preflight-and-pitfalls.md`](references/preflight-and-pitfalls.md).

---

## Step 0 — Obtain Bundle & Run One-Shot Preflight Gate

Before running preflight, obtain from `@megamanics` via your private channel (never post hostnames, IPs, topologies, or tokens in public issues):
- Either a window-scoped **GHCR digest reference** (`ghcr.io/builtwithai/ghes-loadgen@sha256:<DIGEST>`) **or** an **air-gap `.tar.gz` archive** + out-of-band SHA256 checksum.
- Expected `org.opencontainers.image.revision` commit SHA (and image config ID for air-gap). Confirm the release includes runtime `repo.csv` override support and Git `pull.rebase=false` defaults.
- Your private **workload configuration bundle** (44-vector smoke config file with its exact filename, fixture templates, `repository_dispatch` workflow file path/template, and cluster harness if targeting a cluster).

Run the batched preflight script in [`references/preflight-and-pitfalls.md`](references/preflight-and-pitfalls.md#one-shot-preflight-script) on your dedicated Linux `x86_64` loadgen host to verify every prerequisite before pulling or launching load:

| Check | Requirement | Action if Flagged |
|---|---|---|
| **Host OS, arch & Docker** | Linux `x86_64`, non-root user in `docker` group | Switch to a dedicated Linux `x86_64` loadgen VM (never a laptop, macOS host, or GHES prod node). |
| **Docker bridge mode** | Inspect `/etc/docker/daemon.json` for `"bridge": "none"` | Set `DOCKER_NETWORK="host"` (or a routed custom bridge) for single-node `direct-probe`/`direct`, or use the networkless cluster harness. **Never** edit `/etc/docker/daemon.json` or host firewall/NAT rules. |
| **SELinux mode** | Check `getenforce` (`Enforcing` vs `Permissive`/`Disabled`) | If `Enforcing`, set `SELINUX_RO=":ro,z"` and `SELINUX_RW=":z"` for every `-v` bind mount (including offline `report`). |
| **Host headroom & tools** | `>= 4` vCPU, `>= 8 GiB` available RAM, `>= 10 GiB` free disk on both workspace and Docker storage root; `docker`, `curl`, `sha256sum`, `gunzip`, `ssh-keygen`, `ssh-keyscan` | Free space or move workspace/Docker storage before staging. |
| **Target `/status` & TLS** | `curl -sS https://<GHES_HOST>/status` returns `HTTP 200` | Prefer trusted CA certs. Only if you have confirmed a self-signed non-prod cert, opt in explicitly with `GHES_INSECURE_TLS=1` and pass it consistently to **both** `direct-probe` and `direct`. |
| **Cluster / repl health** | *(Cluster targets only)* `ghe-cluster-status` exit `0` and `ghe-repl-status` all healthy | Repair non-prod cluster health before generating load. |
| **Test user token auth** | `GET /api/v3/user` returns `HTTP 200` (single-line `0600` token file) | Rotate or fix the non-prod token before staging fixtures. |
| **Repo GHAS & workflow** | Every repo in `repo.csv` returns `200`, has GHAS enabled (verified with a repo-admin token), and contains the bundle's `repository_dispatch` workflow trigger | Enable GHAS and commit the exact dispatch workflow on every missing repo before starting. |
| **Self-hosted runner** | Online Linux `x64` self-hosted runner on the test org carrying the workflow's exact `runs-on` labels (with referenced actions reachable on the appliance) | Register a matching runner, or plan to run the cluster harness's `runner-up` step and re-verify before load. |

Persist non-secret run settings (no tokens!) in a dedicated per-run directory so separate shell invocations share exact flags. Set `GHES_HOST` to your real non-production hostname first, and in any new shell export `WORK_DIR` (printed below) before running `source "${WORK_DIR}/run.env"`:

```bash
export GHES_HOST="ghes-nonprod.example.internal"   # Replace with your non-prod GHES hostname
export WORK_DIR="$(pwd)/ghes-loadgen-work-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "${WORK_DIR}" && chmod 0700 "${WORK_DIR}"
cat > "${WORK_DIR}/run.env" <<EOF
export WORK_DIR=$(printf '%q' "${WORK_DIR}")
export GHES_HOST=$(printf '%q' "${GHES_HOST}")
export DOCKER_NETWORK="bridge"       # Set to "host" if daemon sets "bridge": "none"
export GHES_INSECURE_TLS="0"         # Set to "1" ONLY for confirmed self-signed non-prod certs
export SELINUX_RO=":ro"              # Set to ":ro,z" if SELinux is Enforcing
export SELINUX_RW=""                 # Set to ":z" if SELinux is Enforcing
EOF
chmod 0600 "${WORK_DIR}/run.env"
echo "Saved non-secret run settings to: ${WORK_DIR}/run.env"
```

Do not proceed past Step 0 with any failing prerequisite.

---

## Step 1 — Path Selection & Image Verification

| Decision | Option A | Option B |
|---|---|---|
| **Image delivery** | **GHCR pull** (egress to `ghcr.io` allowed): expiring **classic** PAT with `read:packages` only + SAML SSO authorized for `builtwithai` if prompted + digest-pinned pull (`@sha256:<DIGEST>`). Fine-grained PATs do not support GHCR container pulls. | **Air-gap `.tar.gz`** (no `ghcr.io` egress or no org membership allowed): verify out-of-band SHA256 checksum, then `gunzip -c \| docker load`. No PAT or org membership required. |
| **Target topology** | **Single-node non-prod (`direct` mode)**: standalone container with explicit network opt-in. | **Multi-node non-prod cluster (private cluster harness)**: strictly networkless (`--network=none`) container behind the bundle's host round-robin relay and headroom watchdog. Never fire unguarded `direct` traffic at a single cluster node. |

**Path A (GHCR pull):** Run `docker login` in an interactive terminal (if executing inside a non-interactive agent shell, have the human operator run `docker login` directly — never pass a PAT in command arguments):

```bash
source "${WORK_DIR}/run.env"
export GH_USER="<your-github-username>"
export LOADGEN_IMAGE="ghcr.io/builtwithai/ghes-loadgen@sha256:<DIGEST_FROM_MAINTAINER>"

read -rsp "Enter expiring read:packages classic PAT: " GHCR_PAT
echo
printf '%s\n' "${GHCR_PAT}" | docker login ghcr.io -u "${GH_USER}" --password-stdin && \
  docker pull --platform linux/amd64 "${LOADGEN_IMAGE}"
unset GHCR_PAT
printf 'export LOADGEN_IMAGE=%q\n' "${LOADGEN_IMAGE}" >> "${WORK_DIR}/run.env"
```

**Path B (Air-gap `.tar.gz`):**

```bash
source "${WORK_DIR}/run.env"
export ARCHIVE_FILE="/path/to/ghes-loadgen-release.tar.gz"
export EXPECTED_SHA256="<SHA256_FROM_PRIVATE_THREAD>"

printf '%s  %s\n' "${EXPECTED_SHA256}" "${ARCHIVE_FILE}" | sha256sum -c - && \
  gunzip -c "${ARCHIVE_FILE}" | docker load
export LOADGEN_IMAGE="<LOADED_IMAGE_REF_OR_ID>"
printf 'export LOADGEN_IMAGE=%q\nexport ARCHIVE_FILE=%q\n' "${LOADGEN_IMAGE}" "${ARCHIVE_FILE}" >> "${WORK_DIR}/run.env"
```

Run the mandatory image metadata check:

```bash
docker image inspect \
  --format 'arch={{.Architecture}} id={{.Id}} size={{.Size}} rev={{index .Config.Labels "org.opencontainers.image.revision"}}' \
  "${LOADGEN_IMAGE}"
```

Confirm `arch=amd64`, and verify `rev` (plus `id` for air-gap) matches the values shared privately.

---

## Step 2 — Fail-Closed Gate (`exit 64`) & `direct-probe`

### 2.1 Verify Fail-Closed Interlock

Verify the container refuses to start without `GHES_LOADGEN_ALLOW_NETWORK=1` (use `--network=none --pull=never` and capture the exit code safely under `set -e`):

```bash
source "${WORK_DIR}/run.env"

rc=0
docker run --rm --network=none --pull=never --platform linux/amd64 --name ghes-loadgen-probe \
  -e "GHES_HOST=${GHES_HOST}" \
  "${LOADGEN_IMAGE}" direct-probe || rc=$?
echo "Fail-closed exit code: ${rc}"   # Must be 64 with ENTRYPOINT_ERROR
test "${rc}" -eq 64
```

### 2.2 Run `direct-probe` Reachability and TLS Check

```bash
source "${WORK_DIR}/run.env"

docker run --rm --pull=never --platform linux/amd64 --name ghes-loadgen-probe \
  --network="${DOCKER_NETWORK:-bridge}" \
  -e GHES_LOADGEN_ALLOW_NETWORK=1 \
  -e "GHES_HOST=${GHES_HOST}" \
  -e "GHES_INSECURE_TLS=${GHES_INSECURE_TLS:-0}" \
  "${LOADGEN_IMAGE}" direct-probe
```

Verify **both** conditions before launching load:
1. The line preceding `DIRECT_PROBE_OK` shows `DIRECT_PROBE_HTTPS ok=5 total=5` (**`ok == total`**).
2. `https://${GHES_HOST}/status` returns **`HTTP 200 OK`** via `curl` (`direct-probe` validates network reachability and TLS only; non-2xx responses such as `403`/`503` still count as reachable contacts).

Keep `GHES_INSECURE_TLS` identical between `direct-probe` and `direct` so Git-over-HTTPS workloads do not fail TLS checks after the probe passes.

---

## Step 3 — Workspace & Fixture Staging

Create the locked-down workspace subdirectories owned by your **regular non-root host user** (never create `run/` as `root`, or `direct` mode will not drop privileges and outputs will be root-owned):

```bash
source "${WORK_DIR}/run.env"
mkdir -p "${WORK_DIR}/resources" "${WORK_DIR}/secrets" "${WORK_DIR}/ssh" "${WORK_DIR}/run"
chmod 0700 "${WORK_DIR}" "${WORK_DIR}/resources" "${WORK_DIR}/secrets" "${WORK_DIR}/ssh"
```

### Stage Runtime Inputs

Populate these files from your private configuration bundle (SARIF payloads are already bundled inside the image):

| Path | Mode & Ownership | Why This Matters |
|---|---|---|
| `${WORK_DIR}/resources/users.csv` | Single-node `direct`: `0600` owned by the same non-root UID as `run/`. Multi-node cluster: staged by harness `stage`. | In `direct` mode the container drops to the UID owning `run/`, so matching ownership keeps CSVs strictly `0600` inside `0700`. In multi-node harness runs, let `stage` handle mounts/permissions (a different container UID cannot traverse a `0700` host directory). |
| `${WORK_DIR}/resources/repo.csv` | Same as `users.csv` | Target non-prod `org,repo` list; requires a release image without a baked default `repo.csv`. |
| `${WORK_DIR}/resources/admin_token.csv` | Same as `users.csv` | Non-prod admin token row for org/admin checks. |
| `${WORK_DIR}/secrets/admin_token` | `0600` (strictly private) | Single-line non-prod admin token mounted read-only. |
| `${WORK_DIR}/run/<exact-bundle-config-filename>` | `0600` | **Keep the exact unmodified filename from the private bundle.** Missing or renamed config silently falls back to the image's default workload instead of the 44-vector smoke profile. |

### Generate, Upload, and Pre-Verify an Ephemeral SSH Key

```bash
source "${WORK_DIR}/run.env"
ssh-keygen -t ed25519 -N "" -C "ghes-loadgen-ephemeral" -f "${WORK_DIR}/ssh/id_ed25519"
ssh-keyscan -p 22 -T 10 "${GHES_HOST}" 2>/dev/null | sort -u > "${WORK_DIR}/ssh/known_hosts"
chmod 0600 "${WORK_DIR}/ssh/id_ed25519"
chmod 0644 "${WORK_DIR}/ssh/known_hosts"
```

Upload `${WORK_DIR}/ssh/id_ed25519.pub` to the non-prod test user account and **record the returned SSH key ID** for Step 6 cleanup. Verify SSH auth before launching load:

```bash
ssh -i "${WORK_DIR}/ssh/id_ed25519" \
  -o UserKnownHostsFile="${WORK_DIR}/ssh/known_hosts" \
  -o IdentitiesOnly=yes -o BatchMode=yes \
  -T "git@${GHES_HOST}" || true
# Expect: "Hi <username>! You've successfully authenticated, but GitHub does not provide shell access."
```

---

## Step 4 — Execute the 44-Vector Smoke Test

The all-vector smoke profile runs 1 bounded pass across **44 vectors** spanning all 8 workload families within the container's 6 GiB memory cap. Use a fresh `run/` directory for each smoke run.

### Option A — Single-Node Non-Production Target (`direct` Mode)

```bash
source "${WORK_DIR}/run.env"

docker run --rm --init --pull=never --platform linux/amd64 --name ghes-loadgen \
  --cpus=3 --memory=6g --memory-swap=6g \
  --network="${DOCKER_NETWORK:-bridge}" \
  -e GHES_LOADGEN_ALLOW_NETWORK=1 \
  -e "GHES_HOST=${GHES_HOST}" \
  -e "GHES_INSECURE_TLS=${GHES_INSECURE_TLS:-0}" \
  -e GHES_HARD_LEASE_SECS=1800 \
  -v "${WORK_DIR}/resources:/workspace/resources${SELINUX_RO:-:ro}" \
  -v "${WORK_DIR}/secrets/admin_token:/workspace/secrets/admin_token${SELINUX_RO:-:ro}" \
  -v "${WORK_DIR}/ssh:/workspace/ssh${SELINUX_RO:-:ro}" \
  -v "${WORK_DIR}/run:/workspace/run${SELINUX_RW:-}" \
  "${LOADGEN_IMAGE}" direct
```

### Option B — Multi-Node Non-Production Cluster (Private Cluster Harness)

Never fire unguarded `direct` traffic at a single node of a multi-node cluster. Run the private cluster harness supplied with your intake bundle (it runs the container strictly networkless under `--network=none` over a host round-robin relay with zero host firewall/NAT mutations):

1. **`stage`:** Stage the verified image, fixtures, ephemeral SSH key, and host round-robin relay; confirm zero host firewall/NAT changes.
2. **`verify-routing`:** Prove before sending real load that HTTPS and SSH split evenly (`1/N`) across all web frontends, Git read traffic reaches the Git storage replica, and host NAT rules remain untouched.
3. **`runner-up` (if needed):** If Step 0 found no matching online self-hosted runner on the test org, launch the harness's ephemeral runner and re-verify it reports `online` with the expected labels before starting load.
4. **Supervised 44-vector smoke run:** Set **role-calibrated memory headroom floors** before starting — stateful primary/database nodes naturally carry far less idle free RAM than stateless web frontends, so applying one high stateless-node floor to a primary node causes false-positive watchdog aborts. Keep the run directory if you plan to verify offline `report` mode in Step 5.

---

## Step 5 — Deterministic Verification & Offline `report` Mode

### 5.1 Check Exit Code

| Exit Code | Meaning | Next Action |
|:---:|---|---|
| **`0`** | Completed within `GHES_HARD_LEASE_SECS`. | Verify all four signals below. |
| **`1`** | Runtime or probe failure (unreadable fixture, failed request/probe). | Inspect run logs under `${WORK_DIR}/run/` and check [`references/preflight-and-pitfalls.md`](references/preflight-and-pitfalls.md#symptom--root-cause--fastest-fix). |
| **`2`** | Configuration or input error (`ENTRYPOINT_ERROR`). | Fix missing `/workspace/run` mount, invalid mode, or missing raw run data. |
| **`64`** | Fail-closed safety interlock triggered. | Pass `GHES_LOADGEN_ALLOW_NETWORK=1` (or switch to cluster harness if on a guarded cluster monitor host). |
| **`124` / `143` / `137`** | Hard-lease (`GHES_HARD_LEASE_SECS`) or stale-heartbeat cutoff, or external stop (`143`/`137` forwarded from terminated child). | Check run error logs under `${WORK_DIR}/run/` for `DEADMAN_ABORT reason=`, investigate stalled calls, and run offline `report` below. |

### 5.2 Verify All Four Signals

1. **Summary JSON & HTML report:** Inspect **both** `${WORK_DIR}/run/results/<run-dir>/js/stats.json` and `index.html` for this run. Confirm **44/44** vectors recorded `OK`, **0** unexpected `KO`s, and **0** `401`/`403`/`5xx` responses. *(If 42/44 pass and only the 2 Git LFS pull vectors fail with a divergent-branch Git error, upgrade to a release revision with `pull.rebase=false` configured.)*
2. **Actions completion:** Verify dispatched workflow runs created since run start reached `status=completed` and `conclusion=success` (not stuck `queued`).
3. **Post-run target health:** Re-check `https://${GHES_HOST}/status` (`HTTP 200`) and (for clusters) `ghe-cluster-status` exit `0` plus green `ghe-repl-status`.
4. **Offline `report` mode (fills in missing reports):** `report` skips any run directory that already has a non-empty `js/stats.json` (even if `index.html` is missing) and still prints `REPORT_OK`. To force a full rebuild, delete `<run-dir>/js/stats.json` and `<run-dir>/index.html` first:

```bash
source "${WORK_DIR}/run.env"
export RUN_DIR="${WORK_DIR}/run"

docker run --rm --network=none --pull=never --platform linux/amd64 \
  --user "$(id -u):$(id -g)" \
  -v "${RUN_DIR}:/workspace/run${SELINUX_RW:-}" \
  "${LOADGEN_IMAGE}" report
```

Confirm stdout ends with `REPORT_OK results=/workspace/run/results`, exit code is `0`, and **both** `index.html` and `js/stats.json` exist and are owned by your host UID/GID. Ignore any harmless user-preferences file-lock warning emitted when the host UID has no `/etc/passwd` entry in the container (`$HOME=/`).

---

## Step 6 — Mandatory Clean Closeout

Execute every applicable step when the test window ends, then run the [`closeout verification script`](references/preflight-and-pitfalls.md#closeout-verification-script):

1. **Drain and deregister ephemeral Actions runner** (if launched in Step 4B); cancel any queued runs and verify `0` ephemeral runners remain on the test org.
2. **Delete the ephemeral SSH key** uploaded in Step 3 using the recorded key ID.
3. **Clean non-prod target test state:** Delete disposable test tokens, test repos/branches/objects created by write vectors, and revert any test-org settings modified by Projects v2 runs.
4. *(GHCR pull path only)* **Delete your `read:packages` classic PAT** immediately on `github.com` (do not wait for expiry), run `docker logout ghcr.io`, verify `${DOCKER_CONFIG:-$HOME/.docker}/config.json` contains no `ghcr.io` entry, and clear any configured OS credential helper (`credsStore` / `credHelpers`).
5. **Remove containers, image, dedicated workspace, and air-gap archive (if used):**

```bash
source "${WORK_DIR}/run.env"
docker rm -f ghes-loadgen ghes-loadgen-probe 2>/dev/null || true
if [[ -n "${LOADGEN_IMAGE:-}" ]]; then
  docker rmi "${LOADGEN_IMAGE}"
fi
if [[ -n "${ARCHIVE_FILE:-}" && -f "${ARCHIVE_FILE}" ]]; then
  rm -f "${ARCHIVE_FILE}"
fi
# Verify WORK_DIR points to your dedicated per-run directory before removing
[[ -n "${WORK_DIR:-}" && "${WORK_DIR}" == *ghes-loadgen-work* ]] && rm -rf "${WORK_DIR}"
```

6. **Verify clean state:** Run the closeout verification script (confirming Docker reachability, no `ghcr.io` credential entry, and zero leftover `ghes-loadgen` containers, images, workspaces, or archives); also confirm zero ephemeral Actions runners remain on the test org, host firewall/NAT rules remain untouched on clusters, and cluster/replication health is green.
7. **Notify `@megamanics`** in your private thread to revoke window-scoped registry access.

