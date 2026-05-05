# Question 2: Secure Email Configuration (SMTP/IMAP)

## Objective

Install and configure a secure email server using **Postfix** (SMTP) and **Dovecot** (IMAP) on the Rocky Linux server. Local users must be able to send and receive emails. TLS/SSL encryption is implemented to secure communications. Thunderbird on the Ubuntu client is used to connect to, send, and receive mail.

---

## Lab Environment Setup

| Role | Hostname | IP Address |
|---|---|---|
| Rocky Linux Server | `server.kawkaw.com` | `192.168.200.3` |
| Ubuntu Client | `client.kawkaw.com` | `192.168.200.4` |

---

## Part 1: Rocky Linux Server Configuration

### Step 1: Verify Operating System & Update

Launch your Rocky VM and verify the environment before starting.

```bash
# Verify OS version
cat /etc/os-release

# Update system repositories
sudo dnf update -y
```

---

### Step 2: Install OpenSSL & Mail Packages

OpenSSL is required to generate the cryptographic keys used for securing communications.

```bash
# Install OpenSSL
sudo dnf install openssl -y

# Install Postfix (MTA) and Dovecot (IMAP server)
sudo dnf install postfix dovecot -y
```

---

### Step 3: Generate a New Dedicated Mail Certificate

We will create a separate directory to ensure isolation from other services.

```bash
# Create directory for mail certificates
sudo mkdir -p /etc/ssl/mail

# Generate self-signed certificate and private key
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/mail/mailserver.key \
  -out /etc/ssl/mail/mailserver.crt
```

> **When prompted, enter:**
> - Country: `MY`
> - Locality: `KL`
> - Common Name: `server.kawkaw.com`

**Set correct file permissions** so Postfix and Dovecot can read the key:

```bash
sudo chmod 644 /etc/ssl/mail/mailserver.crt
sudo chmod 640 /etc/ssl/mail/mailserver.key
sudo chown root:postfix /etc/ssl/mail/mailserver.key
```

---

### Step 4: Configure Postfix (SMTP)

#### 4a — Edit the main configuration file to enable the server to route mail and use SSL/TLS. `main.cf`

```bash
sudo nano /etc/postfix/main.cf
```

Add or modify the following lines:

```ini
myhostname = server.kawkaw.com
mydomain = kawkaw.com
myorigin = $mydomain
inet_interfaces = all # <- uncomment this
# comment out inet_interfaces = localhost
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain # <- uncomment this
# comment out mydestination = $myhostname, localhost.$mydomain, localhost
home_mailbox = Maildir/
smtpd_tls_cert_file = /etc/ssl/mail/mailserver.crt
smtpd_tls_key_file = /etc/ssl/mail/mailserver.key
smtpd_tls_security_level = may
smtpd_tls_loglevel = 1
```

> **Why these additions matter:**
> - `myorigin` — sets the domain appended to outgoing mail addresses.
> - `mydestination` — tells Postfix which domains to accept mail for locally. Without this, Postfix will **reject** all mail addressed to `kawkaw.com`.
> - `smtpd_tls_security_level = may` — the correct modern directive. The old `smtpd_use_tls = yes` is deprecated in current Postfix versions.

#### 4b — Enable Port 587 (Submission) in `master.cf`

Port 587 is **commented out by default** on Rocky Linux. Thunderbird sends mail on this port — the lab will silently fail if this step is skipped.

```bash
sudo nano /etc/postfix/master.cf
```

Find the line beginning with `#submission` and **uncomment and edit** it to look exactly like this:

```
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
```

> **Important:** The two continuation lines must be indented with **spaces** (not tabs), and there must be **no blank line** between them and the `submission` line.

---

### Step 5: Configure Dovecot (IMAP)

We must tell Dovecot where the mailbox is and point it to our new certificates.

#### 5a — Set Mailbox Location

```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Set or modify:

```ini
mail_location = maildir:~/Maildir
```

#### 5b — Enable SSL

```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

Set or modify:

```ini
ssl = required
ssl_cert = </etc/ssl/mail/mailserver.crt
ssl_key = </etc/ssl/mail/mailserver.key
```

> The `<` character is **intentional Dovecot syntax** — it reads the file contents from the given path. Do not remove it.

#### 5c — Configure Authentication

```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Set or modify:

```ini
disable_plaintext_auth = no
auth_mechanisms = plain login
```

> `disable_plaintext_auth = no` is required so Thunderbird can authenticate using the standard `PLAIN` mechanism over the STARTTLS-encrypted connection. Without this line, Dovecot will reject all logins from Thunderbird.

---

### Step 6: Create a Local Test User

```bash
sudo useradd -m mailuser1
sudo passwd mailuser1
```

Manually initialise the Maildir folder structure so Dovecot can deliver mail immediately:

```bash
sudo mkdir -p /home/mailuser1/Maildir/{new,cur,tmp}
sudo chown -R mailuser1:mailuser1 /home/mailuser1/Maildir
```

---

### Step 7: Firewall & Service Activation

```bash
# Open ports: SMTP (25), Submission (587), IMAP (143)
sudo firewall-cmd --permanent --add-service=smtp
sudo firewall-cmd --permanent --add-service=smtp-submission
sudo firewall-cmd --permanent --add-service=imap
sudo firewall-cmd --reload

# Enable and start both services
sudo systemctl enable --now postfix dovecot

