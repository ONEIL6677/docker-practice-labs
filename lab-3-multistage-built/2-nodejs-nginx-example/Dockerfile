# ==============================================================================
# STAGE 1: Build the Static Assets
# Purpose: Installs dependencies and compiles raw source code into pure HTML/JS/CSS.
# ==============================================================================

# Start with a lightweight Node.js runtime environment to handle package assembly
FROM node:20-alpine AS builder

# Create and switch into a dedicated working folder inside the builder image
WORKDIR /app

# Copy the dependency manifest file first to take advantage of Docker caching
COPY package.json .

# Download and install all library dependencies listed inside package.json
RUN npm install

# Copy all remaining local source files into the image workspace
COPY . .

# Run the project build script to compile and minify the files for production
RUN npm run build
# Result: Your raw assets are optimized and saved inside the /app/dist/ directory


# ==============================================================================
# STAGE 2: Serve with Nginx (No Node.js Included)
# Purpose: Efficiently delivers the compiled static pages to web browsers.
# ==============================================================================

# Drop Node.js completely and start fresh with a high-performance web server
FROM nginx:alpine

# Copy the compiled production assets from the builder stage straight into the server's public folder
COPY --from=builder /app/dist /usr/share/nginx/html

# Document that this web server will listen for web traffic on port 80
EXPOSE 80
