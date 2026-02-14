Linux System Administration & DevOps Tasks
A collection of real-world Linux system administration and DevOps scenarios solved in a production-like environment (Stratos Datacenter).
📋 Table of Contents

Task 1: MariaDB Service Troubleshooting
Task 2: Automated Backup Scripts
Task 3: Apache Tomcat Deployment
Task 4: Apache Service Network Troubleshooting
Task 5: Firewall Security Implementation


Task 1: MariaDB Service Troubleshooting
Problem
MariaDB service failed to start on the database server, preventing the Nautilus application from connecting to the database.
Error Analysis
bash[ERROR] InnoDB: Operating system error number 13 in a file operation.
[ERROR] InnoDB: The error means mysqld does not have the access rights to the directory.
[ERROR] InnoDB: Cannot open datafile './ibtmp1'
Root Cause

Permission denied (Error 13)
/var/lib/mysql directory owned by root:root instead of mysql:mysql
MariaDB daemon runs as mysql user and couldn't access its data directory

Solution
bash# Fix ownership of MySQL data directory
chown -R mysql:mysql /var/lib/mysql

# Start MariaDB service
systemctl start mariadb
systemctl enable mariadb

# Verify service status
systemctl status mariadb
Key Learnings

Linux services run under specific user accounts for security
Error code 13 = Permission denied
File ownership is critical for database operations
Always check logs: /var/log/mariadb/mariadb.log


Task 2: Automated Backup Scripts
Problem
Create bash scripts to automate website backups with the following requirements:

Backup /var/www/html/media directory
Store locally in /backup/ as temporary storage
Copy to remote Nautilus Backup Server
No password prompts during execution
Non-root user should be able to run the script

Solution
SSH Key-Based Authentication Setup
bash# Generate SSH key pair (no passphrase)
ssh-keygen -t rsa -N ""

# Copy public key to backup server
ssh-copy-id clint@stbkp01

# Test passwordless authentication
ssh clint@stbkp01 "echo 'Connection successful'"
Backup Script: media_backup.sh
bash#!/bin/bash

# Variables
SOURCE_DIR="/var/www/html/media"
BACKUP_DIR="/backup"
ARCHIVE_NAME="xfusioncorp_media.zip"
BACKUP_SERVER_USER="clint"
BACKUP_SERVER_HOST="stbkp01"
BACKUP_SERVER_DIR="/backup"

# Create zip archive
cd /var/www/html
zip -r ${BACKUP_DIR}/${ARCHIVE_NAME} media

# Check if archive was created successfully
if [ -f "${BACKUP_DIR}/${ARCHIVE_NAME}" ]; then
    echo "Archive created successfully: ${BACKUP_DIR}/${ARCHIVE_NAME}"
    
    # Copy archive to Backup Server
    scp ${BACKUP_DIR}/${ARCHIVE_NAME} ${BACKUP_SERVER_USER}@${BACKUP_SERVER_HOST}:${BACKUP_SERVER_DIR}/
    
    # Check if copy was successful
    if [ $? -eq 0 ]; then
        echo "Archive copied successfully to Backup Server"
    else
        echo "Failed to copy archive to Backup Server"
        exit 1
    fi
else
    echo "Failed to create archive"
    exit 1
fi
Deployment
bash# Create script directory
mkdir -p /scripts

# Create and set permissions
vi /scripts/media_backup.sh
chmod +x /scripts/media_backup.sh
chown tony:tony /scripts/media_backup.sh

# Set proper directory permissions
chown tony:tony /backup
chmod -R 755 /var/www/html/media
Key Learnings

SSH key authentication enables passwordless automation
Exit codes ($?) validate command success
Proper error handling prevents silent failures
Script ownership and permissions affect execution capability


Task 3: Apache Tomcat Deployment
Problem
Deploy a Java-based application (ROOT.war) on Apache Tomcat running on port 5000.
Solution
bash# Install Tomcat
yum install tomcat -y

# Configure Tomcat port
vi /etc/tomcat/server.xml
# Change: <Connector port="8080" to port="5000"

# Copy WAR file from jump host
scp thor@jump_host:/tmp/ROOT.war /tmp/

# Deploy application
rm -rf /usr/share/tomcat/webapps/ROOT
cp /tmp/ROOT.war /usr/share/tomcat/webapps/

# Start Tomcat service
systemctl start tomcat
systemctl enable tomcat

# Verify deployment
curl http://localhost:5000
Architecture Understanding

Tomcat: Java servlet container and web server
WAR file: Web Application Archive (packaged Java application)
ROOT.war: Special naming convention for base URL deployment

ROOT.war → accessible at http://server:5000/
app.war → accessible at http://server:5000/app/



Key Learnings

