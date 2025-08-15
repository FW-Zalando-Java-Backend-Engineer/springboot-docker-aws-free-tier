# Deploy a Dockerized Spring Boot Microservice to AWS (Free-Tier)

> Works for any containerized Spring Boot service.
> Two paths for OAuth2/Google Sign-In:
> **A)** public HTTPS with a free DuckDNS subdomain + Let’s Encrypt, or
> **B)** EC2-only + SSH tunnel using `localhost` redirect (no domain).

---

## 1) Prerequisites

* An AWS account with **Free Tier** enabled. New customers typically get **750 hours/month of t2.micro** (or t3.micro in some regions) for **12 months** and other benefits. ([Amazon Web Services, Inc.][1])
* **EBS** Free Tier includes **30 GB** of SSD storage + **2M I/Os** + **1 GB** of snapshots. ([Amazon Web Services, Inc.][2])
* From Feb 1, 2024, the EC2 Free Tier also includes **750 hours/month of in-use public IPv4** for the first 12 months (so one instance with one public IPv4 is covered). ([Amazon Web Services, Inc.][3])
* Turn on **AWS Budgets** alerts (free) so you get an email if costs appear. Budgets without actions are **free**; the first **two action-enabled** budgets are free as well. ([Amazon Web Services, Inc.][4])

> Optional extras (still Free-Tier-friendly):
> **ECR**: 500 MB private image storage for 12 months. ([Amazon Web Services, Inc.][5])
> **CodeBuild**: 100 build minutes/month, and this free tier **does not expire** after 12 months. ([Amazon Web Services, Inc.][6])

---

## 2) Cost guardrails (read this!)

* Use **one** `t2.micro` (or `t3.micro`) only. ([Amazon Web Services, Inc.][1])
* Keep disk ≤ **30 GB**. ([Amazon Web Services, Inc.][2])
* Avoid **NAT Gateways** (hourly **and** per-GB fees). ([AWS Documentation][7], [Amazon Web Services, Inc.][8])
* Avoid **Load Balancers** (hourly + LCU usage). ([Amazon Web Services, Inc.][9])
* Stop or terminate the instance when not needed (cleanup steps at the end).

---

## 3) Launch a Free-Tier EC2 instance

1. Console → **EC2 → Launch instance**
2. Name: `my-microservice`
3. AMI: **Amazon Linux 2023**
4. Instance type: **t2.micro** (Free Tier). ([Amazon Web Services, Inc.][1])
5. Storage: **30 GB gp3** (fits EBS Free Tier). ([Amazon Web Services, Inc.][2])
6. Network: default VPC, **Auto-assign public IPv4: Enabled** (covered by the Free Tier in your first 12 months). ([Amazon Web Services, Inc.][3])
7. Security Group (inbound):

   * **22/tcp** from *your IP only* (for SSH).
   * If you’ll expose HTTP directly: **80/tcp** from `0.0.0.0/0`.
   * If you’ll use HTTPS with Nginx: **443/tcp** from `0.0.0.0/0`.
8. Create/download a **key pair**.

---

## 4) Connect & install Docker

SSH in (EC2 Instance Connect or your key):

```bash
sudo dnf -y update
sudo dnf -y install docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
newgrp docker
docker --version
```

---

## 5) Put your container on the instance

**Option 1 – Build on the VM**

```bash
# If your repo is private, use a deploy key or token
git clone <YOUR_REPO_URL> app
cd app
docker build -t my-svc:latest .
```

**Option 2 – Pull from ECR (optional)**
If you pushed images to **ECR** (500 MB/month free for 12 months), log in and `docker pull` your image. ([Amazon Web Services, Inc.][5])

Run it (bind to localhost if you’ll put Nginx in front):

```bash
# publish only to loopback; Nginx will face the internet
docker run -d --name my-svc \
  --env-file .env \
  -p 127.0.0.1:8080:8080 \
  my-svc:latest
```

---

## 6) Path A — Public HTTPS with DuckDNS + Let’s Encrypt (recommended for Google OAuth)

Google requires **HTTPS** for web app redirect URIs (except `localhost`) and **does not allow raw public IP hosts**. So if your service includes Google Sign-In, use a domain + TLS. ([Google for Developers][10])

### 6.1 Get a free DuckDNS subdomain

* Create an account at **duckdns.org** and add a domain, e.g. `myteam.duckdns.org`.
* Point it at your EC2 public IPv4 (visible in the EC2 console).
* **About the “5 subdomains” note:** DuckDNS doesn’t publish a formal limit page, but multiple community guides state **up to 5 domains per account**. Always check your DuckDNS dashboard for the current quota. ([swizzin.ltd][11], [MakeUseOf][12], [Medium][13])

