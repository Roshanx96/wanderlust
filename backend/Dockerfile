# Stage 1: Install dependencies and build (if needed)
FROM node:18-alpine AS builder

WORKDIR /app

# Install dependencies (cache package.json and lock file)
COPY package.json package-lock.json* pnpm-lock.yaml* yarn.lock* ./
RUN npm ci --prefer-offline

# Copy source files
COPY . .

# Stage 2: Production image
FROM node:18-alpine

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder /app .

# Expose backend port
EXPOSE 8080

# Use non-root user for security
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser

# Start the server
CMD ["node", "server.js"]
