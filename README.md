# bits Console

A GitHub Pages single-page application for browsing software recipe repositories,
triggering GitHub Actions build-and-publish workflows, and managing CVMFS software
stacks.

---

## Features

| Feature | Description |
|---------|-------------|
| **Package browser** | Search and filter packages across multiple recipe providers |
| **Dependency graph** | Interactive SVG visualisation colour-coded by provider |
| **Build dispatch** | Trigger `cvmfs-publish.yml` for any package, version, and platform |
| **Provider management** | Select providers from a shared registry, set search priority, add custom repos |
| **Scheduled builds** | Manage nightly JSON schedule files stored in the repo |
| **CVMFS status** | Live badge per package from `cvmfs-status.json` |
| **Config persistence** | Export/import browser config; optional localStorage "Remember me" |

---

## Quick start

1. Fork or clone **bits-console** into your GitHub organisation and enable
   **Settings → Pages → Deploy from branch (main / root)**.
2. Edit `ui-config.yaml` in the repo root — set `cvmfs_repo`, `cvmfs_prefix`,
   and `platforms`.  See [REFERENCE.md](REFERENCE.md) for all fields.
3. Open `https://<org>.github.io/bits-console/` in your browser.
4. Click **Connect** in the header and paste a GitHub PAT
   (needs `repo` and `workflow` scopes — see [Personal Access Token](#personal-access-token)).
5. Go to **Settings → Providers** and select the recipe repositories you want
   to build from.
6. Register self-hosted runners and add the required secrets
   (see [Runner setup](#runner-setup) and [Repository secrets](#repository-secrets)).
7. Return to **Packages**, pick a recipe, and click **Build & Publish**.

---

## Personal Access Token

The console requires a GitHub PAT to call the GitHub API on your behalf.
The token is stored in the browser only — it is never sent to any server other
than `api.github.com`.

**Required scopes**

- `repo` — read `ui-config.yaml`, recipe files, and `cvmfs-status.json`
- `workflow` — trigger and cancel GitHub Actions runs

**Create a token**

1. Go to **GitHub → Settings → Developer settings → Personal access tokens →
   Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Select the `repo` and `workflow` checkboxes.
4. Set an expiry, generate, and copy the token.
5. Paste it into the **Connect** dialog in the console header.

Use **Settings → Persistence → Export config** to save the token to a file so
you do not need to re-enter it on every visit.

---

## Providers and search order

The **Packages** tab is populated from _recipe repositories_ — Git repos that
contain `.sh` files with bits YAML front matter.

### Provider registry

Available providers are fetched from `bitsorg/bits-providers/registry.json`
(public repo).  Each entry resolves to a GitHub `owner/repo` via the `homepage`
URL field when the `owner` field contains a display name rather than a plain
GitHub slug.

### Search order

When the same package name exists in multiple providers, the provider listed
first (lowest priority number) wins.  Re-order providers with the **↑ ↓** buttons
in **Settings → Providers → Search order**.  The order is saved in the browser
and applied both to the package list and to the dependency graph.

### Dependency graph colour coding

Each provider is assigned a distinct colour in order of priority
(blue, amber, green, pink, purple, orange, cyan, lime, …).
The same colour appears:

- as a coloured dot next to the provider name in the package list and detail panel,
- as the node fill colour in the SVG dependency graph.

A legend below the graph identifies each provider's colour.  Special node styles:

| Style | Meaning |
|-------|---------|
| Grey fill, dashed border | Package not found in any active provider |
| Red fill | Circular dependency |
| Amber fill | Depth limit reached |

### Adding a provider not in the registry

Go to **Settings → Providers → Add a repository not in the registry** and enter
the GitHub owner, repository name, and branch.

### Recipe file format

```yaml
package: ROOT
version: 6.30.08
description: Data Analysis Framework
homepage: https://root.cern
license: LGPL-2.1
requires:
  - Python
  - libpng
---
# Bash build script follows the --- separator
cmake -DCMAKE_INSTALL_PREFIX="$INSTALLROOT" ...
```

---

## Runner setup

The publishing pipeline uses four runner types, each registered in
**Settings → Actions → Runners** in the bits-console repository.

### 1 · Build runner — `bits-build-<platform>`

One runner per target platform (e.g. `bits-build-x86_64-el9`).

- OS: the target Linux distribution (EL9, EL8, …)
- Must have `bits` CLI installed and on `PATH`
- Must have SSH access to the ingestion host (for `rsync`)
- Runner labels: `self-hosted`, `bits-build-<platform>`

```bash
# Register on the build host:
cd /opt/actions-runner
./config.sh --url https://github.com/<org>/bits-console \
            --token <RUNNER_TOKEN> \
            --labels "self-hosted,bits-build-x86_64-el9" \
            --unattended
./svc.sh install && ./svc.sh start
```

### 2 · Ingest runner — `bits-ingest`

- Host where `cvmfs-ingest` (the Go daemon) is installed
- Needs access to the spool directory and the object store
- Runner labels: `self-hosted`, `bits-ingest`

### 3 · Publisher runner — `bits-cvmfs-publisher`

- Host with CVMFS stratum-0 access (can open catalog transactions)
- Must have `cvmfs_server` CLI available
- Runner labels: `self-hosted`, `bits-cvmfs-publisher`

### 4 · Admin runner — `bits-admin`

A lightweight runner for administrative tasks that do not require build or
CVMFS infrastructure (e.g. updating the `bits` CLI on other runners).

- Any Linux host with `pip` / `pip3` / `pipx` and network access
- Runner labels: `self-hosted`, `bits-admin`

```bash
# Register on the admin host:
cd /opt/actions-runner
./config.sh --url https://github.com/<org>/bits-console \
            --token <RUNNER_TOKEN> \
            --labels "self-hosted,bits-admin" \
            --unattended
./svc.sh install && ./svc.sh start
```

Use the `runner_group` field in `ui-config.yaml` to add an additional label that
routes jobs to experiment-specific runners in a shared pool.

---

## Repository secrets

Add these in **Settings → Secrets and variables → Actions** inside the
bits-console repository.

| Secret | Description |
|--------|-------------|
| `SPOOL_SSH_KEY` | SSH private key (PEM) for the build runner to rsync to the ingestion host |
| `SPOOL_USER` | SSH username on the ingestion host |
| `SPOOL_HOST` | Hostname or IP of the ingestion host |
| `SPOOL_PATH` | Absolute path to the spool root directory on the ingestion host |
| `CVMFS_REPO` | CVMFS repository name (e.g. `sft.cern.ch`) |

Optional repository **variables** (Settings → Secrets → Variables):

| Variable | Description |
|----------|-------------|
| `BITS_CONFIG_REPO` | Default `owner/repo` for the recipe config directory |
| `CVMFS_BACKEND_TYPE` | `local` (default) or `s3` |
| `CVMFS_BACKEND_PATH` | Path or bucket URL for the object store |
| `INGEST_CONCURRENCY` | Worker threads for cvmfs-ingest (0 = auto) |

---

## Updating bits on runners

The `bits` CLI running on self-hosted runners can be updated directly from the
console without logging into each machine.

1. Go to **Settings → Runner administration** in the console.
2. Select the target runner from the dropdown
   (`bits-build-<platform>`, `bits-ingest`, `bits-cvmfs-publisher`, or `bits-admin`).
3. Optionally enter a **Runner group** if your runners use experiment-specific labels.
4. Optionally pin a **bits version** (e.g. `1.2.3`).  Leave blank to install the
   latest published release.
5. Click **Update bits**.

The console dispatches the `update-bits.yml` workflow.  The selected runner
installs (or upgrades) `bits` via `pip` / `pip3` / `pipx` and reports the
before/after version in the workflow log.  A link to the Actions tab is shown
in the console after dispatch.

> **Tip:** To update multiple runner types, click **Update bits** once per
> runner label.  Dispatch once per label; the workflow runs on whichever host
> carries that label.

### Private PyPI index

If `BITS_PYPI_INDEX` is set in the `bits-console.rc` block of `ui-config.yaml`, the
update workflow reads it and passes `--index-url` to pip automatically.

---

## Scheduled builds

1. Go to the **Schedules** tab and click **+ New schedule file**.
2. Set a name (e.g. `nightly.json`), a cron expression for documentation,
   and an optional description.
3. Click **+ Add entry** to add packages — fill in the package name,
   version (blank = latest), platform, and CVMFS target path.
4. Enable individual entries with the checkbox and enable the file with the
   **Enable** button.
5. The `scheduled-build.yml` workflow runs daily at 02:00 UTC and dispatches
   `cvmfs-publish.yml` for every enabled entry in every enabled schedule file.
6. Use **workflow_dispatch** on `scheduled-build.yml` to run manually and use
   the `dry_run` input to preview without actually triggering builds.

> The cron expression stored in the JSON file is informational only.
> To change the actual trigger schedule, edit the `schedule: - cron:` line
> in `.github/workflows/scheduled-build.yml`.

---

## Troubleshooting

| Symptom | Resolution |
|---------|------------|
| "Workflow not found" | Check that `publish_workflow` in `ui-config.yaml` matches the actual filename in `.github/workflows/` |
| Empty package list | No providers selected — go to **Settings → Providers** and enable at least one; if still empty, verify the GitHub slug is correct (hover over the → arrow in the provider row) |
| Wrong packages from a provider | The `homepage` field in `bitsorg/bits-providers/registry.json` controls which GitHub repo is fetched; verify the resolved path matches the intended repo |
| Authentication error | Regenerate the PAT with `repo` and `workflow` scopes; use **Connect** in the header to update it |
| Build stuck / no runner | Verify the self-hosted runner is online and the label exactly matches the platform, e.g. `bits-build-x86_64-el9` |
| "Update bits" dispatch fails | Ensure `update-bits.yml` exists in `.github/workflows/` and the PAT has the `workflow` scope |
| bits version unchanged after update | Check the workflow log — pip may have silently used a cached wheel; try pinning a specific version |
| Pages deploy counted as build | Ensure you are running the latest version of bits-console |
| Packages from wrong provider | Check **Settings → Providers → Search order** — the top-most provider wins when the same package appears in multiple providers |

---

## Configuration reference

See [REFERENCE.md](REFERENCE.md) for a quick-reference card covering all
`ui-config.yaml` fields, runner label conventions, secrets, and the
`bits-console.rc` config files passed to runners.
