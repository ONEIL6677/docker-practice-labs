# Understanding Docker Intermediate Testing Stages

This project utilizes a multi-stage `Dockerfile` pattern that includes an **Intermediate Testing Stage** (or Artifact Pipeline Gate). 

Instead of just compiling code, this setup runs automated safety checks and test suites *inside* the isolated build environment. If a test fails, the build halts immediately, ensuring that broken or untested code can never accidentally reach your production environment or Docker Hub.

---

## The Build Pipeline Architecture

Our `Dockerfile` splits the build lifecycle into isolated, specialized layers:

```text
 ┌───────────────┐
 │   1. BASE     │ ──► Installs Go & copies raw source code.
 └───────┬───────┘
         │
         ├──► [Branch A] ──► 2. TESTER   ──► Runs 'go vet' & checks code quality.
         │                                   (🛑 Blocks build if code fails!)
         │
         └──► [Branch B] ──► 3. BUILDER  ──► Compiles source code into a binary.
                                 │
                                 ▼
                             4. FINAL    ──► Drops Go completely, copies binary,
                                             and runs the application (~9MB total).
```

---

## Why Use This Pattern?

*   **Zero-Dependency Production Images:** Your testing frameworks, linting tools, and raw source files are left behind. Your final image contains nothing but the runtime and the compiled binary.
*   **Guaranteed Consistency:** Tests run in the exact same container environment every time, eliminating the classic *"it works on my machine"* problem.
*   **Pipeline Fail-Safe:** If developer code contains syntactical bugs or failing logic, `docker build` will fail immediately with a non-zero exit code.

---

## How to Interact with the Stages

You can selectively trigger specific paths of this pipeline depending on whether you are working locally or configuring an automated Continuous Integration (CI) server.

### 1. Run the Complete Pipeline (Production Build)
To trigger the entire workflow—including running the quality checks, compiling the application, and producing the ultra-lean final image:
```bash
docker build -t my-app:latest .
```
*If `go vet` or your tests fail during this command, Docker will stop and will **not** output the `my-app:latest` image.*

### 2. Run Only the Tests (CI Pull Request Gates)
When a developer opens a pull request, you might only care if the tests pass, without needing to waste time compiling a final production image. You can tell Docker to stop running after a specific stage using the `--target` flag:
```bash
docker build --target tester .
```
*This is incredibly fast because it stops execution the moment the `tester` stage commands finish running.*

### 3. Debug the Test Environment Interactively
If your container tests are failing and you want to jump inside the container workspace to look around, inspect files, or run commands manually:
```bash
docker build --target tester -t test-debug .
docker run --rm -it test-debug /bin/sh
```

---

## Best Practices for Scaling This Template

To get the most out of your intermediate testing layer as your project grows, consider implementing these extensions:

1.  **Add Your Test Suite:** Expand the `tester` stage to run your unit tests alongside the compiler checks:
    ```dockerfile
    FROM base AS tester
    RUN go vet ./...
    RUN go test -v ./...
    ```
2.  **Utilize Build Cache:** Keep your source files separated from your dependency files (`go.mod` and `go.sum`). This ensures that re-running tests only takes a fraction of a second if you didn't add new external libraries.
