# TOMCAT DEPLOYMENT WITH DOCKER & KUBERNETES

## Project Overview

This project demonstrates deploying Apache Tomcat applications using Docker and Kubernetes. Apache Tomcat is a popular open-source Java servlet container used for Java web applications.

### Objectives
1. Create Docker images for Tomcat applications
2. Configure Tomcat with various settings
3. Deploy Tomcat on Kubernetes
4. Implement clustering and load balancing
5. Configure SSL/TLS for Tomcat
6. Monitor Tomcat applications

## Prerequisites
- Docker installed
- Kubernetes cluster (optional)
- Java applications (WAR files)
- Basic understanding of Java web applications

## Step 1: Dockerizing Tomcat Applications

### Basic Tomcat Docker Image

```
dockerfile
# Dockerfile
FROM tomcat:9.0-jdk11

# Set Java options
ENV JAVA_OPTS="-Xms512m -Xmx1024m"

# Remove default webapps
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy WAR file
COPY myapp.war /usr/local/tomcat/webapps/

# Copy custom configuration
COPY server.xml /usr/local/tomcat/conf/

# Set port
EXPOSE 8080

# Start Tomcat
CMD ["catalina.sh", "run"]
```

### Multi-Stage Build for Java Application

```
dockerfile
# Multi-stage Dockerfile
# Build stage
FROM maven:3.8-openjdk-11 AS builder

WORKDIR /app

# Copy pom.xml first for dependency caching
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests

# Production stage
FROM tomcat:9.0-jdk11

# Remove default apps
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy WAR from builder
COPY --from=builder /app/target/myapp.war /usr/local/tomcat/webapps/

# Copy custom context.xml
COPY context.xml /usr/local/tomcat/webapps/myapp/META-INF/

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

### Tomcat with Custom Configuration

```
dockerfile
# Advanced Dockerfile with JVM options and logging
FROM tomcat:9.0-jdk11

# Labels
LABEL maintainer="admin@example.com"
LABEL version="1.0"

# Environment variables for JVM
ENV CATALINA_OPTS="-Xms512m -Xmx2048m -XX:+UseG1GC"
ENV JAVA_OPTS="-Djava.security.egd=file:/dev/./urandom"
ENV JPDA_ADDRESS="5005"

# Install additional tools
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Create directory for logs
RUN mkdir -p /usr/local/tomcat/logs

# Copy application
COPY myapp.war /usr/local/tomcat/webapps/

# Configure Tomcat users
COPY tomcat-users.xml /usr/local/tomcat/conf/

# Configure server.xml
COPY server.xml /usr/local/tomcat/conf/

# Configure logging
COPY logging.properties /usr/local/tomcat/conf/

# Expose ports
EXPOSE 8080 8443 8009 5005

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8080/ || exit 1

# Start Tomcat
CMD ["catalina.sh", "run"]
```

### Docker Compose for Tomcat

```
yaml
# docker-compose.yml
version: '3.8'

services:
  tomcat:
    build: .
    image: myapp/tomcat:1.0
    container_name: myapp-tomcat
    ports:
      - "8080:8080"
      - "8443:8443"
      - "8009:8009"
    environment:
      - JAVA_OPTS=-Xms512m -Xmx1024m
      - CATALINA_OPTS=-Xdebug -Xrunjdwp:server=y,transport=dt_socket,suspend=n
      - JAVA_HOME=/usr/local/openjdk-11
    volumes:
      - ./logs:/usr/local/tomcat/logs
      - ./webapps:/usr/local/tomcat/webapps
      - ./conf:/usr/local/tomcat/conf
    networks:
      - tomcat-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:14
    container_name: myapp-db
    environment:
      POSTGRES_DB: myappdb
      POSTGRES_USER: dbuser
      POSTGRES_PASSWORD: dbpassword
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - tomcat-network
    restart: unless-stopped

  nginx:
    image: nginx:latest
    container_name: myapp-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - tomcat
    networks:
      - tomcat-network
    restart: unless-stopped

networks:
  tomcat-network:
    driver: bridge

volumes:
  postgres-data:
