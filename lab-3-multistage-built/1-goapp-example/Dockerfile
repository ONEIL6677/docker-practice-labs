# ==============================================================================
# STAGE 1: The Build Environment
# Purpose: Compiles the source code into a single, independent executable binary.
# ==============================================================================

# Use the official Go image based on Alpine Linux to keep the build stage small
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
