# Question 4: Containerised Web Application Stack (Podman + Flask + MariaDB + Cockpit)

## Objective

Deploy a containerised dynamic web application stack on the Rocky Linux server using **Podman**. The stack consists of a **Flask** (Python) web application with a **user registration and login system**, storing credentials securely in a **MariaDB** database. Both services run inside a shared Podman Pod. **Cockpit** provides a web-based GUI for real-time container monitoring. The Ubuntu client interacts with the application through a browser, demonstrating full client-server-database operation.

---

## Justification for Selection

| Justification | Detail |
|---|---|
| **Not in excluded list** | Podman, Flask, MariaDB, and Cockpit are not among the prohibited services |
| **Industry relevance** | Container-based deployment is the dominant pattern in modern DevOps and cloud infrastructure |
| **Security** | Podman is daemonless — superior security model. Passwords are hashed using pbkdf2:sha256, never stored in plaintext |
| **Demonstrability** | Client registers a user → credentials stored in DB → client logs in → server retrieves and verifies from DB |
| **Research depth** | Combines four distinct technologies into one cohesive, production-relevant architecture |

---

## Lab Environment

| Role | Hostname | IP Address |
|---|---|---|
| Rocky Linux Server | `server.kawkaw.com` | `192.168.200.3` |
| Ubuntu Client | `client.kawkaw.com` | `192.168.200.4` |

---

## Architecture Overview

```
Ubuntu Client (192.168.200.4)
        |
        |-- http://192.168.200.3:8080  -->  Flask App (port 5000 inside Pod)
        |                                        |
        |                                        +--> MariaDB (localhost inside Pod)
        |                                              Table: users (id, username, password_hash)
        |
        |-- https://192.168.200.3:9090 -->  Cockpit Dashboard (container monitoring)
```

All containers run inside a single **Podman Pod** named `web-stack`. Containers within the same Pod share a network namespace — Flask connects to MariaDB via `localhost` with no extra networking needed.

> **Port note:** Port `8080` is used instead of `80` to avoid conflict with the Apache web server configured in Question 3.

---

## Part 1: Rocky Linux Server Configuration

### Step 1: Verify Operating System & Update

```bash
cat /etc/os-release
sudo dnf update -y
```

---

### Step 2: Install Cockpit

Cockpit provides the web-based monitoring dashboard for managing containers and viewing logs.

```bash
# Install Cockpit and the Podman container management plugin
sudo dnf install cockpit cockpit-podman -y

# Enable and start the Cockpit socket
sudo systemctl enable --now cockpit.socket

# Open port 9090 in the firewall
sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --reload

# Verify it is running
sudo systemctl status cockpit.socket
```

---

### Step 3: Install Podman

Podman is the container runtime. On Rocky Linux it is the native, preferred container tool.

```bash
# Install Podman
sudo dnf install podman -y

# Verify installation
podman --version
```

---

### Step 4: Create the Pod

A **Pod** groups containers so they share the same network namespace. Flask and MariaDB inside the same Pod can reach each other over `localhost`.

```bash
# Map host port 8080 to internal pod port 5000 (where Flask listens)
podman pod create --name web-stack -p 8080:5000
```

**Verify the pod was created:**

```bash
podman pod list
```

Expected output:

```
POD ID        NAME       STATUS    INFRA ID
xxxxxxxxxxxx  web-stack  Created   xxxxxxxxxxxx
```

---

### Step 5: Deploy the MariaDB Container

```bash
podman run -d --pod web-stack \
  --name db-server-container \
  -e MARIADB_ROOT_PASSWORD=apu_admin_123 \
  -e MARIADB_DATABASE=sna_project \
  -v mariadb_data:/var/lib/mysql:Z \
  mariadb:latest
```

> The `:Z` flag on the volume mount sets the correct SELinux label so the container can write to that storage location. Do not omit it.

**Wait for MariaDB to be ready before continuing:**

```bash
podman logs -f db-server-container
```

Press `Ctrl+C` once you see:

```
[Note] mariadbd: ready for connections.
```

This takes approximately 15–20 seconds on first run.

---

### Step 6: Build the Flask Application

#### 6a — Create the application directory

```bash
mkdir -p ~/flask-app
```

#### 6b — Create `app.py`

```bash
nano ~/flask-app/app.py
```

Paste the following content exactly:

