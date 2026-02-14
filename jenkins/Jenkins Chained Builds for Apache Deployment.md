# Jenkins Chained Builds for Apache Deployment - DevOps Project

## 📋 Project Overview

This project demonstrates the implementation of **Jenkins chained builds** (upstream/downstream jobs) to automate application deployment and service management across multiple servers in the Stratos Datacenter environment.

### Business Requirement
The DevOps team needed a solution to automatically restart Apache (httpd) service on all application servers only when deployment succeeds. The solution uses Jenkins pipeline chaining where a successful deployment job triggers a service management job.

---

## 🏗️ Architecture

### Infrastructure
- **3 Application Servers** (stapp01, stapp02, stapp03)
- **1 Storage Server** (ststor01) - Shared NFS mount for `/var/www/html`
- **1 Jenkins Server** - CI/CD automation
- **1 Gitea Server** - Git repository hosting
- **1 Load Balancer** - Traffic distribution

### Data Flow
```
Developer commits to Gitea
         ↓
Jenkins: nautilus-app-deployment
         ↓
Pull from Git → Deploy to /var/www/html (shared storage)
         ↓
Build Status: STABLE?
         ↓
       YES
         ↓
Jenkins: manage-services (triggered automatically)
         ↓
SSH to all 3 app servers → Restart httpd service
         ↓
Application updated and live on Load Balancer
```

---

## 🎯 Objectives

1. ✅ Create Jenkins job `nautilus-app-deployment` to pull from Git master branch
2. ✅ Deploy code to `/var/www/html` on Storage server (shared volume)
3. ✅ Create Jenkins job `manage-services` as downstream job
4. ✅ Restart `httpd` service on all app servers only if deployment is stable
5. ✅ Ensure jobs can run repeatedly without failure
6. ✅ Content accessible on main URL (no subdirectories)

---

## 🛠️ Technical Implementation

### Phase 1: SSH Key Setup

**Challenge**: Jenkins needed passwordless SSH access to multiple servers for automation.

**Solution**: Generated RSA SSH keys on Jenkins server and distributed public key to all target servers.

```bash
# On Jenkins server
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# Distributed to:
- user1@appserver01 (App Server 1)
- user2@appserver02 (App Server 2)  
- user3@appserver03 (App Server 3)
- storageuser@storageserver (Storage Server)
```

**Configuration**: Added passwordless sudo permissions for httpd service management on each app server:

```bash
# Example for appserver01
echo 'user1 ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd' | sudo tee /etc/sudoers.d/user1
sudo chmod 0440 /etc/sudoers.d/user1
```

---

### Phase 2: Git Repository Configuration

**Challenge**: Git repository ownership and safe directory permissions.

**Solution**: 
- Repository located at `/var/www/html` on Storage server
- Owned by `storageuser` user
- Configured git safe directory to allow operations

```bash
git config --global --add safe.directory /var/www/html
```

**Remote Configuration**:
```
origin: http://<git-server>/sarah/web.git
Branch: master
```

---

### Phase 3: Jenkins Job Configuration

#### Job 1: nautilus-app-deployment (Upstream Job)

**Purpose**: Pull latest code from Git and deploy to shared storage

**Configuration**:
- **Type**: Freestyle Project
- **Source Code Management**: None (using Execute Shell with SSH)
- **Build Trigger**: Manual or webhook
- **Build Action**: Execute Shell Script

```bash
#!/bin/bash
set -e

echo "Deploying from Git repository..."

ssh -o StrictHostKeyChecking=no storageuser@storageserver << 'ENDSSH'
cd /var/www/html
git config --global --add safe.directory /var/www/html
git fetch origin master
git reset --hard origin/master
chmod -R 755 /var/www/html
echo "Deployment completed!"
ENDSSH

echo "Deployment successful!"
exit 0
```

**Post-build Action**:
- Trigger downstream project: `manage-services`
- Condition: **Only if build is stable**

**Key Design Decision**: Used SSH to storage server instead of Git SCM plugin to avoid local checkout security restrictions.

---

#### Job 2: manage-services (Downstream Job)

**Purpose**: Restart Apache service on all application servers

**Configuration**:
- **Type**: Freestyle Project
- **Trigger**: Automatically triggered by `nautilus-app-deployment` (when stable)
- **Build Action**: Execute Shell Script

```bash
#!/bin/bash
set -e

ssh -o StrictHostKeyChecking=no user1@appserver01 "sudo systemctl restart httpd"
echo "✅ appserver01 done"

ssh -o StrictHostKeyChecking=no user2@appserver02 "sudo systemctl restart httpd"
echo "✅ appserver02 done"

ssh -o StrictHostKeyChecking=no user3@appserver03 "sudo systemctl restart httpd"
echo "✅ appserver03 done"

echo "All services restarted!"
```

**Error Handling**: Script exits on first failure (`set -e`) to prevent partial deployments.

---

## 🔧 Troubleshooting & Solutions

### Issue 1: SSH Authentication Failure
**Error**: `Auth fail for methods 'publickey,gssapi-keyex,gssapi-with-mic,password'`

**Root Cause**: Attempted to use "Publish Over SSH" plugin which had authentication issues.

**Solution**: 
- Abandoned plugin-based approach
- Used native SSH with Execute Shell
- Simplified configuration and improved reliability

---

### Issue 2: Git Local Checkout Security
**Error**: `Checkout of Git remote 'file:///var/www/html' aborted because it references a local directory`

**Root Cause**: Jenkins Git plugin blocks local file:// URLs for security.

**Solution**: 
- Instead of using Git SCM in Jenkins
- Used SSH to remote server and executed git commands there
- Maintained security while enabling required functionality

