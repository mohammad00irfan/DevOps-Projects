```markdown
# Nginx Load Balancer Configuration - Nautilus Infrastructure

## Task Overview
Configure Nginx load balancer (LBR server) for high availability stack in Stratos DC with three Apache app servers.

## Server Details
- **LBR Server (Nginx)**: 172.16.238.14 (stlb01)
- **App Servers (Apache)**: 
  - 172.16.238.11:6300 (stapp01)
  - 172.16.238.12:6300 (stapp02)
  - 172.16.238.13:6300 (stapp03)

## Steps Performed

### 1. Install Nginx on LBR Server
```bash
ssh loki@stlb01
sudo yum install nginx -y
```

### 2. Verify Apache on App Servers
```bash
# Confirmed Apache running on port 6300 on all app servers
curl http://172.16.238.11:6300
curl http://172.16.238.12:6300
curl http://172.16.238.13:6300
```

### 3. Configure Nginx Load Balancer
Edited `/etc/nginx/nginx.conf` and added in the `http` block:

```nginx
http {
    # ... existing configuration ...
    
    upstream app_servers {
        server 172.16.238.11:6300;
        server 172.16.238.12:6300;
        server 172.16.238.13:6300;
    }
    
    server {
        listen 80;
        server_name _;
        
        location / {
            proxy_pass http://app_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### 4. Test and Start Nginx
```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

### 5. Verify Load Balancing
```bash
curl http://172.16.238.14
curl http://localhost
# Output: Welcome to xFusionCorp Industries!
```

## Result
✅ Nginx load balancer successfully configured on port 80  
✅ Traffic distributed across three Apache servers on port 6300  
✅ Apache port unchanged (requirement met)  
✅ Application accessible via StaticApp button

## Configuration Notes
- Nginx uses round-robin load balancing by default
- Apache remains on original port 6300 (not modified)
- Proxy headers configured for proper client information forwarding
```
