# Question 3: Secure Web Server Configuration (Apache HTTPS)

## Objective

Install and configure a web server using **Apache (httpd)** on the Rocky Linux server. Websites must be accessible from the Ubuntu client, and all traffic must be secured using **HTTPS (SSL/TLS)** with a self-signed certificate.

---

## Lab Environment Setup

| Role | Hostname | IP Address |
|---|---|---|
| Rocky Linux Server | `server.kawkaw.com` | `192.168.200.3` |
| Ubuntu Client | `client.kawkaw.com` | `192.168.200.4` |

---

## Part 1: Rocky Linux Server Configuration

### Step 1: Verify Operating System & Update

```bash
# Verify OS version
cat /etc/os-release

# Update system repositories
sudo dnf update -y
```

---

### Step 2: Install Apache and the SSL Module

On Rocky Linux, Apache is packaged as `httpd`. The SSL/TLS support is provided by a **separate package** called `mod_ssl` — both must be installed.

```bash
# Install Apache web server
sudo dnf install httpd -y

# Install the SSL module (also auto-creates /etc/httpd/conf.d/ssl.conf)
sudo dnf install mod_ssl -y
```

> **Why mod_ssl matters:** Installing `mod_ssl` automatically creates the file `/etc/httpd/conf.d/ssl.conf`, which contains the default HTTPS VirtualHost on port 443. Without this package, HTTPS will not work at all.

---

### Step 3: Generate a Dedicated Web Server Certificate

Create an isolated directory for the web server certificates.

```bash
# Create directory for Apache certificates
sudo mkdir -p /etc/ssl/httpd

# Generate self-signed certificate and private key
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/httpd/server.key \
  -out /etc/ssl/httpd/server.crt
```

> **When prompted, enter:**
> - Country: `MY`
> - Locality: `KL`
> - Common Name: `server.kawkaw.com`

**Set correct file permissions** so Apache can read the key:

```bash
sudo chmod 644 /etc/ssl/httpd/server.crt
sudo chmod 640 /etc/ssl/httpd/server.key
sudo chown root:apache /etc/ssl/httpd/server.key
```

**Fix SELinux labels** so the confined `httpd` process is allowed to read the certificate files:

```bash
sudo restorecon -Rv /etc/ssl/httpd/
```

---

### Step 4: Create a Test Web Page

Create a simple HTML page to verify the web server is serving content correctly.

```bash
sudo nano /var/www/html/index.html
```

Paste the following content:

```html
<!DOCTYPE html>
<html>
  <head><title>kawkaw.com</title></head>
  <body>
    <h1>Welcome to server.kawkaw.com</h1>
    <p>Apache HTTPS is working correctly.</p>
  </body>
</html>
```

---

### Step 5: Configure HTTPS in `ssl.conf`

`mod_ssl` creates a default VirtualHost in `/etc/httpd/conf.d/ssl.conf` that points to placeholder certificates. We must update it to use our own certificate.

```bash
sudo nano /etc/httpd/conf.d/ssl.conf
```

Find and modify these three lines inside the existing `<VirtualHost _default_:443>` block:

```apache
# Add or set the server name
ServerName server.kawkaw.com

# Point to our certificate
SSLCertificateFile /etc/ssl/httpd/server.crt

# Point to our private key
SSLCertificateKeyFile /etc/ssl/httpd/server.key
```

> **Important:** Do **not** create a new `<VirtualHost *:443>` block. `ssl.conf` already contains one. Adding a second will cause a conflict and Apache will fail to start. Only modify the lines listed above within the existing block.

---

### Step 6: Configure HTTP to HTTPS Redirect

To ensure all plain HTTP traffic on port 80 is automatically redirected to HTTPS, add a redirect VirtualHost to the main Apache configuration.

```bash
sudo nano /etc/httpd/conf/httpd.conf
```

Add the following block at the **end** of the file:

```apache
<VirtualHost *:80>
    ServerName server.kawkaw.com
    Redirect permanent / https://server.kawkaw.com/
</VirtualHost>
```

> This uses Apache's built-in `mod_alias` (loaded by default) to issue a `301 Permanent Redirect`, sending any HTTP request to the HTTPS equivalent.

---

### Step 7: Verify Apache Configuration Syntax

Before starting the service, always test the configuration for errors.

```bash
sudo apachectl configtest
```

Expected output:

```
Syntax OK
```

If you see any errors, re-check Steps 5 and 6 for typos or duplicate VirtualHost blocks.

---

### Step 8: Firewall & Service Activation

```bash
# Open ports: HTTP (80) and HTTPS (443)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Enable Apache to start on boot and start it now
sudo systemctl enable --now httpd

# Verify the service is running (look for "active (running)")
sudo systemctl status httpd
```

---

## Part 2: Ubuntu Client Verification

### Step 1: Install curl (if not already present)

