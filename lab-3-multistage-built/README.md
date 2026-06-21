# Docker Multi-Stage Builds Practice Guide
---

## 1. The Core Concept

Without multi-stage builds, your final image often contains everything used to *build* the app — compilers, source code, package managers, dev dependencies even though none of that is needed to *run* it.

A multi-stage build lets you use multiple `FROM` statements in one Dockerfile. Each `FROM` starts a new, isolated **stage**. You can selectively copy only the files you need (like a compiled binary) from an earlier stage into a later, much smaller final stage and Docker discards everything else.

**Why it matters:**
- **Smaller images** final image only has the runtime, not the whole toolchain.
- **Smaller attack surface** no compilers, source code, or secrets baked into the shipped image.
- **One Dockerfile, no extra scripts** no need for separate build scripts that build outside Docker and then `COPY` artifacts in.
- **Faster pulls/deploys** fewer layers, less data to transfer.

---

## 2. Basic Syntax

```dockerfile
# Stage 1: give it a name with "AS"
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o myapp .

# Stage 2: a fresh, minimal base
FROM alpine:3.20
WORKDIR /app
# Copy ONLY the compiled binary from the "builder" stage
COPY --from=builder /src/myapp .
CMD ["./myapp"]
```

Key points:
- `AS builder` names the stage so it can be referenced later.
- `COPY --from=builder <path-in-that-stage> <path-here>` pulls files across stages.
- You can also `COPY --from=<image-name>` to copy from a completely separate, unrelated image (not just a stage in the same Dockerfile).
- Only the **last** stage in the file becomes the final image, unless you target an earlier one explicitly (see Section 5).

---

## 3. Practice Example — Go app, builder vs. final image

This is the clearest way to *see* multi-stage builds working, because the size difference between stages is dramatic.

**`practice/main.go`**
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello from a multi-stage build!")
}
```

**`practice/Dockerfile`**
```dockerfile
# ==============================================================================
# STAGE 1: The Build Environment
# Purpose: Compiles the source code into a single, independent executable binary.
# ==============================================================================

# Use the official Go image based on Alpine Linux to keep the build stage small
# rename stage one as builder so that it can be called in stage2
FROM golang:1.22-alpine AS builder 

# Create and switch into a dedicated project folder inside the build environment
WORKDIR /src

# Copy your local Go source code file from your computer into the current container directory (/src)
COPY main.go .

# Compile the Go application with specific cross-compilation environment variables:
# - CGO_ENABLED=0: Disables C libraries to make the binary fully static and portable
# - GOOS=linux: Forces the compiler to output a Linux-compatible executable
# - -o myapp: Names the outputted production binary file "myapp"
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp main.go


# ==============================================================================
# STAGE 2: The Final, Minimal Production Runtime
# Purpose: Runs the application inside an ultra-lean image without Go tools.
# ==============================================================================

# Start entirely fresh with a clean, lightweight Alpine Linux operating system base image
FROM alpine:3.20

# Create and switch into a production-ready application directory inside this final container
WORKDIR /app

# Reach back into the 'builder' stage, grab ONLY the compiled binary, and copy it here
# This leaves behind the large Go compiler, SDK, and source code files to save space
COPY --from=builder /src/myapp .

# Define the default startup command to execute your binary when the container boots up
CMD ["./myapp"]

```

**Build and run:**
```bash
cd /docker-practice-labs/lab-3-multistage-built/1-goapp-example
```
```bash
docker build -t multistage-demo .
```
```bash
docker run --rm multistage-demo
```
```
# Output: Hello from a multi-stage build!
```

**Compare sizes the real lesson here:**
```bash
# See the final image size
docker images multistage-demo

