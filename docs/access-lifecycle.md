# Access, Authentication, and Revocation Runbook

`ghcr.io/builtwithai/ghes-loadgen` is a **private, access-gated** `linux/amd64` container package on GitHub Container Registry (GHCR). Anonymous `docker pull` returns `403 Forbidden` / `denied` by design.

Access is granted per named GitHub account for a bounded testing window, authenticated with each operator's **own expiring classic `read:packages` PAT**, and closed out on both sides when testing ends.

---

## 1. Access Model Summary

> **GitHub Packages Rule:** GitHub Container Registry only allows private organization-scoped packages (`ghcr.io/builtwithai/ghes-loadgen`) to be shared with **members or teams of that organization**, and packages with inherited repository permissions ignore standalone package grants until inheritance is disconnected.

Delivery uses one of two least-privilege paths:

| Delivery Path | How It Works | Best For |
|---|---|---|
| **GHCR Pull (Zero-Repo Secret Team)** | Maintainer locks org base repository permissions to `No permission`, disconnects repository inheritance on the package, adds approved GitHub account(s) to a **Secret** zero-repository-access team in `builtwithai` with package `Read` only, and revokes those window-scoped grants at closeout. | Environments with direct or allowlisted HTTPS egress to `ghcr.io`. |
| **Offline `.tar.gz` Archive (Out-of-Band SHA256)** | Maintainer exports the exact `linux/amd64` image as `.tar.gz` and sends the SHA256 digest out-of-band; operator verifies checksum and runs `docker load`. No org membership or `read:packages` PAT is required. | Air-gapped networks, environments where temporary org membership is not permitted, or when cross-org member anonymity is required. |

| Security Control | What It Enforces |
|---|---|
| **No anonymous pulls** | Package stays private so unguided traffic is not pointed at production appliances. |
| **No shared static tokens** | Every GHCR operator authenticates with their own GitHub account and expiring classic `read:packages` PAT. |
| **Verified zero repository access** | Org base repository permission is `No permission`, member repo creation/forking are disabled, no internal repos are exposed, and the **Secret** access team grants package `Read` on `ghes-loadgen` only (note: GitHub org members can still see other org members on the org People page — use the `.tar.gz` archive path when operator anonymity across concurrent test windows is required). |
| **Digest-pinned pulls** | Each window gets an immutable `@sha256:<DIGEST>` reference for reproducible runs, regardless of any moving tags. |
| **Dual closeout** | Operator deletes the PAT (if used), logs out, removes local image/workspace files, and cleans up target test objects; maintainer revokes only the window-scoped team/org grants recorded at intake. |

---

## 2. Host Prerequisites (Load Generator VM)

Run the container on a **dedicated Linux `x86_64` load generator host** — never from a laptop and never on a production GHES node:

- **Docker Engine** on Linux `x86_64` running as a regular non-root user.
- **SELinux (RHEL / Rocky / Fedora):** If SELinux is enforcing, append `:z` (or `:ro,z` for read-only mounts) to bind-mount flags.
- **Workspace ownership:** In direct mode, the container writes outputs using the host UID/GID owning the mounted run directory, so normal cleanup uses plain `rm -rf` without elevated privileges. (When running the offline `report` mode, pass `--user "$(id -u):$(id -g)"` with a writable results directory. Multi-node cluster runs use the private host harness.)
- **Multi-node cluster runs:** When testing a multi-node non-production cluster with balanced per-node traffic distribution, a pre-run headroom gate, and a polled watchdog abort, host setup instructions and configuration bundles are shared in the private intake thread.

---

## 3. Maintainer Steps (`@megamanics`)

### One-Time Org & Repo Prerequisites

Before inviting any external operator to `builtwithai`:

1. In **Organization settings** → **Member privileges**:
   - Set **Base permissions** to **No permission**.
   - Disable **Repository creation** and **Repository forking** for members.
   - Verify no internal repositories are visible to newly invited members.
