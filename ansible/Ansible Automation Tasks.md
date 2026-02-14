```markdown
# Ansible Automation Tasks - Nautilus DevOps Lab

A comprehensive collection of Ansible playbooks and configurations for managing infrastructure in the Stratos Datacenter. This repository demonstrates various Ansible concepts including inventory management, playbooks, roles, templates, conditionals, and ACLs.

## 📋 Table of Contents

- [Infrastructure Overview](#infrastructure-overview)
- [Prerequisites](#prerequisites)
- [Tasks Completed](#tasks-completed)
  - [1. Basic Inventory Setup](#1-basic-inventory-setup)
  - [2. File Creation with Ansible](#2-file-creation-with-ansible)
  - [3. Password-less SSH Setup](#3-password-less-ssh-setup)
  - [4. Package Installation (httpd)](#4-package-installation-httpd)
  - [5. Web Server Setup with Content](#5-web-server-setup-with-content)
  - [6. CSV File Copy](#6-csv-file-copy)
  - [7. File Creation with Different Owners](#7-file-creation-with-different-owners)
  - [8. Package Installation (vsftpd)](#8-package-installation-vsftpd)
  - [9. ACL Permissions Management](#9-acl-permissions-management)
  - [10. Web Server with blockinfile](#10-web-server-with-blockinfile)
  - [11. Ansible Roles with Jinja2 Templates](#11-ansible-roles-with-jinja2-templates)
  - [12. Conditional File Copy](#12-conditional-file-copy)
- [Key Learnings](#key-learnings)
- [Best Practices](#best-practices)

## 🏗️ Infrastructure Overview

### Server Details

| Server Name | IP Address    | Hostname                           | User   | Password  | Purpose              |
|-------------|---------------|------------------------------------|--------|-----------|----------------------|
| stapp01     | 172.16.238.10 | stapp01.stratos.xfusioncorp.com   | tony   | Ir0nM@n   | Nautilus App 1       |
| stapp02     | 172.16.238.11 | stapp02.stratos.xfusioncorp.com   | steve  | Am3ric@   | Nautilus App 2       |
| stapp03     | 172.16.238.12 | stapp03.stratos.xfusioncorp.com   | banner | BigGr33n  | Nautilus App 3       |
| jump_host   | Dynamic       | jump_host.stratos.xfusioncorp.com | thor   | mjolnir123| Jump Server/Ansible Controller |

## 🔧 Prerequisites

- Ansible installed on jump host
- SSH access to all app servers
- Sudo privileges on managed nodes
- Basic understanding of YAML syntax

## ✅ Tasks Completed

### 1. Basic Inventory Setup

**Objective:** Create an inventory file for App Server 1 and test Ansible connectivity.

**Files:**
- Location: `/home/thor/playbook/inventory`

**Solution:**
```ini
stapp01 ansible_host=172.16.238.10 ansible_user=tony ansible_password=Ir0nM@n ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

**Validation:**
```bash
ansible -i inventory stapp01 -m ping
```

**Key Concepts:**
- Inventory file structure
- Ansible variables (ansible_host, ansible_user, ansible_password)
- SSH options configuration

---

### 2. File Creation with Ansible

**Objective:** Create a blank file on App Server 1 using Ansible playbook.

**Files:**
- Inventory: `/home/thor/playbook/inventory`
- Playbook: `/home/thor/playbook/playbook.yml`

**Playbook:**
```yaml
---
- name: Create empty file on App Server 1
  hosts: stapp01
  become: yes
  tasks:
    - name: Create empty file /tmp/file.txt
      file:
        path: /tmp/file.txt
        state: touch
```

**Execution:**
```bash
ansible-playbook -i inventory playbook.yml
```

**Key Concepts:**
- Playbook structure
- File module usage
- Privilege escalation with `become`

---

### 3. Password-less SSH Setup

**Objective:** Configure SSH key-based authentication between jump host and App Server 3.

**Steps:**

1. Generate SSH key pair:
```bash
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa
```

2. Copy public key to App Server 3:
```bash
sshpass -p 'BigGr33n' ssh-copy-id -o StrictHostKeyChecking=no banner@stapp03
```