Tomcat default port is 8080 (customizable in server.xml)
WAR files auto-extract when placed in webapps directory
Service should be enabled to start on boot


Task 4: Apache Service Network Troubleshooting
Problem
Apache service running on port 8085 was not reachable from jump host, showing "No route to host" error.
Troubleshooting Process
Step 1: Verify Apache Status
bashsystemctl status httpd
# Result: Active and running ✓
Step 2: Check Port Binding
bashnetstat -tulpn | grep 8085
# Result: 0.0.0.0:8085 (listening on all interfaces) ✓
Step 3: Verify Apache Configuration
bashgrep -i "^Listen" /etc/httpd/conf/httpd.conf
# Result: Listen 8085 ✓
Step 4: Check Firewall
bashsystemctl status firewalld
# Result: Service not found (not the issue)
Step 5: Check SELinux
bashgetenforce
# Result: Disabled (not the issue)
Step 6: Check iptables
bashiptables -L -n -v
```

**Found the issue:**
```
Chain INPUT (policy ACCEPT)
ACCEPT     tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  state NEW tcp dpt:22
REJECT     all  --  *  *  0.0.0.0/0  0.0.0.0/0  reject-with icmp-host-prohibited
The REJECT rule was blocking all traffic except SSH!
Solution
bash# Insert rule BEFORE the REJECT rule
iptables -I INPUT -p tcp --dport 8085 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

# Save rules permanently
service iptables save

# Verify rule order
iptables -L -n -v --line-numbers
```

**Correct rule order:**
```
1  ACCEPT  tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  tcp dpt:8085
2  ACCEPT  tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  state NEW tcp dpt:22
3  REJECT  all  --  *  *  0.0.0.0/0  0.0.0.0/0  reject-with icmp-host-prohibited
```

### **Key Learnings**
- "No route to host" error often indicates firewall blocking
- iptables rules are processed top-to-bottom
- `-I` (insert) vs `-A` (append) affects rule placement
- Rule order is critical when REJECT/DROP rules exist
- Always save iptables rules for persistence

---

## **Task 5: Firewall Security Implementation**

### **Problem**
Secure Apache servers by implementing firewall rules to allow access only from Load Balancer while blocking all other traffic to port 8087.

### **Requirements**
1. Install iptables on all app servers
2. Block incoming port 8087 for everyone except Load Balancer
3. Ensure rules persist after system reboot

### **Solution**

#### **Architecture**
```
[Internet/Users]
        ↓
[Load Balancer - stlb01]
    172.16.238.14
        ↓
    ✓ ALLOWED
        ↓
┌───────┬───────┬───────┐
│stapp01│stapp02│stapp03│
│:8087  │:8087  │:8087  │
└───────┴───────┴───────┘
        ↑
    ✗ BLOCKED
        ↑
[Direct Access Attempts]
Implementation
bash# Install iptables
yum install iptables iptables-services -y

# Start and enable service
systemctl start iptables
systemctl enable iptables

# Get Load Balancer IP
LBR_IP=$(getent hosts stlb01 | awk '{print $1}')
echo "Load Balancer IP: $LBR_IP"

# Configure firewall rules (ORDER MATTERS!)
# Rule 1: ALLOW Load Balancer (must be FIRST)
iptables -A INPUT -p tcp -s ${LBR_IP} --dport 8087 -j ACCEPT

# Rule 2: DROP everyone else (must be SECOND)
iptables -A INPUT -p tcp --dport 8087 -j DROP

# Save rules for persistence
service iptables save

# Verify configuration
iptables -L INPUT -n -v --line-numbers
```

#### **Expected Output**
```
Chain INPUT (policy ACCEPT)
num   pkts bytes target     prot opt source               destination
1        0     0 ACCEPT     tcp  --  172.16.238.14        0.0.0.0/0            tcp dpt:8087
2        0     0 DROP       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8087
Testing
bash# Test from Load Balancer (should WORK)
ssh loki@stlb01
curl http://stapp01:8087  # ✓ Success
curl http://stapp02:8087  # ✓ Success
curl http://stapp03:8087  # ✓ Success

# Test from Jump Host (should FAIL)
curl http://stapp01:8087  # ✗ Connection timed out
curl http://stapp02:8087  # ✗ Connection timed out
curl http://stapp03:8087  # ✗ Connection timed out
Common Issues & Solutions
Issue: Rules in Wrong Order
bash# Wrong order - DROP blocks everything!
1  DROP    tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8087
2  ACCEPT  tcp  --  172.16.238.14  0.0.0.0/0  tcp dpt:8087
Fix:
bash# Clear and rebuild
iptables -F INPUT
iptables -A INPUT -p tcp -s 172.16.238.14 --dport 8087 -j ACCEPT
iptables -A INPUT -p tcp --dport 8087 -j DROP
service iptables save
Issue: Duplicate Rules
bash# Check for duplicates
iptables -L -n -v --line-numbers | grep 8087

