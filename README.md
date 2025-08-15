# Deploy a Dockerized Spring Boot Microservice to AWS (Free‑Tier)

> Works for any containerized Spring Boot service (including an OAuth2 **Sign in with Google** auth service).  
> Two ways to run:
> - **Path A (Public HTTPS)** — EC2 + free DuckDNS subdomain + Let’s Encrypt on the same EC2.
> - **Path B (EC2‑only)** — EC2 + SSH tunnel using a **localhost** redirect (no domain, great for demos).

---

## 0) What you’ll build

- **One Free‑Tier EC2 instance** (t2.micro/t3.micro) running your **Dockerized** Spring Boot microservice.
- Optional **Nginx** reverse proxy + **Let’s Encrypt** TLS cert (Path A).
- No NAT Gateway, no Load Balancer, no managed databases — all chosen to stay Free‑Tier friendly.

> If your app uses Google Sign‑In: Google requires **HTTPS** and **a real domain** for public users. The **only** exception is `localhost` (Path B with SSH tunnel).

---

## 1) Prerequisites

- An AWS account with Free‑Tier enabled.
- Git, Docker installed on your laptop.
- (If using Google Sign‑In) A Google Cloud project with an **OAuth 2.0 Client (Web)**.
- Your app containerized with a `Dockerfile`. Example ports:
  - Generic microservice: `8080`
  - Sample auth service: `9080`

**Create a Budget (free):**
- AWS Console → **Billing** → **Budgets** → Cost budget → Threshold `$1` with email alerts.

---

## 2) Free‑Tier guardrails (read this first)

- Run **one** micro instance: **t2.micro** (or **t3.micro** if t2 isn’t available).
- Keep EBS disk size ≤ **30 GB**.
- Don’t create a **NAT Gateway** (hourly + data fees) or a **Load Balancer** (hourly + LCU).  
- When idle: **Stop** the instance (compute charges stop; small EBS storage remains). For $0: **Terminate** and delete leftover volumes/IPs/snapshots.

---

## 3) Launch a Free‑Tier EC2 instance

1. Console → **EC2 → Launch instance**
2. Name: `my-microservice`
3. **AMI**: Amazon Linux 2023
4. **Instance type**: `t2.micro` (or `t3.micro`)
5. **Storage**: 20–30 GB **gp3**
6. **Network**: Default VPC, **Auto‑assign public IP: Enabled**
7. **Security Group**:
   - **22/tcp** from *your IP only* (SSH)
   - If using HTTPS (Path A): **80/tcp** and **443/tcp** from `0.0.0.0/0`
8. Create/download a **key pair** (PEM). Launch the instance.

---

## 4) Connect & install Docker (on the EC2)

SSH (EC2 Instance Connect or your key):
```bash
sudo dnf -y update
sudo dnf -y install docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
newgrp docker
docker --version
```

(Optional) install Git and Java if you’ll build on the VM:
```bash
sudo dnf -y install git
```

---

## 5) Put your container on the instance

**Option A — Build on the VM**
```bash
git clone <YOUR_REPO_URL> app
cd app
docker build -t my-svc:latest .
```

**Option B — Pull from your registry (e.g., ECR)**
```bash
# after 'aws ecr get-login-password | docker login ...'
docker pull <AWS_ACCOUNT_ID>.dkr.ecr.<region>.amazonaws.com/my-svc:latest
```

**Run your container** (bind to loopback if using Nginx in front):
```bash
# choose the port your app listens on (8080 or 9080)
APP_PORT=8080          # or 9080 for your auth service

# create an .env file (DO NOT commit secrets)
cat > .env <<'ENVVARS'
# Example for Google OAuth (if you use it)
GOOGLE_CLIENT_ID=replace_me
GOOGLE_CLIENT_SECRET=replace_me
# If you know your public HTTPS domain (Path A), set explicit redirect:
# SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_GOOGLE_REDIRECT_URI=https://<your-domain>/login/oauth2/code/google
ENVVARS

# run the container (loopback only — public access goes through Nginx)
docker run -d --name my-svc \
  --env-file .env \
  -p 127.0.0.1:${APP_PORT}:${APP_PORT} \
  my-svc:latest
```

---

## 6) Path A — Public HTTPS with DuckDNS + Let’s Encrypt (for Google Sign‑In in public)

Google requires **HTTPS** + a real host (no raw public IP) for public web redirects. This path gives you a **free subdomain** and a **free TLS cert**.

### 6.1 Get a free DuckDNS subdomain
1. Create an account at **duckdns.org** and add a domain, e.g., `myteam.duckdns.org`.
2. Point it to your EC2 **public IPv4**.
3. **Note on limits:** DuckDNS commonly allows **up to five subdomains per account**; verify the current quota in your DuckDNS dashboard (policies can change).

### 6.2 Install Nginx and obtain a TLS certificate
```bash
sudo dnf -y install nginx python3 augeas-libs
# Install certbot in a venv (recommended on AL2023)
sudo python3 -m venv /opt/certbot
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-nginx
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot

# Temporary HTTP server block (for the ACME challenge)
sudo tee /etc/nginx/conf.d/default.conf >/dev/null <<'NGINX'
server {
  listen 80;
  server_name myteam.duckdns.org;
  location / { return 200 "ok"; add_header Content-Type text/plain; }
}
NGINX
sudo systemctl enable --now nginx

# Issue the cert and enable HTTPS + 80→443 redirect
sudo certbot --nginx -d myteam.duckdns.org --agree-tos -m you@example.com --redirect
```