3. Update inventory (remove password):
```ini
stapp03 ansible_host=172.16.238.12 ansible_user=banner ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

4. Test connectivity:
```bash
ansible -i inventory stapp03 -m ping
```

**Key Concepts:**
- SSH key generation
- Key-based authentication
- Security best practices

---

### 4. Package Installation (httpd)

**Objective:** Install httpd package on all app servers using Ansible.

**Files:**
- Inventory: `/home/thor/playbook/inventory`
- Playbook: `/home/thor/playbook/playbook.yml`

**Inventory:**
```ini
stapp01 ansible_host=172.16.238.10 ansible_user=tony ansible_password=Ir0nM@n ansible_ssh_common_args='-o StrictHostKeyChecking=no'
stapp02 ansible_host=172.16.238.11 ansible_user=steve ansible_password=Am3ric@ ansible_ssh_common_args='-o StrictHostKeyChecking=no'
stapp03 ansible_host=172.16.238.12 ansible_user=banner ansible_password=BigGr33n ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

**Playbook:**
```yaml
---
- name: Install httpd package on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install httpd package using yum
      yum:
        name: httpd
        state: present
```

**Key Concepts:**
- Multi-host inventory
- YUM package management
- Idempotency in Ansible

---

### 5. Web Server Setup with Content

**Objective:** Install httpd, start service, and deploy a sample web page.

**Files:**
- Inventory: `/home/thor/ansible/inventory`
- Playbook: `/home/thor/ansible/playbook.yml`

**Playbook:**
```yaml
---
- name: Setup httpd web server on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install httpd package
      yum:
        name: httpd
        state: present
    
    - name: Start and enable httpd service
      service:
        name: httpd
        state: started
        enabled: yes
    
    - name: Add content to index.html using blockinfile
      blockinfile:
        path: /var/www/html/index.html
        create: yes
        block: |
          Welcome to XfusionCorp!
          This is Nautilus sample file, created using Ansible!
          Please do not modify this file manually!
    
    - name: Set owner and group of index.html to apache
      file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode: '0777'
```

**Key Concepts:**
- Service management
- blockinfile module
- File permissions and ownership

---

### 6. CSV File Copy

**Objective:** Copy index.html file from jump host to all app servers.

**Files:**
- Inventory: `/home/thor/ansible/inventory`
- Playbook: `/home/thor/ansible/playbook.yml`

**Playbook:**
```yaml
---
- name: Copy index.html to all application servers
  hosts: stapp01,stapp02,stapp03
  become: yes
  tasks:
    - name: Ensure /opt/data directory exists
      file:
        path: /opt/data
        state: directory
        mode: '0755'
    
    - name: Copy index.html to /opt/data
      copy:
        src: /usr/src/data/index.html
        dest: /opt/data/index.html
        mode: '0644'
```

**Key Concepts:**
- Copy module
- Directory creation
- File transfer between hosts

---

### 7. File Creation with Different Owners

**Objective:** Create files with specific permissions and different owners on each app server.

**Files:**
- Inventory: `/home/thor/playbook/inventory`
- Playbook: `/home/thor/playbook/playbook.yml`

**Playbook:**
```yaml
---
- name: Create file with specific permissions on App Server 1
  hosts: stapp01
  become: yes
  tasks:
    - name: Create /opt/data.txt with tony as owner
      file:
        path: /opt/data.txt
        state: touch
        mode: '0777'
        owner: tony
        group: tony

- name: Create file with specific permissions on App Server 2
  hosts: stapp02
  become: yes
  tasks:
    - name: Create /opt/data.txt with steve as owner
      file:
        path: /opt/data.txt
        state: touch
        mode: '0777'
        owner: steve
        group: steve

- name: Create file with specific permissions on App Server 3
  hosts: stapp03
  become: yes
  tasks:
    - name: Create /opt/data.txt with banner as owner
      file:
        path: /opt/data.txt
        state: touch
        mode: '0777'
        owner: banner
        group: banner
```

**Key Concepts:**
- Multiple plays in one playbook
- Host-specific configurations
- File ownership management

---

### 8. Package Installation (vsftpd)

**Objective:** Install and configure vsftpd service on all app servers.

**Files:**
- Inventory: `/home/thor/ansible/inventory`
- Playbook: `/home/thor/ansible/playbook.yml`

