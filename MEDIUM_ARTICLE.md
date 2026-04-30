# Self-Hosting Behind Starlink (CGNAT) with Dokploy, Traefik, and Cloudflare Tunnels

*No public IP? No problem. Here's how to run production-grade apps from your own hardware, even on Starlink.*

You set up port forwarding. You double-checked the firewall. You pointed your domain at your home IP. And nothing works, because your ISP never gave you a real public IP in the first place.

That's **Carrier-Grade NAT (CGNAT)**, and it's the silent killer of self-hosting dreams. Starlink, most 5G home internet providers, and a growing number of ISPs sit their customers behind shared IPs at the network level. Traditional port forwarding is simply not possible.

But there's a clean, production-grade workaround that doesn't require a VPS, a static IP, or any dark networking magic. By combining **Dokploy** (open-source PaaS), **Traefik** (reverse proxy), and **Cloudflare Tunnels**, you can run real applications from your own hardware with HTTPS, CI/CD, and zero-downtime deploys, completely bypassing CGNAT.

This guide walks you through the full setup, end to end.

## How the Architecture Works

Before touching a terminal, it helps to understand the traffic flow:

```
User's Browser
     |
     v
Cloudflare Edge (HTTPS termination + DDoS protection)
     |  (encrypted tunnel, outbound from your server, no open ports needed)
     v
cloudflared daemon (running on your local server)
     |  (plain HTTP internally)
     v
Traefik (reverse proxy on port 80)
     |  (routes by hostname)
     v
Your App Container (Docker Swarm / Dokploy)
```

The key insight: **Cloudflare Tunnel is an outbound connection from your server to Cloudflare's edge.** Your server initiates it, so CGNAT never blocks it. No inbound ports need to be open.

Cloudflare handles all TLS/HTTPS. Traefik only does internal HTTP routing. This keeps the setup simple and avoids certificate management headaches.

### What You Need

- A machine running **Ubuntu 22.04 LTS or 24.04 LTS** (physical or VM)
- 2+ CPU cores, 4 GB+ RAM, 50 GB+ SSD
- A **Cloudflare account** with a domain name managed there
- `sudo` access on the server

## Step 1: Prepare the Server

SSH in and get the system up to date:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano ufw fail2ban
```

### Configure the Firewall (UFW)

Even though your server is behind CGNAT and the tunnel is outbound-only, running a local firewall is still good practice. **Allow SSH first, before enabling UFW, or you will lock yourself out.**

```bash
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP - Traefik internal'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw allow 3000/tcp comment 'Dokploy UI'

# Docker Swarm ports (needed if you add worker nodes later)
sudo ufw allow 2377/tcp comment 'Swarm Management'
sudo ufw allow 7946/tcp comment 'Swarm Node TCP'
sudo ufw allow 7946/udp comment 'Swarm Node UDP'
sudo ufw allow 4789/udp comment 'Overlay Network'

sudo ufw --force enable
sudo ufw status
```

## Step 2: Install Dokploy

Dokploy is an open-source PaaS, think Heroku or Coolify, that sits on top of Docker Swarm and gives you a clean UI for deploying apps, databases, and managing CI/CD.

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

This single script installs Docker Engine, initializes Docker Swarm, creates the `dokploy-network` overlay, and starts the Dokploy UI on port `3000`. Watch the progress:

```bash
docker logs -f dokploy
```

Once it's done, the UI is accessible at `http://YOUR_SERVER_IP:3000`. Complete the setup wizard to create your admin account.

## Step 3: Install and Configure Cloudflare Tunnel

This is the piece that solves CGNAT. The tunnel daemon (`cloudflared`) runs on your server and maintains a persistent outbound connection to Cloudflare's edge. No open inbound ports required.

### Install cloudflared

```bash
sudo mkdir -p /usr/share/keyrings

curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] \
  https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install cloudflared -y
```

### Authenticate and Create a Tunnel

```bash
cloudflared tunnel login
```

This opens a browser window to authorize your Cloudflare account. Then create the tunnel:

```bash
cloudflared tunnel create dokploy-tunnel
```

Note the path to the credentials JSON file in the output. You will need it in the next step.

### Create the Configuration File

```bash
sudo mkdir -p /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```

```yaml
tunnel: dokploy-tunnel
credentials-file: /root/.cloudflared/<YOUR-TUNNEL-ID>.json

ingress:
  # Route all traffic for your domain to Traefik on port 80
  - hostname: app.yourdomain.com
    service: http://localhost:80
  # Catch-all, required by cloudflared
  - service: http_status:404
```

Replace `app.yourdomain.com` with your actual domain or subdomain, and update the credentials file path with your tunnel ID.

### Run as a System Service

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared

# Verify it's running
sudo systemctl status cloudflared
```

The tunnel will now start automatically on boot and reconnect if the connection drops.

## Step 4: Configure Traefik

Because Cloudflare handles HTTPS at the edge, Traefik only needs to do **internal HTTP routing**. Do not configure ACME or Let's Encrypt here. It's unnecessary and will conflict with Cloudflare's SSL.

### Create the Directory Structure

```bash
sudo mkdir -p /etc/dokploy/traefik/dynamic
```

### Static Config (/etc/dokploy/traefik/traefik.yml)

```yaml
global:
  sendAnonymousUsage: false

entryPoints:
  web:
    address: ":80"

providers:
  docker:
    exposedByDefault: false
    network: dokploy-network
  file:
    directory: /etc/dokploy/traefik/dynamic
    watch: true

log:
  level: INFO

api:
  dashboard: true
  insecure: false
```

### Dynamic Config (/etc/dokploy/traefik/dynamic/apps.yml)

This is where you define routing rules for each app. Replace `app.yourdomain.com` with your domain and `your-app-container` with your Docker container's name and port:

```yaml
http:
  routers:
    my-app-router:
      rule: "Host(`app.yourdomain.com`)"
      service: my-app-service
      entryPoints:
        - web

  services:
    my-app-service:
      loadBalancer:
        servers:
          # Internal container name and port on the dokploy-network
          - url: "http://your-app-container:3000"
        passHostHeader: true
```

> **Tip:** The container name is the Docker service name assigned by Dokploy when you deploy an app. You can find it by running `docker ps` after deployment.

### Start Traefik

```bash
docker run -d \
  --name dokploy-traefik \
  --restart always \
  --network dokploy-network \
  -v /etc/dokploy/traefik/traefik.yml:/etc/traefik/traefik.yml \
  -v /etc/dokploy/traefik/dynamic:/etc/traefik/dynamic \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -p 80:80 \
  traefik:v3.6.7
```

## Step 5: Wire Up Cloudflare DNS

Go to your **Cloudflare Dashboard > DNS > Add Record**:

| Field | Value |
|---|---|
| Type | `CNAME` |
| Name | `app` (or your subdomain) |
| Target | `<YOUR-TUNNEL-ID>.cfargotunnel.com` |
| Proxy status | Proxied (orange cloud) |

Save it. Within a minute or two, the full chain is live:

```
https://app.yourdomain.com
  > Cloudflare Edge (TLS terminated here)
    > Cloudflare Tunnel (outbound from your server)
      > localhost:80 (Traefik)
        > your-app-container:3000
```

Test it with `curl -I https://app.yourdomain.com`. You should get a `200 OK` with Cloudflare headers.

## Step 6: Deploying Your First App

With the tunnel and Traefik running, it's time to actually deploy something. Here's the exact flow inside the Dokploy UI.

### Create a Project

Projects are the top-level container in Dokploy. Everything lives inside one.

1. In the sidebar, click **Projects**
2. Click **Create Project**
3. Give it a name (e.g., `my-apps`) and an optional description
4. Click **Create**

Dokploy creates the project with a default **Production** environment.

### Add an Application

Inside your project:

1. Click **Create Service**
2. Select **Application** as the service type
3. Give it a name (e.g., `web-app`)
4. Click **Create**

### Connect Your Git Repository

In the application settings, go to the **General** tab:

1. Under **Git Provider**, choose GitHub, GitLab, Bitbucket, or Gitea
2. Enter your repository URL
3. Set the branch to deploy (e.g., `main`)
4. Choose your build type:
   - **Nixpacks** for auto-detection (works for most Node, Python, Go, Ruby apps)
   - **Dockerfile** if you have one in the repo
   - **Buildpacks** as a fallback

If your app needs a specific build command or start command, set those in the **Build** tab.

### Set Environment Variables

Go to the **Environment** tab and add any variables your app needs:

```
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@db:5432/mydb
API_SECRET=your-secret-here
```

Variables are stored securely and injected at runtime. They do not get baked into the image.

### Configure the Domain

Go to the **Domains** tab:

1. Click **Add Domain**
2. Set **Host** to your subdomain (e.g., `app.yourdomain.com`)
3. Set **Container Port** to the port your app listens on (e.g., `3000`)
4. Leave **Path** as `/`
5. Toggle **HTTPS** off (Cloudflare handles TLS, not Traefik)
6. Click **Create**

Dokploy writes a Traefik routing config automatically. No redeployment needed for domain changes.

### Deploy

Go to the **Deployments** tab and click **Deploy**. You can watch the build logs in real time:

```
Cloning repository...
Installing dependencies...
Building application...
Creating Docker image...
Deploying to swarm...
Deployment successful!
```

Once done, your app is live at `https://app.yourdomain.com`.

### Enable Auto-Deploy on Git Push

To wire up push-to-deploy:

1. In the **General** tab, toggle **Auto Deploy** on
2. Go to the **Deployments** tab and copy the **Webhook URL**
3. In your GitHub repo, go to **Settings > Webhooks > Add webhook**
4. Paste the URL, set content type to `application/json`, and save

From that point on, every push to your configured branch triggers a new build and a zero-downtime rolling update on Docker Swarm. No GitHub Actions YAML needed.

> **Note:** Make sure the branch set in Dokploy matches the branch you push to. A mismatch causes a "Branch Not Match" error and the deploy won't trigger.

## Step 7: Lock Down the Dokploy UI with Cloudflare Zero Trust

Port 3000, the Dokploy management UI, should never be exposed to the open internet. The cleanest way to protect it is **Cloudflare Access**, which is part of the free Zero Trust tier.

### Set Up Cloudflare Access

1. Go to **Cloudflare Zero Trust Dashboard > Access > Applications > Add an Application**
2. Choose **Self-hosted**
3. Set the application domain to `dokploy.yourdomain.com` (add a separate tunnel ingress rule for this)
4. Under **Policies**, add an **Allow** rule:
   - Rule name: Admin only
   - Include: Emails, then add your admin email address
5. Save and deploy

Now anyone hitting `dokploy.yourdomain.com` gets a Cloudflare login prompt before they ever reach your server. Bots and scanners never make it through.

Update your `config.yml` to add the Dokploy UI route:

```yaml
ingress:
  - hostname: app.yourdomain.com
    service: http://localhost:80
  - hostname: dokploy.yourdomain.com
    service: http://localhost:3000
  - service: http_status:404
```

Restart cloudflared after updating the config:

```bash
sudo systemctl restart cloudflared
```

## Troubleshooting

### Tunnel connects but app returns 502

Traefik can reach the tunnel but can't find the container. Check:

```bash
# Confirm the container is running and on the right network
docker ps
docker network inspect dokploy-network

# Check Traefik logs for routing errors
docker logs dokploy-traefik --tail 50
```

Make sure the container name in your dynamic config matches exactly what `docker ps` shows.

### cloudflared service fails to start

```bash
sudo journalctl -u cloudflared -n 50
```

Most common causes: wrong credentials file path in `config.yml`, or the tunnel name doesn't match the credentials file. Run `cloudflared tunnel list` to confirm the tunnel ID.

### App is unreachable after deploy

```bash
# Check the Dokploy service is running in Swarm
docker service ls
docker service ps <your-service-name>

# Check app logs
docker service logs <your-service-name> --tail 100
```

### Cloudflare shows "SSL handshake failed"

This usually means Traefik is trying to serve HTTPS on port 80, or there's a port mismatch. Confirm your tunnel config points to `http://localhost:80` (not `https`), and that Traefik is only listening on port 80 with no TLS configured.

### UFW blocking Docker Swarm

If you add worker nodes later and they can't join, check that the Swarm ports are open between nodes:

```bash
# Test from worker to manager
nc -zv MANAGER_IP 2377

# If blocked, allow the worker IP on the manager
sudo ufw allow from WORKER_IP to any port 2377 proto tcp
sudo ufw allow from WORKER_IP to any port 7946
sudo ufw allow from WORKER_IP to any port 4789 proto udp
sudo ufw reload
```

## Production Best Practices

**Let Cloudflare own TLS.** Don't configure ACME or Let's Encrypt in Traefik. Cloudflare terminates HTTPS at the edge and sends plain HTTP down the tunnel. This removes certificate renewal complexity entirely.

**Use Cloudflare Access for every admin interface.** The Dokploy UI, any database admin tools, monitoring dashboards, put them all behind Zero Trust. It's free and takes five minutes per app.

**Scale horizontally when ready.** Docker Swarm makes adding capacity straightforward. Install Docker on a new machine, run `docker swarm join` with the token from your manager, and Dokploy will start scheduling containers on it immediately.

**Enable Dokploy's built-in monitoring.** Under Settings > Monitoring, you can enable Prometheus and Grafana with a few clicks. Set alerts for CPU above 80%, memory above 90%, and disk below 10% so you're not caught off guard.

**Back up your Dokploy config.** The Dokploy database (PostgreSQL) holds all your app configs, environment variables, and deployment history. Set up automated backups to S3 or a local destination through the Dokploy UI under Databases > Backups.

## Wrapping Up

CGNAT is a real constraint, but it's not a dead end. The Cloudflare Tunnel approach is genuinely production-grade. It's the same mechanism companies use for Zero Trust network access, just applied to self-hosting.

The full stack here, Dokploy for orchestration, Traefik for routing, Cloudflare for edge and security, gives you most of what a managed cloud platform provides, running on hardware you own, at a fraction of the cost.

If this helped you get unstuck, share it with someone else dealing with the same CGNAT problem.

*Suggested Medium tags: Self Hosting, DevOps, Cloudflare, Docker, Starlink*
