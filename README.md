---
canonical: "https://sph.sh/en/posts/cost-effective-vps-dokploy-cloudflared-setup/"
category: cloud-computing
date: 2025-10-26
description: A practical guide to setting up a secure, affordable
  private server using VPS, Dokploy for deployments, and Cloudflared
  tunnels for secure access without exposing ports
locale: en
tags:
- docker
- vps
- cloudflare
- devops
- self-hosting
- security
- dokploy
title: "Dokploy + Cloudflare Tunnel on a VPS: Setup Guide"
---

## Why This Stack?

Running a private server doesn't have to mean expensive cloud bills or
complex infrastructure. The sweet spot often lies in combining simple,
focused tools rather than reaching for enterprise platforms.

This guide walks through setting up a production-ready private server
using:
- **VPS** for affordable compute (\~\$5-20/month)
- **Dokploy** for Docker-based deployments with a clean UI
- **Cloudflared** for secure access without opening ports

The total cost? About \$5-10/month for a basic setup that handles
multiple applications securely.

## Prerequisites

Before starting, you'll need:
- A domain name (for Cloudflare tunnel)
- Basic terminal/SSH knowledge
- A Cloudflare account (free tier works)
- \~30 minutes of setup time

## Architecture Overview

Here's what we're building:

``` mermaid
graph LR
    A["🌐 User"] --> B["☁️ Cloudflare"]
    B --> C["🔒 Cloudflare Tunnel"]
    C --> D["🖥️ VPS"]
    D --> E["🚦 Traefik (Dokploy)"]

    E --> F["📦 React/Vite App"]
    E --> G["⚡ API Gateway"]
    E --> H["🔐 Your Apps"]
    E --> I["👥 Your Apps"]
```

The beauty of this setup: your server never exposes ports directly. All
traffic flows through Cloudflare's encrypted tunnel.

## Step 1: VPS Selection and Initial Setup

### Choosing a Provider

Budget-friendly options worth knowing:

**Hetzner** (\$5-10/month):
- Excellent price/performance
- European data centers (good GDPR compliance)
- Reliable network
- Great for production workloads

**Contabo** (\$4-8/month):
- Very affordable
- More resources for the price
- Network can be inconsistent during peak times

**DigitalOcean** (\$6-12/month):
- Excellent documentation
- Predictable performance
- Great community support

For this guide, I'll use Hetzner, but commands work across providers.

### Minimum Specs

For Dokploy and a few small apps:
- **RAM**: 2GB minimum (4GB recommended)
- **CPU**: 1-2 cores
- **Storage**: 20GB SSD minimum
- **OS**: Ubuntu 22.04 LTS

### Initial Server Setup

Once your VPS is provisioned, SSH into it:

``` bash
ssh root@your-server-ip  
```

First, update the system:

``` bash
apt update && apt upgrade -y  
```

## Step 2: Security Hardening

Spending 15 minutes on security now saves hours of headaches later.

### Create a Non-Root User

``` bash
# Create user  
adduser deploy  
usermod -aG sudo deploy

# Test sudo access  
su - deploy  
sudo ls -la root  
```

### Setup SSH Key Authentication

On your local machine:

``` bash
# Generate SSH key if you don't have one  
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy to server  
ssh-copy-id deploy@your-server-ip  
```

### Harden SSH Configuration

Back on the server:

``` bash
sudo nano /etc/ssh/sshd_config  
```

Update these settings:

``` text
# Disable root login  
PermitRootLogin no

# Disable password authentication  
PasswordAuthentication no  
PubkeyAuthentication yes

# Change default port (optional but recommended)  
Port 2222

# Disable empty passwords  
PermitEmptyPasswords no  
```

Restart SSH:

``` bash
sudo systemctl restart sshd  
```

> \[!WARNING\]\
> Before closing your current SSH session, test the new configuration in
> a separate terminal. If something's wrong, you still have access to
> fix it.

### Configure Firewall (UFW)

