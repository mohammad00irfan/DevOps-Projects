# Kubernetes Tasks Showcase

This repository documents two Kubernetes troubleshooting and implementation tasks completed on a live Kubernetes cluster.

---

## Task 1: Fixing Nginx and PHP-FPM Pod Configuration

### Problem Statement
The Nginx and PHP-FPM setup on the Kubernetes cluster encountered a critical issue that halted its functionality. The task involved:
- Investigating and fixing the broken `nginx-phpfpm` pod
- Configuring proper communication between Nginx and PHP-FPM containers
- Deploying a PHP application accessible via NodePort service

### Initial Issues Identified

**Pod Configuration Problems:**
- Single container trying to run both Nginx and PHP-FPM
- Using `php:7.2-fpm-alpine` image which doesn't include Nginx
- Command failure: `nginx -g 'daemon off;'` returning Exit Code 127 (command not found)
- Pod in crash loop with multiple restarts

**Configuration Mismatches:**
- Nginx listening on port 8099 instead of standard port 80
- Volume mount path mismatch between containers
- Nginx config root path (`/var/www/html`) didn't match container mount (`/usr/share/nginx/html`)

### Solution Implemented

**1. Multi-Container Pod Architecture**
Created a proper pod with two separate containers:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-phpfpm
  labels:
    app: php-app
spec:
  containers:
  - name: php-fpm-container
    image: php:7.2-fpm-alpine
    volumeMounts:
    - name: shared-files
      mountPath: /var/www/html
  
  - name: nginx-container
    image: nginx:latest
    volumeMounts:
    - name: shared-files
      mountPath: /var/www/html
    - name: nginx-config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  
  volumes:
  - name: shared-files
    emptyDir: {}
  - name: nginx-config-volume
    configMap:
      name: nginx-config
```

**2. Nginx Configuration Fix**
Updated ConfigMap to use correct port and document root:
```nginx
events {}
http {
  server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    root /var/www/html;
    index index.html index.htm index.php;
    server_name _;
    
    location / {
      try_files $uri $uri/ =404;
    }
    
    location ~ \.php$ {
      include fastcgi_params;
      fastcgi_param REQUEST_METHOD $request_method;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      fastcgi_pass 127.0.0.1:9000;
    }
  }
}
```

**3. Service Configuration**
Configured NodePort service for external access:
```bash
kubectl expose pod nginx-phpfpm --name=nginx-service --type=NodePort --port=80 --target-port=80
kubectl patch svc nginx-service --type='json' -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30008}]'
```

**4. Application Deployment**
```bash
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container
```

### Key Technical Decisions

- **Shared Volume Strategy**: Used `emptyDir` volume mounted at the same path (`/var/www/html`) in both containers to ensure file accessibility
- **FastCGI Communication**: Configured `fastcgi_pass 127.0.0.1:9000` since containers in the same pod share network namespace
- **Container Separation**: Separated concerns with dedicated containers for web server and PHP processing

### Results
✅ Pod running successfully (2/2 containers ready)  
✅ Nginx serving on port 80  
✅ PHP-FPM processing PHP files correctly  
✅ Application accessible via NodePort 30008  
✅ Proper file sharing between containers

---

## Task 2: Implementing Shared Volume Between Containers

### Problem Statement
Create a pod demonstrating volume sharing between multiple containers to handle temporary data storage. Requirements:
- Two Debian containers in a single pod
- Shared emptyDir volume mounted at different paths
- Verify data accessibility across containers

### Implementation

**Pod Specification:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-share-datacenter
spec:
  containers:
  - name: volume-container-datacenter-1
    image: debian:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/blog
  
  - name: volume-container-datacenter-2
    image: debian:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/demo
  
  volumes:
  - name: volume-share
    emptyDir: {}
```

### Deployment Steps

```bash
# Create the pod
kubectl apply -f volume-share-pod.yaml

# Wait for pod readiness
kubectl wait --for=condition=ready pod/volume-share-datacenter --timeout=60s

# Create test file in container 1
kubectl exec volume-share-datacenter -c volume-container-datacenter-1 -- \
  sh -c 'echo "This is shared volume test content" > /tmp/blog/blog.txt'

# Verify from container 1
kubectl exec volume-share-datacenter -c volume-container-datacenter-1 -- \
  cat /tmp/blog/blog.txt

# Verify from container 2 (different mount path)
kubectl exec volume-share-datacenter -c volume-container-datacenter-2 -- \
  cat /tmp/demo/blog.txt
```

