# Liquibase VEX Repository

Self-hosted [VEX](https://www.cisa.gov/sites/default/files/2023-04/minimum-requirements-for-vex-508c.pdf)
(Vulnerability Exploitability eXchange) repository for Liquibase products.
This repository provides vulnerability assessments in
[OpenVEX](https://github.com/openvex/spec) format, following the
[Trivy VEX Repository Specification v0.1](https://github.com/aquasecurity/trivy/blob/main/docs/docs/supply-chain/vex/repo.md).

> **This repository is auto-generated.** Do not edit the VEX files directly.
> Changes will be overwritten on the next automated update.
> To add or modify assessments, submit a PR to
> [`liquibase-pro/vex/assessments.yaml`](https://github.com/liquibase/liquibase-pro/blob/master/vex/assessments.yaml).
> See the [contributing guide](https://github.com/liquibase/liquibase-pro/blob/master/vex/CONTRIBUTING.md).

## How it fits together

```
liquibase-pro                           vex-repo                    Customer
+---------------------------+           +--------------------+      +----------------+
| vex/assessments.yaml      |           |                    |      |                |
|   (source of truth)       |           | vex-repository.json|      |  Trivy         |
|                           |           | index.json         |      |  --vex repo    |
| vex/generate-vex-repo.sh  |---------->| pkg/maven/         |----->|  (auto-fetch)  |
|                           | dispatch  |   .../vex.openvex  |      |                |
| .github/workflows/        |           |   .../vex.cdx.vex  |      +----------------+
|   vex-repo-dispatch.yml   |           |                    |
+---------------------------+           +--------------------+
```

Assessments are maintained in `liquibase-pro/vex/assessments.yaml`. When that
file changes on `master`, a GitHub Actions workflow dispatches to this
repository, regenerates all VEX content, and opens a pull request.

## What is VEX?

VEX is an industry-standard format for communicating whether known
vulnerabilities actually affect a software product. When a vulnerability
scanner flags a CVE, a VEX document can declare that the CVE is
**not exploitable** in context -- for example because the vulnerable code
path is not present, or because a newer version of the dependency includes
the fix.

## How to use with Trivy

Configure Trivy to use this repository alongside (or instead of) the
default [VEX Hub](https://github.com/aquasecurity/vexhub):

```yaml
# ~/.trivy/vex/repository.yaml
repositories:
  - name: default
    url: https://github.com/aquasecurity/vexhub
    enabled: true
  - name: liquibase
    url: https://github.com/liquibase/vex-repo
    enabled: true
```

Then run Trivy with the `--vex repo` flag:

```bash
# Scan a Liquibase Docker image
trivy image --vex repo liquibase/liquibase:latest

# Scan a local Liquibase installation
trivy fs --vex repo /path/to/liquibase/

# Scan the root filesystem of a Liquibase container
trivy rootfs --vex repo /liquibase/

# Show suppressed vulnerabilities alongside active ones
trivy image --vex repo --show-suppressed liquibase/liquibase:latest
```

Trivy will automatically download and cache the VEX documents from this
repository, suppressing any CVEs that have been assessed as not exploitable.

## Formats available

Two VEX formats are generated for each package:

| File | Format | Purpose |
|------|--------|---------|
| `vex.openvex.json` | OpenVEX 0.2.0 | Primary format for Trivy `--vex repo` suppression. Indexed in `index.json`. |
| `vex.cdx.vex.json` | CycloneDX 1.6 | Full advisory data: CVSS ratings, CWEs, recommendations, version ranges. |

Trivy uses the OpenVEX file via the index. The CycloneDX file is provided for
consumers that need richer advisory content (severity ratings, remediation
guidance, external references).

## Repository structure

```
vex-repo/
  vex-repository.json                              # VEX Repo Spec v0.1 manifest
  index.json                                       # PURL-to-file mapping
  pkg/
    maven/
      org.liquibase/
        liquibase-core/
          vex.openvex.json                          # OpenVEX document (Trivy suppression)
          vex.cdx.vex.json                          # CycloneDX 1.6 VEX (full advisory)
```

| File | Purpose |
|------|---------|
| `vex-repository.json` | Repository manifest required by the VEX Repo Spec. Declares the repository name, spec version, download location, and update interval (24h). |
| `index.json` | Maps package PURLs to their VEX document paths. Trivy uses this to locate the correct VEX file for a given package. |
| `pkg/.../vex.openvex.json` | OpenVEX document containing all vulnerability assessments for `pkg:maven/org.liquibase/liquibase-core`. Used by Trivy for suppression. |
| `pkg/.../vex.cdx.vex.json` | CycloneDX 1.6 VEX document with full advisory fields (ratings, recommendations, CWEs). |

## Current assessments

The following vulnerabilities are currently assessed:

| CVE | Package | Status | Justification |
|-----|---------|--------|---------------|
| CVE-2022-0839 / GHSA-jvfv-hrrc-6q72 | org.liquibase:liquibase-core | not_affected | code_not_present |
| CVE-2014-8180 | com.liquibase.ext:liquibase-commercial-mongodb | not_affected | code_not_present |
| CVE-2022-0839 | com.liquibase:liquibase-license-utility | not_affected | code_not_present |
| CVE-2023-36415 | com.azure:azure-identity | not_affected | code_not_present |
| CVE-2024-35255 | com.azure:azure-identity | not_affected | code_not_present |
| CVE-2024-35255 | com.microsoft.azure:msal4j | not_affected | code_not_present |
| CVE-2024-45394 | com.instaclustr:cassandra-driver-kerberos | not_affected | code_not_present |

## Verifying VEX documents manually

```bash
# Inspect the OpenVEX document
jq '.statements[] | {vuln: .vulnerability["@id"], status, justification}' \
  pkg/maven/org.liquibase/liquibase-core/vex.openvex.json

# Inspect the CycloneDX VEX document
jq '.vulnerabilities[] | {id, status: .analysis.state, justification: .analysis.justification}' \
  pkg/maven/org.liquibase/liquibase-core/vex.cdx.vex.json

# Verify OpenVEX with vexctl (requires vexctl installed)
vexctl verify pkg/maven/org.liquibase/liquibase-core/vex.openvex.json
```

## How this repository is updated

This repository is auto-generated from
[`vex/assessments.yaml`](https://github.com/liquibase/liquibase-pro/blob/master/vex/assessments.yaml)
in the liquibase-pro repository. The update flow:

1. A change to `assessments.yaml` is merged to `master` in liquibase-pro
2. The [`vex-repo-dispatch.yml`](https://github.com/liquibase/liquibase-pro/blob/master/.github/workflows/vex-repo-dispatch.yml) workflow triggers
3. It dispatches to this repository's [`update-vex.yaml`](.github/workflows/update-vex.yaml) workflow
4. That workflow sparse-checkouts `liquibase-pro/vex/`, runs `generate-vex-repo.sh`, validates the output, and opens a PR

The workflow can also be triggered manually via `workflow_dispatch`.

## References

- [VEX Repository Specification v0.1](https://github.com/aquasecurity/trivy/blob/main/docs/docs/supply-chain/vex/repo.md)
- [OpenVEX Specification](https://github.com/openvex/spec)
- [Trivy VEX documentation](https://aquasecurity.github.io/trivy/latest/docs/supply-chain/vex/)
- [CISA VEX Minimum Requirements](https://www.cisa.gov/sites/default/files/2023-04/minimum-requirements-for-vex-508c.pdf)
- [Contributing assessments](https://github.com/liquibase/liquibase-pro/blob/master/vex/CONTRIBUTING.md)

## License

This repository is licensed under the Apache License, Version 2.0.
See [LICENSE](LICENSE) for the full text.
