# KUBERNETES ADVANCED - NETWORKING, STORAGE & SECURITY

## Project Overview

This project covers advanced Kubernetes topics including networking, persistent storage, security best practices, and operational tools.

### Objectives
1. Understand Kubernetes networking model
2. Implement persistent storage with PV/PVC
3. Configure Kubernetes security (RBAC, Network Policies)
4. Use Helm for package management
5. Implement service mesh with Istio
6. Configure ingress controllers

## Prerequisites
- Kubernetes cluster (minikube, kind, or cloud)
- kubectl installed
- Helm 3 installed

## Step 1: Kubernetes Networking

### Pod Networking

```
bash
# Check pod networking
kubectl get pods -o wide

# Describe pod network details
kubectl describe pod <pod-name>

# Test pod-to-pod communication
kubectl exec -it <pod-name> -- ping <another-pod-ip>

# View CNI configuration
cat /etc/cni/net.d/10-flannel.conflist

# Check network policies
kubectl get networkpolicies
```

### Cluster DNS

```
bash
# Check CoreDNS
kubectl get pods -n kube-system | grep coredns

# Test DNS resolution
kubectl exec -it <pod-name> -- nslookup kubernetes.default
kubectl exec -it <pod-name> -- nslookup <service-name>

# View DNS configuration
kubectl exec -it <pod-name> -- cat /etc/resolv.conf

# Debug DNS issues
kubectl run dnsutils --image=tutum/dnsutils --restart=Never -- sleep 3600
kubectl exec -it dnsutils -- nslookup kubernetes.default
```

## Step 2: Ingress Controllers

### Install NGINX Ingress Controller

```
bash
# Install NGINX Ingress Controller using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Or install using manifests
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Configure Ingress Resources

```
yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
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
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    - admin.example.com
    secretName: tls-secret
```

### Secure Ingress with TLS

```
bash
# Create TLS secret
kubectl create secret tls tls-secret \
  --key server.key \
  --cert server.crt

# Create wildcard certificate
kubectl create secret tls wildcard-tls \
  --key wildcard.key \
  --cert wildcard.crt \
  --namespace default

# Use cert-manager for automatic TLS
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create ClusterIssuer for Let's Encrypt
yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

## Step 3: Persistent Storage

### Create PersistentVolume

```
yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Create PersistentVolumeClaim

```
yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
  selector:
    matchLabels:
      type: "local"
```

### Use PVC in Pod

```
yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /var/www/html
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### StatefulSet with Storage

```
yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Step 4: Kubernetes Security

### RBAC Configuration

```
bash
# Create namespace
kubectl create namespace myapp

# Create service account
kubectl create serviceaccount myapp-sa -n myapp

# Create role
kubectl create role pod-reader \
  --verb=get \
  --verb=list \
  --verb=watch \
  --resource=pods \
  -n myapp

# Create cluster role
kubectl create clusterrole pod-cluster-reader \
  --verb=get \
  --verb=list \
  --verb=watch \
  --resource=pods

# Create role binding
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=myapp:myapp-sa \
  -n myapp

# Create cluster role binding
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=myapp:myapp-sa

# Check permissions
kubectl auth can-i get pods --as=system:serviceaccount:myapp:myapp-sa
```

### Network Policies

```
yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend
      - namespaceSelector:
          matchLabels:
            name: monitoring
      ports:
      - protocol: TCP
        port: 8080
  egress:
    - to:
      - podSelector:
          matchLabels:
            app: database
      ports:
      - protocol: TCP
        port: 5432
    - to:
      - namespaceSelector: {}
      ports:
      - protocol: TCP
        port: 53
      - protocol: UDP
        port: 53
```

### Pod Security Standards

```
yaml
# pod-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
```

## Step 5: Helm Package Manager

### Install and Use Helm

```
bash
# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Search charts
helm search repo nginx
helm search hub wordpress

# Install chart
helm install my-release bitnami/nginx
helm install my-release bitnami/nginx \
  --namespace my-namespace \
  --create-namespace

# Upgrade
helm upgrade my-release bitnami/nginx \
  --set service.type=LoadBalancer

# Rollback
helm rollback my-release 1

# Uninstall
helm uninstall my-release
```

### Create Custom Helm Chart

```
bash
# Create chart
helm create mychart

# Chart structure
# mychart/
#   Chart.yaml
#   values.yaml
#   charts/
#   templates/
#   LICENSE
#   README.md
```

```
yaml
# Chart.yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application
version: 1.0.0
appVersion: "1.0"
```

```
yaml
# values.yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.21"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

```
yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: http
        readinessProbe:
          httpGet:
            path: /
            port: http
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

## Step 6: Service Mesh with Istio

### Install Istio

```
bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Add to path
export PATH=$PATH:$HOME/istio-1.18.0/bin

# Install Istio with demo profile
istioctl install --set profile=demo -y

# Enable automatic sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify installation
kubectl get pods -n istio-system
```

### Configure Istio Resources

```
yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /api/v1
    route:
    - destination:
        host: myapp-backend
        port:
          number: 8080
      weight: 80
    - destination:
        host: myapp-backend-v2
        port:
          number: 8080
      weight: 20
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: myapp-frontend
        port:
          number: 80

---
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp-backend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: LEAST_REQUEST
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### Observability

```
bash
# Install Kiali for service visualization
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/kiali.yaml

# Install Jaeger for tracing
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/jaeger.yaml

# Install Grafana for metrics
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/grafana.yaml

# Install Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/addons/prometheus.yaml

# Access Kiali dashboard
istioctl dashboard kiali
```

## Step 7: Kubernetes Operations

### Resource Management

```
bash
# Resource limits
kubectl limitrange default-mem-limit --mem-hard=1Gi --cpu-hard=2000m -n default
kubectl limitrange default-mem-request --mem-request=100Mi --cpu-request=100m -n default

# Resource quotas
kubectl create quota my-quota \
  --hard=cpu=4,memory=8Gi,pods=10,services=5 \
  -n myapp

# Pod disruption budget
kubectl create pdb my-pdb \
  --selector=app=myapp \
  --min-available=2

# Vertical Pod Autoscaler
kubectl autoscale deployment myapp \
  --cpu-percent=80 \
  --min=2 \
  --max=10
```

### Debugging

```
bash
# Describe resources
kubectl describe pod <pod-name>
kubectl describe service <service-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs -f <pod-name> --tail=100

# Execute into container
kubectl exec -it <pod-name> -- /bin/bash

# Port forward
kubectl port-forward <pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80

# Debug with ephemeral container
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# Check events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod-name>
```

## Conclusion

This project covers:
- Kubernetes networking and DNS
- NGINX Ingress Controller setup
- TLS/HTTPS configuration
- Persistent storage (PV/PVC)
- RBAC configuration
- Network policies
- Pod security
- Helm package management
- Istio service mesh
- Kubernetes operations and debugging

## Additional Resources
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