### Verification Results

```bash
# Container 1 perspective
$ kubectl exec volume-share-datacenter -c volume-container-datacenter-1 -- cat /tmp/blog/blog.txt
This is shared volume test content

# Container 2 perspective
$ kubectl exec volume-share-datacenter -c volume-container-datacenter-2 -- cat /tmp/demo/blog.txt
This is shared volume test content

# File listing from container 2
$ kubectl exec volume-share-datacenter -c volume-container-datacenter-2 -- ls -la /tmp/demo/
total 12
drwxrwxrwx 2 root root 4096 Jan 16 00:40 .
drwxrwxrwt 1 root root 4096 Jan 16 00:40 ..
-rw-r--r-- 1 root root   35 Jan 16 00:40 blog.txt
```

### Key Concepts Demonstrated

**Volume Sharing Mechanism:**
- Single `emptyDir` volume mounted at different paths in each container
- Container 1: `/tmp/blog`
- Container 2: `/tmp/demo`
- Both paths point to same underlying storage

**Use Cases:**
- Sharing temporary data between application and sidecar containers
- Log aggregation scenarios
- Data processing pipelines
- Inter-container communication via filesystem

### Results
✅ Pod created with 2/2 containers running  
✅ EmptyDir volume successfully shared  
✅ File written in container 1 immediately accessible in container 2  
✅ Data persistence verified across different mount paths  
✅ Demonstrates proper volume configuration in multi-container pods

---

## Technical Skills Demonstrated

### Kubernetes Concepts
- Multi-container pod design patterns
- Volume management (emptyDir)
- ConfigMap usage for configuration injection
- Service exposure (NodePort)
- Container networking and communication
- Pod lifecycle management

### Troubleshooting Skills
- Pod failure diagnosis using `kubectl describe` and `kubectl logs`
- Exit code analysis (Exit Code 127 - command not found)
- Configuration debugging
- Path and mount point validation
- Service connectivity testing

### Configuration Management
- YAML manifest creation and management
- ConfigMap patching and updates
- Service configuration and port mapping
- Volume mount configuration
- Container command specification

### Tools Used
- kubectl (pod management, exec, cp, logs, describe)
- curl (endpoint testing)
- YAML (resource definition)
- Shell commands (file operations, testing)

---

## Lessons Learned

1. **Container Image Selection**: Ensure base images contain all required binaries (nginx not in php-fpm image)
2. **Volume Mount Consistency**: Mount shared volumes at the same path to avoid confusion
3. **Pod vs Deployment**: Standalone pods aren't automatically recreated; use Deployments for production
4. **Network Namespace**: Containers in same pod share network, enabling localhost communication
5. **Configuration Validation**: Always verify config changes with test commands before exposing services

---

## Environment
- **Platform**: Kubernetes Cluster (KodeKloud)
- **kubectl**: Configured on jump host
- **Images Used**: 
  - `nginx:latest` / `nginx:alpine`
  - `php:7.2-fpm-alpine`
  - `debian:latest`

---

## Commands Reference

### Task 1 - Nginx PHP-FPM Fix
```bash
# Delete broken pod
kubectl delete pod nginx-phpfpm

# Create corrected pod
kubectl apply -f nginx-phpfpm-pod.yaml

# Update ConfigMap
kubectl patch configmap nginx-config --type merge -p '{...}'

# Copy application files
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container

# Verify deployment
kubectl exec nginx-phpfpm -c nginx-container -- curl localhost/index.php
```

### Task 2 - Shared Volume
```bash
# Create pod with shared volume
kubectl apply -f volume-share-pod.yaml

# Test file sharing
kubectl exec volume-share-datacenter -c volume-container-datacenter-1 -- \
  sh -c 'echo "test" > /tmp/blog/blog.txt'

kubectl exec volume-share-datacenter -c volume-container-datacenter-2 -- \
  cat /tmp/demo/blog.txt
```
