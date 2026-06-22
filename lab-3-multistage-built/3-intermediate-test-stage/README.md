# High-Performance Docker Artifact & Testing Pipeline

This repository implements a production-hardened, multi-stage `Dockerfile` pipeline. It optimizes build speeds using layer caching, enforces code quality with dead-end testing gates, and minimizes attack vectors by shipping a non-root, ultra-lean final runtime image (~9MB).

---

## The Optimized Pipeline Pipeline

```text
       ┌────────────────────────┐
       │        1. BASE         │ ──► Copies go.mod ➔ Runs 'go mod download'
       └───────────┬────────────┘     (Cached securely until modules change!)
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
   ┌───────────┐       ┌───────────┐
   │ 2. TESTER │       │3. BUILDER │ ──► Compiles fully static production
   └───────────┘       └─────┬─────┘     binary with CGO completely disabled.
   Runs 'go vet'             │
   Runs 'go test'            ▼
   (Halts build        ┌───────────┐
    on failure)        │ 4. FINAL  │ ──► Drops Go SDK, runs as non-root
                       └───────────┘     'appuser' on a bare Alpine base.
```

---

## Core Performance & Security Enhancements

1. **Deterministic Layer Caching**: By processing `go.mod` and `go.sum` before copying source code, changing a single line of text in `main.go` will **not** trigger a re-download of your external libraries. Subsquent local builds drop from minutes to under 2 seconds.
2. **Non-Root Execution (`appuser`)**: Standard containers run as `root` inside their namespace. This configuration strips away root privileges (`USER appuser`) in the final image, containing runtime exploits from breaching your host machine.
3. **Static Binary Compilation**: Disabling CGO (`CGO_ENABLED=0`) packs all required execution dependencies directly inside the `myapp` binary file, removing any shared system library prerequisites.

---

## Production & CI/CD Command Guide

### Run Full Pipeline (Local Release / Production Build)
Compiles, tests, and builds the production-ready secure container:
```bash
docker build -t my-app:latest .
```
*Note: If unit tests or lints fail inside the container, Docker breaks execution immediately and skips generating the output image.*

### Execute Tests Only (Fast Pull-Request Verification)
Run this command inside automated CI environments (like GitHub Actions) to validate code health without wasting CPU cycles compiling production distributions:
```bash
docker build --target tester .
```

### Debug the Test Environment Manually
If automated container tests fail and you need to drop into a live terminal inside the test container setup to run files manually:
```bash
docker build --target tester -t app-debug .
```
```bash
docker run --rm -it app-debug /bin/sh
```