---

### Issue 3: Jenkins User Sudo Permissions
**Error**: `jenkins is not in the sudoers file`

**Root Cause**: Jenkins server didn't allow sudo for jenkins user.

**Solution**: 
- All privileged operations executed via SSH on target servers
- Jenkins server only orchestrates, doesn't execute privileged commands
- Cleaner security model with principle of least privilege

---

### Issue 4: Git Repository Ownership
**Error**: `fatal: detected dubious ownership in repository at '/var/www/html'`

**Root Cause**: Git security check for repository ownership mismatch.

**Solution**:
```bash
git config --global --add safe.directory /var/www/html
```

---

## 📊 Results & Validation

### Successful Build Output

**nautilus-app-deployment Console Output**:
```
Deploying from Git repository...
Deployment completed!
Deployment successful!
Finished: SUCCESS

Triggering downstream project: manage-services
```

**manage-services Console Output**:
```
✅ appserver01 done
✅ appserver02 done
✅ appserver03 done
All services restarted!
Finished: SUCCESS
```

### Application Verification
- Accessed via Load Balancer URL
- Content displayed: `Welcome to Application! Deployment Test`
- No subdirectory in URL (requirement met)
- Changes from Git immediately reflected

### Repeatability Test
- Ran deployment job 3+ times consecutively
- All builds: **SUCCESS**
- No state conflicts or permission issues
- Idempotent operations confirmed

---

## 🔐 Security Considerations

1. **SSH Key Management**
   - RSA 4096-bit keys generated
   - Private keys secured on Jenkins server
   - Public keys distributed to authorized servers only

2. **Sudo Restrictions**
   - Limited to specific commands (systemctl restart/status httpd)
   - NOPASSWD only for required operations
   - Separate sudoers.d files per user

3. **Git Credentials**
   - Embedded in remote URL (acceptable for internal network)
   - Could be improved with credential helpers for production

4. **Jenkins Security**
   - No sudo on Jenkins server (principle of least privilege)
   - All privileged operations delegated to target servers
   - Build isolation maintained

---

## 📚 Key Learnings

### Technical Insights

1. **Plugin vs Native Approach**
   - Jenkins plugins can introduce complexity
   - Native shell scripts offer more control and transparency
   - SSH is reliable for cross-server automation

2. **Job Chaining Best Practices**
   - Always use "Trigger only if build is stable"
   - Prevents cascading failures
   - Maintains deployment integrity

3. **Shared Storage Benefits**
   - Single deployment point updates all app servers
   - Eliminates need for multiple deployments
   - Ensures consistency across environment

4. **Idempotent Operations**
   - `git reset --hard` ensures clean state
   - Service restarts are naturally idempotent
   - Jobs can safely run multiple times

### DevOps Principles Applied

- ✅ **Automation**: Eliminated manual service restarts
- ✅ **Reliability**: Deployment only triggers restart on success
- ✅ **Consistency**: All app servers updated simultaneously
- ✅ **Traceability**: Jenkins provides complete audit trail
- ✅ **Repeatability**: Jobs tested for multiple executions

---

## 🚀 Future Enhancements

### Potential Improvements

1. **Health Checks**
   - Add post-restart HTTP health checks
   - Verify services are responding before marking success
   - Implement retry logic for transient failures

2. **Notifications**
   - Email/Slack notifications on deployment success/failure
   - Real-time alerts for DevOps team

3. **Rollback Capability**
   - Store previous Git commit hash
   - Automated rollback on deployment failure
   - Quick recovery mechanism

4. **Blue-Green Deployment**
   - Deploy to staging servers first
   - Automated traffic switching after validation
   - Zero-downtime deployments

5. **Monitoring Integration**
   - Post-deployment metrics collection
   - Performance comparison before/after
   - Automated performance regression detection

6. **GitOps Workflow**
   - Webhook triggers from Git commits
   - Automatic deployment on merge to master
   - Full CI/CD pipeline integration

---

## 📖 Documentation

### Server Inventory

| Server | IP | Hostname | User | Purpose |
|--------|-----|----------|------|---------|
| appserver01 | 172.16.238.xx | appserver01.domain.com | user1 | App Server 1 |
| appserver02 | 172.16.238.xx | appserver02.domain.com | user2 | App Server 2 |
| appserver03 | 172.16.238.xx | appserver03.domain.com | user3 | App Server 3 |
| storageserver | 172.16.238.xx | storageserver.domain.com | storageuser | Storage Server |
| jenkinsserver | 172.16.238.xx | jenkins.domain.com | jenkins | Jenkins CI/CD |

### Access Credentials

- **Jenkins UI**: Admin user with secure password
- **Git UI**: Git user with secure credentials
- **Git Repository**: web repository (master branch)

---

## 🎓 Skills Demonstrated

- ✅ Jenkins administration and job configuration
- ✅ Shell scripting and automation
- ✅ SSH key-based authentication
- ✅ Git operations and repository management
- ✅ Linux system administration (sudoers, permissions)
- ✅ Multi-server orchestration
- ✅ CI/CD pipeline design
- ✅ Troubleshooting and problem-solving
- ✅ DevOps best practices
- ✅ Infrastructure as Code concepts

---

## 🏆 Conclusion

This project successfully implemented an automated deployment pipeline using Jenkins chained builds, demonstrating practical DevOps skills in:
- Continuous Integration/Continuous Deployment (CI/CD)
- Infrastructure automation
- Service orchestration across multiple servers
- Security-conscious system administration

The solution is production-ready, repeatable, and follows DevOps best practices for automated deployments.

---