2. Create a **Secret** team with **zero repository permissions** for package-only `Read` grants. *(Note: GitHub organization members can still view other members of the same organization on the org People page; when operator anonymity across concurrent test windows is required, use the offline `.tar.gz` delivery path in Section 5 instead of inviting to the org.)*
3. On this public showcase repository (`builtwithai/ghes-loadgen`), disable **Issues** in repository **Settings** if you want to strictly prevent public issue filing (`.github/ISSUE_TEMPLATE/config.yml` disables blank issues, not the Issues feature itself).

### Grant Access

1. Confirm private intake is complete: non-production target verified, test window end date recorded, and delivery path chosen (GHCR pull with named GitHub username(s), or air-gapped `.tar.gz` archive).
2. For **GHCR Pull** requests, in GitHub Package Settings for `ghcr.io/builtwithai/ghes-loadgen`:
   - Open **Packages** → `ghes-loadgen` → **Package settings**.
   - Ensure **Inherit access from source repository** is **unchecked / disconnected**.
   - Under **Manage access** → **Invite teams or people**, grant **`Read`** to the dedicated **Secret** zero-repository-access team.
3. Add each approved GitHub username to that secret team, record in the private thread whether this window created a new org membership (vs. adding a pre-existing member to the team), and verify in **Organization → People** that the invitee sees **0 repositories**.
4. Share in the private thread:
   - Exact digest-pinned image reference: `ghcr.io/builtwithai/ghes-loadgen@sha256:<DIGEST>` (or `.tar.gz` archive + out-of-band SHA256 for air-gap)
   - Expected `org.opencontainers.image.revision` commit SHA (and image config ID for offline loads)
   - Runnable workload configuration bundle and target-specific invocation steps
   - Window end date (for setting PAT expiration on GHCR pulls).

### Revoke Access (GHCR Pulls)

1. Revoke **only** the grants recorded for this window: remove the GitHub username(s) from the secret access team, and remove them from the `builtwithai` organization **only if** org membership was created specifically for this testing window (do not remove pre-existing org members).
2. Confirm in **Package settings** → **Manage access** that no stale external accounts remain.
3. Post closeout confirmation in the private thread once the operator confirms local closeout.

---

## 4. Operator Steps (GHCR Pull Path): Expiring `read:packages` PAT, Pull, and Probe

*(Skip Steps 4.1–4.2 if you received the air-gapped `.tar.gz` archive — jump straight to Section 5, then return to Step 4.3.)*

> **Classic PAT required:** GitHub Container Registry (`ghcr.io`) requires a **Personal access token (classic)** with the `read:packages` scope — fine-grained PATs do not currently support GHCR container pulls. Do **not** grant `repo`, `write:packages`, `delete:packages`, or `admin:*` scopes.

### Step 4.1: Generate Your Expiring Classic PAT

1. On `github.com`, open **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
2. Click **Generate new token** → **Generate new token (classic)**.
3. Set **Note** to something traceable, e.g. `ghes-loadgen-nonprod-2026-12`.
4. Set **Expiration** to **Custom** matching your test window end date (typically **7–14 days**, up to 30 days; never "No expiration").
5. Select **only** one scope:
   - `[x] read:packages`
