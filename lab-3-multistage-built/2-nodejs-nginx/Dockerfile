# ==============================================================================
# STAGE 1: Build the Static Assets
# Purpose: Compiles source code into pure, optimized web assets (HTML/JS/CSS).
# ==============================================================================

# Start with a modern, lightweight Node.js runtime environment
FROM node:20-alpine AS builder

# Create and switch into a dedicated workspace folder inside the builder image
WORKDIR /app

# Copy package manifests first to lock versions and maximize build cache efficiency
COPY package*.json ./

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
