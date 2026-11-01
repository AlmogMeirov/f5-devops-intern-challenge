# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output

- `./scripts/run-tests.sh` stops at `docker compose build`; the last line is only
  `apt-get install ... did not complete successfully: exit code: 100`.
- The real cause is ~100 lines higher, in the middle of the apt output:

```
Setting up nginx-common (1.24.0-2ubuntu7.x) ...
Configuration file '/etc/nginx/nginx.conf'
 ==> File on system created by you or by a script.
 ==> File also in package provided by package maintainer.
   What would you like to do about it ?  Your options are:
    Y or I  : install the package maintainer's version
    N or O  : keep your currently-installed version
*** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
 end of file on stdin at conffile prompt
...
Errors were encountered while processing:
 nginx-common
 nginx
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

### How I diagnosed it

- `docker compose build proxy --progress=plain 2>&1 | grep -n -iE "conffile|dpkg: error|E: "`
  to find the *first* error instead of the last line.
- `apt-get` resolved and downloaded every package fine; the failure is in the
  dpkg **configure** phase of `nginx-common`, so it is not a missing package.

### Root cause

- The Dockerfile ran `COPY nginx.conf /etc/nginx/nginx.conf` **before**
  `apt-get install nginx`.
- `/etc/nginx/nginx.conf` is a dpkg *conffile* of `nginx-common`. When dpkg finds
  a conffile already on disk that it didn't install, it assumes an admin edited
  it and asks interactively whether to keep it or replace it.
- `docker build` has no stdin, so dpkg reads EOF, aborts the package, and apt exits 100.
  `-y` doesn't help: it answers apt's prompts, not dpkg's conffile prompts.

### Fix

- Moved `COPY nginx.conf` to **after** the `apt-get install` layer: the package
  installs cleanly with its default file, then we overwrite it.
- Rejected alternative: `-o Dpkg::Options::="--force-confold"`. It works, but it
  only hides the ordering problem behind a flag.
- (The final image no longer uses apt at all — see Bonus 1 — but this was the
  minimal fix for the defect as reported.)

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output

- `/nginx-health` passes (nginx answers it itself), `/api/info` fails:

```
test-runner-1 | TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway
proxy-1       | [error] connect() failed (111: Connection refused) while connecting to upstream,
                request: "GET /api/info HTTP/1.1", upstream: "http://127.0.0.1:8080/api/info"