``` bash
# Set defaults  
sudo ufw default deny incoming  
sudo ufw default allow outgoing

# Allow SSH (use your custom port if changed)  
sudo ufw allow 2222/tcp

# Enable firewall  
sudo ufw enable

# Check status  
sudo ufw status verbose  
```

### Install fail2ban

Protects against brute force attacks:

``` bash
sudo apt install fail2ban -y

# Create local config  
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local  
sudo nano /etc/fail2ban/jail.local  
```

Update the SSH section:

``` ini
[sshd]  
enabled = true  
port = 2222  
filter = sshd  
logpath = /var/log/auth.log  
maxretry = 3  
bantime = 3600  
```

Start fail2ban:

``` bash
sudo systemctl enable fail2ban  
sudo systemctl start fail2ban  
```

## Step 3: Dokploy Installation

Dokploy provides a Heroku-like deployment experience with Docker. Here's
what makes it useful: simple web UI, built-in database support, and
straightforward app deployment.

### Install Docker

``` bash
# Install Docker  
curl -fsSL https://get.docker.com -o get-docker.sh  
sudo sh get-docker.sh

# Add user to docker group  
sudo usermod -aG docker deploy

# Enable Docker service  
sudo systemctl enable docker  
sudo systemctl start docker

# Verify installation  
docker --version  
```

> \[!TIP\]\
> Log out and back in for the docker group change to take effect.

### Install Dokploy

``` bash
# One-line installation  
curl -sSL https://dokploy.com/install.sh | sh
# Or
curl -sSL https://dokploy.com/install.sh | sudo sh    
```

This script:
1. Sets up Docker if not installed
2. Installs Dokploy services
3. Configures the web interface
4. Starts the Dokploy dashboard

After installation, Dokploy runs on port 3000. But here's the thing:
we're not going to expose this port directly. That's where Cloudflared
comes in.

## Step 4: Cloudflared Tunnel Setup

Instead of opening ports, create a secure tunnel through Cloudflare.
This approach has several advantages:
- No port forwarding needed
- Built-in DDoS protection
- Free SSL certificates
- Traffic analytics

### Install Cloudflared

``` bash
# Download and install  
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb  
sudo dpkg \-i cloudflared-linux-amd64.deb  
```

### Authenticate with Cloudflare

``` bash
cloudflared tunnel login  
```

This opens a browser to authorize the tunnel. Select your domain.

### Create and Configure Tunnel

``` bash
# Create tunnel  
cloudflared tunnel create dokploy-server

# Note the tunnel ID from output  
```

Create configuration file:

``` bash
sudo mkdir -p /etc/cloudflared  
sudo nano /etc/cloudflared/config.yml  
```

Add this configuration:

``` yaml
tunnel: YOUR-TUNNEL-ID  
credentials-file: /etc/cloudflared/YOUR-TUNNEL-ID.json

ingress:  
  # Dokploy dashboard  
  - hostname: dokploy.yourdomain.com  
    service: http://localhost:3000

  # Catch-all rule (required)  
  - service: http_status:404  
```

### Setup DNS

``` bash
# Create DNS record  
cloudflared tunnel route dns dokploy-server dokploy.yourdomain.com  
```

### Run Tunnel as Service

``` bash
# Install as service  
sudo cloudflared service install

# Start service  
sudo systemctl start cloudflared  
sudo systemctl enable cloudflared

# Check status  
sudo systemctl status cloudflared  
```

Now visit `https://dokploy.yourdomain.com` - you should see the Dokploy
dashboard, accessed securely through the Cloudflare tunnel.

## Step 5: Deploying Your First Application

Let's deploy a simple Node.js application to verify everything works.

### Create App in Dokploy

1\. Open Dokploy dashboard (`https://dokploy.yourdomain.com`)\
2. Complete initial setup (create admin account)\
3. Click "Create Project"\
4. Choose "Application"

### Example: Deploy Node.js App

Here's a basic deployment configuration:

**Project Structure:**

    my-app/  
    ├── Dockerfile  
    ├── package.json  
    └── src/  
        └── index.js  

