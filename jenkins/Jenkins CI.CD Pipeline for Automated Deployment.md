# Jenkins CI/CD Pipeline for Automated Deployment

## Project Overview

This project demonstrates the implementation of a complete CI/CD pipeline using Jenkins to automate the deployment of a web application across multiple servers. The solution includes automated builds triggered by Git commits and deployment to shared storage accessible by multiple application servers.

## Architecture

```
┌─────────────┐         ┌──────────────┐         ┌─────────────────┐
│   Git Repo  │ ──────> │   Jenkins    │ ──────> │ Storage Server  │
│   (Gitea)   │         │   Server     │         │ (/var/www/html) │
└─────────────┘         └──────────────┘         └─────────────────┘
                                                           │
                                                           │ NFS Share
                                                           │
                        ┌──────────────────────────────────┴─────────┐
                        │                                            │
                   ┌────▼─────┐      ┌──────────┐      ┌───────────▼┐
                   │  App     │      │  App     │      │  App        │
                   │ Server 1 │      │ Server 2 │      │  Server 3   │
                   │ (httpd)  │      │ (httpd)  │      │  (httpd)    │
                   └──────────┘      └──────────┘      └─────────────┘
```

## Technologies Used

- **Jenkins**: Automation server for CI/CD
- **Git/Gitea**: Version control system
- **Apache HTTP Server (httpd)**: Web server
- **SSH/SFTP**: Secure file transfer
- **NFS**: Network File System for shared storage
- **Linux (CentOS/RHEL)**: Operating system

## Requirements

### Infrastructure Setup

1. **Multiple Application Servers** running Apache httpd on port 8080
2. **Storage Server** with NFS-shared directory
3. **Jenkins Server** with required plugins
4. **Git Repository** (Gitea/GitHub/GitLab)

### Prerequisites

- Jenkins installed and running
- Git repository accessible from Jenkins
- SSH access between Jenkins and deployment servers
- Sudo privileges on deployment servers

## Implementation Steps

### Phase 1: Web Server Configuration

Configure Apache httpd on all application servers to serve content on port 8080:

```bash
# Install Apache HTTP Server
sudo yum install httpd -y

# Configure httpd to listen on port 8080
sudo sed -i 's/Listen 80/Listen 8080/g' /etc/httpd/conf/httpd.conf

# Start and enable httpd service
sudo systemctl start httpd
sudo systemctl enable httpd

# Verify service status
sudo systemctl status httpd
```

### Phase 2: Storage Server Preparation

Prepare the shared storage location:

```bash
# Set appropriate ownership for deployment user
sudo chown -R deployuser:deployuser /var/www/html

# Set proper permissions
sudo chmod -R 755 /var/www/html

# Verify NFS share is mounted on app servers
df -h | grep /var/www/html
```

### Phase 3: Jenkins Configuration

#### 3.1 Install Required Plugins

Navigate to **Manage Jenkins** → **Manage Plugins** and install:

- Git Plugin
- Publish Over SSH Plugin
- Gitea Plugin (optional)

#### 3.2 Configure SSH Credentials

1. Go to **Manage Jenkins** → **Manage Credentials**
2. Add SSH credentials for deployment user:
   - **Kind**: SSH Username with private key (or Username with password)
   - **ID**: `deploy-user-ssh`
   - **Username**: `deployuser`
   - **Private Key/Password**: [Configure as needed]

#### 3.3 Configure Publish Over SSH

1. Navigate to **Manage Jenkins** → **Configure System**
2. Scroll to **Publish over SSH** section
3. Add SSH Server:
   - **Name**: `storage-server`
   - **Hostname**: `storage.example.com`
   - **Username**: `deployuser`
   - **Remote Directory**: `/var/www/html`
4. Test configuration and save

### Phase 4: Create Jenkins Pipeline Job

#### 4.1 Create New Job

1. Click **New Item**
2. Enter job name: `web-app-deployment`
3. Select **Freestyle project**
4. Click **OK**

