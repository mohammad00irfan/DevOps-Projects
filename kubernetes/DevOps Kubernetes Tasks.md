---

# Nautilus DevOps Kubernetes Tasks

This repository documents the step-by-step tasks performed by the Nautilus DevOps team for Kubernetes application management.

---

## 1️⃣ Task: Create a Pod `pod-httpd`

**Objective:**
Create a pod named `pod-httpd` using the `httpd:latest` image. Set the label `app=httpd_app` and container name `httpd-container`.

### Steps:

1. **Verify kubectl access**

```bash
kubectl get nodes
```

2. **Create pod YAML file**
   File: `pod-httpd.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    app: httpd_app
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
```

3. **Deploy the pod**

```bash
kubectl apply -f pod-httpd.yaml
```

Output:

```
pod/pod-httpd created
```

4. **Verify the pod**

```bash
kubectl get pods
kubectl describe pod pod-httpd
```

✅ Confirmed:

* Pod name: `pod-httpd`
* Container name: `httpd-container`
* Image: `httpd:latest`
* Label: `app=httpd_app`
* Pod is running

---

## 2️⃣ Task: Create a Deployment `httpd`

**Objective:**
Create a deployment named `httpd` using the `httpd:latest` image.

### Steps:

1. **Verify kubectl access**

```bash
kubectl get nodes
```

2. **Create deployment YAML file**
   File: `httpd-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
        - name: httpd-container
          image: httpd:latest
```

3. **Deploy the deployment**

```bash
kubectl apply -f httpd-deployment.yaml
```

Output:

```
deployment.apps/httpd created
```

4. **Verify deployment and pods**

```bash
kubectl get deployments
kubectl get pods
```

Expected pod name: `httpd-xxxxxxxxxx-xxxxx`

5. **Describe deployment**

```bash
kubectl describe deployment httpd
```

✅ Confirmed:

* Deployment name: `httpd`
* Container name: `httpd-container`
* Image: `httpd:latest`
* Replicas: 1 (available and running)
* Deployment conditions: Available and Progressing ✅

---

### ✅ Task Completion Status

| Task               | Status              |
| ------------------ | ------------------- |
| Pod `pod-httpd`    | Completed & Running |
| Deployment `httpd` | Completed & Running |

---

This document can be **committed to GitHub** as the official record of Nautilus DevOps Kubernetes lab tasks.

```bash
git add .
git commit -m "Add Kubernetes pod and deployment tasks documentation"
git push origin master
```

---