```

### How I diagnosed it

- `docker compose logs proxy` → the `upstream:` field shows nginx dialing `127.0.0.1:8080`.
- `nginx.conf` → `upstream app_backend { server 127.0.0.1:8080; }`.

### Root cause

- Every container has its **own network namespace**. Inside the proxy container,
  `127.0.0.1` is the proxy container itself, where nothing listens on 8080.
  (It "works" on a laptop only when everything runs on one host.)

### Fix

- `server backend:8080;`. Compose puts all services on a shared network where
  Docker's embedded DNS (127.0.0.11) resolves the **service name** `backend` to
  the backend container's IP.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output

- Same `502 Bad Gateway` / `Connection refused`, but now against the *correct* IP
  (`upstream: "http://172.18.0.2:8080/api/info"`). It failed on 3/3 runs.
- In the same log, the backend prints `Backend listening on port 8080...` only
  **after** the test runner had already crashed.

### How I diagnosed it

- `docker compose logs -t backend` (timestamps): one `loaded config key` line
  every 2s, and `listening` ~8s after start.
- Polled `curl localhost:8080/health` from a fresh `docker compose up -d backend`:
  first success after **8.2s**.
- `backend/app.py`: `load_config()` does `time.sleep(CONFIG_FETCH_SECONDS=2)` for
  each of 4 keys **before** `HTTPServer(...)` binds the port.

### Root cause

- A startup **race**. `depends_on` (short form) only waits until the dependency
  container has *started*, not until it is *ready*. test-runner started ~1s after
  the backend, which needs ~8s before it accepts connections.

### Fix, and why I chose it over the alternatives

- **Fix:** Compose healthchecks plus `depends_on: condition: service_healthy`:
  - `backend`: probe its own `GET /health` (with Python, which is guaranteed to be
    in a python image, so no dependency on curl/wget).
  - `proxy`: probe `GET /nginx-health`; it waits for `backend` to be healthy.
  - `test-runner`: waits for `proxy` to be healthy, so the whole chain is ready.
  - `start_period: 60s` is a ceiling, not a delay: the service becomes healthy on
    the first successful probe (every 2s). If the backend never comes up, compose
    exits non-zero with `dependency failed to start: container ... is unhealthy`
    instead of hanging (verified by breaking the backend port: clean exit 1 after 66s).
- **Verified:** 10/10 consecutive green runs, plus a run from a fresh clone.
- **Alternatives rejected:**
  - `sleep 10` in the script or in test-runner: a guess. It's slow when the backend
    is fast and flaky when it's slow (e.g. a loaded CI runner).
  - Retry loop inside `test_integration.py`: changes the tests and can mask real
    502s. The ground rules forbid weakening the tests.
  - `CONFIG_FETCH_SECONDS=0`: changes the third-party service's behaviour to dodge
    the problem. Production would still take 8s to start, so the stack would be
    tested in an unrealistic state.
  - A `wait-for-it` script in test-runner: re-implements what Compose already
    provides, and test-runner would need to know the backend's address (it should
    only know the proxy).
- **Related finding:** `docker compose logs` showed `backend-1 exited with code 137`
  on every teardown. `python app.py` runs as **PID 1** with no SIGTERM handler, and
  the kernel ignores default-action signals for PID 1, so `docker stop` waited the
  full 10s grace period and then SIGKILLed it. Added `init: true` (tini as PID 1
  forwards SIGTERM): exit 143, and each run is ~9s faster (28s → 19s).

---

## Defect 4: CI pipeline fails

### Symptom / error output

- First CI run on `main` (CI Pipeline #1, "Initial commit"), step *Execute test suite*:

```
Run ./scripts/run-tests.sh
/home/runner/work/_temp/1343ff25-ab2e-4eb5-bdcd-c117a5dabb9f.sh: line 1: ./scripts/run-tests.sh: Permission denied
Error: Process completed with exit code 126.
```

### How I diagnosed it

- Exit code **126** means "found but not executable" (127 would be "not found").
- `git ls-files -s scripts/run-tests.sh` → `100644`: git stores the file
  **without** the executable bit, so every fresh checkout (CI, a new clone) gets a
  non-executable script.

### Root cause

- The script was committed with mode `100644`. A local `chmod +x` makes it work on
  that one machine only; git tracks the mode in the index/tree, and that's what
  every clone receives.

### Fix

- `git update-index --chmod=+x scripts/run-tests.sh` (plus `chmod +x` on the
  working copy) and committed → mode `100755`. Verified with a fresh `git clone`.
- Gotcha I hit: `update-index` changes only the index. With the working copy still
  `644`, a later `git add -A` silently staged `644` again and reverted the fix.
  Caught it with `git show --summary` (`mode change 100755 => 100644`) and amended.
- Added `.gitattributes` with `*.sh text eol=lf`, so a Windows clone with
  `core.autocrlf=true` can't turn the shebang into `bash\r` (another "works for me,
  fails for the next person" failure).
- Not chosen: changing CI to `bash ./scripts/run-tests.sh` or adding `chmod +x` in
  the workflow. That makes CI green but leaves the README's `./scripts/run-tests.sh`
  broken for every developer, and treats the symptom in one place only.
- Small addition: CI now prints built image sizes after the tests (`if: always()`).

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

> **1. Smaller proxy image**
>
> - `FROM ubuntu:24.04` + `apt-get install nginx curl htop vim git` →
>   `FROM nginx:1.30-alpine-slim` (official image, Alpine, nginx only — no njs/perl
>   modules), pinned to the 1.30 *stable* branch (gets patch releases, never a
>   surprise minor/major jump).
> - Removed curl, htop, vim and git: debugging/attack tooling that the proxy never
>   uses (git + curl inside a compromised container are an attacker's toolkit).
>   166 dpkg packages → a handful of apk packages.
> - Removed the unused stock `conf.d/default.conf`.
> - **Size** (`docker image ls`, Docker Desktop on Windows, same machine):
>
>   | Image | Size |
>   |---|---|
>   | before — ubuntu:24.04 + nginx curl htop vim git | **443 MB** |
>   | after — nginx:1.30-alpine-slim | **20.6 MB** |
>
>   **~95% smaller (~21x).** Less to pull on every deploy, faster cold starts, and a
>   much smaller attack surface (fewer packages = fewer CVEs to patch).
>
>   How to reproduce: `git checkout HEAD~2 -- proxy && docker compose build proxy && docker image ls`,
>   then `git checkout HEAD -- proxy` and repeat. CI also prints the "after" size on every run.
>
> **2. Non-root proxy**
>
> - `USER 101:101` (the `nginx` user that ships in the official image; numeric so
>   Kubernetes `runAsNonRoot` can verify it).
> - `listen 8080` instead of 80. Binding ports <1024 needs `CAP_NET_BIND_SERVICE`,
>   which a non-root process doesn't have on a default Linux host or in Kubernetes.
>   Compose maps `8081:8080`, and the tests reach it via `PROXY_URL=http://proxy:8080`
>   (an environment value in docker-compose.yml; the test code is unchanged).
> - `pid /tmp/nginx.pid` and all `*_temp_path` → `/tmp`; the defaults
>   (`/var/run`, `/var/cache/nginx`) are root-owned.
> - No `user` directive: it only matters when the master starts as root.
> - Verified: `id` in the image → `uid=101(nginx)`; master and workers run as uid
>   101; `/proc/1/status` → `CapEff: 0`.
>
> **3. Additional security / performance**
>
> - **Runtime (Compose):** `read_only: true` with `tmpfs: /tmp` (an attacker can't
>   drop files or modify config; verified with `touch` → "Read-only file system"),
>   `cap_drop: [ALL]`, and `no-new-privileges:true` (verified `NoNewPrivs: 1`).
> - **Config is root-owned and read-only** for the runtime user: a compromised
>   worker can't rewrite its own config.
> - **nginx:** `server_tokens off` (no version in the `Server` header or error
>   pages); `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` headers;
>   `limit_except GET` (the backend only implements GET; `POST` → 403 at the edge);
>   `client_max_body_size 1k`; `X-Forwarded-Proto` passed upstream.
> - **Performance/ops:** `worker_processes auto`; upstream timeouts of 5s
>   connect / 15s read instead of the 60s defaults (fail fast); `/nginx-health`
>   made an exact-match location (`location =`) and excluded from the access log
>   (it's probed every 2s).
> - **Considered and skipped:** upstream `keepalive`. `app.py` uses
>   `BaseHTTPRequestHandler`, which speaks HTTP/1.0 and closes every connection,
>   so pooling would be a no-op.

---

## Anything you would do differently with more time

- **Upstream re-resolution:** nginx resolves `backend` once at startup. If the
  backend container is recreated with a new IP, the proxy keeps the stale IP
  until it restarts. Fix on nginx ≥ 1.27.3: `zone` + `server backend:8080 resolve;`
  + `resolver 127.0.0.11 valid=10s;`.
- **Pin base images by digest** (`nginx:1.30-alpine-slim@sha256:...`) with
  Dependabot/Renovate bumping it, for reproducible builds.
- **Image scanning in CI** (Trivy/Grype) and `hadolint` for Dockerfiles; fail the
  build on critical CVEs.
- **Backend hardening from the outside** (without touching `app.py`): run it as
  a non-root user with `read_only` and `cap_drop: ALL` in Compose too.
- **Rate limiting** (`limit_req`) on `/api/`, once real traffic patterns are known.
- **CI:** a `concurrency` group to cancel superseded runs, and dump
  `docker compose logs` as an artifact on failure.
