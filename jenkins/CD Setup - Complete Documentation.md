# Jenkins CI/CD Setup Guide - Complete Documentation

## Overview
This repository contains comprehensive documentation for setting up and configuring Jenkins server for CI/CD pipelines at xFusionCorp Industries under the Stratos Datacenter infrastructure.

---

## Table of Contents
1. [Initial Jenkins Installation](#1-initial-jenkins-installation)
2. [Plugin Management](#2-plugin-management)
3. [User Management and Access Control](#3-user-management-and-access-control)
4. [Automated Package Installation Job](#4-automated-package-installation-job)
5. [Troubleshooting](#troubleshooting)

---

## 1. Initial Jenkins Installation

### Prerequisites
- CentOS/RHEL-based system
- Root access to the Jenkins server
- Jump host with SSH access
- Java 11 or 17 installed

### Task Requirements
- Install Jenkins using `yum` utility only
- Start and enable Jenkins service
- Create admin user with specific credentials

### Installation Steps

#### Step 1: Connect to Jenkins Server
```bash
ssh root@jenkins
# Password: S3curePass
```

#### Step 2: Install Java (if not already installed)
```bash
# Check Java version
java -version

# Install Java 17 if needed
sudo yum install java-17-openjdk java-17-openjdk-devel -y
```

#### Step 3: Add Jenkins Repository
```bash
# Download Jenkins repository file
sudo curl -o /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

# Import Jenkins GPG key
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

#### Step 4: Install Jenkins
```bash
# Update yum cache
sudo yum clean all
sudo yum makecache

# Install Jenkins
sudo yum install jenkins -y
```

#### Step 5: Start Jenkins Service
```bash
# Reload systemd daemon
sudo systemctl daemon-reload

# Start Jenkins (takes 2-3 minutes)
sudo systemctl start jenkins

# Enable Jenkins on boot
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins
```

**Expected Output:**
```
● jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled)
     Active: active (running)
```

#### Step 6: Retrieve Initial Admin Password
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Initial Setup via Web UI

1. **Access Jenkins UI**: Click the Jenkins button on the top bar
2. **Unlock Jenkins**: Paste the initial admin password
3. **Plugin Installation**: 
   - Option 1: Click "Install suggested plugins" (slower)
   - Option 2: Click "Select plugins to install" → "None" → "Install" (faster)
4. **Create First Admin User**:
   - Username: `theadmin`
   - Password: `Adm!n321`
   - Full name: As specified (e.g., `Yousuf`, `Kareem`)
   - Email: As specified (e.g., `yousuf@jenkins.stratos.xfusioncorp.com`)
5. **Instance Configuration**: Keep default Jenkins URL
6. **Complete Setup**: Click "Start using Jenkins"

### Common Issues

**Issue: Jenkins service fails to start**
```bash
# Check logs
sudo journalctl -xeu jenkins.service

# Common fix: Ensure Java is installed
java -version
```

**Issue: Timeout during startup**
- Jenkins startup can take 2-3 minutes
- The service extends timeout at each initialization milestone
- Wait patiently for "Jenkins is fully up and running" message

---

## 2. Plugin Management

### Task: Install Git and GitLab Plugins

#### Step 1: Access Plugin Manager
1. Login to Jenkins UI
2. Click **Manage Jenkins** → **Plugins**
3. Click **Available plugins** tab

#### Step 2: Search and Select Plugins
1. Search for **"Git plugin"**
2. Check the checkbox
3. Search for **"GitLab plugin"**
4. Check the checkbox

#### Step 3: Install with Restart
1. Click **"Install"** button
2. ☑ Check: **"Restart Jenkins when installation is complete and no jobs are running"**
3. Wait 1-2 minutes for automatic restart
4. Login again once the login page reappears

#### Verification
1. Go to **Manage Jenkins** → **Plugins** → **Installed plugins**
2. Verify both plugins are listed

---

## 3. User Management and Access Control

### Task: Configure User Access with Matrix Authorization

#### Requirements
- Create specific users with limited permissions
- Implement Project-based Matrix Authorization Strategy
- Remove anonymous access
- Maintain admin privileges

### Step 1: Create New User

1. **Navigate to User Management**:
   - Manage Jenkins → Manage Users → Create User

2. **User Details** (Example):
   - Username: `ammar`
   - Password: `ksH85UJjhb`
   - Full name: `Ammar`
   - Email: (optional)

3. Click **"Create User"**

### Step 2: Install Matrix Authorization Plugin

1. **Manage Jenkins** → **Plugins** → **Available plugins**
2. Search: **"Matrix Authorization Strategy Plugin"**
3. Install and restart Jenkins

### Step 3: Configure Global Security

1. **Manage Jenkins** → **Security** → **Configure Global Security**
2. Under **Authorization**, select: **"Project-based Matrix Authorization Strategy"**
3. Configure permissions matrix:

#### Permission Matrix Configuration

| User | Overall Read | Overall Administer | Job Read | Job Configure |
|------|--------------|-------------------|----------|---------------|
| admin | ✓ | ✓ | ✓ | ✓ |
| ammar | ✓ | ✗ | (set at job level) | ✗ |
| Anonymous | ✗ | ✗ | ✗ | ✗ |

**Adding Users to Matrix:**
```
1. In "User/group to add" field, type: admin
2. Click "Add"
3. Check "Administer" under Overall column
4. Repeat for other users with appropriate permissions
```

**Remove Anonymous Access:**
- Uncheck all boxes for Anonymous user, OR
- Click delete/X button to remove Anonymous row entirely

4. Click **"Save"**

### Step 4: Configure Job-Level Permissions

1. Click on existing job (or create new one)
2. Click **"Configure"**
3. Check: ☑ **"Enable project-based security"**
4. Select: **"Do not inherit permissions from other ACLs"**
5. Add users and permissions:
   - `admin`: All permissions
   - `ammar`: Job → Read only
6. Click **"Save"**

### Security Configuration Summary

```yaml
Global Permissions:
  admin:
    - Overall: Administer (full access)
  user (e.g., ammar, mark):
    - Overall: Read only
  Anonymous:
    - All permissions: REMOVED

Job-Level Permissions:
  admin:
    - All job permissions
  user:
    - Job: Read only
```

---

## 4. Automated Package Installation Job

### Task: Create Jenkins Job for Package Installation

#### Requirements
- Job name: `install-packages`
- String parameter: `PACKAGE`
- Install packages on storage server
- Repeatable and reliable execution

### Step 1: Install SSH Plugin (if needed)

1. **Manage Jenkins** → **Plugins** → **Available plugins**
2. Search: **"Publish Over SSH"** or **"SSH"**
3. Install and restart if needed

### Step 2: Create New Job

1. From Dashboard: **New Item**
2. Item name: `install-packages`
3. Select: **"Freestyle project"**
4. Click **"OK"**

### Step 3: Configure String Parameter

1. ☑ Check: **"This project is parameterized"**
2. Click **"Add Parameter"** → **"String Parameter"**
3. Configure:
   - **Name**: `PACKAGE`
   - **Default Value**: (optional, e.g., `httpd`)
   - **Description**: `Package name to install on storage server`

### Step 4: Configure Build Step

#### Option A: Direct SSH Execution
```bash
# Add build step: Execute shell
ssh root@ststor01 "yum install -y $PACKAGE"
```

#### Option B: Using SSH Plugin
1. Configure SSH server in **Manage Jenkins** → **Configure System** → **SSH Servers**
2. Add build step: **"Execute shell script on remote host using ssh"**
3. Script:
```bash
yum install -y $PACKAGE
```

### Step 5: Complete Job Configuration

```yaml
Job Configuration:
  Name: install-packages
  Type: Freestyle project
  
  Parameters:
    - Type: String Parameter
      Name: PACKAGE
      Description: Package name to install
  
  Build Steps:
    - Execute shell on remote host (storage server)
    - Command: yum install -y $PACKAGE
  
  Post-build Actions: (optional)
    - Email notification on failure
    - Build history retention
```

### Step 6: Test the Job

1. Click **"Build with Parameters"**
2. Enter package name (e.g., `wget`, `curl`, `htop`)
3. Click **"Build"**
4. Monitor **"Console Output"**

**Expected Output:**
```
Started by user admin
Building in workspace /var/lib/jenkins/workspace/install-packages
[install-packages] $ /bin/sh -xe /tmp/jenkins123.sh
+ ssh root@ststor01 'yum install -y wget'
Loaded plugins: fastestmirror
Installing: wget
Complete!
Finished: SUCCESS
```

### Step 7: Verify Repeated Executions

Test with multiple packages:
```
Build #1: PACKAGE=wget
Build #2: PACKAGE=curl
Build #3: PACKAGE=htop
```

All should complete successfully.

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Jenkins Service Won't Start
```bash
# Check Java installation
java -version

# Check Jenkins logs
sudo journalctl -u jenkins -n 50

# Verify Jenkins is installed
rpm -qa | grep jenkins

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl start jenkins
```

#### 2. Timeout During Startup
- **Cause**: Jenkins initialization takes time
- **Solution**: Wait 2-3 minutes, check status periodically
```bash
sudo systemctl status jenkins
```

#### 3. Plugin Installation Issues
- Clear browser cache
- Try different browser
- Check internet connectivity on Jenkins server
- Manual plugin installation via `.hpi` file

#### 4. Permission Denied Errors
- Ensure proper user permissions in matrix
- Check job-level security settings
- Verify admin has "Administer" permission

#### 5. SSH Connection to Storage Server Failed
```bash
# Test SSH connectivity
ssh root@ststor01

# Check SSH key permissions
ls -la ~/.ssh/

# Verify storage server is reachable
ping ststor01
```

#### 6. Email Field Not Showing
- Install **Mailer plugin**
- Go to Manage Jenkins → Plugins → Available
- Search "Mailer" and install
- Configure user email in Manage Users

---

## Best Practices

### Security
1. ✅ Always remove Anonymous access
2. ✅ Use principle of least privilege
3. ✅ Enable project-based security for sensitive jobs
4. ✅ Regularly update Jenkins and plugins
5. ✅ Use strong passwords
6. ✅ Enable CSRF protection

### Job Configuration
1. ✅ Use parameterized builds for flexibility
2. ✅ Add descriptive job names and descriptions
3. ✅ Implement error handling in shell scripts
4. ✅ Set up notifications for build failures
5. ✅ Archive important build artifacts
6. ✅ Configure build retention policies

### Maintenance
1. ✅ Regular backups of Jenkins home directory
2. ✅ Monitor disk space usage
3. ✅ Review and clean old builds
4. ✅ Keep plugins updated
5. ✅ Monitor Jenkins logs for errors

---

## Quick Reference Commands

### Service Management
```bash
# Start Jenkins
sudo systemctl start jenkins

# Stop Jenkins
sudo systemctl stop jenkins

# Restart Jenkins
sudo systemctl restart jenkins

# Check status
sudo systemctl status jenkins

# View logs
sudo journalctl -u jenkins -f
```

### File Locations
```bash
# Jenkins home
/var/lib/jenkins/

# Initial admin password
/var/lib/jenkins/secrets/initialAdminPassword

# Service configuration
/usr/lib/systemd/system/jenkins.service

# Override configuration
/etc/systemd/system/jenkins.service.d/override.conf
```

### Common Jenkins CLI Operations
```bash
# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Check Jenkins version
java -jar jenkins-cli.jar -s http://localhost:8080/ version

# List installed plugins
java -jar jenkins-cli.jar -s http://localhost:8080/ list-plugins
```

---

## Resources

### Official Documentation
- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [Jenkins Plugin Index](https://plugins.jenkins.io/)
- [Jenkins Security](https://www.jenkins.io/doc/book/security/)

### Useful Plugins
- Git Plugin
- GitLab Plugin
- Matrix Authorization Strategy Plugin
- Publish Over SSH
- Email Extension Plugin
- Build Timeout Plugin

### Community
- [Jenkins Community Forums](https://community.jenkins.io/)
- [Jenkins JIRA](https://issues.jenkins.io/)
- [Jenkins GitHub](https://github.com/jenkinsci/jenkins)

---

## Contributing
Feel free to contribute to this documentation by submitting pull requests or opening issues for corrections and improvements.

## License
This documentation is provided as-is for educational and operational purposes.

---

**Last Updated**: January 2026  
**Maintained by**: DevOps Team - xFusionCorp Industries  
**Environment**: Stratos Datacenter