**Dockerfile:**

``` dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package\*.json ./  
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "src/index.js"]  
```

**package.json:**

``` json
{  
  "name": "my-app",  
  "version": "1.0.0",  
  "main": "src/index.js",  
  "dependencies": {  
    "express": "^4.18.2"  
  }  
}  
```

**src/index.js:**

``` javascript
const express = require('express');  
const app = express();  
const port = 3000;

app.get('/', (req, res) => {  
  res.json({  
    message: 'Hello from Dokploy!',  
    timestamp: new Date().toISOString()  
  });  
});

app.listen(port, () => {  
  console.log(`Server running on port ${port}`);  
});  
```

### Configure Deployment

In Dokploy:
1. Connect your Git repository (GitHub, GitLab, etc.)
2. Set build settings (Dockerfile-based)
3. Configure port: 3000
4. Deploy!

### Add Tunnel Route for Your App

Update `/etc/cloudflared/config.yml`:

``` yaml
tunnel: YOUR-TUNNEL-ID  
credentials-file: /etc/cloudflared/YOUR-TUNNEL-ID.json

ingress:  
  # Dokploy dashboard  
  - hostname: dokploy.yourdomain.com  
    service: http://localhost:3000

  # Your app  
  - hostname: app.yourdomain.com  
    service: http://localhost:8080

  - service: http_status:404  
```
Pros:

-   Simple to understand.
-   Good for a single application.

Cons:

-   Every new application requires editing `config.yml`, creating a DNS
    record and restarting the tunnel.

Create DNS and restart:

``` bash
cloudflared tunnel route dns dokploy-server app.yourdomain.com  
sudo systemctl restart cloudflared  
```

## Or Deploying Applications: Recommended Approaches

There are one ways to expose applications when using Cloudflare Tunnel
with Dokploy.
### Recommended: Traefik + Wildcard

Instead of routing every application directly from Cloudflare Tunnel,
route all requests to Traefik and let Dokploy decide which container
should receive the request.

``` mermaid
graph TD
    A["🌐 User"] --> B["☁️ Cloudflare"]
    B --> C["🔒 Cloudflare Tunnel"]
    C --> D["🚦 Traefik"]

    D --> E["profile.example.com"]
    D --> F["api.example.com"]
    D --> G["admin.example.com"]
    D --> H["nextjs.example.com"]
```

Example configuration:

``` yaml
tunnel: YOUR-TUNNEL-ID

credentials-file: /etc/cloudflared/YOUR-TUNNEL-ID.json

ingress:
  - hostname: "*.ollama-serve.shop"
    service: http://localhost:80

  - hostname: ollama-serve.shop
    service: http://localhost:80

  - service: http_status:404
```
Create DNS and restart:

``` bash
cloudflared tunnel route dns dokploy-server "*.ollama-serve.shop"  
sudo systemctl restart cloudflared  
```

With this configuration, every subdomain is forwarded to Traefik, and
Dokploy routes requests to the correct container based on the configured
domain.

> \[!TIP\] This approach is recommended for hosting multiple
> applications because the Cloudflare Tunnel configuration rarely needs
> to change.

------------------------------------------------------------------------

## Framework Port Reference

  Framework                      Recommended Port
  ---------------------------- ------------------
  React + Vite (Caddy/Nginx)                   80
  Next.js                                    3000
  Express                                    3000
  Fastify                                    3000
  NestJS                                     3000

> \[!IMPORTANT\] Static React + Vite applications deployed by Dokploy
> are typically served by Caddy on port **80**. Configuring port
> **3000** will usually result in a **502 Bad Gateway** error.

## Cost Breakdown

Here's what this setup actually costs:

  Component           Monthly Cost
  ------------------- ---------------------------
  VPS (Hetzner 2GB)   \$5-6
  Domain name         \$1-2 (yearly, amortized)
  Cloudflare Tunnel   Free
  Dokploy             Free (open source)
  \*\*Total\*\*       **\~\$6-8/month**