### 6.2 Install Nginx and get a TLS cert

```bash
sudo dnf -y install nginx python3 augeas-libs
# Install certbot in a venv (AL2023)
sudo python3 -m venv /opt/certbot
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-nginx
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot

# Temporary HTTP server block (for the challenge)
sudo tee /etc/nginx/conf.d/default.conf >/dev/null <<'NGINX'
server {
  listen 80;
  server_name myteam.duckdns.org;
  location / { return 200 "ok"; add_header Content-Type text/plain; }
}
NGINX
sudo systemctl enable --now nginx

# Issue cert and enable HTTPS + redirect
sudo certbot --nginx -d myteam.duckdns.org --agree-tos -m you@example.com --redirect
```

(That sets up auto-renewal and a valid cert for `myteam.duckdns.org`.)

### 6.3 Reverse-proxy to your container

```bash
sudo tee /etc/nginx/conf.d/app.conf >/dev/null <<'NGINX'
server {
  listen 80;
  server_name myteam.duckdns.org;
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl http2;
  server_name myteam.duckdns.org;

  ssl_certificate     /etc/letsencrypt/live/myteam.duckdns.org/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/myteam.duckdns.org/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
  }
}
NGINX
sudo nginx -t && sudo systemctl reload nginx
```

> Spring tip: add `server.forward-headers-strategy=framework` so generated redirects use HTTPS when behind Nginx.

### 6.4 Google OAuth settings (only if your service uses Google Sign-In)

In Google Cloud Console → **Credentials → OAuth client (Web)** set:

* **Authorized JavaScript origin**: `https://myteam.duckdns.org`
* **Authorized redirect URI**: `https://myteam.duckdns.org/login/oauth2/code/google`

Google’s rules: **HTTPS required** (except localhost) and **no raw IP addresses**. ([Google for Developers][10])

---

## 7) Path B — EC2-only, no domain (for demos with Google)

If you want to use **only EC2** (no DNS, no Nginx), you can still use Google Sign-In by tunneling:

1. In Google Console, set the redirect to
   `http://127.0.0.1:9080/login/oauth2/code/google` (or `localhost`). ([Google for Developers][10])
2. Run your app on the VM bound to **127.0.0.1:9080**.
3. From your laptop, open an SSH tunnel:

   ```bash
   ssh -i /path/to/key.pem -N -L 9080:127.0.0.1:9080 ec2-user@<EC2_PUBLIC_IP>
   ```