```bash
sudo apt update
sudo apt install curl -y
```

---

### Step 2: Verify HTTP Redirects to HTTPS

Test that a plain HTTP request is automatically redirected to HTTPS:

```bash
curl -I http://192.168.200.3
```

Expected output shows a `301 Moved Permanently` redirect to HTTPS:

```
HTTP/1.1 301 Moved Permanently
Location: https://server.kawkaw.com/
```

---

### Step 3: Verify HTTPS is Serving Content

The `-k` flag tells curl to accept the self-signed certificate without error:

```bash
curl -k https://192.168.200.3
```

Expected output — the HTML content of your test page:

```html
<!DOCTYPE html>
<html>
  <head><title>kawkaw.com</title></head>
  ...
</html>
```

---

### Step 4: Verify TLS Certificate Details

Use the verbose flag to inspect the TLS handshake and confirm the certificate details:

```bash
curl -kv https://192.168.200.3 2>&1 | grep -E "subject|issuer|SSL|TLS"
```

Expected output confirms TLS is active and the certificate matches your server:

```
* SSL connection using TLSv1.3 / ...
* Server certificate:
*  subject: C=MY; L=KL; CN=server.kawkaw.com
*  issuer: C=MY; L=KL; CN=server.kawkaw.com
```

---

### Step 5: Verify via Firefox (GUI)

1. Open **Firefox** on the Ubuntu client.
2. Navigate to `https://192.168.200.3`.
3. Firefox will show a **"Warning: Potential Security Risk Ahead"** message because the certificate is self-signed.
4. Click **Advanced** → **Accept the Risk and Continue**.
5. Confirm the page displays: **"Welcome to server.kawkaw.com"**.

> To inspect the certificate in Firefox: click the **padlock icon** in the address bar → **Connection Secure** → **More Information** → **View Certificate**.

---

## Summary of Configured Services

| Service | Port | Protocol | Purpose |
|---|---|---|---|
| Apache HTTP | 80 | HTTP | Redirects to HTTPS |
| Apache HTTPS | 443 | HTTPS (TLS) | Serves web content |

---

## Troubleshooting

### Apache Fails to Start — Wrong Certificate Path

**Symptom:** `sudo systemctl status httpd` shows a failed state immediately after enabling.

**Check the error:**

```bash
sudo journalctl -xe | grep httpd
# or
sudo apachectl configtest
```

You may see:

```
AH02561: Failed to configure certificate (with key) for server.kawkaw.com
httpd: Could not reliably determine the server's fully qualified domain name
```

**Fix:** Confirm the paths in `ssl.conf` exactly match the files you created:

```bash
ls -l /etc/ssl/httpd/
```

Both `server.crt` and `server.key` must exist. Re-check Step 5.

---

### SELinux Blocking Certificate Read

**Symptom:** `apachectl configtest` passes (Syntax OK), but `httpd` still fails to start. The audit log shows a denial.

**Check the SELinux audit log:**

```bash
sudo cat /var/log/audit/audit.log | grep denied | grep httpd
```

You will see something like:

```
type=AVC msg=audit(...): avc: denied { read } for pid=... comm="httpd"
name="server.key" scontext=system_u:system_r:httpd_t:s0
tcontext=system_u:object_r:user_home_t:s0 tclass=file
```

The label `user_home_t` is wrong — it should be `cert_t`.

**Fix:**

```bash
sudo restorecon -Rv /etc/ssl/httpd/
```

**Verify the labels are now correct:**

```bash
ls -Z /etc/ssl/httpd/
```

Expected output:

```
system_u:object_r:cert_t:s0 server.crt
system_u:object_r:cert_t:s0 server.key
```

Then restart Apache:

```bash
sudo systemctl restart httpd
sudo systemctl status httpd
```

---

### Duplicate VirtualHost Conflict on Port 443

**Symptom:** Apache starts but logs a warning, and HTTPS may behave unpredictably.

**Check:**

```bash
sudo apachectl configtest 2>&1 | grep -i "overlap\|conflict\|duplicate"
```

You may see:

```
[warn] _default_ VirtualHost overlap on port 443, the first has precedence
```

**Fix:** Ensure there is only **one** `<VirtualHost *:443>` or `<VirtualHost _default_:443>` block across all files in `/etc/httpd/conf.d/`. Remove any custom block you may have added and only edit the existing one in `ssl.conf`.

---

### Port 443 Not Reachable from Ubuntu Client

**Symptom:** `curl -k https://192.168.200.3` hangs or returns `Connection refused`.

**Check the firewall on the Rocky server:**

```bash
sudo firewall-cmd --list-services
```

`https` must appear in the output. If it does not:

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## References

- Red Hat System Administration (RH124)
- Apache HTTP Server Documentation: https://httpd.apache.org/docs/
- OpenSSL Documentation: https://www.openssl.org/docs/