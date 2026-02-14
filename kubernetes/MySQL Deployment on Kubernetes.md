# MySQL Deployment on Kubernetes

This repository contains the complete configuration for deploying a MySQL server on a Kubernetes cluster with persistent storage, secrets management, and NodePort service exposure.

## 📋 Requirements

The Nautilus DevOps team has finalized the following requirements for MySQL deployment:

1. Create a PersistentVolume `mysql-pv` with capacity of `250Mi`
2. Create a PersistentVolumeClaim `mysql-pv-claim` requesting `250Mi` of storage
3. Create a deployment named `mysql-deployment` with MySQL image and PersistentVolume mounted at `/var/lib/mysql`
4. Create a NodePort service named `mysql` with nodePort `30007`
5. Create three secrets for MySQL credentials and database configuration
6. Configure environment variables from secrets

## 🚀 Deployment

### Prerequisites

- Kubernetes cluster up and running
- `kubectl` configured to communicate with the cluster
- Appropriate permissions to create resources

### Quick Start

1. **Clone this repository** (or save the YAML file):
```bash
git clone <your-repo-url>
cd <repo-directory>
```

2. **Apply the configuration**:
```bash
kubectl apply -f mysql-deployment.yaml
```

3. **Verify the deployment**:
```bash
# Check all resources
kubectl get secrets
kubectl get pv
kubectl get pvc
kubectl get deployments
kubectl get pods
kubectl get svc
```

## 📦 Resources Created

### Secrets

Three secrets are created with the following credentials:

| Secret Name | Key | Value |
|-------------|-----|-------|
| `mysql-root-pass` | password | YUIidhb667 |
| `mysql-user-pass` | username | kodekloud_roy |
| `mysql-user-pass` | password | dCV3szSGNA |
| `mysql-db-url` | database | kodekloud_db10 |

### Storage

- **PersistentVolume**: `mysql-pv`
  - Capacity: 250Mi
  - Access Mode: ReadWriteOnce
  - Storage Type: hostPath (`/var/lib/mysql-data`)
  - Reclaim Policy: Retain

- **PersistentVolumeClaim**: `mysql-pv-claim`
  - Request: 250Mi
  - Access Mode: ReadWriteOnce

### Deployment

- **Name**: `mysql-deployment`
- **Image**: mysql:5.7
- **Replicas**: 1
- **Mount Path**: `/var/lib/mysql`
- **Environment Variables**:
  - `MYSQL_ROOT_PASSWORD` (from secret: mysql-root-pass)
  - `MYSQL_DATABASE` (from secret: mysql-db-url)
  - `MYSQL_USER` (from secret: mysql-user-pass)
  - `MYSQL_PASSWORD` (from secret: mysql-user-pass)

### Service

- **Name**: `mysql`
- **Type**: NodePort
- **Port**: 3306
- **NodePort**: 30007

## 🔧 Testing the Deployment

### Method 1: Using MySQL Client Pod (Recommended)

Connect using a temporary MySQL client pod:

```bash
# Connect as root user
kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- mysql -h mysql -u root -pYUIidhb667

# Or connect as application user
kubectl run mysql-client --image=mysql:5.7 -it --rm --restart=Never -- mysql -h mysql -u kodekloud_roy -pdCV3szSGNA
```

### Method 2: Using NodePort (External Access)

If you have MySQL client installed locally:

```bash
# Get node IP
kubectl get nodes -o wide

# Connect to MySQL
mysql -h <NODE_IP> -P 30007 -u kodekloud_roy -p
# Password: dCV3szSGNA
```

### Verify Database Configuration

Once connected to MySQL:

```sql
-- Show all databases
SHOW DATABASES;

-- Check users
SELECT user, host FROM mysql.user;

-- Use the application database
USE kodekloud_db10;

-- Show tables
SHOW TABLES;
```

## 📊 Monitoring

Check pod logs:
```bash
# Get pod name
kubectl get pods -l app=mysql

# View logs
kubectl logs -l app=mysql

# Follow logs
kubectl logs -f -l app=mysql
```

Check pod status:
```bash
kubectl describe pod -l app=mysql
```

## 🔍 Troubleshooting

### Pod not starting
```bash
# Check pod events
kubectl describe pod -l app=mysql

# Check logs
kubectl logs -l app=mysql
```

### PVC not binding
```bash
# Check PV and PVC status
kubectl get pv
kubectl get pvc

# Describe PVC for events
kubectl describe pvc mysql-pv-claim
```

### Service not accessible
```bash
# Check service endpoints
kubectl get endpoints mysql

# Verify service configuration
kubectl describe svc mysql
```

## 🧹 Cleanup

To remove all resources:

```bash
kubectl delete -f mysql-deployment.yaml
```

Or delete resources individually:

```bash
kubectl delete service mysql
kubectl delete deployment mysql-deployment
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv
kubectl delete secret mysql-root-pass mysql-user-pass mysql-db-url
```

## 📝 Configuration Details

### Environment Variables Mapping

| Variable | Secret | Key |
|----------|--------|-----|
| MYSQL_ROOT_PASSWORD | mysql-root-pass | password |
| MYSQL_DATABASE | mysql-db-url | database |
| MYSQL_USER | mysql-user-pass | username |
| MYSQL_PASSWORD | mysql-user-pass | password |

### Volume Mounting

- **Container Path**: `/var/lib/mysql`
- **Volume**: mysql-persistent-storage (from PVC: mysql-pv-claim)

## ⚠️ Security Notes

- The secrets in this configuration use `stringData` for clarity. In production, consider using base64 encoded `data` or external secret management solutions
- Default MySQL root password is stored in plaintext in the YAML. Consider using sealed secrets or external vaults for production
- The hostPath storage is suitable for development/testing. Use proper storage classes for production

## 📚 Additional Resources

- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [MySQL Docker Image](https://hub.docker.com/_/mysql)

## ✅ Verification Checklist

- [ ] All secrets created successfully
- [ ] PersistentVolume created and available
- [ ] PersistentVolumeClaim bound to PV
- [ ] Deployment running with 1 replica
- [ ] Pod in Running state
- [ ] Service created with NodePort 30007
- [ ] MySQL accessible via service
- [ ] Database `kodekloud_db10` exists
- [ ] User `kodekloud_roy` can connect

## 📄 License

This configuration is provided as-is for the Nautilus DevOps team.

---

**Author**: Nautilus DevOps Team  
**Last Updated**: January 2026