```

## Step 2: Tomcat Configuration Files

### server.xml

```
xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">
    
    <!-- HTTP Connector -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxThreads="200"
               minSpareThreads="10"
               enableLookups="false"
               acceptCount="100"
               compression="on"
               compressionMinSize="2048"
               noCompressionUserAgents="gozilla, traviata"
               compressableMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json" />

    <!-- AJP Connector for Apache/Nginx -->
    <Connector protocol="AJP/1.3"
               address="0.0.0.0"
               port="8009"
               redirectPort="8443"
               maxThreads="200"
               connectionTimeout="600000"
               maxParameterCount="10000" />

    <Engine name="Catalina" defaultHost="localhost">
      
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost" appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t "%r" %s %b" />
               
        <!-- Access log configuration -->
        <Valve className="org.apache.catalina.valves.AccessLogValve"
               directory="logs"
               prefix="myapp_access_log"
               suffix=".log"
               pattern="%{X-Forwarded-For}i %l %u %t "%r" %s %b %D" />
      </Host>
    </Engine>
  </Service>
</Server>
```

### context.xml

```
xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>
    <!-- Database connection pool -->
    <Resource name="jdbc/MyAppDB"
              auth="Container"
              type="javax.sql.DataSource"
              factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
              driverClassName="org.postgresql.Driver"
              url="jdbc:postgresql://postgres:5432/myappdb"
              username="dbuser"
              password="dbpassword"
              maxActive="100"
              maxIdle="30"
              maxWait="10000"
              validationQuery="SELECT 1"
              testOnBorrow="true"
              testWhileIdle="true"
              timeBetweenEvictionRunsMillis="30000"
              minEvictableIdleTimeMillis="60000" />
    
    <!-- Session configuration -->
    <Manager pathname="" />
    
    <!-- Prevent memory leaks -->
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
</Context>
```

### Logging Configuration

```
properties
# logging.properties
handlers = 1catalina.org.apache.juli.AsyncFileHandler, 2localhost.org.apache.juli.AsyncFileHandler, 3manager.org.apache.juli.AsyncFileHandler, 4host-manager.org.apache.juli.AsyncFileHandler, java.util.logging.ConsoleHandler

.handlers = 1catalina.org.apache.juli.AsyncFileHandler, java.util.logging.ConsoleHandler

java.util.logging.ConsoleHandler.level = INFO
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].handlers = 2localhost.org.apache.juli.AsyncFileHandler

org.apache.catalina.realm.level = INFO
org.apache.catalina.session.level = INFO
```

## Step 3: Tomcat SSL/TLS Configuration

### Generate SSL Certificate

```
bash
# Generate keystore
keytool -genkeypair \
  -alias tomcat \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365 \
  -keystore keystore.jks \
  -storepass changeit \
  -keypass changeit \
  -dname "CN=localhost, OU=DevOps, O=Example, L=City, ST=State, C=US"

# Convert to PKCS12
keytool -importkeystore \
  -srckeystore keystore.jks \
  -destkeystore keystore.p12 \
  -srcstoretype JKS \
  -deststoretype PKCS12 \
  -srcalias tomcat \
  -destalias tomcat \
  -srcstorepass changeit \
  -deststorepass changeit

# Generate CSR
keytool -certreq \
  -alias tomcat \
  -keystore keystore.jks \
  -file server.csr \
  -storepass changeit
```

### Configure SSL in server.xml

```
xml
<!-- HTTPS Connector -->
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true"
           scheme="https" secure="true"
           keystoreFile="/usr/local/tomcat/conf/keystore.jks"
           keystorePass="changeit"
           clientAuth="false" sslProtocol="TLS"
           ciphers="TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
                    TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
                    TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
                    TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384" />
```

## Step 4: Deploy Tomcat on Kubernetes

### Deployment

```
yaml
# tomcat-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-app
  labels:
    app: tomcat
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
        version: v1
    spec:
      containers:
      - name: tomcat
        image: myapp/tomcat:1.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8443
          name: https
        - containerPort: 8009
          name: ajp
        env:
        - name: JAVA_OPTS
          value: "-Xms512m -Xmx1024m -XX:+UseG1GC"
        - name: CATALINA_OPTS
          value: "-Xdebug -Xrunjdwp:server=y,transport=dt_socket,suspend=n"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
        volumeMounts:
        - name: tomcat-logs
          mountPath: /usr/local/tomcat/logs
        - name: tomcat-webapps
          mountPath: /usr/local/tomcat/webapps
      volumes:
      - name: tomcat-logs
        emptyDir: {}
      - name: tomcat-webapps
        emptyDir: {}