# Now build WITHOUT multi-stage (everything in one stage) to compare
```

**`practice/Dockerfile.singlestage`** (for comparison only)
copy the code bellow and replace with the code in your docker file

```dockerfile
FROM golang:1.22-alpine
WORKDIR /src
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp main.go
CMD ["./myapp"]
```

```bash
docker build -f Dockerfile.singlestage -t singlestage-demo .
```
```bash
docker images | grep -E "multistage-demo|singlestage-demo"
```

You should see the single-stage image is **hundreds of megabytes** larger (it contains the entire Go toolchain), while the multi-stage image is just Alpine + a small binary, often under 15 MB.

---

## 4. Practice Example Node.js build, static files served by nginx

A very common real-world pattern: compile/bundle a frontend app, then ship only the static output in a tiny web server image — no Node.js in the final image at all.

**`2-nodejs-nginx-example/package.json`**
```json
{
  "name": "demo-app",
  "version": "1.0.0",
  "scripts": {
    "build": "mkdir -p dist && echo '<h1>Built with multi-stage Docker!</h1>' > dist/index.html"
  }
}
```

**`2-nodejs-nginx-example/Dockerfile`**
```dockerfile
# ---------- Stage 1: build the static assets ----------
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
RUN npm run build
# Result: /app/dist/index.html

# ---------- Stage 2: serve with nginx, no Node.js included ----------
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

**Build and run:**
```bash
cd /docker-practice-labs/lab-3-multistage-built/2-nodejs-nginx-example
```
```bash
docker build -t web-demo .
```
```bash
docker run --rm -p 8080:80 web-demo
```

Visit `http://localhost:8080` — you'll see the page, even though the final image never had Node.js installed.

```bash
# Confirm Node isn't in the final image
docker run --rm web-demo which node
# Should return nothing / exit with an error — Node was only in the "builder" stage
```

---

## 5. Targeting a Specific Stage

You don't always have to build the whole file. `--target` stops the build at a named stage — useful for debugging an intermediate stage, or for a dedicated "test" stage you don't want shipped to production.

**`practice/Dockerfile.withtests`**
```dockerfile
FROM golang:1.22-alpine AS base
WORKDIR /src
COPY main.go .

# A stage just for running tests/checks — never shipped
FROM base AS tester
RUN go vet ./...

FROM base AS builder
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp main.go

FROM alpine:3.20 AS final
WORKDIR /app
COPY --from=builder /src/myapp .
CMD ["./myapp"]
```

```bash
# Build and stop at the "tester" stage only — won't produce the final runtime image
docker build -f Dockerfile.withtests --target tester -t demo-tests .

# Build the real final image (default target is the last stage, "final")
docker build -f Dockerfile.withtests -t demo-final .
```

**Try it yourself:** add a deliberate bug to `main.go` (like an unused import) and run the `tester` target — it should fail the build before ever reaching the `builder` stage.

---

## 6. Quick Reference Cheat Sheet

```dockerfile
FROM <image> AS <stage-name>      # start/name a stage
COPY --from=<stage-name> src dst  # copy files from another stage
COPY --from=<external-image> src dst   # copy from an unrelated image
```

```bash
docker build -t myimage .                     # builds the LAST stage by default
docker build --target <stage-name> -t x .      # build & stop at a specific stage
docker images                                  # compare resulting image sizes
docker history <image>                         # inspect layers of the final image
```

---

## 7. Common Pitfalls to Practice Spotting

1. **Forgetting `--from=`** a bare `COPY` always copies from the build context (your host), not from another stage. Leaving off `--from=builder` silently copies the wrong files or fails.
2. **Copying too much** copying an entire directory (`COPY --from=builder /src /app`) instead of just the binary defeats the purpose; final image bloats again.
3. **Stage name typos** referencing `--from=buidler` (typo) won't error clearly in older Docker versions in all cases; double check stage names match exactly.
4. **Build cache invalidation** putting `COPY . .` before installing dependencies (as in `npm install`) busts the cache on every source change. Copy dependency manifests first, install, *then* copy the rest of the source.
5. **Assuming the last stage runs by default in CI** if your pipeline uses `--target` for testing, make sure a separate, untargeted build step exists to produce the real shippable image.

---

## 8. Suggested Practice Order

1. Build `1-goapp-example/Dockerfile` and confirm the app runs.
2. Build `Dockerfile.singlestage` and compare `docker images` output see the size gap with your own eyes.
3. Build `2-nodejs-nginx-example/Dockerfile` and confirm Node.js is absent from the final image.
4. Build `Dockerfile.withtests`, targeting `tester` first, then the full final image.
5. Intentionally break the build cache (reorder `COPY` instructions) and watch how many layers get invalidated on a rebuild.

Happy practicing!