#### 4.2 Configure Source Code Management

```
Repository URL: http://git.example.com/developer/webapp.git
Credentials: [Select appropriate Git credentials]
Branch Specifier: */master
```

**Git Credentials Configuration:**
- **Kind**: Username with password
- **Username**: `gituser`
- **Password**: [Your Git password]
- **ID**: `git-credentials`

#### 4.3 Configure Build Triggers

Enable automatic builds on code commits:

```
☑ Poll SCM
Schedule: * * * * *
```

This configuration polls the Git repository every minute for changes.

**How Poll SCM Works:**
- Jenkins checks the repository at the specified interval
- If new commits are detected, a build is triggered automatically
- The schedule uses cron syntax (minute hour day month weekday)

#### 4.4 Configure Build Steps

**Option 1: Using Publish Over SSH Plugin (Recommended)**

Add build step: **Send files or execute commands over SSH**

```
SSH Server: storage-server
Source files: **/*
Remove prefix: [leave empty]
Remote directory: [leave empty - uses configured path]
Exec command:
  sudo chown -R deployuser:deployuser /var/www/html
  ls -la /var/www/html
```

**Option 2: Using Shell Script**

Add build step: **Execute shell**

```bash
#!/bin/bash
# Deploy application to storage server
sshpass -p 'PASSWORD' rsync -avz --delete \
  ${WORKSPACE}/ \
  deployuser@storage.example.com:/var/www/html/

# Verify deployment
sshpass -p 'PASSWORD' ssh deployuser@storage.example.com \
  'ls -la /var/www/html/'
```

### Phase 5: Testing the Pipeline

#### 5.1 Initial Deployment Test

Make a change to the repository:

```bash
# Clone repository
git clone http://git.example.com/developer/webapp.git
cd webapp

# Update index.html
echo "Welcome to My Application" > index.html

# Commit and push
git add index.html
git commit -m "Update welcome message"
git push origin master
```

#### 5.2 Verify Automated Build

1. Wait 1-2 minutes for Poll SCM to detect changes
2. Jenkins job should trigger automatically
3. Check build console output for success
4. Verify deployment on web servers

Expected console output:

```
Started by an SCM change
Building in workspace /var/lib/jenkins/workspace/web-app-deployment
Fetching changes from the remote Git repository
Checking out Revision abc123...
SSH: Transferred 1 file(s)
Finished: SUCCESS
```

#### 5.3 Verify Application Access

Access the application through load balancer:

```bash
curl http://loadbalancer.example.com:8080
# Should display: Welcome to My Application
```

## Key Features

### 1. Automated Deployment
- No manual intervention required
- Builds trigger automatically on Git push
- Consistent deployment process

### 2. Scalable Architecture
- Shared storage (NFS) across multiple app servers
- Easy to add more application servers
- Single deployment point

### 3. Build Repeatability
- Job configured to handle multiple runs
- Idempotent deployment process
- No conflicts on repeated builds

### 4. Security
- SSH-based secure file transfer
- Credential management in Jenkins
- Proper file ownership and permissions

## Troubleshooting Guide

### Issue: Poll SCM Not Triggering Builds

**Solution:**
- Verify Git credentials are configured in SCM section
- Check "Git Polling Log" for error messages
- Ensure repository URL is accessible from Jenkins

### Issue: Permission Denied on Deployment

**Solution:**
```bash
# On storage server
sudo chown -R deployuser:deployuser /var/www/html
sudo chmod -R 755 /var/www/html
```

### Issue: Files Deployed to Wrong Directory

**Solution:**
- Verify "Remote directory" is empty in SSH configuration
- Check that source files pattern is `**/*`
- Ensure no subdirectories are created inadvertently

### Issue: httpd Not Serving Content

**Solution:**
```bash
# On each app server
sudo systemctl restart httpd
sudo systemctl status httpd

# Verify port 8080 is listening
sudo netstat -tlnp | grep 8080
```

## Best Practices

