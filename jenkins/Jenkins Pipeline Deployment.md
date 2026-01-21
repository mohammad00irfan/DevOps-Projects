# Jenkins Pipeline Deployment Guide

## Project Overview

This guide demonstrates how to configure Jenkins for automated deployment of a static website using Jenkins Pipeline and slave nodes. The project involves setting up user permissions, configuring slave nodes, and creating a CI/CD pipeline for automated deployment.

---

## Table of Contents

1. [Task 1: Configure Jenkins User Permissions](#task-1-configure-jenkins-user-permissions)
2. [Task 2: Create Jenkins Pipeline for Web Application Deployment](#task-2-create-jenkins-pipeline-for-web-application-deployment)
3. [Troubleshooting Guide](#troubleshooting-guide)

---

## Task 1: Configure Jenkins User Permissions

### Objective
Grant specific permissions to developers for accessing Jenkins jobs using project-based matrix authorization.

### Requirements
- Existing Jenkins job: `Packages`
- Two users requiring access: `user1` and `user2`
- Use "Inherit permissions from parent ACL" inheritance strategy

### Implementation Steps

#### 1. Configure Global Security

1. Navigate to **Manage Jenkins** → **Security** → **Configure Global Security**
2. Under **Authorization**, ensure **Project-based Matrix Authorization Strategy** is selected
3. Add users with global read permissions:
   - Click **"Add user..."**
   - Add `user1` with **Overall/Read** permission
   - Add `user2` with **Overall/Read** permission
   - Ensure `admin` has **Overall/Administer** permission
4. Save configuration

#### 2. Configure Job-Level Permissions

1. Navigate to the **Packages** job → **Configure**
2. Enable **"Enable project-based security"** checkbox
3. Select **"Inherit permissions from parent ACL"** under Inheritance Strategy
4. Add user permissions:

   **User 1 Permissions:**
   - Job/Build ✓
   - Job/Configure ✓
   - Job/Read ✓

   **User 2 Permissions:**
   - Job/Build ✓
   - Job/Cancel ✓
   - Job/Configure ✓
   - Job/Read ✓
   - SCM/Tag ✓

5. Save configuration

#### 3. Verification

- Logout from admin account
- Login as each user to verify appropriate access levels
- Confirm users can only perform authorized actions

### Key Learnings

- **Global vs. Job-level permissions**: Users need global read access to see the dashboard, plus specific job permissions
- **Inheritance strategy**: "Inherit permissions from parent ACL" allows job-specific permissions while maintaining global security settings
- **Permission granularity**: Jenkins provides fine-grained control over build, configure, read, cancel, and tag operations

---

## Task 2: Create Jenkins Pipeline for Web Application Deployment

### Objective
Set up a Jenkins pipeline to automatically deploy a static website from a Git repository to application servers using a slave node.

### Architecture

```
┌─────────────┐      ┌──────────────┐      ┌─────────────────┐
│   Gitea     │─────>│   Jenkins    │─────>│ Storage Server  │
│ Repository  │      │   Master     │      │  (Slave Node)   │
└─────────────┘      └──────────────┘      └─────────────────┘
                                                     │
                                                     ↓
                                            ┌─────────────────┐
                                            │  App Servers    │
                                            │  (via mount)    │
                                            └─────────────────┘
```

### Requirements

- Jenkins master with admin access
- Git repository containing web application code
- Storage server for deployment (acts as Jenkins slave)
- Apache web servers (pre-configured on port 8080)
- Load balancer (pre-configured)

### Implementation Steps

#### Phase 1: Add Jenkins Slave Node

**Initial Approach (Permission Issues):**

First attempt was to use `/var/www/html` directly as the remote root directory, but this caused permission errors:

```
Caused by: com.trilead.ssh2.SFTPException: Permission denied
Could not copy remoting.jar to '/var/www/html/remoting.jar'
```

**Solution Options:**

**Option A: Fix Permissions (Recommended)**

```bash
# SSH to Storage Server as root
ssh root@storage-server

# Change ownership to allow Jenkins user write access
chown -R jenkins-user:jenkins-user /var/www/html
chmod -R 755 /var/www/html
```

Then configure node with:
- **Remote root directory:** `/var/www/html`

**Option B: Use Alternative Directory**

If you cannot modify `/var/www/html` permissions:
- **Remote root directory:** `/home/jenkins-user`
- Deploy to `/var/www/html` via sudo in pipeline

**Node Configuration:**

1. Navigate to **Manage Jenkins** → **Nodes**
2. Click **"New Node"**
3. Configure node:
   ```
   Node name: Storage Server
   Type: Permanent Agent
   Remote root directory: /var/www/html (or /home/jenkins-user)
   Labels: ststor01
   Usage: Use this node as much as possible
   Launch method: Launch agents via SSH
   Host: storage-server-hostname
   Credentials: SSH Username with password
   Host Key Verification Strategy: Non verifying Verification Strategy
   ```
4. Add SSH credentials:
   ```
   Kind: Username with password
   Username: jenkins-user
   Password: ********
   ID: storage-server-creds
   ```
5. Save and verify node connects successfully

#### Phase 2: Configure Passwordless Sudo

To allow deployment without password prompts:

```bash
# SSH to Storage Server as root
sudo su -

# Add sudoers configuration
echo "jenkins-user ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/jenkins-user
chmod 440 /etc/sudoers.d/jenkins-user
exit

# Test sudo access
sudo ls /var/www/html
```

#### Phase 3: Create Jenkins Pipeline Job

1. Navigate to Jenkins Dashboard
2. Click **"New Item"**
3. Configure:
   ```
   Name: webapp-deployment-job
   Type: Pipeline (NOT Multibranch Pipeline)
   ```
4. Add pipeline script:

**Pipeline Script (Final Working Version):**

```groovy
pipeline {
    agent {
        label 'ststor01'
    }
    
    stages {
        stage('Deploy') {
            steps {
                sh '''
                    # Clone repository to workspace
                    rm -rf web_app
                    git clone http://git-server/user/web_app.git
                    
                    # Deploy to /var/www/html
                    sudo cp -r web_app/* /var/www/html/
                    sudo chmod -R 755 /var/www/html/
                '''
            }
        }
    }
}
```

5. Save configuration

#### Phase 4: Testing and Verification

1. Click **"Build Now"**
2. Monitor **Console Output** for:
   ```
   Started by user admin
   Running on Storage Server
   Cloning into 'web_app'...
   [Pipeline] End of Pipeline
   Finished: SUCCESS
   ```
3. Access application via load balancer URL
4. Verify content loads at main URL (not in subdirectory)

---

## Troubleshooting Guide

### Issue 1: "Access Denied - User is missing Overall/Read permission"

**Cause:** Users lack global read permission in Jenkins

**Solution:**
```
1. Go to Manage Jenkins → Configure Global Security
2. Add user with Overall/Read permission
3. Save configuration
```

### Issue 2: Node Connection Failed - Permission Denied

**Cause:** Jenkins user cannot write to remote root directory

**Solutions:**
- **Option A:** Change directory ownership to Jenkins user
- **Option B:** Use user's home directory as remote root, deploy via sudo
- **Option C:** Verify SSH credentials are correct

### Issue 3: "chown: invalid user: 'apache:apache'"

**Cause:** Incorrect user/group name for web server

**Solution:**
```bash
# Find correct ownership
ls -l /var/www/html

# Common web server users:
# - apache:apache (CentOS/RHEL)
# - www-data:www-data (Ubuntu/Debian)
# - httpd:httpd (Some systems)
# - nginx:nginx (Nginx)

# Or simply use chmod without chown:
sudo chmod -R 755 /var/www/html/
```

### Issue 4: Git Clone Permission Denied in Pipeline

**Cause:** Attempting to clone directly to protected directory

**Solution:**
```groovy
# Clone to workspace first, then copy with sudo
sh '''
    rm -rf web_app
    git clone http://git-server/repo.git
    sudo cp -r web_app/* /var/www/html/
'''
```

### Issue 5: Website Shows Subdirectory (e.g., /web_app)

**Cause:** Files deployed into subdirectory instead of root

**Solution:**
```bash
# Ensure wildcard copies contents, not directory
sudo cp -r web_app/* /var/www/html/

# NOT: sudo cp -r web_app /var/www/html/
```

### Issue 6: Credential Creation Failed

**Cause:** Special characters or UI issues

**Solutions:**
- Use "Username with password" instead of "SSH Username with password"
- Check for extra spaces in password field
- Verify CAPS LOCK is off
- Try creating credential from different page (Manage Jenkins → Credentials)

---

## Best Practices

### Security
- ✅ Use principle of least privilege for user permissions
- ✅ Implement job-level security for sensitive projects
- ✅ Use SSH keys instead of passwords when possible
- ✅ Regularly audit user permissions
- ✅ Configure passwordless sudo only for specific commands if possible

### Pipeline Design
- ✅ Use declarative pipeline syntax for better readability
- ✅ Label agents appropriately for targeted deployment
- ✅ Include error handling in shell scripts
- ✅ Clean workspace before cloning to avoid conflicts
- ✅ Use meaningful stage names (case-sensitive requirements)

### Node Management
- ✅ Verify node connectivity before creating jobs
- ✅ Use descriptive labels for easy identification
- ✅ Set appropriate number of executors based on workload
- ✅ Monitor node disk space and resources
- ✅ Document node configurations for team reference

### Deployment
- ✅ Verify file permissions after deployment
- ✅ Test application accessibility after each deployment
- ✅ Keep deployment scripts idempotent
- ✅ Maintain deployment logs for troubleshooting
- ✅ Implement rollback procedures

---

## Key Takeaways

1. **Permission Hierarchy**: Jenkins requires both global and job-level permissions to be configured correctly
2. **File System Permissions**: Slave node remote root directories must be writable by the Jenkins user
3. **Sudo Configuration**: Passwordless sudo enables automated deployments without manual intervention
4. **Pipeline Flexibility**: Using workspace for operations, then deploying with elevated privileges, provides best security
5. **Testing is Critical**: Always verify each component (node connectivity, permissions, deployment) before final integration
6. **Documentation**: Maintain clear documentation of server credentials, configurations, and deployment procedures

---

## Additional Resources

- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [Matrix Authorization Strategy Plugin](https://plugins.jenkins.io/matrix-auth/)
- [SSH Build Agents Plugin](https://plugins.jenkins.io/ssh-slaves/)
- [Jenkins Best Practices](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/)

---

## Conclusion

This guide demonstrates a complete Jenkins deployment setup, from user permissions to automated pipeline deployment. The implementation showcases real-world troubleshooting scenarios and provides practical solutions for common Jenkins configuration challenges.

**Skills Demonstrated:**
- Jenkins security configuration
- Pipeline as Code
- Linux system administration
- SSH and remote execution
- Permission management
- Troubleshooting and problem-solving
- CI/CD best practices

---

*Last Updated: January 2026*
