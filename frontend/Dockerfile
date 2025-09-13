# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Install dependencies (cache package.json and lock file)
COPY package.json package-lock.json* pnpm-lock.yaml* yarn.lock* ./
RUN npm ci --prefer-offline

# Copy source files
COPY . .

# Build the app
RUN npm run build

# Stage 2: Serve
FROM nginx:1.25-alpine

RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 5173

CMD ["nginx", "-g", "daemon off;"]