### 1. Version Control
- Always commit with meaningful messages
- Use feature branches for development
- Tag releases for easy rollback

### 2. Jenkins Job Configuration
- Use credentials manager for sensitive data
- Enable build history retention
- Configure email notifications for build failures

### 3. Deployment Strategy
- Test in staging environment first
- Implement health checks
- Plan rollback procedures

### 4. Security
- Use SSH keys instead of passwords when possible
- Limit Jenkins user permissions
- Regularly update plugins and Jenkins core

### 5. Monitoring
- Monitor Jenkins build queue
- Track deployment success rates
- Set up alerts for failed builds

## Performance Optimization

### 1. Reduce Build Time
```groovy
// Clean workspace only when necessary
Delete workspace before build starts: [Only when needed]

// Use shallow clone
Advanced clone behaviours:
  Shallow clone: true
  Shallow clone depth: 1
```

### 2. Optimize File Transfer
```bash
# Use rsync with compression
rsync -avz --delete --exclude='.git' \
  ${WORKSPACE}/ user@server:/path/
```

## Validation Checklist

- [ ] httpd installed and running on port 8080 on all app servers
- [ ] Jenkins job created with correct name
- [ ] Poll SCM configured and working
- [ ] Git credentials properly configured
- [ ] Files deploy to correct directory (no subdirectories)
- [ ] Ownership and permissions set correctly
- [ ] Application accessible via load balancer
- [ ] Job passes on repetitive runs
- [ ] Console output shows "Started by an SCM change"

## Metrics and KPIs

Track these metrics for CI/CD effectiveness:

- **Deployment Frequency**: How often deployments occur
- **Build Success Rate**: Percentage of successful builds
- **Mean Time to Deploy**: Average time from commit to production
- **Build Duration**: Time taken for each build
- **Failed Build Recovery Time**: Time to fix failed builds

## Future Enhancements

1. **Blue-Green Deployment**
   - Implement zero-downtime deployments
   - Use load balancer to switch between environments

2. **Automated Testing**
   - Add unit tests before deployment
   - Implement integration tests
   - Run security scans

3. **Docker Integration**
   - Containerize the application
   - Use Docker for consistent environments
   - Implement container orchestration

4. **Advanced Monitoring**
   - Integrate with monitoring tools (Prometheus, Grafana)
   - Set up application performance monitoring
   - Implement log aggregation (ELK stack)

5. **Pipeline as Code**
   - Convert to Jenkins Pipeline (Jenkinsfile)
   - Version control pipeline configuration
   - Use declarative or scripted pipeline

## Sample Jenkinsfile (Advanced)

For those looking to implement Pipeline as Code:

```groovy
pipeline {
    agent any
    
    environment {
        DEPLOY_USER = 'deployuser'
        DEPLOY_SERVER = 'storage.example.com'
        DEPLOY_PATH = '/var/www/html'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master',
                    credentialsId: 'git-credentials',
                    url: 'http://git.example.com/developer/webapp.git'
            }
        }
        
        stage('Deploy') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'storage-server',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: '**/*',
                                    removePrefix: '',
                                    remoteDirectory: '',
                                    execCommand: 'sudo chown -R deployuser:deployuser /var/www/html'
                                )
                            ]
                        )
                    ]
                )
            }
        }
        
        stage('Verify') {
            steps {
                sh '''
                    curl -f http://loadbalancer.example.com:8080 || exit 1
                    echo "Deployment verified successfully"
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed. Please check the logs.'
        }
    }
}
```

## Learning Resources

- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [Git Documentation](https://git-scm.com/doc)
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
- [Jenkins Pipeline Tutorial](https://www.jenkins.io/doc/book/pipeline/)



This project is provided as-is for educational and demonstration purposes.

## Acknowledgments

- DevOps community for best practices
- Jenkins community for excellent documentation
- Open source contributors

---

**Note**: This is a demonstration project. Adapt configurations based on your specific infrastructure and security requirements.
