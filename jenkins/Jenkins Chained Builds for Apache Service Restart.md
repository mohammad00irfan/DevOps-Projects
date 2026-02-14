# Jenkins Chained Builds for Apache Service Restart

## Project Overview

This project demonstrates setting up Jenkins chained builds (upstream/downstream jobs) to automate Apache web server deployment and service management across multiple application servers. When code is deployed successfully, Apache services are automatically restarted on all app servers.

## Architecture

### Infrastructure
- **Jenkins Server**: CI/CD automation server
- **Storage Server**: Hosts Git repository and shared web content (`/var/www/html`)
- **App Servers (3x)**: Web servers running Apache httpd
- **Load Balancer**: Distributes traffic across app servers
- **Git Repository**: Gitea-hosted version control

### Workflow
1. Developer pushes changes to Git repository
2. Jenkins Job 1 (`nautilus-app-deployment`) pulls changes to shared storage
3. On successful deployment, Jenkins Job 2 (`manage-services`) is auto-triggered
4. Job 2 restarts Apache on all app servers
5. Changes are immediately available via load balancer

## Prerequisites

### SSH Key Setup
Jenkins needs passwordless SSH access to all servers:

```bash
# On Jenkins server as jenkins user
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub  # Copy this key
```

### Configure Target Servers

**On Storage Server:**
```bash
# As storage user
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "<JENKINS_PUBLIC_KEY>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Configure git safe directory
cd /var/www/html
git config --global --add safe.directory /var/www/html
```

**On Each App Server:**
```bash
# As respective user (user1, user2, user3)
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "<JENKINS_PUBLIC_KEY>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Passwordless Sudo Configuration

Each app server user needs sudo privileges for systemctl commands:

```bash
# On each app server
echo 'username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd' | sudo tee /etc/sudoers.d/username
sudo chmod 0440 /etc/sudoers.d/username

# Verify
sudo /usr/bin/systemctl status httpd  # Should not ask for password
```

### Verify Connectivity

From Jenkins server:
```bash
# Test SSH (should work without password)
ssh storage-user@storage-server "hostname"
ssh app-user1@app-server1 "hostname"
ssh app-user2@app-server2 "hostname"
ssh app-user3@app-server3 "hostname"

# Test sudo (should not ask for password)
ssh app-user1@app-server1 "sudo /usr/bin/systemctl status httpd"
ssh app-user2@app-server2 "sudo /usr/bin/systemctl status httpd"
ssh app-user3@app-server3 "sudo /usr/bin/systemctl status httpd"
```

## Jenkins Job Configuration

### Job 1: nautilus-app-deployment

**Type:** Freestyle Project

**Description:** Deploy web application from Git repository to storage server

**Build Steps - Execute Shell:**
```bash
#!/bin/bash
set -e

echo "=== Starting Deployment ==="
echo "Pulling latest changes from Git repository on storage server..."

ssh -o StrictHostKeyChecking=no storage-user@storage-server << 'EOF'
cd /var/www/html
git pull origin master
echo "Git pull completed successfully"
EOF

echo "=== Deployment Completed Successfully ==="
```

**Post-build Actions:**
- **Build other projects:** `manage-services`
- **Trigger only if build is stable:** ✅ Enabled

### Job 2: manage-services

**Type:** Freestyle Project

**Description:** Restart Apache httpd service on all application servers

**Build Steps - Execute Shell:**
```bash
#!/bin/bash
set -e

echo "=== Restarting Apache on All App Servers ==="

echo "Restarting Apache on app-server1..."
ssh -o StrictHostKeyChecking=no app-user1@app-server1 "sudo /usr/bin/systemctl restart httpd"
ssh -o StrictHostKeyChecking=no app-user1@app-server1 "sudo /usr/bin/systemctl status httpd --no-pager"
echo "app-server1: Apache restarted successfully"

echo "Restarting Apache on app-server2..."
ssh -o StrictHostKeyChecking=no app-user2@app-server2 "sudo /usr/bin/systemctl restart httpd"
ssh -o StrictHostKeyChecking=no app-user2@app-server2 "sudo /usr/bin/systemctl status httpd --no-pager"
echo "app-server2: Apache restarted successfully"

