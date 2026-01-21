# Kubernetes Deployment Scenarios - Complete Guide

A comprehensive collection of real-world Kubernetes deployment scenarios, including init containers, secrets management, persistent storage, and application deployments.

## 📋 Table of Contents

1. [Init Containers with Shared Volumes](#1-init-containers-with-shared-volumes)
2. [Kubernetes Secrets Management](#2-kubernetes-secrets-management)
3. [Multi-Tier Application Deployment](#3-multi-tier-application-deployment)
4. [Troubleshooting Python Flask Deployment](#4-troubleshooting-python-flask-deployment)
5. [Redis Deployment with ConfigMap](#5-redis-deployment-with-configmap)

---

## 1. Init Containers with Shared Volumes

**Scenario**: Deploy applications with prerequisites that need to be completed before the main container starts.

### Requirements
- Init container to write configuration files
- Shared volume between init and main containers
- Main container reads data created by init container

### Solution

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ic-deploy-datacenter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ic-datacenter
  template:
    metadata:
      labels:
        app: ic-datacenter
    spec:
      initContainers:
      - name: ic-msg-datacenter
        image: fedora:latest
        command:
          - '/bin/bash'
          - '-c'
          - 'echo Init Done - Welcome to xFusionCorp Industries > /ic/ecommerce'
        volumeMounts:
        - name: ic-volume-datacenter
          mountPath: /ic
      containers:
      - name: ic-main-datacenter
        image: fedora:latest
        command:
          - '/bin/bash'
          - '-c'
          - 'while true; do cat /ic/ecommerce; sleep 5; done'
        volumeMounts:
        - name: ic-volume-datacenter
          mountPath: /ic
      volumes:
      - name: ic-volume-datacenter
        emptyDir: {}
```

### Key Concepts
- **Init Containers**: Run before main containers and must complete successfully
- **emptyDir Volumes**: Temporary storage shared between containers in a pod
- **Use Cases**: Configuration setup, data preparation, prerequisite checks

### Verification
```bash
# Check deployment
kubectl get deployment ic-deploy-datacenter

# Verify init container completed
kubectl describe pod -l app=ic-datacenter

# View main container logs
kubectl logs -f <pod-name> -c ic-main-datacenter
```

---

## 2. Kubernetes Secrets Management

**Scenario**: Store sensitive information (licenses, passwords) securely and mount them in containers.

### Requirements
- Create secret from existing file
- Mount secret as volume in pod
- Keep container running for verification

### Solution

#### Step 1: Create Secret from File
```bash
# Create generic secret from file
kubectl create secret generic ecommerce --from-file=/opt/ecommerce.txt

# Verify secret creation
kubectl get secrets
kubectl describe secret ecommerce
```

#### Step 2: Deploy Pod with Secret Mount
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-xfusion
spec:
  containers:
  - name: secret-container-xfusion
    image: fedora:latest
    command: ["/bin/bash", "-c", "sleep infinity"]
    volumeMounts:
    - name: secret-volume
      mountPath: /opt/cluster
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: ecommerce
```

### Key Concepts
- **Generic Secrets**: Created from files, literals, or directories
- **Secret Volumes**: Mount secrets as files in containers
- **Read-Only Mounts**: Security best practice for sensitive data
- **Symbolic Links**: Kubernetes uses symlinks for atomic secret updates

### Verification
```bash
# Exec into container
kubectl exec -it secret-xfusion -c secret-container-xfusion -- /bin/bash

# Inside container, verify secret
ls -la /opt/cluster
cat /opt/cluster/ecommerce.txt

# From outside (one-liner)
kubectl exec secret-xfusion -c secret-container-xfusion -- cat /opt/cluster/ecommerce.txt
```

---

## 3. Multi-Tier Application Deployment

**Scenario**: Deploy a complete web application with frontend (Iron Gallery) and backend (MariaDB) database.

### Architecture
- **Frontend**: Iron Gallery (Nginx-based image gallery)
- **Backend**: MariaDB database
- **Services**: NodePort for external access, ClusterIP for internal communication

### Complete Solution

```yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: iron-namespace-datacenter

---
# Iron Gallery Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-gallery-deployment-datacenter
  namespace: iron-namespace-datacenter
spec:
  replicas: 1
  selector:
    matchLabels:
      run: iron-gallery
  template:
    metadata:
      labels:
        run: iron-gallery
    spec:
      containers:
      - name: iron-gallery-container-datacenter
        image: kodekloud/irongallery:2.0
        resources:
          limits:
            memory: "100Mi"
            cpu: "50m"
        volumeMounts:
        - name: config
          mountPath: /usr/share/nginx/html/data
        - name: images
          mountPath: /usr/share/nginx/html/uploads
      volumes:
      - name: config
        emptyDir: {}
      - name: images
        emptyDir: {}

---
# Iron DB Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-db-deployment-datacenter
  namespace: iron-namespace-datacenter
spec:
  replicas: 1
  selector:
    matchLabels:
      db: mariadb
  template:
    metadata:
      labels:
        db: mariadb
    spec:
      containers:
      - name: iron-db-container-datacenter
        image: kodekloud/irondb:2.0
        env:
        - name: MYSQL_DATABASE
          value: "database_web"
        - name: MYSQL_ROOT_PASSWORD
          value: "R00tP@ssw0rd#2024"
        - name: MYSQL_PASSWORD
          value: "Us3rP@ssw0rd#2024"
        - name: MYSQL_USER
          value: "iron_user"
        volumeMounts:
        - name: db
          mountPath: /var/lib/mysql
      volumes:
      - name: db
        emptyDir: {}

---
# Database Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: iron-db-service-datacenter
  namespace: iron-namespace-datacenter
spec:
  type: ClusterIP
  selector:
    db: mariadb
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306

---
# Gallery Service (NodePort)
apiVersion: v1
kind: Service
metadata:
  name: iron-gallery-service-datacenter
  namespace: iron-namespace-datacenter
spec:
  type: NodePort
  selector:
    run: iron-gallery
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 32678
```

### Key Concepts
- **Namespaces**: Isolate resources and provide logical separation
- **Resource Limits**: Control CPU and memory usage
- **Environment Variables**: Configure application settings
- **Service Types**: 
  - ClusterIP for internal communication
  - NodePort for external access
- **Multi-Volume Pods**: Multiple emptyDir volumes for different data types

### Database Connection Details
```
Host: iron-db-service-datacenter
Port: 3306
Database: database_web
User: iron_user
Password: Us3rP@ssw0rd#2024
```

### Deployment & Verification
```bash
# Deploy all resources
kubectl apply -f iron-gallery-deployment.yaml

# Verify all resources
kubectl get all -n iron-namespace-datacenter

# Check pods are running
kubectl get pods -n iron-namespace-datacenter

# Access the application
curl http://<node-ip>:32678
```

---

## 4. Troubleshooting Python Flask Deployment

**Scenario**: Debug and fix a broken Flask application deployment.

### Common Issues Found
1. ❌ Wrong image name: `poroko/flask-app-demo` → `poroko/flask-demo-app`
2. ❌ Label mismatch: Deployment uses different labels than service selector
3. ❌ Wrong service port: `8080` → `5000` (Flask default)

### Diagnostic Commands
```bash
# Check deployment status
kubectl get deployment python-deployment-datacenter

# Check pod status and errors
kubectl get pods -l app=python-deployment-datacenter
kubectl describe pod <pod-name>

# Check service configuration
kubectl get svc
kubectl describe svc python-service-datacenter
```

### Fixed Solution
```yaml
---
# Fixed Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-deployment-datacenter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python-deployment-datacenter
  template:
    metadata:
      labels:
        app: python-deployment-datacenter
    spec:
      containers:
      - name: python-container-datacenter
        image: poroko/flask-demo-app
        ports:
        - containerPort: 5000

---
# Fixed Service
apiVersion: v1
kind: Service
metadata:
  name: python-service-datacenter
spec:
  type: NodePort
  selector:
    app: python-deployment-datacenter
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
    nodePort: 32345
```

### Key Troubleshooting Steps
1. **Check Image Pull Status**: Look for `ImagePullBackOff` or `ErrImagePull`
2. **Verify Label Selectors**: Ensure deployment labels match service selectors
3. **Validate Port Configuration**: Check containerPort, targetPort, and service port alignment
4. **Review Pod Events**: Use `kubectl describe pod` to see error messages

### Apply Fixes
```bash
# Delete broken resources
kubectl delete deployment python-deployment-datacenter
kubectl delete svc python-service-datacenter

# Apply corrected configuration
kubectl apply -f python-flask-fixed.yaml

# Verify it's working
kubectl get pods
curl http://localhost:32345
```

---

## 5. Redis Deployment with ConfigMap

**Scenario**: Deploy Redis as an in-memory cache with custom configuration for performance testing.

### Requirements
- ConfigMap for Redis configuration
- CPU resource requests
- Multiple volume mounts
- Expose Redis port

### Complete Solution

```yaml
---
# ConfigMap with Redis Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-redis-config
data:
  redis-config: |
    maxmemory 2mb

---
# Redis Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis-container
        image: redis:alpine
        resources:
          requests:
            cpu: "1"
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /redis-master-data
        - name: redis-config
          mountPath: /redis-master
      volumes:
      - name: data
        emptyDir: {}
      - name: redis-config
        configMap:
          name: my-redis-config
```

### Key Concepts
- **ConfigMaps**: Store non-sensitive configuration data
- **Resource Requests**: Guarantee minimum CPU/memory allocation
- **Multiple Volumes**: Separate data and configuration storage
- **ConfigMap as Volume**: Mount configuration files from ConfigMap

### Deployment & Verification
```bash
# Deploy Redis
kubectl apply -f redis-deployment.yaml

# Verify ConfigMap
kubectl get configmap my-redis-config
kubectl describe configmap my-redis-config

# Check deployment
kubectl get deployment redis-deployment
kubectl get pods -l app=redis

# Verify configuration is mounted
kubectl exec -it <redis-pod-name> -- cat /redis-master/redis-config

# Test Redis connectivity
kubectl exec -it <redis-pod-name> -- redis-cli ping
# Expected output: PONG
```

### Resource Verification
```bash
# Check CPU requests
kubectl describe pod -l app=redis | grep -A 5 "Requests"

# Verify volume mounts
kubectl describe pod -l app=redis | grep -A 10 "Mounts"

# Check exposed port
kubectl describe pod -l app=redis | grep "Port"
```

---

## 🎯 Best Practices Applied

### 1. **Resource Management**
- Always set resource requests and limits
- Use appropriate values based on application needs
- Monitor resource usage in production

### 2. **Security**
- Use secrets for sensitive data
- Mount secrets as read-only volumes
- Use strong passwords for databases
- Avoid hardcoding credentials

### 3. **Configuration Management**
- Use ConfigMaps for application configuration
- Separate configuration from code
- Version control your manifests

### 4. **Labeling & Selection**
- Use consistent label naming conventions
- Ensure selectors match labels exactly
- Use meaningful label values

### 5. **Volume Management**
- Use emptyDir for temporary data
- Use ConfigMaps/Secrets for configuration
- Consider persistent volumes for production data

### 6. **Troubleshooting**
- Always check pod events with `kubectl describe`
- Review logs with `kubectl logs`
- Verify service endpoints match pod labels
- Check image names and tags carefully

---

## 🚀 Quick Reference Commands

### Deployment Operations
```bash
# Apply configuration
kubectl apply -f <filename.yaml>

# Delete resources
kubectl delete -f <filename.yaml>
kubectl delete deployment <name>
kubectl delete service <name>

# Scale deployment
kubectl scale deployment <name> --replicas=3
```

### Inspection Commands
```bash
# Get resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get configmaps
kubectl get secrets

# Describe resources
kubectl describe pod <name>
kubectl describe deployment <name>
kubectl describe service <name>

# View logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Follow logs
kubectl logs <pod-name> -c <container-name>
```

### Interactive Debugging
```bash
# Exec into container
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container-name> -- sh

# Port forwarding
kubectl port-forward <pod-name> 8080:80

# Copy files
kubectl cp <pod-name>:/path/to/file ./local-file
```

### Label & Selector Operations
```bash
# Get pods by label
kubectl get pods -l app=myapp

# Add label to resource
kubectl label pod <pod-name> env=production

# Remove label
kubectl label pod <pod-name> env-
```

---

## 📚 Learning Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Patterns](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)

---

## 📝 Notes

- All examples use the `default` namespace unless specified
- `kubectl` must be configured to work with your cluster
- Adjust resource limits based on your cluster capacity
- Always validate YAML syntax before applying
- Use `--dry-run=client -o yaml` to preview changes

---

## 🤝 Contributing

Feel free to submit issues and enhancement requests!

## 📄 License

These examples are provided for educational purposes.

---

**Author**: DevOps Engineer  
**Last Updated**: January 2026
