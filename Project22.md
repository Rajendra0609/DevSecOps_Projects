# CONTAINERIZATION WITH DOCKER — ADVANCED CONCEPTS

## Project Overview

This project builds on Project 20 by covering advanced Docker concepts including writing optimized Dockerfiles, managing multi-container environments, container networking, volume management, and pushing images to container registries.

### Objectives
1. Write optimized, multi-stage Dockerfiles
2. Use Docker volumes for persistent data
3. Understand container networking in depth
4. Push and pull images from Docker Hub and AWS ECR
5. Implement Docker health checks
6. Use environment variables and Docker secrets securely

## Prerequisites
- Docker installed (from Project 20)
- Docker Hub account
- AWS CLI configured (for ECR)
- Basic understanding of Docker fundamentals

## Step 1: Writing Optimized Dockerfiles

### Multi-Stage Build

Multi-stage builds reduce the final image size by separating the build environment from the runtime environment.

```dockerfile
# Stage 1: Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy dependency files first (improves layer caching)
COPY package*.json ./
RUN npm ci --only=production

# Copy application source
COPY . .
RUN npm run build

# Stage 2: Production stage
FROM node:18-alpine AS production

WORKDIR /app

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only built artifacts from builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### Build and verify the image

```bash
# Build the image
docker build -t myapp:1.0 .

# Check image size
docker images myapp

# Inspect image layers
docker history myapp:1.0

# Run the container
docker run -d --name myapp-container -p 3000:3000 myapp:1.0

# Check health status
docker inspect --format='{{.State.Health.Status}}' myapp-container
```

## Step 2: Docker Volumes for Persistent Data

Containers are ephemeral by default. Volumes allow data to persist beyond the container lifecycle.

### Named Volumes

```bash
# Create a named volume
docker volume create myapp-data

# Run container with named volume
docker run -d \
  --name myapp-container \
  -v myapp-data:/app/data \
  myapp:1.0

# Inspect the volume
docker volume inspect myapp-data

# List all volumes
docker volume ls
```

### Bind Mounts (for development)

```bash
# Mount local source code into container
docker run -d \
  --name myapp-dev \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  myapp:1.0
```

### Volume Backup and Restore

```bash
# Backup a volume
docker run --rm \
  -v myapp-data:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/myapp-data-$(date +%Y%m%d).tar.gz -C /source .

# Restore a volume
docker run --rm \
  -v myapp-data:/target \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/myapp-data-20240101.tar.gz -C /target
```

## Step 3: Docker Networking

### Creating Custom Networks

```bash
# Create a bridge network with custom subnet
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/24 \
  --ip-range=172.20.0.0/28 \
  --gateway=172.20.0.1 \
  myapp-network

# List networks
docker network ls

# Inspect the network
docker network inspect myapp-network
```

### Connecting Containers

```bash
# Run a database container on the custom network
docker run -d \
  --name myapp-db \
  --network myapp-network \
  -e POSTGRES_DB=myappdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=apppassword \
  -v db-data:/var/lib/postgresql/data \
  postgres:14-alpine

# Run the application container on the same network
docker run -d \
  --name myapp-container \
  --network myapp-network \
  -e DB_HOST=myapp-db \
  -e DB_PORT=5432 \
  -p 3000:3000 \
  myapp:1.0

# Test connectivity between containers
docker exec myapp-container ping myapp-db
```

## Step 4: Environment Variables and Secrets

### Using .env Files

Create a `.env` file (never commit this to git):

```
DB_HOST=myapp-db
DB_PORT=5432
DB_NAME=myappdb
DB_USER=appuser
DB_PASSWORD=apppassword
NODE_ENV=production
```

```bash
# Run with .env file
docker run -d \
  --name myapp-container \
  --env-file .env \
  -p 3000:3000 \
  myapp:1.0
```

### Docker Secrets (Swarm Mode)

```bash
# Create a secret
echo "superSecretPassword" | docker secret create db_password -

# Use the secret in a service
docker service create \
  --name myapp \
  --secret db_password \
  myapp:1.0
```

## Step 5: Pushing to Container Registries

### Docker Hub

```bash
# Log in to Docker Hub
docker login

# Tag the image for Docker Hub
docker tag myapp:1.0 <dockerhub-username>/myapp:1.0

# Push the image
docker push <dockerhub-username>/myapp:1.0

# Pull the image on another machine
docker pull <dockerhub-username>/myapp:1.0
```

### AWS Elastic Container Registry (ECR)

```bash
# Create an ECR repository
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Tag the image for ECR
docker tag myapp:1.0 <account-id>.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Push to ECR
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

## Step 6: Docker Health Checks

Health checks allow Docker to know when a container is truly ready to serve traffic.

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

```bash
# Check health status
docker ps  # shows health status in the STATUS column

# Get detailed health info
docker inspect --format='{{json .State.Health}}' myapp-container | python3 -m json.tool
```

## Step 7: Cleaning Up Docker Resources

### Remove stopped containers

```bash
docker container prune
```

### Remove unused images

```bash
# Remove dangling (untagged) images
docker image prune

# Remove all unused images
docker image prune -a
```

### Full system cleanup

```bash
# Remove all stopped containers, unused networks, dangling images, and build cache
docker system prune

# Also remove unused volumes (caution: data loss)
docker system prune --volumes

# Check disk usage
docker system df
```

## Conclusion

This project covers:
- Writing optimized multi-stage Dockerfiles
- Using named volumes for persistent storage
- Creating custom Docker networks for container isolation
- Managing environment variables and secrets securely
- Pushing images to Docker Hub and AWS ECR
- Implementing Docker health checks
- Cleaning up Docker resources

## Additional Resources
- [Docker Documentation](https://docs.docker.com/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Hub](https://hub.docker.com/)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
