# Docker & Container Security Practical Assessment

## Project overview

FestPass is a three-tier registration application. An Nginx frontend serves the registration page and proxies API requests to a Flask backend. The backend stores registrations in PostgreSQL.

## Student and release details

| Item | Value |
|---|---|
| Student name | **Satyam Srivastav** |
| Registration number | `202401100100230` |
| Docker Hub username | `satyamsri85` |
| Docker Hub repository | [satyamsri85/fest-backend](https://hub.docker.com/r/satyamsri85/fest-backend) |
| Application URL | `http://localhost:8030` |
| Current Compose backend image | `satyamsri85/fest-backend:1.0.0` |

## Technologies and security tools

- Docker Engine and Docker Compose
- Python 3.13, Flask, and psycopg2
- PostgreSQL 17
- Nginx 1.30
- Trivy 0.76.1 for image vulnerability scanning
- Cosign v3.1.4 for image signing and verification
- Git and GitHub

### AI assistance

GitHub Copilot in VS Code assisted with project configuration, troubleshooting, and assessment documentation. Reported implementation and verification details were checked against repository files and command output. No other AI tool use is documented.

## Architecture

The Compose project contains three services:

| Service | Image / implementation | Purpose and network |
|---|---|---|
| `frontend` | `nginx:1.30-alpine`; serves `frontend/index.html` and mounts `frontend/nginx.conf` read-only | Publishes host port `8030` to container port `80`; joins only the `frontend` network and proxies `/api/` to `backend:5000` |
| `backend` | `satyamsri85/fest-backend:1.0.0` | Flask API on container port `5000`; joins `frontend` and `backend_db`; waits for the database health check |
| `db` | `postgres:17-alpine` | PostgreSQL database on the private `backend_db` network; data persists in the named `festpass_data` volume |

The PostgreSQL service has a `pg_isready` health check. The backend exposes `GET /api/health`, which reports its version, registration number, and database connection status; Compose does not define a separate backend container health check.

## Setup and run

Run these commands from the repository root in PowerShell.

1. Copy the environment template:

   ```powershell
   Copy-Item .env.example .env
   ```

2. Edit `.env` and replace the template value with a unique local `POSTGRES_PASSWORD`. Keep the actual value private; `.env` is ignored by Git.

3. Validate and start the application:

   ```powershell
   docker compose config --quiet
   docker compose up -d
   docker compose ps
   ```

4. Open `http://localhost:8030`, or check the API and frontend:

   ```powershell
   Invoke-RestMethod http://localhost:8030/api/health
   Invoke-RestMethod http://localhost:8030/api/registrations
   Invoke-WebRequest http://localhost:8030 -UseBasicParsing
   ```

The Compose configuration reads `POSTGRES_PASSWORD` from `.env` for PostgreSQL and the backend database connection. It sets `REGNO=202401100100230`, `APP_VERSION=1.0.0`, and the database host/name/user in Compose. `.env.example` contains only a placeholder and must not be used as a real password.

To stop the services without deleting persistent registrations, use `docker compose down` (without `-v`). Do not remove the `festpass_data` volume unless intentionally discarding the database.

## Assessment work and verification

### Stage 1 — Repository and project setup

The assessment project is in a Git repository with `origin` configured to `https://github.com/Satyamsrivastav-ok/202401100100230-docker-assessment.git`. The `.gitignore` excludes `.env`, `*.key`, and other credential-like files. The public verification key `cosign.pub` is tracked; `.env` and the private `cosign.key` are ignored and are not tracked.

Evidence files currently present: [`screenshots/s1-1.png`](./screenshots/s1-1.png), [`screenshots/s1-2.png`](./screenshots/s1-2.png), and [`screenshots/s1-3image.png`](./screenshots/s1-3image.png).

### Stage 2 — Compose application

The Compose file defines the frontend, backend, and database services, two separate bridge networks, a database health check, and the persistent `festpass_data` volume. The frontend is exposed at port `8030`; PostgreSQL is not published to a host port.

Evidence files currently present: [`screenshots/s2-1.png`](./screenshots/s2-1.png), [`screenshots/s2-2.png`](./screenshots/s2-2.png), [`screenshots/s2-3.png`](./screenshots/s2-3.png), and [`screenshots/s2-4.png`](./screenshots/s2-4.png).

### Stage 3 — Trivy scan and remediation

The original `fest-backend:0.1` image was scanned with Trivy 0.76.1. The backend was then rebuilt as `fest-backend:0.2` on `python:3.13.18-slim-bookworm`; dependencies are pinned in `backend/requirements.txt`, pip and bundled installer wheels are removed from the runtime image, and the application runs as UID/GID `10001:10001`.

The counts below are taken from the saved reports. The original report has separate OS and Python-package summaries; its totals below combine those summaries. The fixed report has 242 Debian findings and zero Python-package findings.

| Image | CRITICAL | HIGH | Other severities | Total findings |
|---|---:|---:|---|---:|
| `fest-backend:0.1` | 234 | 2,250 | 158 UNKNOWN, 3,211 LOW, 7,528 MEDIUM | 13,381 |
| `fest-backend:0.2` | 2 | 59 | 1 UNKNOWN, 89 LOW, 91 MEDIUM | 242 |

The fixed image is not vulnerability-free: the report retains two CRITICAL and 59 HIGH Debian findings. In particular, `CVE-2025-7458` in `libsqlite3-0` has no fixed version listed, and `CVE-2023-45853` in `zlib1g` is marked `will_not_fix`.

The required CRITICAL gate was run against `fest-backend:0.2`:

```powershell
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 fest-backend:0.2
```

Its recorded exit code was `0`. A separate HIGH/CRITICAL scan using `--ignore-unfixed` found no remaining fixable HIGH or CRITICAL findings. The full scan still reports the findings listed above; passing the gate does not mean the image has no vulnerabilities.

Full reports: [`reports/trivy-before.txt`](./reports/trivy-before.txt) and [`reports/trivy-after.txt`](./reports/trivy-after.txt).

Evidence files currently present: [`screenshots/s3-1.png`](./screenshots/s3-1.png), [`screenshots/s3-2.png`](./screenshots/s3-2.png), and [`screenshots/s3-3.png`](./screenshots/s3-3.png).

### Stage 4 — Versioned Docker Hub release and Compose deployment

The backend Dockerfile accepts an `APP_VERSION` build argument and the API reports that value from `/api/health`. The versioned image was built with `APP_VERSION=1.0.0`; Docker Hub currently reports both release tags at the same digest:

| Docker Hub tag | Digest |
|---|---|
| `satyamsri85/fest-backend:1.0.0` | `sha256:0ab7a0be268e85152022ec2b45133ddf919c9ccf4a575d0cf8e5d0a71e785c25` |
| `satyamsri85/fest-backend:1.0` | `sha256:0ab7a0be268e85152022ec2b45133ddf919c9ccf4a575d0cf8e5d0a71e785c25` |

The current Compose file uses the registry image directly and does not build the backend locally. The following deployment checks were run during this documentation update: `docker compose config --quiet` succeeded; all three services were running, with PostgreSQL healthy; `/api/health` reported registration number `202401100100230`, version `1.0.0`, and `database: true`; the registrations endpoint returned two records; and the `festpass_data` volume exists. No registration values are reproduced here.

The Docker Hub tags were queried through the public registry API. The screenshots listed below are present in the repository; their presence alone is not a substitute for checking each image's contents during assessment.

#### Build and publish commands

```powershell
docker build --pull --build-arg APP_VERSION=1.0.0 -t fest-backend:1.0.0 ./backend
docker tag fest-backend:1.0.0 satyamsri85/fest-backend:1.0.0
docker tag fest-backend:1.0.0 satyamsri85/fest-backend:1.0
docker push satyamsri85/fest-backend:1.0.0
docker push satyamsri85/fest-backend:1.0
```

#### Deploy from the registry

```powershell
docker compose config --quiet
docker compose pull backend
docker compose up -d
docker compose ps
Invoke-RestMethod http://localhost:8030/api/health
Invoke-RestMethod http://localhost:8030/api/registrations
Invoke-WebRequest http://localhost:8030 -UseBasicParsing
```

#### Rollback

No earlier, different public release was verified in this assessment; `1.0` and `1.0.0` refer to the same digest and are not separate rollback versions. If a rollback is needed, first select a previously verified known-good image tag or digest, update only the backend image reference and matching `APP_VERSION` in `compose.yaml`, then run `docker compose pull backend` and `docker compose up -d backend`. Keep the database service and `festpass_data` volume unchanged. Do not roll back to an unverified image or remove the volume.

Evidence files currently present: [`screenshots/s4-1.png`](./screenshots/s4-1.png), [`screenshots/s4-2.png`](./screenshots/s4-2.png), and [`screenshots/s4-3.png`](./screenshots/s4-3.png).

### Stage 5 — Cosign signing and unsigned-image verification

Cosign v3.1.4 is installed. The private key `cosign.key` is local and ignored by Git; the public key [`cosign.pub`](./cosign.pub) is tracked. The private key material and its password are intentionally not included in this documentation.

The `1.0.0` image was signed by immutable digest, not only by tag:

```text
docker.io/satyamsri85/fest-backend@sha256:0ab7a0be268e85152022ec2b45133ddf919c9ccf4a575d0cf8e5d0a71e785c25
```

Verification with the public key succeeded. The actual `cosign verify` output confirmed the Cosign claims, verified the transparency-log entry (offline from the local transparency-log data), and verified the signature against the supplied public key.

The `1.0.1` image was published without a signature. Docker Hub currently reports this digest:

```text
docker.io/satyamsri85/fest-backend@sha256:8703216dd9375e39e861c863fb8567e98708d505542040494a66dffda4a9f528
```

Verification with `cosign.pub` returned `Error: no signatures found` (exit code `10`), the expected rejection for this unsigned-image test. `1.0.1` is deliberately unsigned for Stage 5.4 and is not the image used by the current Compose deployment.

Reproduce the verification checks with:

```powershell
$cosignPath = "$env:LOCALAPPDATA\Programs\cosign\cosign.exe"
& $cosignPath verify --key cosign.pub "docker.io/satyamsri85/fest-backend@sha256:0ab7a0be268e85152022ec2b45133ddf919c9ccf4a575d0cf8e5d0a71e785c25"
& $cosignPath verify --key cosign.pub "docker.io/satyamsri85/fest-backend@sha256:8703216dd9375e39e861c863fb8567e98708d505542040494a66dffda4a9f528"
```

The first command should verify successfully. The second should fail with `no signatures found`; that expected nonzero exit is evidence that the intentionally unsigned image has no signature.

Evidence files currently present (lowercase filenames on disk): [`screenshots/s5-1.png`](./screenshots/s5-1.png), [`screenshots/s5-2.png`](./screenshots/s5-2.png), [`screenshots/s5-3.png`](./screenshots/s5-3.png), and [`screenshots/s5-4.png`](./screenshots/s5-4.png).

## Screenshot and evidence inventory

The following screenshot files were present under `screenshots/` when this README was reviewed:

- Stage 1: `s1-1.png`, `s1-2.png`, `s1-3image.png`
- Stage 2: `s2-1.png`, `s2-2.png`, `s2-3.png`, `s2-4.png`
- Stage 3: `s3-1.png`, `s3-2.png`, `s3-3.png`
- Stage 4: `s4-1.png`, `s4-2.png`, `s4-3.png`
- Stage 5: `s5-1.png`, `s5-2.png`, `s5-3.png`, `s5-4.png`

Stage 5 evidence is intended to show, respectively: secret-ignore/public-key tracking status without private-key contents; successful signing of the `1.0.0` digest; successful verification of signed `1.0.0`; and rejection of unsigned `1.0.1`. These files exist, but an assessor should confirm their visible contents meet the evidence requirements.

## Known limitations and incomplete items

- The `fest-backend:0.2` Trivy report still contains two CRITICAL and 59 HIGH Debian findings, including findings without a fix and one marked `will_not_fix`.
- `1.0.1` is intentionally unsigned for the Stage 5.4 test; the current Compose deployment remains on signed `1.0.0`.
- No GitHub push was performed during this documentation update; the local branch state should be reviewed before submission.
