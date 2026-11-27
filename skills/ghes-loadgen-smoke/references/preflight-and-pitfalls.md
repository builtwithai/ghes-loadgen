# Preflight, Troubleshooting, and Verification Reference

Use this reference alongside [`SKILL.md`](../SKILL.md) to fail fast before pulling/running containers, diagnose any error on the first hit, and verify clean closeout.

---

## Contents

1. [One-Shot Preflight Script](#one-shot-preflight-script)
2. [Symptom → Root Cause → Fastest Fix](#symptom--root-cause--fastest-fix)
3. [Exit-Code Reference](#exit-code-reference)
4. [44-Vector Coverage & Intentional Exclusions](#44-vector-coverage--intentional-exclusions)
5. [Closeout Verification Script](#closeout-verification-script)

---

## One-Shot Preflight Script

Run this zero-mutation check script on the dedicated Linux `x86_64` load generator host **after receiving your private configuration bundle and non-prod token**, and **before** pulling images or staging workspaces. It validates Linux `x86_64`, Docker daemon networking (`"bridge": "none"`), SELinux mount labeling, CPU/RAM/disk headroom (workspace and Docker storage root), target `/status` and TLS posture, token authentication, GHAS + `repository_dispatch` workflow readiness on every repo in `repo.csv`, and self-hosted Actions runner availability.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Required environment variables:
#   GHES_HOST              - Non-production GHES hostname (e.g. ghes-nonprod.example.internal)
#   TEST_ORG               - Non-production test organization name
#   REPO_CSV               - Path to local repo.csv (org,repo rows; header optional)
#   TOKEN_FILE             - Path to 0600 file containing a 1-line non-prod API token (repo-admin scope recommended for GHAS verification)
#   DISPATCH_WORKFLOW_PATH - Exact workflow path from your private bundle (e.g. .github/workflows/<bundle-workflow>.yml)
# Optional:
#   GHES_INSECURE_TLS      - Set to 1 ONLY after confirming a self-signed non-prod certificate (default: 0)
#   EXPECT_RUNNER_LABEL    - Optional custom runs-on label required by the bundle workflow (default: self-hosted)

: "${GHES_HOST:?Set GHES_HOST to your non-production GHES hostname}"
: "${TEST_ORG:?Set TEST_ORG to your non-production test organization}"
: "${REPO_CSV:?Set REPO_CSV to your local repo.csv path}"
: "${TOKEN_FILE:?Set TOKEN_FILE to a readable 1-line non-prod token file}"
: "${DISPATCH_WORKFLOW_PATH:?Set DISPATCH_WORKFLOW_PATH to the exact workflow path from your private bundle}"
GHES_INSECURE_TLS="${GHES_INSECURE_TLS:-0}"
EXPECT_RUNNER_LABEL="${EXPECT_RUNNER_LABEL:-self-hosted}"

FAIL=0
WARN=0
pass() { printf '[PASS] %s\n' "$*"; }
warn() { printf '[WARN] %s\n' "$*"; WARN=$((WARN + 1)); }
fail() { printf '[FAIL] %s\n' "$*"; FAIL=$((FAIL + 1)); }

# Helper: authenticated curl reading Authorization header from stdin (keeps token out of argv/ps)
api_request() {
  local url="$1"
  local body_file="$2"
  local curl_rc=0
  local http_code="000"
  local -a tls_opt=()
  [[ "${GHES_INSECURE_TLS}" == "1" ]] && tls_opt=(-k)
  http_code="$(
    printf 'header = "Authorization: token %s"\n' "${TOKEN}" | \
      curl -sS ${tls_opt[@]+"${tls_opt[@]}"} -K - -o "${body_file}" -w '%{http_code}' --max-time 10 "${url}" 2>/dev/null
  )" || curl_rc=$?
  printf '%s %s\n' "${curl_rc}" "${http_code:-000}"
}

TMP_DIR="$(mktemp -d)"
trap 'rm -rf "${TMP_DIR}"' EXIT

# 1. OS, architecture & required CLI tools
OS_NAME="$(uname -s)"
ARCH="$(uname -m)"
if [[ "${OS_NAME}" == "Linux" && "${ARCH}" == "x86_64" ]]; then
  pass "Host platform is Linux x86_64"
else
  fail "Host platform is ${OS_NAME} ${ARCH} (dedicated Linux x86_64 host required)"
fi

for bin in docker curl sha256sum gunzip ssh-keygen ssh-keyscan base64; do
  if command -v "${bin}" >/dev/null 2>&1; then
    pass "Found tool: ${bin}"
  else
    fail "Missing required tool: ${bin}"
  fi
done

# Validate local input files early
if [[ ! -r "${REPO_CSV}" || ! -s "${REPO_CSV}" ]]; then
  fail "REPO_CSV is missing, unreadable, or empty: ${REPO_CSV}"
fi
if [[ ! -r "${TOKEN_FILE}" || ! -s "${TOKEN_FILE}" ]]; then
  fail "TOKEN_FILE is missing, unreadable, or empty: ${TOKEN_FILE}"
else
  TOK_LINES="$(wc -l < "${TOKEN_FILE}" | tr -d '[:space:]')"
  TOK_MODE="$(stat -c '%a' "${TOKEN_FILE}" 2>/dev/null || echo '')"
  if [[ "${TOK_LINES}" -gt 1 ]]; then
    fail "TOKEN_FILE must contain a single-line token (found ${TOK_LINES} lines)"
  fi
  if [[ -n "${TOK_MODE}" && "${TOK_MODE}" != "600" && "${TOK_MODE}" != "400" ]]; then
    warn "TOKEN_FILE mode is ${TOK_MODE} (restrict to 0600 before running)"
  fi
fi

# 2. Docker daemon access, non-root check, and "bridge": "none" detection
if [[ "$(id -u)" -eq 0 ]]; then
  fail "Running as root — switch to a regular non-root user in the docker group so run/ outputs stay non-root"
else
  pass "Running as non-root user ($(id -un), uid=$(id -u))"
fi

DOCKER_ROOT=""
if docker info >/dev/null 2>&1; then
  pass "Docker daemon reachable"
  DOCKER_ROOT="$(docker info -f '{{.DockerRootDir}}' 2>/dev/null || true)"
else
  fail "Cannot reach Docker daemon (check service status and docker group membership)"
fi

if [[ -r /etc/docker/daemon.json ]] && grep -Eq '"bridge"[[:space:]]*:[[:space:]]*"none"' /etc/docker/daemon.json; then
  warn 'Docker sets "bridge": "none" -> set DOCKER_NETWORK=host (or routed custom bridge) for single-node direct-probe/direct, or use the networkless cluster harness; do NOT edit daemon.json'
else
  pass "Docker default bridge not disabled in /etc/docker/daemon.json"
fi

# 3. SELinux enforcing check
if command -v getenforce >/dev/null 2>&1 && [[ "$(getenforce 2>/dev/null || true)" == "Enforcing" ]]; then
  warn "SELinux is Enforcing -> set SELINUX_RO=:ro,z and SELINUX_RW=:z on every docker -v bind mount"
else
  pass "SELinux not enforcing (standard -v bind mounts OK)"
fi

# 4. Host CPU, RAM, and disk headroom (workspace + Docker storage root)
if [[ "${OS_NAME}" == "Linux" ]]; then
  VCPU="$(nproc 2>/dev/null || echo 0)"
  AVAIL_KB="$(awk '/^MemAvailable:/ {print $2}' /proc/meminfo 2>/dev/null || echo 0)"
  AVAIL_GIB=$(( AVAIL_KB / 1024 / 1024 ))
  FREE_GIB="$(df -BG --output=avail . 2>/dev/null | tail -n1 | tr -dc '0-9' || echo 0)"
  (( VCPU >= 4 )) && pass "CPU headroom OK (${VCPU} vCPU)" || warn "Low CPU (${VCPU} vCPU; >= 4 recommended)"
  (( AVAIL_GIB >= 8 )) && pass "RAM headroom OK (~${AVAIL_GIB} GiB available)" || warn "Low RAM (~${AVAIL_GIB} GiB available; >= 8 GiB recommended)"
  (( FREE_GIB >= 10 )) && pass "Workspace disk headroom OK (${FREE_GIB} GiB free)" || fail "Low workspace disk (${FREE_GIB} GiB free; >= 10 GiB required)"
  if [[ -n "${DOCKER_ROOT}" && -d "${DOCKER_ROOT}" ]]; then
    DOCKER_FREE_GIB="$(df -BG --output=avail "${DOCKER_ROOT}" 2>/dev/null | tail -n1 | tr -dc '0-9' || echo 0)"
    (( DOCKER_FREE_GIB >= 10 )) && pass "Docker storage headroom OK (${DOCKER_FREE_GIB} GiB free at ${DOCKER_ROOT})" || fail "Low Docker storage disk (${DOCKER_FREE_GIB} GiB free at ${DOCKER_ROOT}; >= 10 GiB required)"
  fi
fi

# 5. Target /status HTTP 200 & explicit TLS check (never auto-downgrade TLS for credentialed requests)
STATUS_RC=0
STATUS_CODE="$(curl -sS -o /dev/null -w '%{http_code}' --max-time 10 "https://${GHES_HOST}/status" 2>/dev/null)" || STATUS_RC=$?
if [[ "${STATUS_RC}" -eq 0 && "${STATUS_CODE}" == "200" ]]; then
  pass "Target /status returned HTTP 200 with CA-verified TLS"
elif [[ "${GHES_INSECURE_TLS}" == "1" ]]; then
  INSEC_RC=0
  INSEC_CODE="$(curl -sk -o /dev/null -w '%{http_code}' --max-time 10 "https://${GHES_HOST}/status" 2>/dev/null)" || INSEC_RC=$?
  if [[ "${INSEC_RC}" -eq 0 && "${INSEC_CODE}" == "200" ]]; then
    warn "Target /status returned HTTP 200 with explicit GHES_INSECURE_TLS=1 -> pass -e GHES_INSECURE_TLS=1 to BOTH direct-probe and direct"
  else
    fail "Target https://${GHES_HOST}/status unreachable or unhealthy even with GHES_INSECURE_TLS=1 (curl_rc=${INSEC_RC}, http=${INSEC_CODE:-000})"
  fi
else
  fail "Target https://${GHES_HOST}/status failed strict TLS check (curl_rc=${STATUS_RC}, http=${STATUS_CODE:-000}); configure trusted CA or re-run with GHES_INSECURE_TLS=1 ONLY for a confirmed self-signed non-prod target"
fi

# Stop before sending authenticated API requests if token file or TLS check failed
if (( FAIL > 0 )); then
  printf '\nPreflight stopped early before authenticated checks: FAIL=%d WARN=%d\n' "${FAIL}" "${WARN}"
  exit 1
fi

TOKEN="$(tr -d '\r\n' < "${TOKEN_FILE}")"

# 6. Token authentication check
read -r AUTH_RC AUTH_CODE < <(api_request "https://${GHES_HOST}/api/v3/user" "${TMP_DIR}/user.json")
if [[ "${AUTH_RC}" -eq 0 && "${AUTH_CODE}" == "200" ]]; then
  pass "API token authenticated against https://${GHES_HOST}/api/v3/user"
else
  fail "API token check failed (curl_rc=${AUTH_RC}, http=${AUTH_CODE})"
fi

# 7. Repo existence, GHAS enabled (multiline JSON safe), and repository_dispatch workflow check
REPO_COUNT=0
while IFS=, read -r r_org r_name _rest || [[ -n "${r_org:-}" ]]; do
  r_org="${r_org//$'\r'/}"
  r_name="${r_name//$'\r'/}"
  r_org_lc="$(printf '%s' "${r_org}" | tr '[:upper:]' '[:lower:]')"
  [[ -z "${r_org}" || -z "${r_name}" || "${r_org}" == "#"* ]] && continue
  [[ "${r_org_lc}" == "org" || "${r_org_lc}" == "organization" ]] && continue
  REPO_COUNT=$((REPO_COUNT + 1))

  read -r REPO_RC REPO_CODE < <(api_request "https://${GHES_HOST}/api/v3/repos/${r_org}/${r_name}" "${TMP_DIR}/repo.json")
  if [[ "${REPO_RC}" -ne 0 || "${REPO_CODE}" != "200" ]]; then
    fail "Repo ${r_org}/${r_name} check failed (curl_rc=${REPO_RC}, http=${REPO_CODE})"
    continue
  fi

  if command -v jq >/dev/null 2>&1; then
    GHAS_STATE="$(jq -r 'if has("security_and_analysis") | not then "UNVERIFIED" elif (.security_and_analysis.advanced_security.status == "enabled" or .security_and_analysis.code_security.status == "enabled") then "ENABLED" else "DISABLED" end' "${TMP_DIR}/repo.json" 2>/dev/null || echo "UNVERIFIED")"
  else
    FLAT="$(tr -d '[:space:]' < "${TMP_DIR}/repo.json")"
    if [[ "${FLAT}" != *'"security_and_analysis":'* ]]; then
      GHAS_STATE="UNVERIFIED"
    elif [[ "${FLAT}" == *'"advanced_security":{"status":"enabled"'* || "${FLAT}" == *'"code_security":{"status":"enabled"'* ]]; then
      GHAS_STATE="ENABLED"
    else
      GHAS_STATE="DISABLED"
    fi
  fi

  case "${GHAS_STATE}" in
    ENABLED)    pass "Repo ${r_org}/${r_name}: HTTP 200, GHAS enabled" ;;
    UNVERIFIED) warn "Repo ${r_org}/${r_name}: security_and_analysis omitted (use a repo-admin token to verify GHAS)" ;;
    *)          fail "Repo ${r_org}/${r_name}: GHAS / Code Scanning is NOT enabled" ;;
  esac

  read -r WF_RC WF_CODE < <(api_request "https://${GHES_HOST}/api/v3/repos/${r_org}/${r_name}/contents/${DISPATCH_WORKFLOW_PATH}" "${TMP_DIR}/wf.json")
  if [[ "${WF_RC}" -ne 0 || "${WF_CODE}" != "200" ]]; then
    fail "Repo ${r_org}/${r_name}: missing dispatch workflow ${DISPATCH_WORKFLOW_PATH} (curl_rc=${WF_RC}, http=${WF_CODE})"
  else
    if command -v jq >/dev/null 2>&1; then
      WF_BODY="$(jq -r '.content // ""' "${TMP_DIR}/wf.json" | tr -d '\n' | base64 -d 2>/dev/null || true)"
    else
      WF_BODY="$(tr -d '\r\n' < "${TMP_DIR}/wf.json" | sed -n 's/.*"content"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' | sed 's/\\n//g' | base64 -d 2>/dev/null || true)"
    fi
    if printf '%s' "${WF_BODY}" | grep -q 'repository_dispatch'; then
      pass "Repo ${r_org}/${r_name}: dispatch workflow present with repository_dispatch trigger"
    else
      fail "Repo ${r_org}/${r_name}: ${DISPATCH_WORKFLOW_PATH} exists but does not declare repository_dispatch"
    fi
  fi
done < "${REPO_CSV}"
(( REPO_COUNT > 0 )) || fail "No valid org,repo rows found in ${REPO_CSV}"

# 8. Online Linux x64 self-hosted Actions runner on test org (check labels too)
read -r RUNNERS_RC RUNNERS_CODE < <(api_request "https://${GHES_HOST}/api/v3/orgs/${TEST_ORG}/actions/runners" "${TMP_DIR}/runners.json")
if [[ "${RUNNERS_RC}" -eq 0 && "${RUNNERS_CODE}" == "200" ]]; then
  if command -v jq >/dev/null 2>&1; then
    MATCH_COUNT="$(jq --arg lbl "${EXPECT_RUNNER_LABEL}" '[.runners[]? | select(.status == "online" and (.os | ascii_downcase) == "linux") | select(([.labels[]?.name | ascii_downcase] | index("x64")) and ([.labels[]?.name] | index($lbl)))] | length' "${TMP_DIR}/runners.json" 2>/dev/null || echo 0)"
    if (( MATCH_COUNT > 0 )); then
      pass "Found ${MATCH_COUNT} online Linux x64 runner(s) with label '${EXPECT_RUNNER_LABEL}' on org ${TEST_ORG}"
    else
      warn "No online Linux x64 runner with label '${EXPECT_RUNNER_LABEL}' on org ${TEST_ORG} -> register a matching runner (and ensure workflow actions are reachable) or run cluster harness runner-up"
    fi
  else
    FLAT_RUNNERS="$(tr -d '[:space:]' < "${TMP_DIR}/runners.json")"
    if [[ "${FLAT_RUNNERS}" == *'"status":"online"'* && "${FLAT_RUNNERS}" == *"\"name\":\"${EXPECT_RUNNER_LABEL}\""* ]]; then
      pass "Online runner matching label '${EXPECT_RUNNER_LABEL}' detected on org ${TEST_ORG} (install jq for full per-runner OS/arch check)"
    else
      warn "No online runner matching label '${EXPECT_RUNNER_LABEL}' confirmed on org ${TEST_ORG} -> register a matching runner or run cluster harness runner-up"
    fi
  fi
else
  warn "Unable to query org ${TEST_ORG} runners (curl_rc=${RUNNERS_RC}, http=${RUNNERS_CODE}) -> verify org-admin token or plan cluster harness runner-up"
fi

printf '\nPreflight summary: FAIL=%d WARN=%d\n' "${FAIL}" "${WARN}"
(( FAIL == 0 )) || exit 1
```

*(For multi-node clusters, also confirm `ghe-cluster-status` exits `0` and `ghe-repl-status` reports all services healthy on the non-production cluster before staging.)*

---

## Symptom → Root Cause → Fastest Fix

Common setup and runtime issues and their fastest fixes:

| Symptom | Root Cause | Fastest Fix |
|---|---|---|
| `docker pull` returns `403 Forbidden` / `denied` | Anonymous pull attempted, fine-grained PAT used instead of classic PAT, missing `read:packages` scope, or SAML SSO not authorized for `builtwithai`. | Create an expiring **classic** PAT with `read:packages` only, authorize SAML SSO for `builtwithai` if prompted, confirm access grant is active, or switch to the air-gap `.tar.gz` delivery path. |
| `ENTRYPOINT_ERROR: Mode 'direct-probe' requires explicit GHES_LOADGEN_ALLOW_NETWORK=1 opt-in` (exit `64`) | Built-in fail-closed networking gate triggered. | Pass `-e GHES_LOADGEN_ALLOW_NETWORK=1` explicitly for `direct-probe` and `direct`. |
| Exit `64` refusing `direct` / `direct-probe` on a cluster monitor host | Safety interlock detected a guarded cluster monitor host or cluster relay socket. | Use the private networkless cluster harness (`--network=none`) instead of `direct` mode. |
| `direct-probe` fails with `Temporary failure in name resolution` (`ok=0 total=5`) while host `curl` works | Host `/etc/docker/daemon.json` disables default bridge networking (`"bridge": "none"`) or DNS is host-local. | Set `DOCKER_NETWORK="host"` (or attach a routed custom bridge) for single-node `direct-probe`/`direct`, or use the networkless cluster harness. **Never** edit `/etc/docker/daemon.json` or host firewall rules. |
| `direct-probe` fails TLS handshake on self-signed staging cert, **or** `direct-probe` passes but Git-over-HTTPS vectors fail TLS in `direct` | TLS verification is ON by default, or `GHES_INSECURE_TLS=1` was passed to `direct-probe` only and omitted on `direct`. | Pass `-e GHES_INSECURE_TLS=1` consistently to **both** `direct-probe` and `direct` (confirmed self-signed non-prod targets only). |
| `direct-probe` prints `DIRECT_PROBE_OK`, yet actual load fails with `403`/`503` | `direct-probe` checks network reachability and TLS (HTTP error status codes still count as reachable contact, and any `ok >= 1` passes). | Always verify `ok == total` on the `DIRECT_PROBE_HTTPS` line **and** run `curl` against `https://<GHES_HOST>/status` to confirm `HTTP 200 OK`. |
| Immediate startup crash: fixture file `users.csv` (or `repo.csv` / `admin_token.csv`) cannot be read | Mounted CSVs are `0600` owned by a UID that differs from the container's runtime UID, or the parent `0700` directory cannot be traversed by that runtime UID. | In single-node `direct` mode, ensure `run/` and `resources/*` are owned by the same regular non-root host UID (so `0600` inside `0700` works cleanly). On multi-node harness runs, let `stage` manage mounts/permissions or upgrade to a patched image/harness while keeping token files strictly private. |
| `direct` mode hits unknown repositories (`404`) instead of your `repo.csv` | Running an older pre-release image that still contains a baked default `repo.csv`, shadowing `/workspace/resources/repo.csv`. | Pull or load a current release image revision that strips bundled fixture CSVs (request the latest digest from `@megamanics`). |
| `direct` mode runs default traffic instead of the 44-vector smoke profile | The workload config file in `${WORK_DIR}/run/` is missing or renamed from the exact filename shipped in your private bundle. | Place the exact unmodified config filename from your private bundle inside `${WORK_DIR}/run/` and re-run. |
| `42/44` vectors pass; only the 2 Git LFS pull vectors fail with `fatal: Need to specify how to reconcile divergent branches` | Running an older image revision where container Git config omits `pull.rebase=false` when concurrent writes update the same branch. | Upgrade to a current release image revision with `pull.rebase=false` configured. |
| Code Scanning (SARIF) vector fails with HTTP `403` or `404` | GHAS / Code Scanning is disabled on one or more repos listed in `repo.csv`. | Enable GHAS on every repo in `repo.csv` (Step 0 catches this up front when run with a repo-admin token). |
| Actions dispatch returns `204`, but workflow runs stay `queued` or fail immediately | No online Linux `x64` self-hosted runner matches the workflow's `runs-on` labels, referenced actions are unreachable on the appliance, or `repository_dispatch` is missing from the repo. | Register a runner with the exact `runs-on` labels required by the bundle workflow (or run cluster harness `runner-up`), confirm referenced actions are available on the target, and commit the dispatch workflow to every repo in `repo.csv`. |
| Multi-node cluster smoke aborts early on primary node memory watchdog | Stateful primary/database nodes naturally carry lower idle free RAM than stateless web nodes; applying one uniform high stateless floor triggers false-positive aborts. | Calibrate per-role memory headroom floors (lower floor for stateful primary/DB nodes, higher floor for stateless web frontends) before launching the batch. |
| Bind-mounted files fail with `Permission denied` on RHEL / Rocky / Fedora | SELinux is `Enforcing` and container volume mounts lack `:z` / `:ro,z` relabeling. | Set `SELINUX_RO=":ro,z"` and `SELINUX_RW=":z"` on every `-v` bind mount (including offline `report`). |
| Output files under `${WORK_DIR}/run/` are owned by `root` and resist plain `rm -rf` | `${WORK_DIR}/run` was created as `root` (UID `0`), so `direct` mode did not drop privileges to a non-root UID. | Always create `${WORK_DIR}` and `${WORK_DIR}/run` as your regular non-root user before running Docker. |
| Container exits with `124`, `143`, or `137` | Outer safety cutoff (`GHES_HARD_LEASE_SECS` or stale heartbeat) stopped the container (`143`/`137` forwarded from terminated workload process), or an external `docker stop` interrupted the run. | Grep run error logs under `${WORK_DIR}/run/` for `DEADMAN_ABORT reason=`, investigate stalled requests, raise `GHES_HARD_LEASE_SECS` only if intentional, and build any missing HTML report with offline `report` mode. |
| Offline `report` prints `REPORT_OK` but `index.html` is still missing | `report` skips any run directory that already contains a non-empty `js/stats.json`. | Delete `<run-dir>/js/stats.json` and `<run-dir>/index.html`, then re-run `report` and verify both files exist and are host-owned. |
| Offline `report` prints a user-preferences file-lock warning | `--user "$(id -u):$(id -g)"` maps to a host UID absent from container `/etc/passwd`, leaving `$HOME=/`. | Harmless warning — verify `REPORT_OK results=/workspace/run/results`, exit code `0`, and host-owned `index.html` + `js/stats.json`. |

---

## Exit-Code Reference

| Exit Code | Mode(s) | Meaning |
|:---:|---|---|
| **`0`** | All | Completed cleanly (`DIRECT_PROBE_OK`, smoke finished within lease, or `REPORT_OK`). |
| **`1`** | `direct-probe`, `direct`, cluster run | Runtime or probe failure (e.g., `direct-probe` had `0` reachable HTTPS attempts, fixture unreadable, or non-zero workload exit). |
| **`2`** | All | Container input/config error (`ENTRYPOINT_ERROR`: unsupported mode, missing `/workspace/run`, missing raw run data in `report` mode, or networkless check failed in cluster mode). |
| **`64`** | `direct-probe`, `direct` | Fail-closed safety interlock: missing `GHES_LOADGEN_ALLOW_NETWORK=1`, or direct network mode blocked on a guarded cluster monitor host. |
| **`124` / `143` / `137`** | `direct`, cluster run | Polled outer safety cutoff (`GHES_HARD_LEASE_SECS` or stale heartbeat) terminated the workload (`124` if child exited `0`, otherwise `143` on SIGTERM or `137` on SIGKILL; `143`/`137` also occur on external stop). Confirm `DEADMAN_ABORT reason=` in the run error logs under `${WORK_DIR}/run/`. |

---

## 44-Vector Coverage & Intentional Exclusions

A healthy all-vector smoke pass exercises **44 vectors** (1 pass per vector) across all 8 workload families:

| Workload Family | Vectors | What Is Exercised |
|---|:---:|---|
| **REST API** | 14 | Repos, branches, commits, contents, metadata, low-level Git refs/objects, issues, PRs, search, commit statuses, orgs, notifications, repository dispatch |
| **GraphQL** | 1 | Authenticated issue pagination query |
| **Git & Git LFS (HTTPS + SSH)** | 16 | Full, shallow, and partial clones; fetch, pull, push, mixed workflows, and LFS push/pull over both HTTPS and SSH |
| **Web UI & Search** | 8 | `/status`, raw file & ZIP downloads, commit history, web search, web commits, repo create & fork |
| **Projects v2** | 2 | Organization Projects v2 creation and GraphQL issue-to-board mutation (GHES `>= 3.18`) |
| **Code Scanning (SARIF)** | 1 | Bundled SARIF upload acceptance (`HTTP 202`) and branch/commit lookup |
| **Actions & Deployments** | 2 | Deployment creation/status flow + throttled workflow dispatch completed on a self-hosted runner |
| **Load Shaping** | *(mode)* | 1-pass bounded smoke shaping (`1` user per vector, `1` iteration, outer hard lease) |
| **Total** | **44** | |

**Intentionally excluded from the default 44-vector smoke pass (9 vectors):**
- **Webhooks (2):** Disabled by default until you supply an internal, operator-controlled non-prod receiver URL.
- **Packages (4):** Out of scope for this container image.
- **Legacy Web UI issue/PR forms (3):** Excluded due to UI form changes on modern GHES releases.

---

## Closeout Verification Script

Run after completing Step 6 cleanup to verify zero credentials, containers, images, or workspace files remain on the load generator host. Pass the exact `WORK_DIR`, `LOADGEN_IMAGE`, and `ARCHIVE_FILE` (if air-gap) recorded during the test window:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Recommended: export WORK_DIR, LOADGEN_IMAGE, and ARCHIVE_FILE (if air-gap) before deleting run.env
: "${WORK_DIR:?Set WORK_DIR to the dedicated workspace directory you removed}"
DOCKER_CFG="${DOCKER_CONFIG:-$HOME/.docker}/config.json"
ISSUES=0
WARNS=0

pass() { printf '[PASS] %s\n' "$*"; }
warn() { printf '[WARN] %s\n' "$*"; WARNS=$((WARNS + 1)); }
fail() { printf '[FAIL] %s\n' "$*"; ISSUES=$((ISSUES + 1)); }

# 1. Verify Docker daemon is reachable (never silently pass on Docker errors)
if docker info >/dev/null 2>&1; then
  pass "Docker daemon reachable for closeout checks"
else
  fail "Cannot reach Docker daemon to verify container/image cleanup"
fi

# 2. Verify Docker config file (and warn if OS credential helpers need manual check)
if [[ -e "${DOCKER_CFG}" ]]; then
  if [[ ! -r "${DOCKER_CFG}" ]]; then
    fail "Docker config exists but is unreadable: ${DOCKER_CFG}"
  else
    if grep -q 'ghcr.io' "${DOCKER_CFG}"; then
      fail "ghcr.io entry still present in ${DOCKER_CFG}"
    else
      pass "No ghcr.io entry in ${DOCKER_CFG}"
    fi
    if grep -Eq '"(credsStore|credHelpers)"' "${DOCKER_CFG}"; then
      warn "Docker credential helper configured in ${DOCKER_CFG} — also verify ghcr.io was cleared from your OS keychain/helper"
    fi
  fi
else
  pass "No Docker config file present at ${DOCKER_CFG}"
fi

# 3. Verify zero leftover loadgen containers and images (checked without swallowing Docker errors)
if docker info >/dev/null 2>&1; then
  LEFTOVER_CTRS="$(docker ps -a --format '{{.Names}} {{.Image}}' | grep -E 'ghes-loadgen' || true)"
  if [[ -z "${LEFTOVER_CTRS}" ]]; then
    pass "No leftover ghes-loadgen containers"
  else
    fail "Leftover containers found: ${LEFTOVER_CTRS}"
  fi

  if [[ -n "${LOADGEN_IMAGE:-}" ]]; then
    if docker image inspect "${LOADGEN_IMAGE}" >/dev/null 2>&1; then
      fail "Recorded image still present locally: ${LOADGEN_IMAGE}"
    else
      pass "Recorded image removed (${LOADGEN_IMAGE})"
    fi
  fi

  LEFTOVER_IMGS="$(docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}' | grep -E 'ghes-loadgen' || true)"
  if [[ -z "${LEFTOVER_IMGS}" ]]; then
    pass "No local ghes-loadgen images remain"
  else
    fail "Local ghes-loadgen images still present: ${LEFTOVER_IMGS}"
  fi
fi

# 4. Verify workspace and air-gap archive removed
if [[ ! -e "${WORK_DIR}" ]]; then
  pass "Workspace directory removed (${WORK_DIR})"
else
  fail "Workspace directory still exists: ${WORK_DIR}"
fi

if [[ -n "${ARCHIVE_FILE:-}" ]]; then
  if [[ ! -e "${ARCHIVE_FILE}" ]]; then
    pass "Air-gap archive removed (${ARCHIVE_FILE})"
  else
    fail "Air-gap archive still exists: ${ARCHIVE_FILE}"
  fi
fi

printf '\nCloseout verification: FAIL=%d WARN=%d (also confirm PAT deleted, ephemeral SSH key/runner removed, and @megamanics notified)\n' "${ISSUES}" "${WARNS}"
(( ISSUES == 0 )) || exit 1
```
