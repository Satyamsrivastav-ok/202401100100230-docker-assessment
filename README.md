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

## Stage 4: Version and deploy from Docker Hub

The backend image uses the pinned `python:3.13.18-slim-bookworm` base, runs as UID/GID `10001:10001`, and exposes container port 5000. `backend/Dockerfile` accepts `APP_VERSION` as a build argument and stores it in the runtime environment; the application reports that value from `GET /api/health`.

Build and tag the release from the `backend/` build context:

```powershell
docker build --pull --build-arg APP_VERSION=1.0.0 -t fest-backend:1.0.0 ./backend
docker tag fest-backend:1.0.0 satyamsri85/fest-backend:1.0.0
docker tag fest-backend:1.0.0 satyamsri85/fest-backend:1.0
```

The Docker Hub repository is [satyamsri85/fest-backend](https://hub.docker.com/r/satyamsri85/fest-backend). Both release tags were verified through Docker Hub's public repository API as pointing to the same manifest digest:

| Tag | Registry digest |
|---|---|
| `1.0.0` | `sha256:0ab7a0be268e85152022ec2b45133ddf919c9ccf4a575d0cf8e5d0a71e785c25` |
| `1.0` | `sha256:0ab7a0be268e85152022ec2b45133ddf919c9ccf4a575d0cf8e5d0a71e785c25` |

To publish the tags from an authenticated Docker CLI:

```powershell
docker push satyamsri85/fest-backend:1.0.0
docker push satyamsri85/fest-backend:1.0
```

Docker Hub currently reports both tags as public. The remote digest above matches the local versioned image ID. A successful push terminal capture is still required as assessment evidence; this README records the verified registry state, not a claim that a push was performed during this work.

Compose runs the published image without a local build. Its backend service retains the REGNO, database settings, dependency on the healthy database, and both isolated networks. Deploy with:

```powershell
docker compose config --quiet
docker compose pull backend
docker compose up -d
docker compose ps
Invoke-RestMethod http://localhost:8030/api/health
Invoke-RestMethod http://localhost:8030/api/registrations
Invoke-WebRequest http://localhost:8030 -UseBasicParsing
```

Verify the health response has `regno` `202401100100230`, `version` `1.0.0`, and `database` `true`; verify the registrations response still contains the existing records. The database uses the named `festpass_data` volume; do not use `docker compose down -v` during deployment or rollback.

### Rollback

No earlier public release tag was verified when this section was written. If a rollback is needed, select a previously verified known-good image tag or digest, update only the backend `image` and matching `APP_VERSION` in `compose.yaml`, then run `docker compose pull backend` and `docker compose up -d backend`. Keep the DB service and `festpass_data` volume unchanged. Do not roll back to an unverified tag or remove the volume.

The Stage 4 screenshots (`screenshots/S4-1.png`, `screenshots/S4-2.png`, and `screenshots/S4-3.png`) must be captured manually; they are not generated or included by these instructions.