4. Visit **[http://127.0.0.1:9080](http://127.0.0.1:9080)** locally and log in.
   (Google allows `localhost`/`127.0.0.1` over HTTP; everything else must be HTTPS.) ([Google for Developers][10])

---

## 8) Quick test checklist

* `docker ps` shows your container running.
* If using Nginx/TLS: `curl -I https://myteam.duckdns.org` returns `200`/`301`.
* For OAuth: the **exact** redirect URI in Google equals what your app sends (inspect the `redirect_uri` parameter in the request to Google).

---

## 9) (Optional) CI/CD on Free-Tier

* **CodeBuild**: 100 build minutes/month **and it doesn’t expire after 12 months**. ([Amazon Web Services, Inc.][6])
* **CodePipeline**: one V1 pipeline free per month **or** 100 minutes of V2 action time. ([Amazon Web Services, Inc.][14])
* **ECR**: 500 MB private storage for 12 months. ([Amazon Web Services, Inc.][5])

---

## 10) How to avoid charges

* Run **one** micro instance (t2.micro/t3.micro). ([Amazon Web Services, Inc.][1])
* Keep EBS ≤ **30 GB**. ([Amazon Web Services, Inc.][2])
* Public IPv4 usage for one instance is covered during your first 12 months (750 hours/month); don’t allocate extra Elastic IPs. ([Amazon Web Services, Inc.][3])
* **Do not create** NAT Gateways or Load Balancers for this exercise (they incur hourly charges). ([AWS Documentation][7], [Amazon Web Services, Inc.][9])
* Set **Budgets** alerts (free) so you get an email if any cost appears. ([Amazon Web Services, Inc.][4])

---

## 11) Stopping and cleaning up (to pay \$0)

**Pause (no compute charges):**

* EC2 → Instances → select → **Stop**.
  (Storage still accrues, but that’s within the Free Tier if ≤30 GB.) ([Amazon Web Services, Inc.][2])

**Zero cost:**

* Back up anything you need, then **Terminate** the instance.
* EC2 → **Volumes**: delete any **available** (orphaned) volumes.
* EC2 → **Elastic IPs**: if you allocated one, **Disassociate** then **Release**.
* Delete leftover snapshots/AMIs if you created any.

---

## 12) FAQ

**Q: Can I expose multiple microservices on one EC2?**
Yes. Run each container on a different local port and let Nginx route by path (e.g., `/auth → :9000`, `/api/catalog → :9010`). This keeps one public IP and no load balancer costs.

**Q: Why not an ALB?**
Great for production, but it adds hourly + LCU charges that aren’t Free-Tier. ([Amazon Web Services, Inc.][9])

**Q: Do I really need a domain for Google Sign-In?**
For public users, **yes**—Google requires HTTPS and not a raw IP. For personal testing, use the **localhost** exception with an SSH tunnel. ([Google for Developers][10])

---

### Note on DuckDNS “five subdomains”

You asked for confirmation: DuckDNS does not prominently publish an official limit document. However, multiple community docs and tutorials indicate **up to 5 subdomains per account**—and this matches what most users see in their dashboard. Please verify in your DuckDNS account for the current limit, as policies can change. ([swizzin.ltd][11], [MakeUseOf][12], [Medium][13])

---

### Appendix: Optional useful links

* **EC2 Free-Tier (t2.micro)**: 750 hours/month for 12 months. ([Amazon Web Services, Inc.][1])
* **Public IPv4 Free-Tier addition**: 750 hours/month for 12 months. ([Amazon Web Services, Inc.][3])
* **EBS Free-Tier**: 30 GB + 2M I/Os + 1 GB snapshots. ([Amazon Web Services, Inc.][2])
* **Budgets pricing/FAQ** (budgets without actions are free; first two action-enabled free). ([Amazon Web Services, Inc.][4])
* **CodeBuild Free-Tier**: 100 minutes/month, doesn’t expire. ([Amazon Web Services, Inc.][6])
* **ECR Free-Tier**: 500 MB/month for 12 months. ([Amazon Web Services, Inc.][5])
* **NAT Gateway pricing (hourly + per-GB)**. ([AWS Documentation][7])
* **ALB pricing (hourly + LCU)**. ([Amazon Web Services, Inc.][9])
* **Google OAuth redirect rules** (HTTPS required; localhost exception; no raw IPs). ([Google for Developers][10])

---


[1]: https://aws.amazon.com/ec2/instance-types/t2/?utm_source=chatgpt.com "Amazon EC2 T2 Instances"
[2]: https://aws.amazon.com/ebs/pricing/?utm_source=chatgpt.com "Amazon EBS pricing"
[3]: https://aws.amazon.com/about-aws/whats-new/2024/02/aws-free-tier-750-hours-free-public-ipv4-addresses/?utm_source=chatgpt.com "AWS Free Tier now includes 750 hours of free Public IPv4 ..."
[4]: https://aws.amazon.com/aws-cost-management/aws-budgets/faqs/?utm_source=chatgpt.com "AWS Budgets FAQs - Amazon Web Services"
[5]: https://aws.amazon.com/ecr/pricing/?utm_source=chatgpt.com "Amazon ECR Pricing - Elastic Container Registry"
[6]: https://aws.amazon.com/codebuild/pricing/?utm_source=chatgpt.com "Managed Build Server - AWS CodeBuild Pricing"
[7]: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html?utm_source=chatgpt.com "Pricing for NAT gateways"
[8]: https://aws.amazon.com/vpc/pricing/?utm_source=chatgpt.com "Amazon VPC Pricing"
[9]: https://aws.amazon.com/elasticloadbalancing/pricing/?utm_source=chatgpt.com "Elastic Load Balancing pricing"
[10]: https://developers.google.com/identity/protocols/oauth2/web-server?utm_source=chatgpt.com "Using OAuth 2.0 for Web Server Applications | Authorization"
[11]: https://swizzin.ltd/applications/duckdns?utm_source=chatgpt.com "Duck DNS | swizzin community edition"
[12]: https://www.makeuseof.com/tag/5-best-dynamic-dns-providers-can-lookup-free-today/?utm_source=chatgpt.com "The 6 Best Free Dynamic DNS Providers"
[13]: https://medium.com/%404get.prakhar/connecting-your-local-network-to-the-internet-with-duckdns-f8179f7cc20e?utm_source=chatgpt.com "Connecting your Local Network to the Internet with DuckDNS"
[14]: https://aws.amazon.com/codepipeline/pricing/?utm_source=chatgpt.com "CI/CD Pipeline – AWS CodePipeline Pricing"
