# ==============================================================================
# STAGE 1: Base Environment & Dependency Caching
# Purpose: Installs dependencies once and shares them with downstream stages.
# ==============================================================================
FROM golang:1.22-alpine AS base
WORKDIR /src

# Copy dependency manifests first to leverage Docker's layer caching
COPY go.mod go.sum* ./

# Download Go module dependencies (this layer stays cached unless go.mod changes)
RUN go mod download

# Copy the rest of the application source code files
COPY . .


# ==============================================================================
# STAGE 2: Automated Testing Gate
# Purpose: Isolates lints and unit tests. If this fails, the build halts.
# ==============================================================================
FROM base AS tester

# Analyze code for structural bugs and common mistakes
RUN go vet ./...

# Run the complete unit test suite with verbose logging enabled
RUN go test -v ./...


# ==============================================================================
# STAGE 3: Production Compilation
# Purpose: Compiles a lightweight, standalone static binary.
# ==============================================================================
FROM base AS builder

# Compile to a static binary:
# - CGO_ENABLED=0 removes C bindings for maximum portability across Linux envs
# - GOOS=linux targets the Linux kernel explicitly
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o myapp main.go


# ==============================================================================
# STAGE 4: Final, Minimal Runtime
# Purpose: Secure, ultra-lean image with zero build tools or source code.
# ==============================================================================
FROM alpine:3.20 AS final

# Run as a non-privileged system user for container security hardening
RUN adduser -D -u 10001 appuser

WORKDIR /app

# Safely copy only the compiled static binary from the builder stage
COPY --from=builder --chown=appuser:appuser /src/myapp .

# Switch away from root access
USER appuser

# Start the application
CMD ["./myapp"]
