# Kubernetes Guestbook Application

A multi-tier guestbook application deployed on Kubernetes, featuring Redis backend and PHP frontend with proper resource management and scaling.

## Architecture Overview

This application consists of two main tiers:

### Backend Tier (Redis)
- **Redis Master**: Single instance for write operations
- **Redis Slave**: Two replicas for read operations with DNS-based service discovery

### Frontend Tier (PHP)
- **PHP Frontend**: Three replicas for high availability
- Web interface for guest entries management

## Prerequisites

- Kubernetes cluster (v1.19+)
- kubectl configured to communicate with your cluster
- Access to pull images from Google Container Registry

## Deployment Specifications

### Backend Components

#### Redis Master
- **Deployment**: `redis-master`
- **Replicas**: 1
- **Container**: `master-redis-devops`
- **Image**: `redis`
- **Resources**: 
  - CPU: 100m
  - Memory: 100Mi
- **Port**: 6379

#### Redis Slave
- **Deployment**: `redis-slave`
- **Replicas**: 2
- **Container**: `slave-redis-devops`
- **Image**: `gcr.io/google_samples/gb-redisslave:v3`
- **Resources**: 
  - CPU: 100m
  - Memory: 100Mi
- **Environment**: 
  - `GET_HOSTS_FROM=dns`
- **Port**: 6379

### Frontend Components

#### PHP Frontend
- **Deployment**: `frontend`
- **Replicas**: 3
- **Container**: `php-redis-devops`
- **Image**: `gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff`
- **Resources**: 
  - CPU: 100m
  - Memory: 100Mi
- **Environment**: 
  - `GET_HOSTS_FROM=dns`
- **Port**: 80
- **Service Type**: NodePort (30009)

## Installation

### Quick Start

1. **Clone the repository**
```bash
git clone <your-repo-url>
cd kubernetes-guestbook
```

2. **Deploy the application**
```bash
kubectl apply -f guestbook.yaml
```

3. **Verify deployment**
```bash
# Check all resources
kubectl get all

# Check deployments
kubectl get deployments

# Check pods
kubectl get pods

# Check services
kubectl get services
```

### Step-by-Step Deployment

1. **Deploy Redis Master**
```bash
kubectl apply -f guestbook.yaml
```
This creates the Redis master deployment and service.

2. **Deploy Redis Slaves**
The manifest automatically creates the Redis slave deployment with 2 replicas.

3. **Deploy Frontend**
The frontend deployment with 3 replicas is created automatically.

4. **Verify all pods are running**
```bash
kubectl get pods -w
```
Wait until all pods show `Running` status.

## Accessing the Application

The frontend service is exposed via NodePort on port **30009**.

### Access Methods

**Via NodePort:**
```bash
# Get node IP
kubectl get nodes -o wide

# Access the application
http://<NODE_IP>:30009
```

**Via Port Forward (for testing):**
```bash
kubectl port-forward service/frontend 8080:80
# Access at http://localhost:8080
```

**Via LoadBalancer (if available):**
```bash
# Modify service type to LoadBalancer
kubectl patch service frontend -p '{"spec":{"type":"LoadBalancer"}}'

# Get external IP
kubectl get service frontend
```

## Configuration

### Resource Limits

All containers have resource requests configured:
- **CPU**: 100m per container
- **Memory**: 100Mi per container

To modify resources, edit the `guestbook.yaml` file and update the resources section.

### Scaling

**Scale Frontend:**
```bash
kubectl scale deployment frontend --replicas=5
```

**Scale Redis Slaves:**
```bash
kubectl scale deployment redis-slave --replicas=3
```

**Note**: Redis master should remain at 1 replica for data consistency.

## Monitoring

### Check Pod Status
```bash
kubectl get pods -l app=guestbook
kubectl get pods -l app=redis
```

### View Logs
```bash
# Frontend logs
kubectl logs -l app=guestbook -f

# Redis master logs
kubectl logs -l role=master -f

# Redis slave logs
kubectl logs -l role=slave -f
```

### Describe Resources
```bash
kubectl describe deployment frontend
kubectl describe service frontend
kubectl describe deployment redis-master
```

## Troubleshooting

### Pods Not Starting
```bash
# Check pod events
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>
```

### Service Not Accessible
```bash
# Verify service endpoints
kubectl get endpoints

# Check service configuration
kubectl describe service frontend
```

### Redis Connection Issues
```bash
# Test Redis master connectivity
kubectl exec -it <redis-master-pod> -- redis-cli ping

# Test Redis slave connectivity
kubectl exec -it <redis-slave-pod> -- redis-cli ping
```

## Cleanup

Remove all resources created by this application:

```bash
kubectl delete -f guestbook.yaml
```

Or delete individual components:
```bash
kubectl delete deployment frontend redis-master redis-slave
kubectl delete service frontend redis-master redis-slave
```

## Architecture Diagram

```
┌─────────────────────────────────────────┐
│         Frontend Tier (NodePort)        │
│  ┌───────────────────────────────────┐  │
│  │  PHP Frontend (3 replicas)        │  │
│  │  Port: 80 → NodePort: 30009       │  │
│  └───────────────────────────────────┘  │
└─────────────────┬───────────────────────┘
                  │
                  │ DNS Service Discovery
                  │
┌─────────────────┴───────────────────────┐
│           Backend Tier (Redis)          │
│  ┌──────────────┐   ┌────────────────┐  │
│  │ Redis Master │   │ Redis Slave    │  │
│  │ (1 replica)  │   │ (2 replicas)   │  │
│  │ Port: 6379   │   │ Port: 6379     │  │
│  └──────────────┘   └────────────────┘  │
└─────────────────────────────────────────┘
```

## Features

- ✅ High availability with multiple frontend replicas
- ✅ Read scalability with Redis slave replicas
- ✅ DNS-based service discovery
- ✅ Resource management with requests/limits
- ✅ NodePort service for external access
- ✅ Production-ready configuration

## Technology Stack

- **Container Orchestration**: Kubernetes
- **Backend Database**: Redis
- **Frontend**: PHP
- **Service Discovery**: Kubernetes DNS
- **Load Balancing**: Kubernetes Service


- Uses official Redis and Google sample images