echo "Restarting Apache on app-server3..."
ssh -o StrictHostKeyChecking=no app-user3@app-server3 "sudo /usr/bin/systemctl restart httpd"
ssh -o StrictHostKeyChecking=no app-user3@app-server3 "sudo /usr/bin/systemctl status httpd --no-pager"
echo "app-server3: Apache restarted successfully"

echo "=== All Apache Services Restarted Successfully ==="
```

**Upstream Projects:** `nautilus-app-deployment` (automatically configured)

## Testing the Pipeline

### 1. Manual Test
```bash
# Trigger deployment job in Jenkins UI
# Click "Build Now" on nautilus-app-deployment
# Verify both jobs complete successfully
```

### 2. Git Push Test
```bash
# On storage server
cd /var/www/html
echo "<h1>Test Deployment - $(date)</h1>" > index.html
git add index.html
git commit -m "Test Jenkins deployment"
git push origin master

# Trigger Jenkins job
# Verify changes appear on load balancer URL
```

### Expected Console Output

**nautilus-app-deployment:**
```
Started by user admin
=== Starting Deployment ===
Pulling latest changes from Git repository on storage server...
Already up to date.
Git pull completed successfully
=== Deployment Completed Successfully ===
Finished: SUCCESS
Triggering a new build of manage-services
```

**manage-services:**
```
Started by upstream project "nautilus-app-deployment" build number 5
=== Restarting Apache on All App Servers ===
Restarting Apache on app-server1...
● httpd.service - The Apache HTTP Server
   Active: active (running)
app-server1: Apache restarted successfully
Restarting Apache on app-server2...
● httpd.service - The Apache HTTP Server
   Active: active (running)
app-server2: Apache restarted successfully
Restarting Apache on app-server3...
● httpd.service - The Apache HTTP Server
   Active: active (running)
app-server3: Apache restarted successfully
=== All Apache Services Restarted Successfully ===
Finished: SUCCESS
```

## Troubleshooting

### Issue: Permission denied (SSH)
**Solution:** Verify SSH keys are properly configured
```bash
ssh jenkins-user@target-server "hostname"  # Should work without password
```

### Issue: sudo asks for password
**Solution:** Check sudoers configuration
```bash
# Verify file exists and has correct permissions
sudo cat /etc/sudoers.d/username
ls -l /etc/sudoers.d/username  # Should be -r--r----- (0440)

# Test sudo
sudo /usr/bin/systemctl status httpd  # Should not ask for password
```

### Issue: Git pull fails with "dubious ownership"
**Solution:** Add safe directory
```bash
git config --global --add safe.directory /var/www/html
```

### Issue: Downstream job doesn't trigger
**Solution:** Verify post-build action in upstream job
- Check "Build other projects" is configured
- Ensure "Trigger only if build is stable" is enabled
- Verify downstream project name matches exactly

## Security Considerations

1. **SSH Keys:** Use dedicated SSH keys for Jenkins, not personal keys
2. **Sudo Access:** Limit NOPASSWD sudo to specific commands only
3. **Git Credentials:** Use token-based authentication or SSH for Git
4. **File Permissions:** Ensure sudoers files are mode 0440
5. **Network:** Consider using bastion hosts for production environments

## Benefits

- **Automation:** Eliminates manual Apache restarts after deployments
- **Consistency:** Ensures all app servers are restarted uniformly
- **Safety:** Only restarts services after successful deployment
- **Visibility:** Jenkins provides clear logs of all operations
- **Scalability:** Easy to add more app servers to the pipeline

## Future Enhancements

- [ ] Add health checks before restarting services
- [ ] Implement rolling restarts to maintain availability
- [ ] Add Slack/email notifications for build status
- [ ] Create rollback job for failed deployments
- [ ] Integrate automated testing before deployment
- [ ] Add deployment approval gates for production
- [ ] Implement blue-green deployment strategy

---
