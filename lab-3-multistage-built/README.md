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

**`/docker-practice-labs/lab-3-multistage-built/1-goapp/main.go`**
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello from a multi-stage build!")
}
```

**`/docker-practice-labs/lab-3-multistage-built/1-goapp/Dockerfile`**
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
cd /docker-practice-labs/lab-3-multistage-built/1-goapp
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

A very common real-world pattern: compile/bundle a frontend app, then ship only the static output in a tiny web server image no Node.js in the final image at all.

**`2-nodejs-nginx/package.json`**
```json
{
  "name": "demo-app",
  "version": "1.0.0",
  "scripts": {
    "build": "mkdir -p dist && echo '<h1>Built with multi-stage Docker!</h1>' > dist/index.html"
  }
}
```

**`2-nodejs-nginx/Dockerfile`**
```dockerfile
# ==============================================================================
# STAGE 1: Build the Static Assets
# Purpose: Compiles source code into pure, optimized web assets (HTML/JS/CSS).
# ==============================================================================

# Start with a modern, lightweight Node.js runtime environment
FROM node:20-alpine AS builder

# Create and switch into a dedicated workspace folder inside the builder image
WORKDIR /app

# Copy package manifests first to lock versions and maximize build cache efficiency
COPY package.json package-lock.json* ./

# Install exact dependency versions cleanly for automation (faster than npm install)
RUN npm ci

# Copy all remaining local source files into the build container
COPY . .

# Compile and minify the application assets for production delivery
RUN npm run build
# Result: Your optimized application files are generated inside '/app/dist'


# ==============================================================================
# STAGE 2: Serve with Nginx (No Node.js Included)
# Purpose: Delivers the compiled static assets efficiently to user browsers.
# ==============================================================================

# Drop the bulky Node.js runtime entirely and boot a high-performance web server
FROM nginx:1.25-alpine

# Copy the compiled production assets from the builder stage into Nginx's public root
COPY --from=builder /app/dist /usr/share/nginx/html

# OPTIONAL: Uncomment the line below if you add a custom 'nginx.conf' to handle SPA routing
# COPY nginx.conf /etc/nginx/conf.d/default.conf

# Document that this container accepts incoming web traffic on HTTP port 80
EXPOSE 80

# Start the web server in the foreground so the container continues running
CMD ["nginx", "-g", "daemon off;"]

```

**Build and run:**
```bash
cd /docker-practice-labs/lab-3-multistage-built/2-nodejs-nginx
```
```bash
docker build -t web-demo .
```
```bash
docker run --rm -p 8080:80 web-demo
```

Visit `http://localhost:8080` — you'll see the page, even though the final image never had Node.js installed.

### Confirm Node isn't in the final image
```bash
docker run --rm web-demo which node
```
```bash
# Should return nothing / exit with an error — Node was only in the "builder" stage
```

## 5. Targeting a Specific Stage

You don't always have to build the whole file. `--target` stops the build at a named stage useful for debugging an intermediate stage, or for a dedicated "test" stage you don't want shipped to production.

**`docker-practice-labs/lab-3-multistage-built/3-intermediate-test-stage/Dockerfile`**
```dockerfile
# ==============================================================================
# STAGE 1: Base Configuration
# Purpose: Shared foundation to avoid repeating commands in subsequent stages.
# ==============================================================================

# Start with an official Go image on Alpine Linux and name this stage 'base'
FROM golang:1.22-alpine AS base

# Create and move into a dedicated workspace folder inside the container
WORKDIR /src

# Copy your local Go source code file into the current directory (/src)
COPY main.go .


# ==============================================================================
# STAGE 2: Automated Testing & Code Analysis
# Purpose: A quality gate to validate code safety. This stage is NEVER shipped.
# ==============================================================================

# Inherit everything from the 'base' stage (includes Go, your folder, and main.go)
FROM base AS tester

# Run 'go vet', a built-in static analysis tool that inspects code for common bugs
# If 'go vet' finds errors, the build crashes right here, protecting production.
RUN go vet ./...


# ==============================================================================
# STAGE 3: Production Compilation
# Purpose: Compiles the vetted source code into a standalone executable.
# ==============================================================================

# Inherit again from the clean 'base' stage to isolate the compilation process
FROM base AS builder

# Compile the Go application with environment variables optimized for Alpine containers:
# - CGO_ENABLED=0: Disables dynamic C libraries to make the binary statically linked
# - GOOS=linux: Explicitly targets Linux operating systems
# - -o myapp: Names the outputted production binary file "myapp"
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp main.go


# ==============================================================================
# STAGE 4: Final, Minimal Runtime
# Purpose: The ultra-lean, lightweight image that actually runs in production.
# ==============================================================================

# Start fresh with a bare-bones Alpine operating system (no Go tools or compilers)
FROM alpine:3.20 AS final

# Create a secure application directory inside this final container
WORKDIR /app

# Reach directly into the 'builder' stage, extraction only the compiled 'myapp' binary
# All raw code, compilers, testing environments, and temp files are discarded
COPY --from=builder /src/myapp .

# Set the default startup command to run the application binary on container boot
CMD ["./myapp"]
```
## build and run

```bash
cd docker-practice-labs/lab-3-multistage-built/3-intermediate-test-stage
```
### Build and stop at the "tester" stage only won't produce the final runtime image
```bash
docker build --target tester -t demo-tests .
```
### use this comand if your dockerfile is name differently -f followed by file name
```bash
docker build -f DockerfileName --target tester -t demo-tests . 
```
### Build the real final image (default target is the last stage, "final")
```bash
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

Happy practicing!