Compare this to managed platforms:\
- Heroku: \$25-50/month for similar resources\
- AWS Lightsail: \$10-20/month (without tunnel/UI)\
- Managed Kubernetes: \$50-100/month minimum

## Lessons Learned

Working with this stack over several deployments, here's what stands
out:

**What Works Well:**\
- **Security first approach**: Cloudflared eliminates a whole class of
security concerns. No open ports means no port scanning attacks.\
- **Simple updates**: Dokploy handles container orchestration without
Kubernetes complexity.\
- **Cost control**: Fixed monthly cost, no surprise bills from traffic
spikes.\
- **Developer experience**: Git push to deploy feels modern without
vendor lock-in.

**Trade-offs to Consider:**\
- **Single point of failure**: One VPS means downtime if hardware fails.
For critical apps, consider multiple regions.\
- **Manual scaling**: Unlike managed platforms, scaling means creating
additional VPS instances and load balancing.\
- **Backup responsibility**: You own the backup strategy. Automate it or
risk data loss.\
- **Limited resources**: A \$5 VPS won't handle thousands of concurrent
users. Know your limits.

**Worth Doing Earlier:**\
- Set up monitoring (Uptime Robot or similar)\
- Document the initial setup steps immediately\
- Create a staging environment from day one\
- Automate security updates with unattended-upgrades

## Next Steps

Once your basic setup is running, consider:

1.  **Add monitoring**: Set up Uptime Robot or similar for availability
    checks\
2.  **Enable automatic updates**: Configure unattended-upgrades for
    security patches\
3.  **Set up databases**: Dokploy supports PostgreSQL, MySQL, MongoDB
    out of the box\
4.  **Create staging environment**: Clone your setup to a second cheap
    VPS for testing\
5.  **Implement CI/CD**: Connect GitHub Actions to trigger Dokploy
    deployments

## Conclusion

This VPS + Dokploy + Cloudflared stack provides a practical middle
ground between expensive managed platforms and complex self-hosted
setups. The total setup time runs about 30-45 minutes, and ongoing
maintenance is minimal - maybe an hour per month for updates and
monitoring.

The approach works particularly well for:\
- Side projects and small applications\
- Learning DevOps without cloud platform complexity\
- Teams wanting deployment simplicity without vendor lock-in\
- Cost-conscious production workloads with moderate traffic

Is it perfect? No. You're trading managed platform convenience for cost
savings and control. But for many use cases, that's exactly the right
trade-off.

The infrastructure patterns used here - containerization, reverse
proxying, secure tunneling - are the same patterns used at scale. You're
learning production-grade concepts on a budget-friendly platform.

Start simple, monitor your resources, and scale when you actually need
it. That's often more practical than over-engineering from day one.

## References

-   \[Cloudflare Tunnel
    Documentation\](https://developers.cloudflare.com/tunnel/) -
    Official guide for setting up cloudflared to expose services
    securely without opening inbound ports.\
-   \[Dokploy Documentation\](https://docs.dokploy.com/docs/core) -
    Official documentation for the self-hosted PaaS, covering
    installation, application deployment, and database management.\
-   \[Dokploy Installation
    Guide\](https://docs.dokploy.com/docs/core/installation) -
    Step-by-step setup instructions for deploying Dokploy on a VPS with
    Docker.\
-   \[Cloudflare Tunnel Setup
    Guide\](https://developers.cloudflare.com/tunnel/setup/) - Reference
    for configuring tunnels, ingress rules, and routing traffic to local
    services.\
-   \[Traefik Proxy Documentation\](https://doc.traefik.io/traefik/) -
    Documentation for the reverse proxy used internally by Dokploy to
    route traffic between containers.\
-   \[Next.js Self-Hosting
    Guide\](https://nextjs.org/docs/app/guides/self-hosting) - Official
    Next.js documentation for deploying applications on your own
    infrastructure with Node.js or Docker.

------------------------------------------------------------------------