**Playbook:**
```yaml
---
- name: Install and configure vsftpd on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install vsftpd package
      yum:
        name: vsftpd
        state: present
    
    - name: Start and enable vsftpd service
      service:
        name: vsftpd
        state: started
        enabled: yes
```

**Key Concepts:**
- Service installation
- Service enablement for boot
- Package state management

---

### 9. ACL Permissions Management

**Objective:** Create files with ACL permissions for specific users/groups on each app server.

**Files:**
- Inventory: `/home/thor/ansible/inventory`
- Playbook: `/home/thor/ansible/playbook.yml`

**Playbook:**
```yaml
---
- name: Create blog.txt with ACL on App Server 1
  hosts: stapp01
  become: yes
  tasks:
    - name: Ensure /opt/dba directory exists
      file:
        path: /opt/dba
        state: directory
        mode: '0755'
    
    - name: Create empty file blog.txt
      file:
        path: /opt/dba/blog.txt
        state: touch
        owner: root
        group: root
    
    - name: Set ACL for group tony on blog.txt
      acl:
        path: /opt/dba/blog.txt
        entity: tony
        etype: group
        permissions: r
        state: present

- name: Create story.txt with ACL on App Server 2
  hosts: stapp02
  become: yes
  tasks:
    - name: Ensure /opt/dba directory exists
      file:
        path: /opt/dba
        state: directory
        mode: '0755'
    
    - name: Create empty file story.txt
      file:
        path: /opt/dba/story.txt
        state: touch
        owner: root
        group: root
    
    - name: Set ACL for user steve on story.txt
      acl:
        path: /opt/dba/story.txt
        entity: steve
        etype: user
        permissions: rw
        state: present

- name: Create media.txt with ACL on App Server 3
  hosts: stapp03
  become: yes
  tasks:
    - name: Ensure /opt/dba directory exists
      file:
        path: /opt/dba
        state: directory
        mode: '0755'
    
    - name: Create empty file media.txt
      file:
        path: /opt/dba/media.txt
        state: touch
        owner: root
        group: root
    
    - name: Set ACL for group banner on media.txt
      acl:
        path: /opt/dba/media.txt
        entity: banner
        etype: group
        permissions: rw
        state: present
```

**Verification:**
```bash
# Check ACL on each server
ansible -i inventory stapp01 -m shell -a "getfacl /opt/dba/blog.txt" -b
ansible -i inventory stapp02 -m shell -a "getfacl /opt/dba/story.txt" -b
ansible -i inventory stapp03 -m shell -a "getfacl /opt/dba/media.txt" -b
```

**Key Concepts:**
- ACL module usage
- Entity types (user vs group)
- Fine-grained permissions
- Root ownership with ACL permissions

---

### 10. Web Server with blockinfile

**Objective:** Setup httpd with custom index.html using blockinfile and lineinfile modules.

**Files:**
- Inventory: `/home/thor/ansible/inventory`
- Playbook: `/home/thor/ansible/playbook.yml`

**Playbook:**
```yaml
---
- name: Setup httpd web server with sample page on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install httpd package
      yum:
        name: httpd
        state: present
    
    - name: Start and enable httpd service
      service:
        name: httpd
        state: started
        enabled: yes
    
    - name: Create index.html with initial content
      copy:
        content: "This is a Nautilus sample file, created using Ansible!\n"
        dest: /var/www/html/index.html
    
    - name: Add welcome message at the top using lineinfile
      lineinfile:
        path: /var/www/html/index.html
        line: "Welcome to xFusionCorp Industries!"
        insertbefore: BOF
    
    - name: Set owner and group of index.html to apache
      file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode: '0777'
```

**Expected Content:**
```
Welcome to xFusionCorp Industries!
This is a Nautilus sample file, created using Ansible!
```

**Key Concepts:**
- lineinfile module
- insertbefore: BOF (Beginning Of File)
- Content manipulation
- Multiple content modification methods

---

### 11. Ansible Roles with Jinja2 Templates

**Objective:** Use Ansible roles and Jinja2 templates to deploy dynamic web content.

