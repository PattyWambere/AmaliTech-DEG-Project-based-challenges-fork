# Stage 1: Install all dependencies (including dev for tests)
FROM node:20-alpine AS deps
WORKDIR /app
COPY app/package*.json ./
RUN npm ci

# Stage 2: Run tests — if tests fail, the entire build fails
FROM deps AS test
COPY app/ ./
RUN npm test

# Stage 3: Production image — lean, non-root, no test tools
FROM node:20-alpine AS production
WORKDIR /app

# Copy package files and install production dependencies only
COPY app/package*.json ./
RUN npm ci --omit=dev

# Copy application source
COPY app/ ./

# Create and switch to a non-root user (security requirement)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Accept PORT as environment variable
ENV PORT=3000
EXPOSE $PORT

# Health check so Docker knows when the container is ready
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:$PORT/health || exit 1

CMD ["node", "index.js"]