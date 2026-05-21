# DEPLOY-SMA — Gotify Server

> Source-build and browser-smoke notes for the `gotify` RL environment.

## Build

```bash
cd apps/gotify
docker build \
  --build-arg GO_VERSION=1.26.0 \
  --build-arg BUILD_JS=1 \
  --build-arg LD_FLAGS="-w -s -X main.Version=local -X main.BuildDate=$(date '+%F-%T') -X main.Commit=$(git rev-parse HEAD) -X main.Mode=prod" \
  --build-arg DEBIAN=bookworm-slim \
  -f docker/Dockerfile \
  -t tester-env-gotify:latest \
  .
```

- `GO_VERSION` is required (read from `go.mod` via `go mod edit -json`).
- `BUILD_JS=1` forces the JS frontend to rebuild inside the image.
- `DEBIAN=bookworm-slim` avoids the default `sid-slim` which can drift.

## Run

```bash
docker run -d --name tester-env-gotify \
  -p 8087:80 \
  -e GOTIFY_DEFAULTUSER_PASS=admin \
  -v /tmp/opencode/gotify-data:/app/data \
  tester-env-gotify:latest
```

- Local URL: `http://localhost:8087`
- Health check: `curl http://localhost:8087/health` → `{"health":"green","database":"green"}`

## Credentials

- Username: `admin`
- Password: `admin`
- Set via env var `GOTIFY_DEFAULTUSER_PASS` on first startup only.

## Database / Services

- Embedded SQLite (`data/gotify.db`) — no extra service required.
- Data directory also contains uploaded images and plugin files.

## Reset

```bash
docker stop tester-env-gotify
docker rm tester-env-gotify
# Volume files are root-owned; clean them from inside a container:
docker run --rm -v /tmp/opencode/gotify-data:/data busybox sh -c "rm -rf /data/*"
docker run -d --name tester-env-gotify \
  -p 8087:80 \
  -e GOTIFY_DEFAULTUSER_PASS=admin \
  -v /tmp/opencode/gotify-data:/app/data \
  tester-env-gotify:latest
```

With the RL lifecycle CLI:

```bash
apps/gotify/tester-env reset --run-id gotify-seed --port 8087
apps/gotify/tester-env deploy --run-id gotify-seed --port 8087
apps/gotify/tester-env seed --run-id gotify-seed --port 8087
apps/gotify/tester-env verify --run-id gotify-seed --port 8087
apps/gotify/tester-env reset --run-id gotify-seed --port 8087
```

## Deterministic Seed Data

- Seed command: `apps/gotify/tester-env seed --run-id <id> --port <port>`.
- Verify command: `apps/gotify/tester-env verify --run-id <id> --port <port>`.
- Credentials: `admin / admin`.
- Theme: commerce operations notification hub.
- Seeded applications: `Checkout API`, `Billing Worker`, `Security Alerts`, `Release Monitor`, `Warehouse Scanner`.
- Seeded clients: `Operations Wallboard`, `iPhone - Ops Lead`, `Warehouse Tablet Dock 4`, `Release Room Display`.
- Seeded messages: 18 total across the five applications, including `Payment provider latency high`, `Invoice retry queue draining`, `Privileged login from new location`, `Canary deploy paused`, and `Dock 4 scanner offline`.
- Message titles, bodies, priorities, application names, and client names are fixed. Gotify assigns message timestamps at API creation time.
- Browser-checker targets: after login, verify the message list is populated; the Apps view contains the five named applications; the Clients view contains the four named clients; filtering/opening app-specific messages shows known titles such as `Dock 4 scanner offline` under `Warehouse Scanner`.

## Browser Smoke Test

1. Open `http://localhost:8087` — login page loads without fatal errors.
2. Log in with `admin / admin`.
3. Click **Apps** → **Create Application** → name it `SmokeTestApp` → **Create**.
4. Assert `SmokeTestApp` appears in the apps list.
5. Reset, restart, and confirm the app is gone (clean state restored).

## Mutation Smoke

Example edit: `ui/src/user/Login.tsx` — change `<DefaultPage title="Login" …>` to `<DefaultPage title="Mutated Login" …>`. Rebuild the image, restart with a clean volume, and confirm the new title is visible on the login page.

## Baseline Ref

`ec8ce07152b10805165a98625defdd5b3ea34cd0`
