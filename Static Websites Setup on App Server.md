#FusionCorp Industries – Static Websites Setup on App Server 3

This repository documents the setup of two static websites (`beta` and `apps`) for **xFusionCorp Industries** on **App Server 3** in the Stratos Datacenter.

---

## Tasks Completed

1. **Install Apache HTTP Server**

   ```bash
   sudo yum install -y httpd
   ```

2. **Configure Apache to listen on port 6200**

   ```bash
   sudo vi /etc/httpd/conf/httpd.conf
   # Change 'Listen 80' to 'Listen 6200'
   sudo systemctl restart httpd
   sudo systemctl enable httpd
   ```

3. **Copy website backups from jump_host to App Server 3**

   ```bash
   # From stapp03
   scp thor@jump_host:/tmp/beta.tar /tmp/
   scp thor@jump_host:/tmp/apps.tar /tmp/

   # Extract into Apache root
   sudo tar -xvf /tmp/beta.tar -C /var/www/html/
   sudo tar -xvf /tmp/apps.tar -C /var/www/html/

   # Move directories to correct location
   sudo mv /var/www/html/home/thor/beta /var/www/html/
   sudo mv /var/www/html/home/thor/apps /var/www/html/
   sudo rm -rf /var/www/html/home
   ```

4. **Set correct permissions and ownership**

   ```bash
   sudo chown -R apache:apache /var/www/html/beta /var/www/html/apps
   sudo chmod -R 755 /var/www/html/beta /var/www/html/apps
   ```

5. **Verify websites are accessible locally**

   ```bash
   curl http://localhost:6200/beta/
   curl http://localhost:6200/apps/
   ```

---

## Outcome

* Apache HTTP Server is running and serving content on **port 6200**.
* The two websites are accessible at:

  * `http://localhost:6200/beta/`
  * `http://localhost:6200/apps/`
* File ownership and permissions are correctly set for Apache.
* Verified using `curl` commands on App Server 3.

---

## Notes

* During the setup, SCP of directories sometimes hangs due to lab restrictions. Using **tar archives** and copying them as single files is a reliable workaround.
* Always ensure Apache is running and listening on the configured port before testing with `curl`.
