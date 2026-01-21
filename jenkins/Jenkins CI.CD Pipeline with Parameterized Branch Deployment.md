# Jenkins CI/CD Pipeline with Parameterized Branch Deployment

A comprehensive guide to setting up Jenkins pipelines for automated deployment of web applications across multiple branches, demonstrated through a real-world DevOps scenario.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Part 1: Initial Setup - Storage Server Node](#part-1-initial-setup---storage-server-node)
- [Part 2: Simple Pipeline Deployment](#part-2-simple-pipeline-deployment)
- [Part 3: Parameterized Multi-Branch Pipeline](#part-3-parameterized-multi-branch-pipeline)
- [Troubleshooting](#troubleshooting)
- [Key Learnings](#key-learnings)

## 🎯 Overview

This project demonstrates the implementation of Jenkins CI/CD pipelines for deploying a static website to application servers. The solution evolved from a basic deployment pipeline to a parameterized pipeline supporting conditional branch deployment.

### Technologies Used
- **Jenkins** - CI/CD automation server
- **Gitea** - Git repository management
- **Apache HTTP Server** - Web server (port 8080)
- **SSH** - Secure communication for agent nodes
- **Linux** - RHEL-based system administration

## 🏗️ Architecture

```
┌─────────────────┐
│  Jenkins Master │
└────────┬────────┘
         │
         │ SSH Connection
         ▼
┌─────────────────┐
│ Storage Server  │
│  (Agent Node)   │
│ Label: ststor01 │
└────────┬────────┘
         │
         │ Shared Mount
         ▼
┌─────────────────────────────────┐
│   Application Servers (1-3)     │
│   Apache HTTP Server :8080      │
│   Document Root: /var/www/html  │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  Load Balancer  │
└─────────────────┘
```

## ✅ Prerequisites

- Jenkins installed and accessible
- Git repository (Gitea or similar)
- SSH access to storage/deployment server
- Apache web servers configured on application servers
- Basic understanding of:
  - Jenkins pipelines
  - Linux permissions
  - Git version control
  - Shell scripting

## 🚀 Part 1: Initial Setup - Storage Server Node

### Step 1: Configure Jenkins Slave Node

Navigate to **Manage Jenkins** → **Manage Nodes and Clouds** → **New Node**

**Node Configuration:**
```yaml
Name: Storage Server
Description: Storage Server for web deployments
Number of executors: 1
Remote root directory: /var/www/html
Labels: ststor01
Usage: Use this node as much as possible
Launch method: Launch agents via SSH
```

**SSH Launch Configuration:**
```yaml
Host: storage-server-hostname
Credentials: [SSH credentials]
Host Key Verification Strategy: Non verifying Verification Strategy
Port: 22
Connection Timeout: 60
Maximum Number of Retries: 10
Seconds To Wait Between Retries: 15
```

### Step 2: Fix Permission Issues

**Problem:** Jenkins agent couldn't write to `/var/www/html`

**Error:**
```
com.trilead.ssh2.SFTPException: Permission denied
Could not copy remoting.jar to '/var/www/html/remoting.jar'
```

**Solution:**

```bash
# SSH to storage server as root
ssh root@storage-server

# Check current ownership
ls -ld /var/www/html

# Add Jenkins user to the web group
usermod -a -G webgroup jenkins-user

# Set proper permissions
chown -R webuser:webgroup /var/www/html
chmod 775 /var/www/html
chmod -R 775 /var/www/html

# Verify permissions
ls -ld /var/www/html
groups jenkins-user
```

### Step 3: Verify Node Connection

**Expected Log Output:**
```
[SSH] Opening SSH connection to storage-server:22.
[SSH] Authentication successful.
[SSH] Starting sftp client.
[SSH] Copying latest remoting.jar...
Agent successfully connected and online
```

## 📦 Part 2: Simple Pipeline Deployment

### Create Pipeline Job

**Job Configuration:**
- **Name:** `devops-webapp-job`
- **Type:** Pipeline (not Multibranch)

### Pipeline Script

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
                    cp -rf web_app/* /var/www/html/
                '''
            }
        }
    }
}
```

### Common Issues and Solutions

**Issue 1: Permission Denied on chmod**

```
chmod: changing permissions of '/var/www/html/index.html': Operation not permitted
```

**Solution:** Remove unnecessary chmod commands or only target deployed files:

```groovy
sh '''
    # Clone repository
    rm -rf web_app
    git clone http://git-server/user/web_app.git
    
    # Deploy (no chmod on Jenkins workspace files)
    cp -rf web_app/* /var/www/html/
'''
```

**Issue 2: Files Copying to Wrong Location**

Ensure the pipeline copies files directly to document root, not creating subdirectories:

```bash
# Correct - copies contents
cp -rf web_app/* /var/www/html/

# Incorrect - creates subdirectory
cp -rf web_app /var/www/html/
```

## 🔀 Part 3: Parameterized Multi-Branch Pipeline

### Enhanced Requirements

- **Parameter:** String parameter named `BRANCH`
- **Conditional Logic:** Deploy different branches based on parameter value
- **Supported Branches:** `master` and `feature`
- **Single Stage:** Named `Deploy` (case-sensitive)

### Configure Parameterized Pipeline

**Step 1: Enable Parameters**

In job configuration, check **"This project is parameterized"**

Add **String Parameter:**
```yaml
Name: BRANCH
Default Value: master
Description: Branch to deploy (master or feature)
```

### Final Pipeline Script

```groovy
pipeline {
    agent {
        label 'ststor01'
    }
    
    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to deploy (master or feature)')
    }
    
    stages {
        stage('Deploy') {
            steps {
                script {
                    echo "Deploying branch: ${params.BRANCH}"
                    
                    sh """
                        # Remove existing clone if present
                        rm -rf web_app
                        
                        # Clone the repository
                        git clone http://git-server/user/web_app.git
                        
                        # Navigate into repository
                        cd web_app
                        
                        # Checkout the specified branch
                        if [ "${params.BRANCH}" = "master" ]; then
                            echo "Checking out master branch"
                            git checkout master
                        elif [ "${params.BRANCH}" = "feature" ]; then
                            echo "Checking out feature branch"
                            git checkout feature
                        else
                            echo "Invalid branch specified. Use 'master' or 'feature'"
                            exit 1
                        fi
                        
                        # Deploy to /var/www/html
                        cd ..
                        cp -rf web_app/* /var/www/html/
                        
                        echo "Deployment of ${params.BRANCH} branch completed successfully"
                    """
                }
            }
        }
    }
}
```

### Alternative Implementation (More Concise)

```groovy
pipeline {
    agent {
        label 'ststor01'
    }
    
    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to deploy (master or feature)')
    }
    
    stages {
        stage('Deploy') {
            steps {
                script {
                    echo "Deploying branch: ${params.BRANCH}"
                    
                    // Validate branch parameter
                    if (params.BRANCH != 'master' && params.BRANCH != 'feature') {
                        error("Invalid branch '${params.BRANCH}'. Please use 'master' or 'feature'")
                    }
                    
                    sh """
                        # Clone specific branch directly
                        rm -rf web_app
                        git clone -b ${params.BRANCH} http://git-server/user/web_app.git
                        
                        # Deploy to /var/www/html
                        cp -rf web_app/* /var/www/html/
                        
                        echo "Successfully deployed ${params.BRANCH} branch"
                    """
                }
            }
        }
    }
}
```

### Usage

**Deploy Master Branch:**
```
Build with Parameters → BRANCH: master → Build
```

**Deploy Feature Branch:**
```
Build with Parameters → BRANCH: feature → Build
```

### Expected Console Output

**Master Branch Deployment:**
```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Storage Server in /var/www/html/workspace/pipeline-job
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] script
[Pipeline] {
[Pipeline] echo
Deploying branch: master
[Pipeline] sh
+ rm -rf web_app
+ git clone http://git-server/user/web_app.git
Cloning into 'web_app'...
+ cd web_app
+ [ master = master ]
+ echo Checking out master branch
Checking out master branch
+ git checkout master
Already on 'master'
+ cd ..
+ cp -rf web_app/* /var/www/html/
+ echo Deployment of master branch completed successfully
Deployment of master branch completed successfully
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

## 🔧 Troubleshooting

### Issue: Node Not Found

**Error:**
```
Still waiting to schedule task
'Jenkins' doesn't have label 'ststor01'
```

**Solution:**
1. Go to **Manage Jenkins** → **Manage Nodes**
2. Click on the Storage Server node
3. Verify **Labels** field contains: `ststor01`
4. Save configuration

### Issue: SSH Connection Failed

**Error:**
```
[SSH] Authentication failed
```

**Solution:**
1. Verify SSH credentials are correct
2. Test SSH connection manually: `ssh user@host`
3. Check if SSH key is properly configured
4. Ensure host is reachable from Jenkins master

### Issue: Permission Denied

**Error:**
```
Permission denied (SSH_FX_PERMISSION_DENIED)
```

**Solution:**
```bash
# On storage server
chown -R deploy-user:deploy-group /var/www/html
chmod 775 /var/www/html

# Verify user can write
su - deploy-user
touch /var/www/html/test.txt
rm /var/www/html/test.txt
```

### Issue: Wrong Content Location

**Problem:** Content accessible at `https://server/web_app/` instead of `https://server/`

**Solution:** Use wildcard in copy command:
```bash
# Correct
cp -rf web_app/* /var/www/html/

# Incorrect (creates subdirectory)
cp -rf web_app /var/www/html/
```

## 📚 Key Learnings

### 1. Jenkins Agent Configuration
- Labels are crucial for agent selection
- Remote root directory must have proper permissions
- SSH key authentication is more secure than password

### 2. Linux Permissions
- Understanding user/group ownership is essential
- Use `chmod 775` for shared directories
- Add users to groups with `usermod -a -G`

### 3. Pipeline Best Practices
- Validate parameters before execution
- Use `script` blocks for complex logic
- Provide clear echo statements for debugging
- Clean up workspace before cloning

### 4. Git Operations
- Clone specific branches with `-b` flag
- Always checkout explicitly to avoid ambiguity
- Clean previous clones to avoid conflicts

### 5. Deployment Strategy
- Use shared storage for multi-server deployment
- Avoid unnecessary permission changes
- Verify deployment with actual application access

## 🎯 Success Criteria Checklist

- ✅ Jenkins slave node configured with correct label
- ✅ SSH connection established successfully
- ✅ Pipeline job created (not Multibranch)
- ✅ String parameter `BRANCH` configured
- ✅ Single stage named `Deploy` implemented
- ✅ Conditional branch deployment working
- ✅ Files deployed to correct location
- ✅ Application accessible from root URL
- ✅ Both master and feature branches deployable

## 📝 Additional Notes

### Security Considerations
- Use SSH keys instead of passwords
- Implement least privilege principle for file permissions
- Consider using Jenkins credentials binding for sensitive data
- Enable host key verification in production

### Scalability Improvements
- Implement parallel deployment to multiple servers
- Add health checks after deployment
- Include rollback mechanisms
- Add notification on build status

### Monitoring & Logging
- Enable Jenkins build logs
- Monitor disk space on storage server
- Track deployment history
- Set up alerts for failed deployments


---

**Created as part of DevOps learning journey** 🚀