# Verify both services are running (look for "active (running)")
sudo systemctl status postfix
sudo systemctl status dovecot
```

---

## Part 2: Ubuntu Client Configuration

### Step 1: Install Thunderbird

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install thunderbird -y
```

---

### Step 2: Add DNS server

```bash
sudo nano /etc/hosts
```
 
Add:
 
```
192.168.200.3   server.kawkaw.com
```
 
Alternatively, use the IP address `192.168.200.3` directly in the Thunderbird server hostname fields instead of the hostname. Both approaches work identically.

---

### Step 3: Configure Account in Thunderbird

1. Launch **Thunderbird**.
2. Go to **Account Setup** and enter:
   - **Full Name:** `Mail User 1`
   - **Email:** `mailuser1@kawkaw.com`
   - **Password:** *(the password set with `passwd mailuser1`)*
3. Click **Configure Manually** and enter the following settings:

**Incoming Mail (IMAP):**

| Field | Value |
|---|---|
| Protocol | IMAP |
| Server Hostname | `192.168.200.3` |
| Port | `143` |
| Connection Security | `STARTTLS` |
| Authentication | `Normal password` |
| Username | `mailuser1` |

**Outgoing Mail (SMTP):**

| Field | Value |
|---|---|
| Server Hostname | `192.168.200.3` |
| Port | `587` |
| Connection Security | `STARTTLS` |
| Authentication | `Normal password` |
| Username | `mailuser1` |

4. Click **Done** (or **Re-test** first to confirm connectivity).

---

### Step 4: Accept the Self-Signed Certificate

Because the certificate is self-signed, Thunderbird will show an **"Unknown Certificate"** security warning.

1. Click **View Certificate** and confirm the Common Name is `server.kawkaw.com`.
2. Check **"Always trust this certificate in future sessions."**
3. Click **Confirm Security Exception**.

> You may see this prompt **twice** — once for the IMAP connection and once for the SMTP connection. Accept **both**.

---

### Step 5: Validation

**Send a test email:**

1. Click **Write** in Thunderbird.
2. Set the recipient to `mailuser1@kawkaw.com`.
3. Enter any subject and body text.
4. Click **Send**.

**Receive the test email:**

1. Right-click **Inbox** in the left panel and select **Get Messages**.
2. Confirm the sent email appears in the Inbox.

**Verify TLS is active on the server (recommended):**

```bash
# On the Rocky server — tail the mail log and look for TLS confirmation
sudo tail -f /var/log/maillog
```

A successful TLS connection will show a line containing `TLS connection established`.

---

## Summary of Configured Ports

| Service | Port | Security |
|---|---|---|
| Postfix SMTP | 25 | STARTTLS |
| Postfix Submission | 587 | STARTTLS (Encrypted) |
| Dovecot IMAP | 143 | STARTTLS |

---

## Troubleshooting

### SELinux Blocking Certificate Access

**When to apply this fix:** Rocky Linux runs SELinux in enforcing mode by default. Even if your Unix file permissions (`chmod`/`chown`) are correct, SELinux maintains a separate layer of access control based on file labels. If Postfix or Dovecot were given the wrong SELinux label when the certificate files were created, the services will be blocked from reading them and will fail to start.

**How to spot the problem — check the mail log:**

```bash
sudo tail -50 /var/log/maillog
```

Look for any of these error messages:

```
# Postfix cannot read the private key:
postfix/smtpd: cannot load certificate authority data: disabling TLS support
postfix/smtpd: warning: cannot get RSA private key from file /etc/ssl/mail/mailserver.key: disabling TLS support

# Dovecot cannot read the certificate or key:
dovecot: ssl-params: Fatal: Failed to read file /etc/ssl/mail/mailserver.key: Permission denied
dovecot: master: Error: service(imap-login) initialization failed, waited 0 secs
dovecot: imap-login: Fatal: Failed to initialize SSL server context
```

**How to spot the problem — check SELinux audit log:**

```bash
sudo cat /var/log/audit/audit.log | grep denied | grep ssl
```

If SELinux is the cause, you will see lines like:

```
type=AVC msg=audit(...): avc: denied { read } for pid=... comm="smtpd"
name="mailserver.key" dev="..." ino=...
scontext=system_u:system_r:postfix_smtpd_t:s0
tcontext=system_u:object_r:user_home_t:s0 tclass=file
```

The key part is `tcontext=...user_home_t` — this means the file has the wrong label (`user_home_t`) instead of the correct one (`cert_t`).

**The fix:**

```bash
sudo restorecon -Rv /etc/ssl/mail/
```

This resets the SELinux labels on all files inside `/etc/ssl/mail/` to the correct policy context (`cert_t`), which Postfix and Dovecot are permitted to read.

**Verify the fix worked:**

```bash
ls -Z /etc/ssl/mail/
```

Expected output — both files must show `cert_t`:

```
system_u:object_r:cert_t:s0 mailserver.crt
system_u:object_r:cert_t:s0 mailserver.key
```

Then restart both services:

```bash
sudo systemctl restart postfix dovecot
sudo systemctl status postfix dovecot
```

Check the mail log once more to confirm the TLS errors are gone:

```bash
sudo tail -20 /var/log/maillog
```

---

## References

- Red Hat System Administration (RH124)
- Postfix Documentation: https://www.postfix.org/documentation.html
- Dovecot Documentation: https://doc.dovecot.org/