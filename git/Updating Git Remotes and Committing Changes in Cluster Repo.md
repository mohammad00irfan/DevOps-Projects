---

# Updating Git Remotes and Committing Changes in Cluster Repo

The DevOps team recently updated the `cluster` repository to reflect new changes from the Git server. The following steps were performed:

---

### **1. Navigate to the local repository**

```bash
cd /usr/src/repos/cluster
```

---

### **2. Add a new remote**

A new remote named `dev_cluster` was added pointing to the updated Git repository:

```bash
git remote add dev_cluster /opt/showcase_cluster.git
```

Verify the remote:

```bash
git remote -v
```

Expected output:

```
dev_cluster  /opt/showcase_cluster.git (fetch)
dev_cluster  /opt/showcase_cluster.git (push)
origin       <previous_remote> (fetch)
origin       <previous_remote> (push)
```

---

### **3. Copy file into repository**

The file `/tmp/index.html` from the server was copied into the repository root:

```bash
cp /tmp/index.html .
```

---

### **4. Stage and commit changes**

```bash
git add index.html
git commit -m "Add index.html to master branch"
```

---

### **5. Push master branch to the new remote**

```bash
git push dev_cluster master
```

This successfully pushed the `master` branch with the new changes to the `dev_cluster` remote.

---