**Directory Structure:**
```
/home/thor/ansible/
├── inventory
├── playbook.yml
└── role/
    └── httpd/
        ├── tasks/
        │   └── main.yml
        └── templates/
            └── index.html.j2
```

**Files:**

**playbook.yml:**
```yaml
---
- name: Run httpd role on App Server 2
  hosts: stapp02
  become: yes
  roles:
    - role/httpd
```

**role/httpd/templates/index.html.j2:**
```jinja2
This file was created using Ansible on {{ inventory_hostname }}
```

**role/httpd/tasks/main.yml:**
```yaml
---
- name: Install httpd package
  yum:
    name: httpd
    state: present

- name: Start and enable httpd service
  service:
    name: httpd
    state: started
    enabled: yes

- name: Copy index.html template to web server
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0777'
```

**Expected Output on stapp02:**
```
This file was created using Ansible on stapp02
```

**Key Concepts:**
- Ansible roles structure
- Jinja2 templating
- Template module
- Variable substitution (inventory_hostname, ansible_user)
- Dynamic content generation

---

### 12. Conditional File Copy

**Objective:** Use Ansible conditionals to copy different files to different servers.

**Files:**
- Inventory: `/home/thor/ansible/inventory`
- Playbook: `/home/thor/ansible/playbook.yml`

**Playbook:**
```yaml
---
- name: Copy files to app servers using conditionals
  hosts: all
  become: yes
  tasks:
    - name: Ensure /opt/sysops directory exists
      file:
        path: /opt/sysops
        state: directory
        mode: '0755'
    
    - name: Copy blog.txt to App Server 1
      copy:
        src: /usr/src/sysops/blog.txt
        dest: /opt/sysops/blog.txt
        owner: tony
        group: tony
        mode: '0777'
      when: ansible_nodename == "stapp01.stratos.xfusioncorp.com"
    
    - name: Copy story.txt to App Server 2
      copy:
        src: /usr/src/sysops/story.txt
        dest: /opt/sysops/story.txt
        owner: steve
        group: steve
        mode: '0777'
      when: ansible_nodename == "stapp02.stratos.xfusioncorp.com"
    
    - name: Copy media.txt to App Server 3
      copy:
        src: /usr/src/sysops/media.txt
        dest: /opt/sysops/media.txt
        owner: banner
        group: banner
        mode: '0777'
      when: ansible_nodename == "stapp03.stratos.xfusioncorp.com"
```

**Execution Flow:**
- Playbook runs on all hosts
- Each task checks the `ansible_nodename` variable
- Only matching tasks execute on each server
- Non-matching tasks are skipped

**Key Concepts:**
- Conditional execution with `when`
- ansible_nodename variable
- Fact gathering
- Task skipping based on conditions
- Single playbook, multiple behaviors

---

## 🎓 Key Learnings

### Ansible Modules Used

1. **file** - Create files/directories, set permissions
2. **copy** - Copy files from controller to managed nodes
3. **yum** - Package management
4. **service** - Service management (start/stop/enable)
5. **template** - Deploy Jinja2 templates
6. **lineinfile** - Modify specific lines in files
7. **blockinfile** - Insert/update blocks of text
8. **acl** - Manage Access Control Lists

### Ansible Concepts Covered

- ✅ Inventory management (static)
- ✅ Playbook structure and syntax
- ✅ Variables (built-in and custom)
- ✅ Privilege escalation (become)
- ✅ Conditionals (when)
- ✅ Loops and iterations
- ✅ Handlers
- ✅ Roles and directory structure
- ✅ Jinja2 templating
- ✅ Fact gathering
- ✅ Idempotency
- ✅ ACL management

### Variables Used

| Variable | Description | Example |
|----------|-------------|---------|
| ansible_host | IP address of managed node | 172.16.238.10 |
| ansible_user | SSH username | tony, steve, banner |
| ansible_password | SSH password | Ir0nM@n |
| ansible_nodename | Fully qualified hostname | stapp01.stratos.xfusioncorp.com |
| inventory_hostname | Inventory hostname | stapp01 |
| ansible_ssh_common_args | Additional SSH arguments | '-o StrictHostKeyChecking=no' |

## 📚 Best Practices