```python
from flask import Flask, request, redirect, url_for
from werkzeug.security import generate_password_hash, check_password_hash
import mysql.connector
import time

app = Flask(__name__)

# ── Database helpers ────────────────────────────────────────────────────────

def get_db():
    """Connect to MariaDB with retries (MariaDB needs time to initialise)."""
    for _ in range(10):
        try:
            conn = mysql.connector.connect(
                host="localhost",
                user="root",
                password="apu_admin_123",
                database="sna_project"
            )
            return conn
        except Exception:
            time.sleep(3)
    return None


def init_db():
    """Create the users table if it does not already exist."""
    conn = get_db()
    if conn:
        cursor = conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id         INT AUTO_INCREMENT PRIMARY KEY,
                username   VARCHAR(50)  UNIQUE NOT NULL,
                password   VARCHAR(255) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        conn.commit()
        cursor.close()
        conn.close()


# Initialise the table on first request
_db_initialised = False

@app.before_request
def setup():
    global _db_initialised
    if not _db_initialised:
        init_db()
        _db_initialised = True


# ── Shared page style ────────────────────────────────────────────────────────

STYLE = """
<style>
  body { font-family: Arial, sans-serif; background: #f0f2f5;
         display: flex; justify-content: center; padding-top: 60px; }
  .card { background: white; padding: 32px 40px; border-radius: 10px;
          box-shadow: 0 2px 12px rgba(0,0,0,0.12); width: 340px; }
  h2 { text-align: center; margin-bottom: 24px; color: #333; }
  input { width: 100%; padding: 10px; margin: 8px 0 16px;
          border: 1px solid #ccc; border-radius: 6px; box-sizing: border-box; }
  button { width: 100%; padding: 11px; background: #1877f2; color: white;
           border: none; border-radius: 6px; font-size: 15px; cursor: pointer; }
  button:hover { background: #155fcb; }
  .link { text-align: center; margin-top: 16px; font-size: 14px; }
  .link a { color: #1877f2; text-decoration: none; }
  .msg-ok  { color: green;  text-align: center; margin-bottom: 12px; }
  .msg-err { color: red;    text-align: center; margin-bottom: 12px; }
  table { width: 100%; border-collapse: collapse; margin-top: 16px; }
  th, td { padding: 10px; border: 1px solid #ddd; text-align: left; }
  th { background: #f5f5f5; }
</style>
"""


# ── Routes ───────────────────────────────────────────────────────────────────

@app.route('/')
def index():
    return redirect(url_for('login'))


@app.route('/register', methods=['GET', 'POST'])
def register():
    message = ''
    msg_class = ''

    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()

        if not username or not password:
            message = 'Username and password are required.'
            msg_class = 'msg-err'
        else:
            hashed = generate_password_hash(password)
            conn = get_db()
            if not conn:
                message = 'Database not available. Please try again shortly.'
                msg_class = 'msg-err'
            else:
                try:
                    cursor = conn.cursor()
                    cursor.execute(
                        "INSERT INTO users (username, password) VALUES (%s, %s)",
                        (username, hashed)
                    )
                    conn.commit()
                    cursor.close()
                    conn.close()
                    message = f'User "{username}" registered successfully!'
                    msg_class = 'msg-ok'
                except mysql.connector.errors.IntegrityError:
                    message = f'Username "{username}" is already taken.'
                    msg_class = 'msg-err'
                    conn.close()

    return f"""
    <!DOCTYPE html><html><head><title>Register</title>{STYLE}</head><body>
    <div class="card">
      <h2>Register</h2>
      <p class="{msg_class}">{message}</p>
      <form method="POST">
        <label>Username</label>
        <input type="text" name="username" required>
        <label>Password</label>
        <input type="password" name="password" required>
        <button type="submit">Create Account</button>
      </form>
      <div class="link">Already have an account? <a href="/login">Login</a></div>
    </div>
    </body></html>
    """


@app.route('/login', methods=['GET', 'POST'])
def login():
    message = ''
    msg_class = ''

    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()

        conn = get_db()
        if not conn:
            message = 'Database not available. Please try again shortly.'
            msg_class = 'msg-err'
        else:
            cursor = conn.cursor()
            cursor.execute(
                "SELECT password FROM users WHERE username = %s", (username,)
            )
            row = cursor.fetchone()
            cursor.close()
            conn.close()

            if row and check_password_hash(row[0], password):
                return f"""
                <!DOCTYPE html><html><head><title>Welcome</title>{STYLE}</head><body>
                <div class="card">
                  <h2>&#10003; Login Successful</h2>
                  <p class="msg-ok">Welcome, <strong>{username}</strong>!</p>
                  <p style="text-align:center;font-size:14px;color:#555;">
                    Your credentials were verified against the MariaDB database
                    running inside the Podman pod on <strong>server.kawkaw.com</strong>.
                  </p>
                  <div class="link"><a href="/login">Back to Login</a> &nbsp;|&nbsp;
                  <a href="/users">View All Users</a></div>
                </div>
                </body></html>
                """
            else:
                message = 'Invalid username or password.'
                msg_class = 'msg-err'

    return f"""
    <!DOCTYPE html><html><head><title>Login</title>{STYLE}</head><body>
    <div class="card">
      <h2>Login</h2>
      <p class="{msg_class}">{message}</p>
      <form method="POST">
        <label>Username</label>
        <input type="text" name="username" required>
        <label>Password</label>
        <input type="password" name="password" required>
        <button type="submit">Login</button>
      </form>
      <div class="link">No account? <a href="/register">Register</a></div>
    </div>
    </body></html>
    """


@app.route('/users')
def users():
    """Show all registered users — proof of database storage for the demo."""
    conn = get_db()
    if not conn:
        return "<p>Database not available.</p>"

    cursor = conn.cursor()
    cursor.execute("SELECT id, username, created_at FROM users ORDER BY id")
    rows = cursor.fetchall()
    cursor.close()
    conn.close()

    rows_html = ''.join(
        f"<tr><td>{r[0]}</td><td>{r[1]}</td><td>{r[2]}</td></tr>"
        for r in rows
    )

    return f"""
    <!DOCTYPE html><html><head><title>Registered Users</title>{STYLE}</head><body>
    <div class="card" style="width:500px;">
      <h2>Registered Users</h2>
      <p style="font-size:13px;color:#555;text-align:center;">
        Stored in MariaDB &mdash; <code>sna_project.users</code>
      </p>
      <table>
        <tr><th>ID</th><th>Username</th><th>Registered At</th></tr>
        {rows_html if rows_html else '<tr><td colspan="3">No users yet.</td></tr>'}
      </table>
      <div class="link" style="margin-top:20px;">
        <a href="/register">Register</a> &nbsp;|&nbsp; <a href="/login">Login</a>
      </div>
    </div>
    </body></html>
    """


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

#### 6c — Create the `Containerfile`

```bash
nano ~/flask-app/Containerfile
```

Paste the following:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install flask mysql-connector-python

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

> `werkzeug` (used for password hashing) is automatically installed as part of Flask — no separate install needed.

#### 6d — Build the image

```bash
podman build -t flask-app ~/flask-app/
```

**Verify the image exists:**

```bash
podman images
```

You should see `localhost/flask-app` in the list.

---

### Step 7: Deploy the Flask Container

```bash
podman run -d --pod web-stack \
  --name flask-app-container \
  flask-app
