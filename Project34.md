# NGINX ADVANCED - WEB SERVER, REVERSE PROXY & LOAD BALANCER

## Project Overview

This project covers advanced NGINX configuration including as a web server, reverse proxy, load balancer, caching, and security features. NGINX is one of the most popular web servers used for high-performance web delivery.

### Objectives
1. Configure NGINX as a web server
2. Set up NGINX as a reverse proxy
3. Implement load balancing
4. Configure caching and compression
5. Implement SSL/TLS
6. Set up rate limiting and security
7. Use NGINX with Docker and Kubernetes

## Prerequisites
- NGINX installed or Docker available
- Basic understanding of HTTP/HTTPS
- Terminal access

## Step 1: NGINX Basics and Configuration

### Install NGINX

```
bash
# Install on Ubuntu/Debian
sudo apt update
sudo apt install nginx

# Install on CentOS/RHEL
sudo yum install epel-release
sudo yum install nginx

# Install on macOS
brew install nginx

# Start NGINX
sudo systemctl start nginx
sudo systemctl enable nginx

# Test installation
nginx -v
nginx -t
```

### Basic Configuration Structure

```
nginx
# /etc/nginx/nginx.conf

# User and group
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;

events {
    worker_connections 1024;
    use epoll;  # Linux
    multi_accept on;
}

http {
    # Basic settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Include MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging settings
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;
    
    # Include virtual host configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Basic Site Configuration

```
nginx
# /etc/nginx/sites-available/example.com

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    
    # Document root
    root /var/www/html;
    index index.html index.htm index.php;
    
    # Logging
    access_log /var/log/nginx/example_access.log;
    error_log /var/log/nginx/example_error.log;
    
    # Location blocks
    location / {
        try_files $uri $uri/ =404;
    }
    
    location /about {
        alias /var/www/about;
        index about.html;
    }
    
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Deny access to hidden files
    location ~ /\.ht {
        deny all;
    }
}
```

## Step 2: Reverse Proxy Configuration

### Basic Reverse Proxy

```
nginx
# /etc/nginx/conf.d/reverse-proxy.conf

server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
    }
    
    # WebSocket support
    location /ws/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
    }
}
```

### Load Balancer

```
nginx
# /etc/nginx/conf.d/load-balancer.conf

upstream backend {
    least_conn;  # Load balancing method
    
    server backend1.example.com:8080 weight=5;
    server backend2.example.com:8080 weight=3;
    server backend3.example.com:8080 weight=2;
    
    # Health check
    server backend4.example.com:8080 backup;
    
    keepalive 32;
}

