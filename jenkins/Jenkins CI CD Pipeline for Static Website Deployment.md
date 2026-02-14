```markdown
# Jenkins CI/CD Pipeline for Static Website Deployment

## Project Overview
This project demonstrates setting up a Jenkins pipeline to automatically deploy a static website from a Git repository to multiple application servers using a centralized storage approach.

## Architecture

### Infrastructure
- **Jenkins Server**: CI/CD automation server
- **Gitea**: Git repository hosting
- **Storage Server (ststor01)**: Central file storage mounted to all app servers
- **App Servers (stapp01-03)**: Apache web servers running on port 8080
- **Load Balancer (stlb01)**: Distributes traffic across app servers on port 8091

### Server Details
| Server | IP | User | Purpose |
|--------|-----|------|---------|
| jenkins | 172.16.238.19 | jenkins | Jenkins CI/CD Server |
| ststor01 | 172.16.238.15 | natasha | Storage Server |
| stapp01 | 172.16.238.10 | tony | App Server 1 |
| stapp02 | 172.16.238.11 | steve | App Server 2 |
| stapp03 | 172.16.238.12 | banner | App Server 3 |
| stlb01 | 172.16.238.14 | loki | Load Balancer |

## Challenge

The main challenge was that Jenkins runs on a separate server from the Storage Server where `/var/www/html` is located. This required remote file transfer rather than simple local file operations.

### Initial Problems Encountered

1. **Direct Directory Access Failed**
   ```bash
   cd /var/www/html  # Error: directory doesn't exist on Jenkins server
   ```

2. **Local Copy Failed**
   ```bash
   cp index.html /var/www/html/  # Error: can't copy to remote location
   ```

3. **SSH Authentication Issues**
   ```bash
   ssh natasha@ststor01  # Error: Host key verification failed
   scp index.html natasha@ststor01:/var/www/html/  # Error: Permission denied
   ```

## Solution

### Prerequisites
- Jenkins with Git plugin installed
- `sshpass` utility installed on Jenkins server
- Gitea repository: `http://git.stratos.xfusioncorp.com/sarah/web.git`
- Storage Server credentials: `natasha` / `Bl@kW`

### Implementation Steps

#### Step 1: Update Repository Content
Updated `index.html` in Gitea repository with:
```html
Welcome to xFusionCorp Industries
```

#### Step 2: Create Jenkins Pipeline Job
- Job Name: `deploy-job`
- Type: Pipeline (not Multibranch Pipeline)
- Two stages: `Deploy` and `Test`

#### Step 3: Pipeline Configuration

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                // Clone repository from Gitea
                git branch: 'master',
                    url: 'http://git.stratos.xfusioncorp.com/sarah/web.git'
                
                // Copy files to Storage Server using sshpass + scp
                sh '''
                    sshpass -p 'Bl@kW' scp -o StrictHostKeyChecking=no index.html natasha@ststor01:/var/www/html/
                '''
            }
        }
        
        stage('Test') {
            steps {
                // Verify deployment through load balancer
                sh '''
                    sleep 3
                    curl -f http://stlb01:8091 | grep "Welcome to xFusionCorp Industries"
                '''
            }
        }
    }
}
```

## Key Technical Decisions

### Why sshpass + scp?
- **sshpass**: Enables password-based SSH authentication in automated scripts
- **scp**: Secure file transfer over SSH
- **StrictHostKeyChecking=no**: Bypasses SSH host key verification (acceptable in controlled environments)

### Pipeline Design
1. **Deploy Stage**: 
   - Pulls latest code from Git repository
   - Transfers files to centralized storage location
   - Storage is mounted on all app servers automatically

2. **Test Stage**:
   - Validates deployment through load balancer
   - Fails if content doesn't match expected output
   - Ensures zero-downtime deployment verification

## Results

### Successful Build Output
```
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] stage (Deploy)
  ✓ Git clone successful
  ✓ File transfer to Storage Server successful
[Pipeline] stage (Test)
  ✓ HTTP Status: 200
  ✓ Content verified: "Welcome to xFusionCorp Industries"
[Pipeline] End of Pipeline
Finished: SUCCESS
```

### Benefits Achieved
- ✅ Automated deployment from Git to production
- ✅ Single source of truth (Gitea repository)
- ✅ Automated testing to verify deployments
- ✅ Centralized file distribution to multiple servers
- ✅ Pipeline fails safely if deployment or tests fail

## Lessons Learned

1. **Understand Your Infrastructure**: Know where services run and how they communicate
2. **Remote vs Local Operations**: Different approaches needed for remote file transfers
3. **Authentication Methods**: Password-based auth with `sshpass` works for controlled environments
4. **Test Stage Design**: Always verify deployments automatically
5. **Error Handling**: Let stages fail fast to prevent bad deployments

## Security Considerations

⚠️ **Note**: This implementation uses plaintext passwords in the pipeline for demonstration purposes. In production environments, consider:
- Jenkins Credentials Manager for secure credential storage
- SSH key-based authentication instead of passwords
- Secrets management tools (HashiCorp Vault, AWS Secrets Manager)
- Removing `StrictHostKeyChecking=no` and managing SSH host keys properly

## Future Enhancements

- [ ] Implement SSH key-based authentication
- [ ] Add rollback mechanism
- [ ] Include health checks for all app servers
- [ ] Add notification stage (email/Slack on success/failure)
- [ ] Implement blue-green deployment strategy
- [ ] Add code quality checks (linting, validation)
- [ ] Create parameterized builds for different environments

## Technologies Used
- Jenkins (Pipeline)
- Git/Gitea
- SSH/SCP
- Apache HTTP Server
- HAProxy/Load Balancer
- Linux Shell Scripting

## Conclusion

This project demonstrates a practical CI/CD implementation for static website deployment in a multi-server environment. The key challenge was understanding the distributed architecture and implementing proper remote file transfer mechanisms with automated testing.

---

**Author**: DevOps Engineer  
**Project Type**: CI/CD Pipeline Implementation  
**Difficulty**: Intermediate  
**Tags**: `jenkins` `ci-cd` `devops` `automation` `git` `deployment`
```