```

**Verify both containers are running:**

```bash
podman ps --pod
```

Both `db-server-container` and `flask-app-container` should show status `Up`.

---

### Step 8: Open the Firewall

```bash
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

**Test locally on the server:**

```bash
curl -s http://localhost:8080/register | grep -o "<title>.*</title>"
```

Expected: `<title>Register</title>`

---

## Part 2: Ubuntu Client Demonstration

### Step 1: Register a New User

Open **Firefox** on the Ubuntu client and go to:

```
http://192.168.200.3:8080/register

or

http://server.kawkaw.com:8080/register
```

1. Enter a **Username** (e.g. `student1`) and a **Password**.
2. Click **Create Account**.
3. You will see: **User "student1" registered successfully!**

> What happened on the server: Flask received the form, hashed the password using `pbkdf2:sha256`, and inserted the record into the `users` table in MariaDB.

---

### Step 2: Login with the Registered Credentials

Navigate to:

```
http://192.168.200.3:8080/login

or

http://server.kawkaw.com:8080/login
```

1. Enter the same username and password used during registration.
2. Click **Login**.
3. You will see the **Login Successful** page confirming the credentials were verified against the MariaDB database.

Try logging in with a **wrong password** to confirm rejection — you will see: `Invalid username or password.`

---

### Step 3: View All Stored Users (Proof of DB Storage)

Navigate to:

```
http://192.168.200.3:8080/users

or

http://server.kawkaw.com:8080/users
```

This page queries the `sna_project.users` table directly and displays every registered user with their ID and registration timestamp. The password column is intentionally excluded — only the hashed value is stored in the database, never the plaintext password.

This page serves as direct proof that data entered on the client is being persisted in the server-side database.

---

### Step 4: Access Cockpit for Container Monitoring

Open Firefox and go to:

```
https://192.168.200.3:9090

Or

https://server.kawkaw.com:9090
```

1. Accept the self-signed certificate warning (**Advanced → Accept the Risk and Continue**).
2. Log in with the Rocky Linux server credentials.
3. In the left sidebar, click **Podman containers**.
4. Both `db-server-container` and `flask-app-container` will be visible as **Running** inside the `web-stack` pod.
5. Click `flask-app-container` → **Logs** to view live application output including each login and register request.

---

## Summary of Application Routes

