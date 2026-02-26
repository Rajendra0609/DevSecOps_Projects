# DOCKER ADVANCED - COMPOSE, SWARM & REGISTRY

## Project Overview

This project covers advanced Docker topics including Docker Compose for multi-container applications, Docker Swarm for orchestration, and private container registries.

### Objectives
1. Master Docker Compose for multi-container applications
2. Understand Docker Swarm architecture and deployment
3. Set up private Docker registry
4. Implement networking between containers
5. Use Docker secrets and configs
6. Implement logging and monitoring

## Prerequisites
- Docker installed
- Docker Compose installed
- Basic understanding of Docker fundamentals

## Step 1: Docker Compose Deep Dive

### Multi-Container Application

```
yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  # Application Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgres://admin:${DB_PASSWORD}@postgres:5432/myapp
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - static_files:/var/www/html:ro
    depends_on:
      - backend
    networks:
      - frontend
    restart: unless-stopped

  # Elasticsearch for logging
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Kibana for visualization
  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - backend
    restart: unless-stopped

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  elasticsearch_data:
  static_files:
```

### Docker Compose Commands

```
bash
# Start all services
docker-compose up -d

# Start with specific file
docker-compose -f docker-compose.prod.yml up -d

# Build images before starting
docker-compose up -d --build

# View logs
docker-compose logs -f
docker-compose logs -f backend
docker-compose logs --tail=100 nginx

# Scale services
docker-compose up -d --scale backend=3

# Stop services
docker-compose down
docker-compose down -v  # Remove volumes too

# Remove images
docker-compose down --rmi local

# Pull latest images
docker-compose pull

# Restart services
docker-compose restart
docker-compose restart nginx

# Execute command in running container
docker-compose exec backend sh
docker-compose exec postgres psql -U admin -d myapp

# Run one-off command
docker-compose run --rm backend npm test

# Check status
docker-compose ps
docker-compose top
```

## Step 2: Docker Swarm

### Initialize Swarm

```
bash
# Initialize Docker Swarm (manager node)
docker swarm init --advertise-addr 192.168.1.100

# Check swarm status
docker info | grep Swarm
docker node ls

# Get worker join token
docker swarm join-token -q worker

# Join worker node
docker swarm join \
  --token SWMTKN-1-xxxxxxxxx \
  --advertise-addr 192.168.1.101 \
  192.168.1.100:2377

# Join as manager (from manager node)
docker swarm join-token manager

# Leave swarm (from worker node)
docker swarm leave

# Remove node from swarm (from manager)
docker node rm node-id
docker node rm --force node-id  # Force remove
```

### Deploy Services in Swarm

```
bash
# Create overlay network
docker network create -d overlay my-overlay

# Deploy service
docker service create \
  --name my-web \
  --replicas 3 \
  --network my-overlay \
  --publish 80:80 \
  --update-delay 10s \
  --update-parallelism 2 \
  --rollback-monitor 5s \
  nginx:latest

# List services
docker service ls
docker service ps my-web

# Inspect service
docker service inspect my-web
docker service inspect --pretty my-web

# Update service
docker service update \
  --image nginx:alpine \
  my-web

# Scale service
docker service scale my-web=5
docker service scale web=3 api=2

# Update service with rollback
docker service update \
  --image nginx:latest \
  --update-failure-action rollback \
  --rollback-failure-action pause \
  my-web

# Remove service
docker service rm my-web
```

### Stack Deploy

```
yaml
# docker-stack.yml
version: '3.8'

services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
        monitor: 5s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      placement:
        constraints:
          - node.role == worker
        preferences:
          - spread: node.labels.datacenter
    networks:
      - frontend

  api:
    build: ./api
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    deploy:
      replicas: 2
      endpoint_mode: vip
    networks:
      - frontend
      - backend
    depends_on:
      - db

  db:
    image: postgres:14
    volumes:
      - db_data:/var/lib/postgresql/data
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - backend

  visualizer:
    image: dockersamples/visualizer
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    driver_opts:
      encrypted: '1'

volumes:
  db_data:
```

```
bash
# Deploy stack
docker stack deploy -c docker-stack.yml myapp

# List stacks
docker stack ls

# List services in stack
docker stack services myapp

# List tasks in stack
docker stack ps myapp

# Remove stack
docker stack rm myapp
```

## Step 3: Private Docker Registry

### Set Up Registry

```
bash
# Start local registry
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

# Test registry
curl http://localhost:5000/v2/

# Pull image and tag for local registry
docker pull nginx:latest
docker tag nginx:latest localhost:5000/my-nginx:latest

# Push to local registry
docker push localhost:5000/my-nginx:latest

# Pull from local registry
docker pull localhost:5000/my-nginx:latest

# List images in registry
curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/my-nginx/tags/list
```

