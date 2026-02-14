# Nginx SSL Deployment - App Server 2

## Task Description
Deploy and configure nginx with SSL on App Server 2 in Stratos Datacenter for xFusionCorp Industries.

## Prerequisites
- Access to App Server 2
- Self-signed SSL certificate at `/tmp/nautilus.crt`
- Self-signed SSL key at `/tmp/nautilus.key`
- Root or sudo access

## Deployment Steps

### Step 1: Connect to App Server 2
```bash
ssh user@app-server-2
```

### Step 2: Install Nginx
```bash
# For RHEL/CentOS systems
sudo yum update -y
sudo yum install -y nginx

# For Debian/Ubuntu systems
# sudo apt update
# sudo apt install -y nginx
```

### Step 3: Configure SSL Certificates
```bash
# Create SSL directory
sudo mkdir -p /etc/nginx/ssl

# Move certificates to appropriate location
sudo mv /tmp/nautilus.crt /etc/nginx/ssl/
sudo mv /tmp/nautilus.key /etc/nginx/ssl/

# Set secure permissions
sudo chmod 644 /etc/nginx/ssl/nautilus.crt
sudo chmod 600 /etc/nginx/ssl/nautilus.key
```

### Step 4: Configure Nginx for HTTPS
Create SSL configuration file:
```bash
sudo vi /etc/nginx/conf.d/ssl.conf
```

Add the following configuration:
```nginx
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/nautilus.crt;
    ssl_certificate_key /etc/nginx/ssl/nautilus.key;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Step 5: Create Welcome Page
```bash
echo "Welcome!" | sudo tee /usr/share/nginx/html/index.html
```

### Step 6: Start and Enable Nginx
```bash
# Start nginx service
sudo systemctl start nginx

# Enable nginx to start on boot
sudo systemctl enable nginx

# Verify nginx is running
sudo systemctl status nginx
```

### Step 7: Configure Firewall (if needed)
```bash
# For firewalld
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# For iptables
# sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
# sudo service iptables save
```

## Testing

### From Jump Host
Test the deployment from the jump host:

```bash
# Test HTTPS connection (headers only)
curl -Ik https://<app-server-2-ip>/

# Test HTTPS connection (full content)
curl -k https://<app-server-2-ip>/
```

### Expected Output
**Headers:**
```
HTTP/1.1 200 OK
Server: nginx/1.x.x
Date: ...
Content-Type: text/html
Content-Length: 9
Connection: keep-alive
```

**Content:**
```
Welcome!
```

## Troubleshooting

### Nginx won't start
```bash
# Check nginx configuration syntax
sudo nginx -t

# Check nginx error logs
sudo tail -f /var/log/nginx/error.log
```

### Certificate errors
```bash
# Verify certificate files exist and have correct permissions
ls -la /etc/nginx/ssl/

# Check certificate details
openssl x509 -in /etc/nginx/ssl/nautilus.crt -text -noout
```

### Connection refused
```bash
# Verify nginx is listening on port 443
sudo netstat -tlnp | grep 443
# or
sudo ss -tlnp | grep 443

# Check firewall status
sudo firewall-cmd --list-all
```

### SELinux issues (RHEL/CentOS)
```bash
# If SELinux is enforcing, you may need to set proper context
sudo semanage fcontext -a -t httpd_sys_content_t "/etc/nginx/ssl(/.*)?"
sudo restorecon -Rv /etc/nginx/ssl
```

## Verification Checklist
- [ ] Nginx installed successfully
- [ ] SSL certificates moved to `/etc/nginx/ssl/`
- [ ] Nginx configuration file created
- [ ] `index.html` contains "Welcome!"
- [ ] Nginx service started and enabled
- [ ] Firewall allows HTTPS traffic
- [ ] HTTPS connection accessible from jump host
- [ ] curl command returns 200 OK status

## Notes
- The `-k` flag in curl bypasses SSL certificate verification (safe for self-signed certificates in testing)
- Default nginx document root is `/usr/share/nginx/html` (may vary by distribution)
- Certificate and key files should never be world-readable for security

## Author
xFusionCorp Industries - System Admins Team

## Date
January 14, 2026
