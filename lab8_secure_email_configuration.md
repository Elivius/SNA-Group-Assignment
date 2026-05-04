# Question 2: Secure Email Configuration (SMTP/IMAP)

## Objective

Secure the email service using encryption (TLS/SSL) and verify that communications between the Rocky Server and Ubuntu Client are protected.

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

# Install Postfix (MTA) and Dovecot (MDA)
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

> **Note:** When prompted, enter your organisation details:
> - **Country:** MY
> - **Locality:** KL
> - **Common Name:** `server.kawkaw.com`

---

### Step 4: Configure Postfix (SMTP)

Edit the main configuration file to enable the server to route mail and use SSL/TLS.

```bash
sudo nano /etc/postfix/main.cf
```

Add or modify the following lines:

```ini
myhostname = server.kawkaw.com
mydomain = kawkaw.com
inet_interfaces = all
home_mailbox = Maildir/
smtpd_tls_cert_file = /etc/ssl/mail/mailserver.crt
smtpd_tls_key_file = /etc/ssl/mail/mailserver.key
smtpd_use_tls = yes
```

---

### Step 5: Configure Dovecot (IMAP)

We must tell Dovecot where the mailbox is and point it to our new certificates.

**Set Mailbox Location:**

```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

```ini
mail_location = maildir:~/Maildir
```

**Enable SSL:**

```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

```ini
ssl = yes
ssl_cert = </etc/ssl/mail/mailserver.crt
ssl_key = </etc/ssl/mail/mailserver.key
```

---

### Step 6: Firewall & Service Activation

Open the ports for SMTP (25), Submission (587), and IMAP (143).

```bash
# Add firewall rules
sudo firewall-cmd --permanent --add-service={smtp,smtp-submission,imap}
sudo firewall-cmd --reload

# Start and enable services
sudo systemctl enable --now postfix dovecot
```

---

### Step 7: Create a Local Test User

```bash
sudo useradd -m mailuser1
sudo passwd mailuser1
```

---

## Part 2: Ubuntu Client Configuration

### Step 1: Install Thunderbird

Update the client and install the graphical mail client.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install thunderbird -y
```

---

### Step 2: Configure Account via GUI

1. Launch **Thunderbird**.
2. Go to **Account Setup** and enter:
   - **Email:** `mailuser1@kawkaw.com`
   - **Password:** *(Your user password)*
3. Click **Configure Manually** and set the following:

| Setting | Value |
|---|---|
| Protocol | IMAP |
| IMAP Port | 143 |
| IMAP Security | STARTTLS |
| SMTP Port | 587 |
| SMTP Security | STARTTLS |
| Hostname | `server.kawkaw.com` |

---

### Step 3: Handle Security Exception

Because the certificate is self-signed (not from an authorised body), Thunderbird will display an **"Unknown Certificate"** warning.

1. Review the certificate details and verify it matches your Rocky Server.
2. Select **"Always trust this certificate in future sessions."**
3. Click **Confirm Security Exception**.

---

### Step 4: Validation

| Action | Details |
|---|---|
| **Send** | Compose an email to `mailuser1@kawkaw.com` |
| **Receive** | Click **Get Messages** and confirm the email appears in the Inbox |

---

## References

- Red Hat System Administration (RH124)
- Lab 7b\_FTP Server - SSL-TLS\_V4.pdf
