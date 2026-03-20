# Liquibase VEX Repository

Self-hosted [VEX](https://www.cisa.gov/sites/default/files/2023-04/minimum-requirements-for-vex-508c.pdf)
(Vulnerability Exploitability eXchange) repository for Liquibase products.
This repository provides vulnerability assessments in
[OpenVEX](https://github.com/openvex/spec) format, following the
[Trivy VEX Repository Specification v0.1](https://github.com/aquasecurity/trivy/blob/main/docs/docs/supply-chain/vex/repo.md).

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
| `vex-repository.json` | Repository manifest required by the VEX Repo Spec. Declares the repository name, spec version, download location, and update interval. |
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

## How this repository is updated

This repository is **auto-generated** from
[`vex/assessments.yaml`](https://github.com/liquibase/liquibase-pro)
in the liquibase-pro repository. When the assessments file is updated on
the `master` branch, a CI workflow dispatches to this repository and
regenerates all VEX content.

**Do not edit the VEX files in this repository directly.** Changes will be
overwritten on the next automated update. To add or modify a vulnerability
assessment, submit a PR to `liquibase-pro/vex/assessments.yaml`.

## References

- [VEX Repository Specification v0.1](https://github.com/aquasecurity/trivy/blob/main/docs/docs/supply-chain/vex/repo.md)
- [OpenVEX Specification](https://github.com/openvex/spec)
- [Trivy VEX documentation](https://aquasecurity.github.io/trivy/latest/docs/supply-chain/vex/)
- [CISA VEX Minimum Requirements](https://www.cisa.gov/sites/default/files/2023-04/minimum-requirements-for-vex-508c.pdf)

## License

This repository is licensed under the Apache License, Version 2.0.
See [LICENSE](LICENSE) for the full text.