# Remove specific rule by line number
iptables -D INPUT 3  # Delete rule at line 3
Persistence Verification
bash# Rules saved to
cat /etc/sysconfig/iptables

# Test after reboot
reboot
iptables -L -n -v  # Rules should still be present
Key Learnings

iptables processes rules sequentially (first match wins)
ACCEPT rules must come BEFORE DROP/REJECT rules
-A appends to end, -I inserts at beginning
service iptables save writes to /etc/sysconfig/iptables
systemctl enable iptables ensures service starts on boot
"Connection timed out" confirms firewall is blocking (expected behavior)


🛠️ Tools & Technologies Used
CategoryToolsOperating SystemCentOS/RHEL LinuxWeb ServersApache HTTP Server, Apache TomcatDatabaseMariaDB 10.5ScriptingBash Shell ScriptingSecurityiptables, SSH Key Authentication, SELinuxNetworkingnetstat, telnet, curl, scpService Managementsystemd (systemctl)Package Managementyum/dnf

📊 Skills Demonstrated
System Administration

✅ Linux service management (systemd)
✅ File permissions and ownership troubleshooting
✅ Log analysis and debugging
✅ Package installation and dependency management

Networking & Security

✅ Firewall configuration (iptables)
✅ Network troubleshooting (netstat, telnet)
✅ SSH key-based authentication
✅ Port management and security hardening

Scripting & Automation

✅ Bash script development
✅ Error handling and validation
✅ Automated backup solutions
✅ Passwordless remote operations

Database Administration

✅ MariaDB troubleshooting
✅ Service recovery procedures
✅ Permission management

Application Deployment

✅ Java application deployment (Tomcat)
✅ WAR file management
✅ Application server configuration


🎯 Problem-Solving Approach
Systematic Troubleshooting Methodology

Identify the Problem

Read error messages carefully
Understand the expected vs actual behavior


Check Service Status

systemctl status <service>
Review service logs


Verify Configuration

Check configuration files
Validate syntax and parameters


Test Connectivity

Local tests first
Then remote connectivity
Use tools: curl, telnet, netstat


Check Security Layers

Firewall rules (iptables/firewalld)
SELinux status
File permissions


Implement Solution

Make targeted changes
Document modifications
Test thoroughly


Ensure Persistence

Save configurations
Enable services for auto-start
Verify after reboot




📝 Best Practices Applied
Security

✅ Principle of least privilege (specific ports, specific sources)
✅ No hardcoded passwords in scripts
✅ SSH key authentication over password authentication
✅ Services run under dedicated user accounts
✅ Firewall rules for network segmentation

Reliability

✅ Error handling in scripts
✅ Exit codes for validation
✅ Persistent configurations (survive reboots)
✅ Service auto-start enabled

Maintainability

✅ Clear variable naming
✅ Commented code
✅ Modular script structure
✅ Consistent configuration management

Operations

✅ Comprehensive testing before production
✅ Local testing before remote
✅ Verification steps documented
✅ Rollback procedures considered


🔍 Common Error Patterns & Solutions
ErrorCauseSolutionPermission denied (13)Wrong file ownershipchown user:group /pathAddress already in usePort conflictChange port or stop conflicting serviceNo route to hostFirewall blockingAdd iptables/firewall ruleConnection refusedService not runningStart service with systemctlConnection timed outFirewall dropping packetsCheck iptables DROP/REJECT rulesPassword prompts in scriptsNo SSH keysSetup ssh-keygen + ssh-copy-id

📚 Documentation References

Apache HTTP Server Documentation
Apache Tomcat Documentation
MariaDB Documentation
iptables Tutorial
Bash Scripting Guide
systemd Documentation


💡 Key Takeaways

Error codes tell a story - Understanding system error codes (like errno 13) speeds up troubleshooting
Rule order matters - In iptables, first match wins; structure rules from specific to general
Always verify - Test locally before testing remotely; verify after each change
Automation requires planning - Passwordless operations need proper key setup
Security by design - Implement least privilege from the start, not as an afterthought
Persistence is crucial - Configurations must survive reboots in production
Documentation saves time - Well-documented solutions become reusable templates


🚀 Future Improvements

 Implement log rotation for backup scripts
 Add monitoring alerts for failed backups
 Create Ansible playbooks for automated deployment
 Implement backup retention policies
 Add database backup to the automation suite
 Create centralized logging solution
 Implement infrastructure as code (Terraform)


📞 Contact
Feel free to reach out if you have questions about these implementations or want to discuss Linux system administration and DevOps practices.

⚖️ License
This documentation is provided for educational and professional showcase purposes.