### 6.3 Reverse‑proxy to your container (keeps only 443 public)
```bash
APP_PORT=8080  # or 9080 for auth service

sudo tee /etc/nginx/conf.d/app.conf >/dev/null <<NGINX
server {
  listen 80;
  server_name myteam.duckdns.org;
  return 301 https://\$host\$request_uri;
}
server {
  listen 443 ssl http2;
  server_name myteam.duckdns.org;

  ssl_certificate     /etc/letsencrypt/live/myteam.duckdns.org/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/myteam.duckdns.org/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:${APP_PORT};
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
  }
}
NGINX

sudo nginx -t && sudo systemctl reload nginx
```

**Spring Boot tip:** add this so generated redirects honor HTTPS behind Nginx:
```properties
# application.properties (or equivalent YAML setting)
server.forward-headers-strategy=framework
```

### 6.4 Configure Google OAuth (only if your service uses it)
In Google Cloud Console → **Credentials → OAuth client (Web)**:
- **Authorized JavaScript origin**: `https://myteam.duckdns.org`
- **Authorized redirect URI**: `https://myteam.duckdns.org/login/oauth2/code/google`

Update your app (env or config) with the same redirect URI.

---

## 7) Path B — EC2‑only (no domain), use a localhost redirect via SSH tunnel

For demos with Google Sign‑In but **no domain**:
1. In Google Console, set redirect to  
   `http://127.0.0.1:9080/login/oauth2/code/google` (or `localhost`).
2. On EC2, run the app bound to loopback (as above).
3. From your laptop, open a tunnel:
```bash
# replace key path and EC2 public IP
ssh -i /path/to/key.pem -N -L 9080:127.0.0.1:9080 ec2-user@<EC2_PUBLIC_IP>
```
4. Browse to `http://127.0.0.1:9080` and log in.  
   (Google allows localhost/127.0.0.1 over HTTP; everything else needs HTTPS.)

---

## 8) Verify

- `docker ps` shows your container healthy.
- If using Nginx/TLS: `curl -I https://myteam.duckdns.org` returns `200`/`301`.
- For OAuth: check the `redirect_uri` param in the request to Google — it must **exactly match** what you registered (scheme, host, port, path).

**Common error**: `Error 400: redirect_uri_mismatch`  
Fix by making the three places identical: Google Console, your app’s configured redirect, and the URL you actually use in the browser.

---

## 9) Optional Free‑Tier CI/CD

- **ECR**: push images to a private repo (fits small images under the free storage).  
- **CodeBuild**: 100 build minutes/month (free tier that doesn’t expire).  
- **CodePipeline**: a small pipeline can be kept within free allowances; or just script `ssh` + `docker pull` + `docker compose up -d`.

---

## 10) How not to get charged

- Stick to **one** micro instance (t2.micro/t3.micro).
- Keep EBS ≤ **30 GB**; prune Docker images occasionally (`docker system prune`).
- Don’t create **NAT Gateways** or **Load Balancers** for this exercise.
- Keep **Budgets** email alerts on.
- Stop the instance when idle; **Terminate** + delete volumes/IPs when done.

---

## 11) Clean‑up to $0

1. **Backup data** you need (env files, uploads).  
2. EC2 → **Instances** → select → **Terminate**.  
3. EC2 → **Volumes** → delete any **available** (orphaned) volumes.  
4. EC2 → **Elastic IPs** → if allocated, **Disassociate** then **Release**.  
5. Delete snapshots/AMIs you created.

---

## 12) FAQ

**Why no Load Balancer?**  
ALBs are excellent in production, but they incur hourly + usage costs. We keep everything on one instance to stay Free‑Tier.

**Multiple microservices on one EC2?**  
Yes. Run each container on a different local port; use Nginx to route by path or host. Keep to one public IP and no ALB.

**DuckDNS “five subdomains” — is it official?**  
DuckDNS commonly allows **up to five** subdomains per account. Treat it as a practical guideline and verify in your account dashboard; policies may change without notice.

**Google Sign‑In without a domain?**  
Use Path B (SSH tunnel) and a `localhost` redirect. For public users you’ll need Path A (HTTPS domain).

---

## Appendix — Quick commands (copy‑paste)

**Run container on EC2 (loopback only):**
```bash
APP_PORT=8080
docker run -d --name my-svc \
  --env-file .env \
  -p 127.0.0.1:${APP_PORT}:${APP_PORT} \
  my-svc:latest
```

**SSH tunnel from your laptop (for localhost redirect):**
```bash
ssh -i /path/to/key.pem -N -L 9080:127.0.0.1:9080 ec2-user@<EC2_PUBLIC_IP>
```

**Nginx reverse proxy (HTTPS public → local container):**
```bash
# after certbot issued the certificate
sudo tee /etc/nginx/conf.d/app.conf >/dev/null <<'NGINX'
server { listen 80; server_name myteam.duckdns.org; return 301 https://$host$request_uri; }
server {
  listen 443 ssl http2; server_name myteam.duckdns.org;
  ssl_certificate /etc/letsencrypt/live/myteam.duckdns.org/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/myteam.duckdns.org/privkey.pem;
  location / { proxy_pass http://127.0.0.1:8080; proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto https; }
}
NGINX
sudo nginx -t && sudo systemctl reload nginx
```