### 1. Security
- Use SSH keys instead of passwords when possible
- Store sensitive data in Ansible Vault (not demonstrated here for lab purposes)
- Use `become` only when necessary
- Implement least privilege principle

### 2. Idempotency
- Always use `state` parameters in modules
- Design playbooks to be run multiple times safely
- Use appropriate modules (e.g., `file` with `state: touch` vs shell commands)

### 3. Organization
- Use roles for complex configurations
- Separate concerns (inventory, playbooks, variables)
- Use meaningful task names
- Comment complex logic

### 4. Variables
- Use built-in variables when available
- Avoid hardcoding values
- Use templates for dynamic content

### 5. Testing
- Test playbooks with `--check` mode
- Use `ansible -m ping` to verify connectivity
- Verify results after execution

### 6. Error Handling
- Plan for transient failures (rc=137 errors)
- Use `async` and `poll` for long-running tasks
- Implement retries when appropriate

## 🚀 Running the Playbooks

### Standard Execution
```bash
ansible-playbook -i inventory playbook.yml
```

### Check Mode (Dry Run)
```bash
ansible-playbook -i inventory playbook.yml --check
```

### Verbose Output
```bash
ansible-playbook -i inventory playbook.yml -v
ansible-playbook -i inventory playbook.yml -vv
ansible-playbook -i inventory playbook.yml -vvv
```

### Limit to Specific Hosts
```bash
ansible-playbook -i inventory playbook.yml --limit stapp01
ansible-playbook -i inventory playbook.yml --limit stapp01,stapp02
```

### Step-by-Step Execution
```bash
ansible-playbook -i inventory playbook.yml --step
```

## 📝 Common Commands

### Ad-hoc Commands
```bash
# Ping all hosts
ansible -i inventory all -m ping

# Check disk space
ansible -i inventory all -m shell -a "df -h" -b

# Check service status
ansible -i inventory all -m shell -a "systemctl status httpd" -b

# Gather facts
ansible -i inventory all -m setup

# Check specific fact
ansible -i inventory all -m setup -a "filter=ansible_nodename"
```

### Verification Commands
```bash
# Check file permissions
ansible -i inventory all -m shell -a "ls -l /path/to/file" -b

# Check file content
ansible -i inventory all -m shell -a "cat /path/to/file" -b

# Check package installation
ansible -i inventory all -m shell -a "rpm -q package_name" -b

# Check ACL
ansible -i inventory all -m shell -a "getfacl /path/to/file" -b
```

## 🔍 Troubleshooting

### Common Issues

**1. Module Failure (rc=137)**
- **Cause:** Memory constraints or timeout
- **Solution:** Run playbook again or use `gather_facts: no` and `async` parameters

**2. Permission Denied**
- **Cause:** Missing `become: yes`
- **Solution:** Add privilege escalation to tasks requiring root

**3. File Not Found**
- **Cause:** Incorrect path or file doesn't exist
- **Solution:** Verify source file exists, check paths

**4. Connection Timeout**
- **Cause:** SSH issues, firewall, or incorrect credentials
- **Solution:** Test SSH manually, verify credentials, check network

**5. Task Skipped Unexpectedly**
- **Cause:** Conditional not met
- **Solution:** Verify variable values with `ansible -m setup`

## 📊 Success Metrics

All tasks completed successfully with:
- ✅ Zero failures in final runs
- ✅ Proper idempotency (running twice shows no changes)
- ✅ Correct file permissions and ownership
- ✅ Services running and enabled
- ✅ Web servers serving content correctly
- ✅ ACLs properly configured

## 🤝 Contributing

This is a learning repository for Ansible automation. Feel free to:
- Suggest improvements
- Add additional tasks
- Optimize existing playbooks
- Share alternative solutions

## 📄 License

This project is for educational purposes as part of the Nautilus DevOps lab environment.

## 👨‍💻 Author

DevOps Engineer working with the Nautilus team in Stratos Datacenter

---

**Note:** All passwords and sensitive information shown here are for lab environment only. Never commit real credentials to version control!
```

This comprehensive README provides:
- Complete documentation of all tasks
- Code examples for each scenario
- Explanations of key concepts
- Best practices and troubleshooting tips
- Command references for common operations
- Professional formatting suitable for GitHub showcase