```

### Service

```
yaml
# tomcat-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  labels:
    app: tomcat
spec:
  type: ClusterIP
  selector:
    app: tomcat
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
---
# tomcat-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: tomcat-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: tomcat
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
```

### Ingress for Tomcat

```
yaml
# tomcat-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tomcat-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "JSESSIONID"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

## Step 5: Tomcat Clustering

### Configure Session Replication

```
xml
<!-- server.xml - Engine configuration -->
<Engine name="Catalina" jvmRoute="node1">
  <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
           channelSendOptions="8">
    <Manager className="org.apache.catalina.ha.session.DeltaManager"
             expireSessionsOnShutdown="false"
             notifyListenersOnReplication="true"/>
    
    <Channel className="org.apache.catalina.tribes.group.GroupChannel">
      <Membership className="org.apache.catalina.tribes.membership.McastService"
                  bind="127.0.0.1"
                  mcastAddr="228.0.0.4"
                  mcastPort="45564"
                  mcastFrequency="500"
                  mcastDropTime="3000"/>
      
      <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"
                address="auto"
                port="4000"
                autoBind="100"
                selectorTimeout="5000"
                maxThreads="6"/>
      
      <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter"
              transport="org.apache.catalina.tribes.transport.nio.PooledParallelSender"/>
      
      <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/>
      <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatchInterceptor"/>
    </Channel>
    
    <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
           filter=".*\.gif;.*\.js;.*\.css;.*\.png;.*\.jpg;.*\.html;.*\.htm"/>
    
    <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/>
  </Cluster>
</Engine>
```

### context.xml for Clustering

```
xml
<Context>
  <!-- Only one Manager element allowed. Use DeltaManager for cluster session replication. -->
  <Manager className="org.apache.catalina.ha.session.DeltaManager"
           pathname=""
           expireSessionsOnShutdown="false"
           notifyListenersOnReplication="true"/>
</Context>
```

## Step 6: Monitoring Tomcat

### Prometheus Metrics

```
xml
<!-- conf/server.xml - Add JMX remote -->
<Listener className="org.apache.catalina.mbeans.JmxRemoteLifecycleListener"
          rmiServerPortPlatform="10001" rmiRegistryPortPlatform="9090" />
```

```
yaml
# prometheus-config.yaml
- job_name: 'tomcat'
  static_configs:
  - targets: ['tomcat-service:8080']
    labels:
      app: 'tomcat'
      env: 'production'
```

### Health Check Script

```
bash
#!/bin/bash
# healthcheck.sh

response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/)

if [ "$response" -eq 200 ]; then
    exit 0
else
    exit 1
fi
```

## Step 7: Complete Application Example

### Complete docker-compose.yml

```
yaml
version: '3.8'

services:
  tomcat:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp/tomcat:latest
    container_name: tomcat-app
    environment:
      - JAVA_OPTS=-Xms512m -Xmx2048m -XX:+UseG1GC
      - CATALINA_OPTS=-Xdebug
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myappdb
      - DB_USERNAME=dbuser
      - DB_PASSWORD=dbpassword
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    ports:
      - "8080:8080"
      - "8443:8443"
      - "9000:5005"
    depends_on:
      - postgres
      - redis
    networks:
      - app-network
    volumes:
      - app-logs:/usr/local/tomcat/logs
      - app-webapps:/usr/local/tomcat/webapps
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    restart: unless-stopped

  postgres:
    image: postgres:14-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: myappdb
      POSTGRES_USER: dbuser
      POSTGRES_PASSWORD: dbpassword
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx
    depends_on:
      - tomcat
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  app-logs:
  app-webapps:
  postgres-data:
  redis-data:
```

## Conclusion

This project covers:
- Creating Docker images for Tomcat applications
- Multi-stage builds for Java applications
- Tomcat configuration files (server.xml, context.xml)
- SSL/TLS configuration
- Kubernetes deployment and service configuration
- Tomcat clustering and session replication
- Monitoring with Prometheus
- Complete Docker Compose setup with Nginx and databases

## Additional Resources
- [Apache Tomcat Documentation](https://tomcat.apache.org/)
- [Tomcat Docker Official Image](https://github.com/docker-library/tomcat)
- [Tomcat Clustering Guide](https://tomcat.apache.org/tomcat-9.0-doc/cluster-howto.html)
