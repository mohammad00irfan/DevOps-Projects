# Kubernetes DevOps Tasks - Portfolio Showcase

A collection of real-world Kubernetes deployment scenarios solved for the Nautilus DevOps team, demonstrating expertise in container orchestration, configuration management, and troubleshooting.

## 📋 Table of Contents

- [Task 1: Sidecar Pattern for Log Aggregation](#task-1-sidecar-pattern-for-log-aggregation)
- [Task 2: High Availability Web Deployment](#task-2-high-availability-web-deployment)
- [Task 3: Environment Variables Configuration](#task-3-environment-variables-configuration)
- [Task 4: Grafana Analytics Deployment](#task-4-grafana-analytics-deployment)
- [Task 5: Redis Deployment Troubleshooting](#task-5-redis-deployment-troubleshooting)
- [Task 6: Persistent Storage Web Application](#task-6-persistent-storage-web-application)

---

## Task 1: Sidecar Pattern for Log Aggregation

### Objective
Implement the sidecar pattern to ship nginx access and error logs to a log-aggregation service without using persistent volumes.

### Requirements
- Create pod `webserver` with two containers
- Use emptyDir volume for log sharing
- Nginx container for serving web pages
- Ubuntu sidecar container for log shipping

### Solution

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  volumes:
    - name: shared-logs
      emptyDir: {}
  
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
    
    - name: sidecar-container
      image: ubuntu:latest
      command: ["sh", "-c", "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
```

### Key Concepts
- **Separation of Concerns**: Each container has a single responsibility
- **Shared Storage**: emptyDir volume enables inter-container communication
- **Log Aggregation**: Sidecar continuously reads and processes logs

### Verification
```bash
kubectl apply -f webserver-pod.yaml
kubectl get pod webserver
kubectl logs webserver -c sidecar-container
```

---

## Task 2: High Availability Web Deployment

### Objective
Deploy a highly available and scalable static website using nginx with multiple replicas.

### Requirements
- Deployment with 3 replicas for high availability
- NodePort service for external access
- Latest nginx image

### Solution

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30011
```

### Key Concepts
- **High Availability**: 3 replicas ensure service continuity
- **Load Balancing**: Service distributes traffic across pods
- **Scalability**: Easy to scale up/down by changing replica count

### Verification
```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
kubectl get service nginx-service
```

---

## Task 3: Environment Variables Configuration

### Objective
Test application configuration using environment variables for greeting messages.

### Requirements
- Single-run pod with environment variables
- Bash container to echo formatted message
- No restart on completion

### Solution

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-envars-greeting
spec:
  restartPolicy: Never
  containers:
    - name: print-env-container
      image: bash
      command: ["/bin/sh", "-c", "echo \"$(GREETING) $(COMPANY) $(GROUP)\""]
      env:
        - name: GREETING
          value: "Welcome to"
        - name: COMPANY
          value: "xFusionCorp"
        - name: GROUP
          value: "Group"
```

### Key Concepts
- **Environment Variables**: Externalizing configuration
- **One-time Jobs**: Using restartPolicy: Never
- **Variable Interpolation**: Shell expansion of environment variables

### Expected Output
```
Welcome to xFusionCorp Group
```

### Verification
```bash
kubectl apply -f print-envars-greeting.yaml
kubectl logs print-envars-greeting
```

---

## Task 4: Grafana Analytics Deployment

### Objective
Deploy Grafana monitoring tool on Kubernetes for application analytics collection.

### Requirements
- Grafana deployment for data visualization
- NodePort service for external access
- Accessible login page

### Solution

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment-datacenter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 32000
```

### Key Concepts
- **Monitoring Tools**: Deploying third-party applications
- **Service Exposure**: NodePort for external access
- **Default Configuration**: Using official images with defaults

### Access Information
- **URL**: `http://<NODE_IP>:32000`
- **Default Username**: admin
- **Default Password**: admin

### Verification
```bash
kubectl apply -f grafana-deployment.yaml
kubectl get deployment grafana-deployment-datacenter
kubectl get pods -l app=grafana
kubectl get service grafana-service
```

---

## Task 5: Redis Deployment Troubleshooting

### Objective
Debug and fix a broken Redis deployment that went down due to configuration errors.

### Problem Identified
The deployment had two critical typos:
1. **Image name**: `redis:alpin` (incorrect) → `redis:alpine` (correct)
2. **ConfigMap name**: `redis-conig` (incorrect) → `redis-config` (correct)

### Error Symptoms
```
Status: ContainerCreating (stuck)
Error: configmap "redis-conig" not found
Warning: FailedMount - MountVolume.SetUp failed
```

### Troubleshooting Steps

```bash
# 1. Check deployment status
kubectl get deployment redis-deployment

# 2. Check pod status and events
kubectl describe pod -l app=redis

# 3. Identify the errors in Events section
# - ConfigMap "redis-conig" not found
# - Image name typo

# 4. Fix the deployment
kubectl edit deployment redis-deployment
```

### Changes Made
```yaml
# Before (INCORRECT)
image: redis:alpin
configMap:
  name: redis-conig

# After (CORRECT)
image: redis:alpine
configMap:
  name: redis-config
```

### Key Concepts
- **Debugging Skills**: Reading pod events and error messages
- **ConfigMap Integration**: Proper volume mounting
- **Image Tags**: Importance of correct image references
- **Rolling Updates**: Kubernetes automatically creates new pods after edit

### Resolution
After editing the deployment:
- Old pod with typos: Terminated
- New pod created with correct configuration
- Status changed from `ContainerCreating` → `Running`
- ConfigMap successfully mounted

---

## Task 6: Persistent Storage Web Application

### Objective
Deploy a web application using persistent volumes to store application code, following Kubernetes best practices for stateful applications.

### Requirements
- PersistentVolume with hostPath storage
- PersistentVolumeClaim for storage request
- Pod mounting PVC at web server document root
- NodePort service for external access

### Solution

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-xfusion
spec:
  storageClassName: manual
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/itadmin
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-xfusion
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-xfusion
  labels:
    app: xfusion
spec:
  containers:
    - name: container-xfusion
      image: httpd:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: storage
          mountPath: /usr/local/apache2/htdocs
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-xfusion
---
apiVersion: v1
kind: Service
metadata:
  name: web-xfusion
spec:
  type: NodePort
  selector:
    app: xfusion
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30008
```

### Key Concepts
- **Persistent Storage**: Data survives pod restarts
- **Storage Classes**: Manual provisioning vs dynamic
- **Volume Binding**: PVC automatically binds to matching PV
- **Mount Points**: Mounting at application-specific paths

### Storage Architecture
```
PersistentVolume (4Gi) 
    ↓ (binds to)
PersistentVolumeClaim (requests 2Gi)
    ↓ (mounted by)
Pod → /usr/local/apache2/htdocs
```

### Verification
```bash
kubectl apply -f xfusion-deployment.yaml

# Check PV status
kubectl get pv pv-xfusion

# Check PVC binding (should show BOUND)
kubectl get pvc pvc-xfusion

# Check pod status
kubectl get pod pod-xfusion

# Verify volume mount
kubectl describe pod pod-xfusion | grep -A 5 Mounts
```

---

## 🛠️ Technologies Used

- **Kubernetes**: Container orchestration platform
- **Docker**: Container images (nginx, httpd, redis, grafana, ubuntu, bash)
- **YAML**: Configuration and manifest files
- **kubectl**: Kubernetes command-line tool

## 🎯 Skills Demonstrated

### Technical Skills
- Kubernetes pod, deployment, and service configuration
- Volume management (emptyDir, hostPath, PVC/PV)
- Multi-container pod patterns (sidecar)
- Service types (NodePort, ClusterIP)
- Environment variable configuration
- Image and tag management

### DevOps Skills
- Troubleshooting containerized applications
- Reading and interpreting Kubernetes events
- Configuration debugging
- Log aggregation patterns
- High availability design
- Persistent storage architecture

### Best Practices
- Separation of concerns
- Resource labeling and selection
- Proper restart policies
- Storage class usage
- Health checks and monitoring
- Documentation and comments

## 📚 Learning Outcomes

1. **Pattern Implementation**: Successfully implemented the sidecar pattern for log aggregation
2. **High Availability**: Deployed scalable applications with multiple replicas
3. **Troubleshooting**: Debugged production issues by analyzing pod events and logs
4. **Storage Management**: Configured persistent volumes for stateful applications
5. **Service Exposure**: Used NodePort services for external access
6. **Configuration Management**: Managed application configuration via environment variables and ConfigMaps

## 🚀 Quick Start

```bash
# Clone the repository
git clone <your-repo-url>
cd kubernetes-devops-tasks

# Apply any configuration
kubectl apply -f <task-name>.yaml

# Verify deployment
kubectl get all

# Check logs
kubectl logs <pod-name>
```

## 📝 Notes

- All configurations tested on Kubernetes cluster
- kubectl utility pre-configured on jump_host
- NodePort services accessible from outside the cluster
- Persistent volumes use hostPath for development/testing

## 🔗 Related Projects

- [Docker Compose Projects](#)
- [CI/CD Pipelines](#)
- [Infrastructure as Code](#)

---

**Date**: January 2026  
**Environment**: Kubernetes Production Cluster  
**Team**: Nautilus DevOps Team
