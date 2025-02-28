# Docker Implementation Guide for Node.js

A comprehensive guide to containerizing Node.js applications using Docker.

## 1. Basic Docker Setup

### Dockerfile for Node.js
```dockerfile
# Base image
FROM node:18-alpine

# Working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Start command
CMD ["npm", "start"]
```

### Docker Ignore
```plaintext
# .dockerignore
node_modules
npm-debug.log
Dockerfile
.dockerignore
.git
.gitignore
README.md
.env
*.test.js
coverage/
```

## 2. Multi-Stage Builds

### Development and Production Builds
```dockerfile
# Development Stage
FROM node:18-alpine AS development

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

# Build Stage
FROM development AS build
RUN npm run build

# Production Stage
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
EXPOSE 3000
CMD ["npm", "start"]
```

## 3. Environment Configuration

### Environment Variables
```dockerfile
# Using ARG for build-time variables
ARG NODE_ENV=production

# Using ENV for runtime variables
ENV NODE_ENV=${NODE_ENV} \
    PORT=3000 \
    REDIS_URL=redis://redis:6379

# Using docker-compose.yml for environment configuration
version: '3.8'
services:
  app:
    build: 
      context: .
      args:
        NODE_ENV: development
    environment:
      - NODE_ENV=development
      - PORT=3000
      - REDIS_URL=redis://redis:6379
```

## 4. Docker Compose Setup

### Basic Compose Configuration
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - redis
      - postgres

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  postgres:
    image: postgres:13-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 5. Production Best Practices

### Security
```dockerfile
# Use specific versions
FROM node:18.17.1-alpine3.18

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Set secure permissions
RUN chmod -R 755 /app

# Use health checks
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

### Performance Optimization
```dockerfile
# Optimize layer caching
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Use .dockerignore effectively
# Minimize image size
RUN npm cache clean --force

# Use multi-stage builds for smaller images
FROM node:18-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
```

## 6. Container Orchestration

### Docker Swarm Configuration
```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    ports:
      - "3000:3000"
    networks:
      - app-network

networks:
  app-network:
    driver: overlay
```

## 7. Monitoring and Logging

### Logging Configuration
```dockerfile
# Configure logging
ENV NODE_OPTIONS="--enable-source-maps"

# Use appropriate logging format
RUN npm install winston

# Example logging setup
CMD ["node", "--trace-warnings", "./dist/server.js"]
```

### Monitoring Setup
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    depends_on:
      - prometheus
```

## 8. Development Workflow

### Hot Reload Setup
```yaml
version: '3.8'
services:
  app:
    build: 
      context: .
      target: development
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev
```

## Related Topics
- [[Kubernetes-Guide]] - Kubernetes orchestration
- [[CICD-Pipeline]] - Continuous Integration/Deployment
- [[Cloud-Deployment]] - Cloud deployment strategies
- [[Security-Best-Practices]] - Security guidelines

## Practice Projects
1. Containerize a Node.js microservices application
2. Set up a development environment with Docker Compose
3. Implement CI/CD with Docker
4. Create a production-ready Docker setup

## Resources
- [Docker Documentation](https://docs.docker.com/)
- [Node.js Docker Best Practices](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/)
- [[Learning-Resources#Docker|Docker Resources]]

## Tags
#docker #nodejs #devops #containerization