### Registry with Authentication

```
bash
# Create password file
mkdir -p auth
htpasswd -Bc auth/htpasswd admin

# Start registry with auth
docker run -d \
  --name registry-auth \
  --restart=always \
  -p 5000:5000 \
  -v $(pwd)/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -v registry-data:/var/lib/registry \
  registry:2

# Login to registry
docker login localhost:5000 -u admin

# Push with auth
docker tag nginx:latest localhost:5000/my-nginx:latest
docker push localhost:5000/my-nginx:latest
```

### Registry with TLS

```
bash
# Create certificates
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -subj "/CN=registry.example.com" \
  -keyout certs/domain.key \
  -out certs/domain.csr

# Self-signed certificate
openssl x509 -req -days 365 \
  -in certs/domain.csr \
  -signkey certs/domain.key \
  -out certs/domain.crt

# Start registry with TLS
docker run -d \
  --name registry-tls \
  --restart=always \
  -p 5000:5000 \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -v registry-data:/var/lib/registry \
  registry:2

# Add certificate to trusted store (on client)
sudo cp certs/domain.crt /usr/local/share/ca-certificates/registry.crt
sudo update-ca-certificates
```

## Step 4: Docker Secrets

```
bash
# Create secrets
echo "mypassword" | docker secret create db_password -
docker secret create my_api_key api_key.txt

# List secrets
docker secret ls

# Inspect secret
docker secret inspect db_password

# Create service with secrets
docker service create \
  --name my-web \
  --secret db_password \
  --secret my_api_key \
  nginx:latest

# Use secrets in docker-compose
yaml
services:
  web:
    image: nginx
    secrets:
      - db_password
      - source: my_api_key
        target: API_KEY

secrets:
  db_password:
    file: ./secrets/db_password.txt
  my_api_key:
    external: true
```

## Step 5: Logging and Monitoring

```
bash
# Configure logging driver
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx

# Use syslog logging
docker run \
  --log-driver syslog \
  --log-opt syslog-address=tcp://localhost:514 \
  nginx

# Use fluentd logging
docker run \
  --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt fluentd-tag=docker.{{.Name}} \
  nginx

# View container logs
docker logs container_id
docker logs --follow --tail 100 container_id

# Docker stats
docker stats
docker stats --no-stream container_id

# Docker events
docker events
docker events --filter 'type=container'
```

### Logging with ELK Stack

```
yaml
# docker-compose.logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
```

## Step 6: Docker Networking

```
bash
# Create network
docker network create my-network
docker network create --driver overlay --attachable my-overlay

# Connect container to network
docker network connect my-network container_name

# Disconnect container
docker network disconnect my-network container_name

# Inspect network
docker network inspect bridge
docker network inspect my-network

# List networks
docker network ls

# Remove network
docker network rm my-network

# Prune unused networks
docker network prune
```

### Container Communication Patterns

```
bash
# DNS-based service discovery
# Containers can reach each other by service name
# backend can reach postgres at: postgres:5432

# Link containers (legacy)
docker run --link redis:redis nginx

# Use alias
docker network connect --alias dbserver my-network postgres_container
```

## Step 7: Docker Volume Plugins

```
bash
# Create volume
docker volume create my-volume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-volume

# Use volume
docker run -v my-volume:/data nginx

# Remove unused volumes
docker volume prune

# NFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/export \
  nfs-volume
```

## Step 8: Docker Build Optimization

```
dockerfile
# BAD: Multiple RUN commands
FROM node:14
RUN apt-get update
RUN apt-get install -y git
RUN npm install -g some-package
RUN rm -rf /var/lib/apt/lists/*

# GOOD: Single RUN with cleanup
FROM node:14
RUN apt-get update && \
    apt-get install -y git && \
    npm install -g some-package && \
    rm -rf /var/lib/apt/lists/*

# Use .dockerignore
# node_modules
# .git
# .env
# *.md

# Use multi-stage builds
FROM node:14 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

# Use specific base image tags
# BAD: node:latest
# GOOD: node:14-alpine
```

## Conclusion

This project covers:
- Docker Compose for multi-container applications
- Docker Swarm for orchestration
- Private Docker registry setup with authentication
- Docker secrets management
- Logging and monitoring
- Docker networking
- Volume management
- Docker build optimization

## Additional Resources
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)
- [Docker Registry Documentation](https://docs.docker.com/registry/)