6. Click **Generate token** and store it temporarily in your password manager (never commit it to git or paste it into issues or chat).
7. **SAML SSO note:** If your account is linked to SAML SSO for the package-owning organization (`builtwithai`), click **Configure SSO** next to the token and authorize it for `builtwithai`. (If **Configure SSO** is not shown yet, sign in through the org's SSO identity provider first.)

### Step 4.2: Authenticate and Pull the Digest-Pinned Image

Run these commands on your dedicated Linux `x86_64` load generator host:

```bash
export GH_USER="<your-github-username>"
export LOADGEN_IMAGE="ghcr.io/builtwithai/ghes-loadgen@sha256:<DIGEST_FROM_MAINTAINER>"

# Prompt securely for the expiring read:packages PAT (keeps it out of shell history and ps)
read -rsp "Enter expiring read:packages PAT: " GHCR_PAT
echo
printf '%s\n' "${GHCR_PAT}" | docker login ghcr.io -u "${GH_USER}" --password-stdin && \
  docker pull --platform linux/amd64 "${LOADGEN_IMAGE}" && \
  docker image inspect \
    --format 'arch={{.Architecture}} id={{.Id}} size={{.Size}} rev={{index .Config.Labels "org.opencontainers.image.revision"}}' \
    "${LOADGEN_IMAGE}"
unset GHCR_PAT
```

Verify that:
- `arch=amd64`
- `rev=<40-char-commit-sha>` matches the revision shared privately by `@megamanics`.

### Step 4.3: Check Target Reachability and TLS Before Generating Load

Run `direct-probe` manually as a pre-flight check against your non-production GHES hostname (`direct` mode does not invoke `direct-probe` automatically). Direct modes fail closed (exit code `64`) unless `GHES_LOADGEN_ALLOW_NETWORK=1` is explicitly passed, and `direct-probe` verifies HTTPS TLS certificates by default. On hardened Docker hosts where `/etc/docker/daemon.json` sets `"bridge": "none"`, add `--network=host` (or attach a routed custom bridge) for single-node `direct-probe`/`direct` runs, or use the networkless cluster harness without modifying `/etc/docker/daemon.json`.

```bash
export GHES_HOST="ghes-nonprod.example.internal"

# TLS verification is enabled by default.
# Pass -e GHES_INSECURE_TLS=1 ONLY on self-signed non-prod targets (disables TLS verification;
# pass the same flag to `direct` runs too so Git-over-HTTPS workloads do not fail TLS checks).
docker run --rm --platform linux/amd64 --name ghes-loadgen-probe \
  -e GHES_LOADGEN_ALLOW_NETWORK=1 \
  -e "GHES_HOST=${GHES_HOST}" \
  "${LOADGEN_IMAGE}" direct-probe
```

How to read the output:
- `direct-probe` is a **reachability and TLS check**, not an appliance health gate: it makes 5 HTTPS requests by default (`https=5` is attempts, not successes), counts HTTP error status codes (such as `403` or `503`) as reachable contacts, and succeeds as long as at least 1 attempt connects.
- Inspect the preceding `DIRECT_PROBE_HTTPS ok=N total=N tls_verify=...` line and confirm **`ok` equals `total`**.
- Separately confirm that `https://${GHES_HOST}/status` returns a healthy `200 OK` response before starting any load run.

### Step 4.4: Run Bounded Workloads and Build Missing Reports Offline

Once reachability and appliance health are verified, run the workload configuration bundle provided in your private thread (`direct` mode for single-node targets, or the cluster run path for multi-node targets).

Every workload run enforces a polled outer cutoff via `GHES_HARD_LEASE_SECS` (followed by graceful termination) and writes a self-contained HTML report on a best-effort basis after completion. If you need to build a missing HTML report offline from an existing run directory (requires readable raw run data and a writable results directory for the supplied UID/GID; skips runs that already have a completed report):

```bash
export RUN_DIR="$(pwd)/work/run-<timestamp>"

docker run --rm --network=none --pull=never --platform linux/amd64 \
  --user "$(id -u):$(id -g)" \
  -v "${RUN_DIR}:/workspace/run" \
  "${LOADGEN_IMAGE}" report
```

A successful run prints `REPORT_OK results=/workspace/run/results`.

---

## 5. Air-Gapped / Restricted-Egress Delivery Fallback

If your load generator host cannot reach `ghcr.io` directly (or if temporary org membership is not permitted) — **no `read:packages` PAT or org membership is needed**:

1. Request the **air-gapped `.tar.gz` delivery** option in the private intake form.
2. Receive the archive (`ghes-loadgen-release.tar.gz`) via your standard transfer channel and receive the expected **SHA256 checksum**, **image config ID (`sha256:...`)**, and **`org.opencontainers.image.revision`** out-of-band in the private thread.
3. Verify the checksum and load the image locally (`docker load` does not preserve registry repo digests, so verify by image config ID and OCI label):

```bash
export ARCHIVE_FILE="./ghes-loadgen-release.tar.gz"
export EXPECTED_SHA256="<SHA256_FROM_PRIVATE_THREAD>"

printf '%s  %s\n' "${EXPECTED_SHA256}" "${ARCHIVE_FILE}" | sha256sum -c - && \
  gunzip -c "${ARCHIVE_FILE}" | docker load

# Verify loaded image config ID and revision label against the values shared by @megamanics
export LOADGEN_IMAGE="<LOADED_IMAGE_REF_OR_ID>"
docker image inspect \
  --format 'arch={{.Architecture}} id={{.Id}} rev={{index .Config.Labels "org.opencontainers.image.revision"}}' \
  "${LOADGEN_IMAGE}"
```

4. Run the `direct-probe` check in Step 4.3, and destroy the `.tar.gz` archive and local Docker image at closeout (skipping the GHCR PAT and `docker logout` steps).

---

## 6. Closeout Revocation Checklist

Complete **all** applicable items when your test window ends. Local cleanup clears credentials, images, run outputs, and target-side test objects; revoking GHCR access blocks future pulls.

### Step 6.1: Operator Checklist

- [ ] *(GHCR pull path only)* **Delete the PAT:** Go to **Settings** → **Developer settings** → **Personal access tokens (classic)** and delete `ghes-loadgen-nonprod-*` immediately (do not wait for expiry).
- [ ] *(GHCR pull path only)* **Log out of GHCR and verify active Docker config & credential store:**
  ```bash
  docker logout ghcr.io || echo "WARNING: docker logout failed" >&2
  DOCKER_CFG="${DOCKER_CONFIG:-$HOME/.docker}/config.json"
  if [[ -f "${DOCKER_CFG}" ]] && grep -q 'ghcr.io' "${DOCKER_CFG}"; then
    echo "WARNING: ghcr.io entry still present in ${DOCKER_CFG}" >&2
  else
    echo "CHECKED_DOCKER_CONFIG (also clear ghcr.io from any OS credsStore/credential helper)"
  fi
  ```
- [ ] **Stop any running containers and remove the local image** (re-export `LOADGEN_IMAGE` if in a fresh shell — use the loaded image ref/ID from Section 5 for air-gap runs):
  ```bash
  export LOADGEN_IMAGE="ghcr.io/builtwithai/ghes-loadgen@sha256:<DIGEST_FROM_MAINTAINER>"  # or loaded image ID for air-gap
  docker rm -f ghes-loadgen ghes-loadgen-probe 2>/dev/null || true
  docker rmi "${LOADGEN_IMAGE}"
  ```
- [ ] **Remove the local workspace directory** (or the exact run path used for your tests) and any air-gap `.tar.gz` archive:
  ```bash
  rm -rf ./work
  ```
- [ ] **Clean up the non-production GHES target:** Remove ephemeral SSH keys, disposable non-prod tokens, test repositories/objects, and revert any test-org settings modified during runs (such as Projects v2 base permissions).
- [ ] **Confirm closeout** in the private thread.

### Step 6.2: Maintainer Checklist (`@megamanics`, GHCR Pull Path)

- [ ] Revoke **only** the grants recorded for this window: remove named GitHub account(s) from the secret zero-repository-access team, and remove them from the `builtwithai` organization only if org membership was created for this window.
- [ ] Confirm in **Package settings** → **Manage access** that no stale external accounts remain on `ghcr.io/builtwithai/ghes-loadgen`.
- [ ] Close the private thread with the closeout timestamp.
