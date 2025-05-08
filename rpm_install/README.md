# 🚀 Coder Server Installation on Fedora (Self-Hosted Setup)

This guide walks through setting up [Coder](https://coder.com/) on a Fedora server or VM, using a systemd service and HTTPS with a locally trusted TLS certificate.

---

## ✅ Prerequisites

- Fedora Linux 36 or later
- User with `sudo` privileges (assumes user is `parallels`)
- Static or accessible IP if accessing externally

---

## 🧰 Step 1: Install Docker

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
```

---

## 👤 Step 2: Add User to Docker Group

```bash
sudo usermod -aG docker parallels
newgrp docker
```

Log out and back in (or use `newgrp`) to activate group membership.

---

## 📦 Step 3: Install Coder (via official RPM)

```bash
curl -fsSL https://coder.com/install.sh | sh
```

This installs the latest Coder binary via RPM and places it at `/usr/bin/coder`.

---

## 🔐 Step 4: Generate a TLS Certificate (for HTTPS)

> You can use `mkcert`, Let's Encrypt, or any other cert. This example assumes a local cert in `/home/parallels/certs`.

If using `mkcert`:

```bash
# Install mkcert
sudo dnf install -y nss-tools
curl -L https://github.com/FiloSottile/mkcert/releases/latest/download/mkcert-v1.4.4-linux-amd64 -o mkcert
chmod +x mkcert && sudo mv mkcert /usr/local/bin/
mkcert -install

# Generate certs
mkdir -p /home/parallels/certs
cd /home/parallels/certs
mkcert coder.local 192.168.1.50
```

---

## 🧾 Step 5: Create systemd Service

```bash
sudo nano /etc/systemd/system/coder.service
```

Paste the following (adjust paths as needed):

```ini
[Unit]
Description=Coder Server
After=network.target

[Service]
User=parallels
WorkingDirectory=/home/parallels
ExecStart=/usr/bin/coder server --http-address 0.0.0.0:3000 --tls-address 0.0.0.0:3443 --tls-enable --tls-cert-file /home/parallels/certs/coder.local+1.pem --tls-key-file /home/parallels/certs/coder.local+1-key.pem --access-url https://coder.local
Restart=always
RestartSec=5
Environment=HOME=/home/parallels

[Install]
WantedBy=multi-user.target
```

Ensure the certificate files are readable by the `parallels` user:

```bash
sudo chown parallels:parallels /home/parallels/certs/*
chmod 600 /home/parallels/certs/coder.local+1-key.pem
chmod 644 /home/parallels/certs/coder.local+1.pem
```

---

## 🔁 Step 6: Reload systemd and Start Coder

```bash
sudo systemctl daemon-reload
sudo systemctl enable coder
sudo systemctl start coder
```

Check status:

```bash
sudo systemctl status coder
```

Tail logs:

```bash
journalctl -u coder -f
```

---
## Step 7: Open ports 3000 and 3443 in the firewall (for internal access)
```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=3443/tcp
sudo firewall-cmd --reload

---

## 🌐 Step 8: Access Coder

From a browser on your host or LAN:

```
https://coder.local:3443
```

If you're using a local domain like `coder.local`, make sure you add an entry to `/etc/hosts` on your host machine:

```
192.168.1.50  coder.local
```

---

## Exposing Coder Over HTTPS and Omitting HTTP

By default, Coder listens on both HTTP (port 80) and HTTPS (port 443/3443). To harden the environment and avoid exposure over HTTP:

### Disable HTTP (Port 80) Access

If you're not using port 80 for redirects, you can disable it entirely:

```bash
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
```

---

### Publishing Coder on a Private HTTPS Port (e.g., 3443)

There are two options to make Coder accessible via HTTPS while ensuring no traffic is allowed on HTTP:

---

### Option A: Use firewalld to Redirect Port 443 → 3443

This silently forwards traffic from port 443 to 3443 at the network layer:

```bash
sudo firewall-cmd --permanent --add-forward-port=port=443:proto=tcp:toport=3443
sudo firewall-cmd --reload
```

> ⚠️ This method does not issue a browser-friendly HTTPS redirect — it only forwards raw TCP.

---

### Option B: Use NGINX as a Secure Reverse Proxy

Install NGINX:

```bash
sudo dnf install nginx -y   # RHEL-based systems
# or
sudo apt install nginx -y   # Debian-based systems
```

Create `/etc/nginx/conf.d/coder.conf`:

```nginx
server {
    listen 80;
    server_name your.domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your.domain.com;

    ssl_certificate /etc/ssl/certs/your-cert.pem;
    ssl_certificate_key /etc/ssl/private/your-key.pem;

    location / {
        proxy_pass https://localhost:3443;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Enable and start NGINX:

```bash
sudo systemctl enable nginx --now
```

This setup provides:
- Automatic redirect from HTTP to HTTPS
- SSL termination at NGINX
- Clean, secure routing to Coder on port 3443


## 🧹 Optional Cleanup

To remove the service:

```bash
sudo systemctl stop coder
sudo systemctl disable coder
sudo rm /etc/systemd/system/coder.service
sudo systemctl daemon-reload
```

---

## 🛠 Troubleshooting

- Cert load errors? Check file paths and permissions
- Not picking up flags? Ensure `ExecStart` is on a **single line**
- Still seeing `127.0.0.1` binding? Add `--http-address` and `--tls-address` explicitly

---

## 📎 Resources

- [Coder Docs](https://coder.com/docs)
- [mkcert GitHub](https://github.com/FiloSottile/mkcert)
- [Docker Fedora Install](https://docs.docker.com/engine/install/fedora/)