| URL | Method | Action |
|---|---|---|
| `/register` | GET | Show registration form |
| `/register` | POST | Insert new user into MariaDB (hashed password) |
| `/login` | GET | Show login form |
| `/login` | POST | Verify credentials against MariaDB |
| `/users` | GET | Display all registered users from MariaDB |

## Summary of Configured Services

| Service | Port | Access From Client | Purpose |
|---|---|---|---|
| Flask Web App | 8080 | `http://192.168.200.3:8080` | Registration, login, user listing |
| Cockpit Dashboard | 9090 | `https://192.168.200.3:9090` | Container monitoring GUI |
| MariaDB (internal) | 3306 | Not exposed externally | User credential storage (pod-internal only) |

---

## After a VM Reboot

Podman containers and pods do **not** restart automatically after a reboot unless configured to do so. This is expected behaviour — it is not an error.

**To manually restart the stack after a reboot:**

```bash
podman pod start web-stack
```

Then verify both containers are running again:

```bash
podman ps --pod
```

**Optional — configure auto-start on boot:**

If you want the pod to start automatically every time the server boots, generate a systemd service unit for it:

```bash
podman generate systemd --new --name web-stack \
  | sudo tee /etc/systemd/system/pod-web-stack.service

sudo systemctl daemon-reload
sudo systemctl enable --now pod-web-stack.service
```

Verify it is enabled:

```bash
sudo systemctl status pod-web-stack.service
```

> This is recommended if you plan to reboot the server between lab sessions or before your demo. With auto-start enabled, the full stack comes up on its own without any manual intervention after the VM is powered on.

---

## Troubleshooting

### Registration Returns "Database not available"

MariaDB may still be initialising. Wait 20 seconds and refresh. Check its status:

```bash
podman logs db-server-container | tail -20
```

Wait for `ready for connections` to appear before using the app.

---

### Flask Container Exits Immediately

Check the logs:

```bash
podman logs flask-app-container
```

If you see an image error, the build in Step 6d did not complete. Re-run:

```bash
podman build -t flask-app ~/flask-app/
podman run -d --pod web-stack --name flask-app-container flask-app
```

---

### SELinux Blocking MariaDB Volume Write

**Symptom:** `podman logs db-server-container` shows permission errors on `/var/lib/mysql`.

**Fix:** The `:Z` flag must be present on the volume mount. Remove and recreate:

```bash
podman stop db-server-container && podman rm db-server-container
podman volume rm mariadb_data
podman run -d --pod web-stack \
  --name db-server-container \
  -e MARIADB_ROOT_PASSWORD=apu_admin_123 \
  -e MARIADB_DATABASE=sna_project \
  -v mariadb_data:/var/lib/mysql:Z \
  mariadb:latest
```

---

### Cannot Reach Port 8080 from Ubuntu Client

```bash
sudo firewall-cmd --list-ports
```

If `8080/tcp` is missing:

```bash
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

---

### Cockpit Podman Tab is Empty

```bash
sudo dnf install cockpit-podman -y
sudo systemctl restart cockpit.socket
```

Refresh the browser.

---

## Appendix: Podman Quick Reference Cheat Sheet

Use these commands on the Rocky Linux server to manage your application stack during the demo.

### 1. Status & Verification
*   **List all Pods**: `podman pod ls`
*   **List all Containers**: `podman ps -a`
*   **View containers organized by Pod**: `podman ps --pod`

### 2. Managing the Entire Stack (Recommended)
Starting or stopping the Pod automatically handles all containers inside it.
*   **Start everything**: `podman pod start web-stack`
*   **Stop everything**: `podman pod stop web-stack`
*   **Restart everything**: `podman pod restart web-stack`

### 3. Managing Individual Containers
*   **Start Flask only**: `podman start flask-app-container`
*   **Stop Flask only**: `podman stop flask-app-container`
*   **Start MariaDB only**: `podman start db-server-container`
*   **Stop MariaDB only**: `podman stop db-server-container`

### 4. Monitoring & Troubleshooting
*   **Follow live Flask logs**: `podman logs -f flask-app-container`
*   **Follow live MariaDB logs**: `podman logs -f db-server-container`
*   **Inspect Pod details**: `podman pod inspect web-stack`

### 5. Emergency Cleanup (Deletes everything)
*   **Remove Pod and Containers**: `podman pod rm -f web-stack`
*   **Remove Volume**: `podman volume rm mariadb_data`

---

## References

- Podman Documentation: https://docs.podman.io/
- Cockpit Project: https://cockpit-project.org/
- Flask Documentation: https://flask.palletsprojects.com/
- Werkzeug Security Utilities: https://werkzeug.palletsprojects.com/en/stable/utils/
- MariaDB Container Image: https://hub.docker.com/_/mariadb