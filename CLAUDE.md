# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Published, auto-generated VEX (Vulnerability Exploitability eXchange) content for Liquibase products, laid out per the Trivy VEX Repository Spec v0.1. Consumers (Trivy `--vex repo`, Grype, Docker Scout) fetch the `main` branch tarball declared in `vex-repository.json`.

**Do not hand-edit anything under `pkg/`, `repo-content/`, `index.json`, or `vex-repository.json`.** They are regenerated from `liquibase-pro/vex/assessments.yaml` and any manual change is overwritten on the next run. To change an assessment, open a PR against `liquibase-pro/vex/assessments.yaml` (see that repo's `vex/CONTRIBUTING.md`). The only things maintained by hand here are `README.md`, `.github/workflows/`, and this file.

## Generation and sync flow

There is no build system, test suite, or source code in this repo. The generator and validator scripts live in liquibase-pro, not here.

1. `assessments.yaml` merges to `master` in liquibase-pro. Its `vex-repo-dispatch.yml` fires a `repository_dispatch` (type `vex-assessments-updated`) at this repo.
2. `.github/workflows/update-vex.yaml` sparse-checks-out `liquibase-pro/vex/`, installs SHA-pinned `vexctl` and `yq`, runs `generate-vex-repo.sh --output-dir ./repo-content`, then `validate-vex-repo.sh ./repo-content`.
3. It copies `repo-content/{vex-repository.json,index.json,pkg}` to the repo root, opens a PR on branch `automation/update-vex`, and **squash-merges it immediately** (`main` has no required checks). The comment in the workflow explains why: a stale VEX repo causes scanners to re-report already-assessed CVEs.
4. `.github/workflows/trigger-trivy-scan.yml` runs on any push to `main` touching `pkg/**`, `index.json`, or `vex-repository.json` and dispatches `trivy-scan-published-images.yml` in liquibase-pro so the new statements are picked up without waiting for the cron.

Notes that follow from this:
- `repo-content/` is committed as a byproduct of step 3 (create-pull-request commits the whole working tree). It is byte-identical to the root copy. Treat it as noise, not a second source of truth.
- `liquibase-pro/` at the root is an empty leftover from local runs of the workflow's sparse checkout. Nothing here depends on it.
- Both workflows get a GitHub App token via AWS OIDC -> Secrets Manager (`/vault/liquibase`), using the double-prefixed env names `LIQUIBASE_LIQUIBASE_GITHUB_APP_ID` / `..._PRIVATE_KEY`. All actions are pinned to commit SHAs with a version comment; keep that convention when bumping.

## Content layout

- `vex-repository.json`: spec manifest. Points at `https://github.com/liquibase/vex-repo/archive/refs/heads/main.tar.gz//vex-repo-main`, 24h update interval. Renaming the repo or default branch breaks every consumer.
- `index.json`: PURL -> file map. Only `vex.openvex.json` files are indexed; multiple PURLs (e.g. `pkg:docker/liquibase/liquibase`, `pkg:docker/liquibase/liquibase-secure`, `pkg:maven/org.liquibase/liquibase-core`) can point at the same document. Ecosystems present: maven, deb (ubuntu), pypi, golang, generic, docker.
- `pkg/<type>/<namespace>/<name>/`: two files per package.
  - `vex.openvex.json` (OpenVEX 0.2.0): what Trivy consumes. Statements list `products` with `subcomponents`; the liquibase-core document is a merged "umbrella" doc covering every transitive dependency assessed for the Liquibase image.
  - `vex.cdx.vex.json` (CycloneDX 1.6): same assessments with full advisory data (CVSS, CWEs, `vers:` ranges, `response`, `firstIssued`/`lastUpdated`). Not indexed; for consumers that want richer data.

## Useful local commands

Inspecting content (only `jq`, `trivy`, and `gh` are typically installed locally; `vexctl` and `yq` are not):

```bash
# List all assessed vulns in the umbrella doc
jq '.statements[] | {vuln: .vulnerability.name, status, justification}' \
  pkg/maven/org.liquibase/liquibase-core/vex.openvex.json

# Same from the CycloneDX side
jq '.vulnerabilities[] | {id, state: .analysis.state, justification: .analysis.justification}' \
  pkg/maven/org.liquibase/liquibase-core/vex.cdx.vex.json

# Which PURLs are indexed, grouped by ecosystem
jq -r '.packages[].id' index.json | cut -d/ -f1 | sort | uniq -c

# Confirm the committed repo-content mirror matches the root (should print nothing)
diff -rq repo-content/pkg pkg && diff -q repo-content/index.json index.json

# Exercise the published repo the way a customer does
trivy image --vex repo --show-suppressed liquibase/liquibase:latest

# Kick a regeneration without waiting for liquibase-pro
gh workflow run update-vex.yaml
```

## README maintenance

`README.md` is hand-written and is the customer-facing doc. Its "Repository structure" tree and "Current assessments" table are snapshots and drift from the generated content (the tree shows only liquibase-core; `pkg/` now has ~60 packages across several ecosystems). When touching the README, regenerate those sections from `index.json` and the openvex documents rather than editing rows by hand.