server {
    listen 80;
    server_name lb.example.com;
    
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Connection pooling
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

### NGINX Load Balancing Methods

```
nginx
# Least connections
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}

# IP Hash (session persistence)
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}

# Generic Hash
upstream backend {
    hash $request_uri;
    server backend1.example.com;
    server backend2.example.com;
}

# Least Time (NGINX Plus)
upstream backend {
    least_time header;
    server backend1.example.com;
    server backend2.example.com;
}
```

## Step 3: SSL/TLS Configuration

### SSL Certificate Setup

```
bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx

# Generate SSL certificate
sudo certbot --nginx -d example.com -d www.example.com

# Test auto-renewal
sudo certbot renew --dry-run
```

### SSL Configuration

```
nginx
# /etc/nginx/conf.d/ssl.conf

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;
    
    # SSL certificate
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers off;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}
```

## Step 4: Caching Configuration

### Proxy Caching

```
nginx
# /etc/nginx/conf.d/caching.conf

proxy_cache_path /var/cache/nginx levels=1:2 
                 keys_zone=my_cache:10m 
                 max_size=100m 
                 inactive=60m 
                 use_temp_path=off;

upstream backend {
    server localhost:3000;
}

server {
    listen 80;
    server_name cache.example.com;
    
    location / {
        proxy_pass http://backend;
        
        # Cache configuration
        proxy_cache my_cache;
        proxy_cache_valid 200 60m;
        proxy_cache_valid 404 1m;
        proxy_cache_valid any 10m;
        
        # Cache key
        proxy_cache_key "$scheme$request_method$host$request_uri";
        
        # Headers
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;
        
        # Add cache headers
        add_header X-Cache-Status $upstream_cache_status;
        
        # Bypass cache
        proxy_cache_bypass $http_cache_control;
        proxy_no_cache $http_cache_control;
    }
}
```

### FastCGI Caching (for PHP)

```
nginx
# /etc/nginx/conf.d/fastcgi-cache.conf

fastcgi_cache_path /var/cache/nginx/fastcgi 
                  levels=1:2 
                  keys_zone=fastcgi_cache:10m 
                  max_size=1000m 
                  inactive=60m 
                  use_temp_path=off;

server {
    listen 80;
    server_name php.example.com;
    
    # PHP configuration
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        
        # FastCGI caching
        fastcgi_cache fastcgi_cache;
        fastcgi_cache_valid 200 60m;
        fastcgi_cache_valid any 10m;
        fastcgi_cache_key "$scheme$request_method$host$fastcgi_script_name";
        fastcgi_cache_use_stale error timeout invalid_header http_500;
        fastcgi_cache_lock on;
        add_header X-FastCGI-Cache $upstream_cache_status;
    }
}
```

## Step 5: Rate Limiting and Security

### Rate Limiting

```
nginx
# /etc/nginx/conf.d/rate-limit.conf

limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

server {
    listen 80;
    server_name rate.example.com;
    
    # General rate limiting
    location / {
        limit_req zone=req_limit burst=20 nodelay;
        limit_req zone=req_limit dryrun;
        
        proxy_pass http://localhost:3000;
    }
    
    # API rate limiting
    location /api/ {
        limit_req zone=api_limit burst=50 nodelay;
        
        proxy_pass http://localhost:3000;
    }
    
    # Connection limiting
    location /download/ {
        limit_conn conn_limit 5;
        
        proxy_pass http://localhost:3000;
    }
}
```

### Security Configuration

```
nginx
# /etc/nginx/conf.d/security.conf

server {
    listen 80;
    server_name secure.example.com;
    
    # Hide NGINX version
    server_tokens off;
    
    # Disable clickjacking
    add_header X-Frame-Options "SAMEORIGIN" always;
    
    # Disable MIME sniffing
    add_header X-Content-Type-Options "nosniff" always;
    
    # XSS Protection
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Referrer Policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Permissions Policy
    add_header Permissions-Policy "geolocation=(),midi=(),sync-xhr=(),microphone=(),camera=(),payment=()" always;
    
    # Content Security Policy
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    
    # Allow only specific IPs
    location /admin/ {
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;
        
        proxy_pass http://localhost:3000;
    }
    
    # Block common exploits
    location ~ /\.(svn|git|hg) {
        deny all;
    }
    
    location ~ /\. {
        deny all;
    }
    
    location ~* \.(jpg|jpeg|gif|png|css|js|ico|xml|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

## Step 6: NGINX with Docker

### Docker Compose Setup

```
yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: my-nginx
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./html:/usr/share/nginx/html:ro
      - ./logs:/var/log/nginx
      - ./ssl:/etc/nginx/ssl:ro
    networks:
      - nginx-network
    depends_on:
      - app
    restart: unless-stopped

  app:
    image: node:18-alpine
    container_name: my-app
    working_dir: /app
    volumes:
      - ./app:/app
    command: npm start
    networks:
      - nginx-network
    restart: unless-stopped

networks:
  nginx-network:
    driver: bridge
```

### Dockerfile for Custom NGINX

```
dockerfile
FROM nginx:alpine

# Remove default config
RUN rm /etc/nginx/nginx.conf
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom configuration
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/*.conf /etc/nginx/conf.d/

# Copy HTML files
COPY html /usr/share/nginx/html

# Copy SSL certificates
COPY ssl /etc/nginx/ssl

# Create log directory
RUN mkdir -p /var/log/nginx

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

## Step 7: NGINX on Kubernetes

### NGINX Ingress Controller

```
bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
```

### Ingress Resource

```
yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

### ConfigMap for NGINX Ingress

```
yaml
# nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
data:
  proxy-body-size: "50m"
  proxy-connect-timeout: "60"
  proxy-read-timeout: "60"
  proxy-send-timeout: "60"
  proxy-buffering: "true"
  proxy-buffers: "8 4k"
  use-forwarded-headers: "true"
  compute-full-forwarded-for: "true"
  enable-underscores-in-headers: "true"
  large-client-header-buffers: "4 16k"
  client-header-buffer-size: "4k"
  keep-alive: "75"
  keep-alive-requests: "1000"
  upstream-keepalive-connections: "1000"
  upstream-keepalive-timeout: "60"
  upstream-keepalive-requests: "10000"
```

## Step 8: Advanced Features

### WebSocket Proxying

```
nginx
# WebSocket configuration
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    # Timeouts for WebSocket
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
}
```

### HTTP/2 and Server Push

```
nginx
# HTTP/2 and Server Push
server {
    listen 443 ssl http2;
    server_name push.example.com;
    
    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;
    
    # HTTP/2
    http2_push_preload on;
    
    # Push resources
    location / {
        proxy_pass http://localhost:3000;
        
        # Push CSS
        link: </css/styles.css>; as=style;
        # Push JS
        link: </js/app.js>; as=script;
        # Push images
        link: </images/logo.png>; as=image;
    }
}
```

### Request Filtering

```
nginx
# Block specific User-Agent
if ($http_user_agent ~* (Python-urllib|wget|curl)) {
    return 403;
}

# Block SQL injection
location ~* union.*select {
    return 403;
}

location ~* union.*from {
    return 403;
}

# Block XSS attempts
location ~* (<script|%3Cscript) {
    return 403;
}

# Allow specific paths
location /api/ {
    # API specific config
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;
    
    limit_req zone=api_limit burst=50 nodelay;
}
```

## Step 9: Monitoring and Troubleshooting

### Health Check Configuration

```
nginx
# Active health checks (NGINX Plus)
upstream backend {
    zone backend 64k;
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
    
    health_check interval=5s fails=3 passes=2;
}
```

### Log Analysis

```
bash
# View access logs
tail -f /var/log/nginx/access.log

# View error logs
tail -f /var/log/nginx/error.log

# Analyze with awk - show top IPs
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head 10

# Analyze with awk - show top URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head 10

# Show request status codes
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr

# Slow requests
awk '{if ($9 > 2) print $7, $9}' /var/log/nginx/access.log | sort -k2 -nr | head 20
```

### Performance Tuning

```
nginx
# /etc/nginx/nginx.conf - Performance optimized

user www-data;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    # Basic settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # File caching
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss;
    
    # Buffers
    client_body_buffer_size 10K;
    client_header_buffer_size 1k;
    client_max_body_size 8m;
    large_client_header_buffers 4 32k;
    
    # Timeouts
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;
    
    # Upstream keepalive
    upstream backend {
        keepalive 32;
        keepalive_requests 100;
        keepalive_timeout 60s;
    }
}
```

## Step 10: Complete Example

### Complete NGINX Configuration

```
nginx
# /etc/nginx/nginx.conf
user www-data;
worker_processes auto;
worker_rlimit_nofile 65535;
pid /run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # File caching
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    
    # Upstream definitions
    upstream app_servers {
        least_conn;
        server 10.0.1.10:8080 weight=5;
        server 10.0.1.11:8080 weight=5;
        server 10.0.1.12:8080 weight=3 backup;
        keepalive 32;
    }
    
    # Include virtual hosts
    include /etc/nginx/conf.d/*.conf;
}
```

```
nginx
# /etc/nginx/conf.d/app.example.com.conf

server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;
    
    # Security headers
    server_tokens off;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name app.example.com;
    
    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # Rate limiting
    limit_req zone=general burst=20 nodelay;
    limit_conn addr 10;
    
    # Logging
    access_log /var/log/nginx/app_access.log;
    error_log /var/log/nginx/app_error.log;
    
    location / {
        proxy_pass http://app_servers;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
        
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Static files caching
    location ~* \.(jpg|jpeg|gif|png|css|js|ico|svg|woff|woff2|ttf|eot)$ {
        proxy_pass http://app_servers;
        proxy_cache_valid 200 60m;
        expires 60d;
        add_header Cache-Control "public, immutable";
    }
}
```

## Conclusion

This project covers:
- NGINX installation and basic configuration
- Reverse proxy setup
- Load balancing with multiple methods
- SSL/TLS configuration with security headers
- Caching strategies (proxy and FastCGI)
- Rate limiting and security features
- NGINX with Docker and Docker Compose
- NGINX Ingress Controller on Kubernetes
- WebSocket support
- HTTP/2 and server push
- Performance tuning
- Monitoring and troubleshooting

## Additional Resources
- [NGINX Documentation](https://nginx.org/en/docs/)
- [NGINX Admin Guide](https://docs.nginx.com/nginx/admin-guide/)
- [NGINX Performance Tuning](https://www.nginx.com/blog/tuning-nginx/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
