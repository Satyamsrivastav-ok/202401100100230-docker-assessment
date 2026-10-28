# 202401100100230-docker-assessment

## Stage 3: Trivy scan and remediation

The original backend image (`fest-backend:0.1`) was scanned with Trivy 0.76.1. The backend image was then rebuilt locally as `fest-backend:0.2` using `python:3.13.18-slim-bookworm`. Python dependencies are pinned in `backend/requirements.txt`; the image runs as UID/GID `10001:10001`. Pip and its bundled installer wheels are removed from the final runtime image after dependency installation.

The before and after scans used the same vulnerability scanner and table format. Counts below sum the OS and Python-package result sections in the full reports.

| Image | CRITICAL | HIGH | Docker image size |
|---|---:|---:|---:|
| `fest-backend:0.1` | 234 | 2,250 | 1,607,231,486 bytes (about 1.61 GB) |
| `fest-backend:0.2` | 2 | 59 | 204,709,108 bytes (about 205 MB) |

The required gate was run against `fest-backend:0.2`:

```powershell
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 fest-backend:0.2
```

Its actual exit code was **0**. A separate HIGH/CRITICAL scan using `--ignore-unfixed` found no remaining fixable HIGH or CRITICAL vulnerabilities. The full scan still reports two CRITICAL and 59 HIGH Debian OS findings: `CVE-2025-7458` in `libsqlite3-0` has no fixed version listed, and `CVE-2023-45853` in `zlib1g` is marked `will_not_fix`. The gate ignores unfixed findings; this image is not vulnerability-free.

Full scan reports:

- [`reports/trivy-before.txt`](./reports/trivy-before.txt) — original `fest-backend:0.1`
- [`reports/trivy-after.txt`](./reports/trivy-after.txt) — fixed `fest-backend:0.2`

The fixed image was functionally tested on the existing Compose database network without inserting or changing registrations. Its health endpoint returned HTTP 200 with the required REGNO and `database: true`; the registrations endpoint returned the two existing records. No Compose services or database volumes were restarted or removed.
