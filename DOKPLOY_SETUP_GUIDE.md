# Dokploy Multi-Server Setup Guide
## Complete Step-by-Step Production Deployment

**Date:** January 8, 2026  
**Version:** Dokploy v0.25.4+ (Latest Stable)  
**Architecture:** Manager-Worker Cluster (Docker Swarm)

**⚠️ CRITICAL:** All nodes must be on the same network subnet OR have proper routing configured between subnets before attempting cluster setup.

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Master Server Setup](#master-server-setup)
4. [Worker Servers Setup](#worker-servers-setup)
5. [Security Hardening](#security-hardening)
6. [Network Configuration](#network-configuration)
7. [SSL/TLS Configuration](#ssltls-configuration)
8. [Application Deployment](#application-deployment)
9. [Monitoring & Maintenance](#monitoring--maintenance)
10. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### Cluster Topology
```
                          Internet
                             |
                    [Firewall/Router]
                             |
                    [Load Balancer - Optional]
                             |
        _____________________|_____________________
       |                     |                     |
[Manager Node]        [Worker Node 1]      [Worker Node 2]
   (Master)              (Worker)            (Worker)
   - Dokploy UI          - Applications      - Applications
   - Traefik             - Databases         - Databases
   - Swarm Manager       - Containers        - Containers
```

### How It Works
- **Manager Node (Master)**: Runs Dokploy UI, manages the cluster, orchestrates deployments
- **Worker Nodes**: Execute application containers, handle workloads
- **Docker Swarm**: Provides clustering, networking, and service discovery
- **Traefik**: Routes external traffic to containers, handles SSL/TLS

---

## Prerequisites

### Hardware Requirements (Minimum per Node)

**Manager Node:**
- CPU: 2 cores (4 cores recommended)
- RAM: 4GB (8GB recommended)
- Storage: 50GB SSD (100GB+ recommended)
- Network: Static IP address, 1Gbps connection

**Worker Nodes:**
- CPU: 2 cores (4+ cores recommended)
- RAM: 4GB (8GB+ recommended)
- Storage: 50GB SSD (depends on workload)
- Network: Static IP address, 1Gbps connection

### Software Requirements

**All Nodes:**
- OS: Ubuntu 22.04 LTS / Ubuntu 24.04 LTS (Recommended)
  - Alternatively: Debian 11/12, CentOS 8+, RHEL 8+
- Docker: Will be installed automatically
- Root or sudo access
- Open ports (details in Network Configuration section)

### Network Requirements

**CRITICAL - Network Connectivity:**
- **All nodes MUST be on the same subnet** (e.g., all on 192.168.1.x)
  - OR have proper routing configured between different subnets
  - Test with `ping` between all nodes BEFORE starting setup
  - Cross-subnet setups require router configuration or VPN (Tailscale recommended)

**Manager Node Must Have:**
- Public IP address (or port forwarding configured)
- Domain name pointed to the server (e.g., dokploy.yourdomain.com)

**All Nodes Must Allow:**
- SSH access (port 22 or custom)
- **Direct network communication between nodes** (same subnet strongly recommended)
- Bidirectional connectivity on Docker Swarm ports (see Port Reference)

### Pre-Installation Checklist

- [ ] All servers are running Ubuntu 22.04+ LTS
- [ ] Root/sudo access verified on all nodes
- [ ] Static IP addresses assigned to all nodes
- [ ] Domain DNS records configured (A record → Manager public IP)
- [ ] SSH key-based authentication configured
- [ ] Firewall rules documented
- [ ] Backup strategy defined

---

## Master Server Setup

### Step 1: Prepare the Manager Node

#### 1.1 Update System
```bash
# SSH into your manager server
ssh root@your-manager-ip

# Update package lists and upgrade system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y curl wget git nano ufw fail2ban
```

#### 1.2 Set Hostname
```bash
# Set a descriptive hostname
sudo hostnamectl set-hostname dokploy-manager

# Update hosts file
echo "127.0.0.1 dokploy-manager" | sudo tee -a /etc/hosts
```

#### 1.3 Configure Firewall (UFW)
```bash
# Enable UFW
sudo ufw --force enable

# Allow SSH (IMPORTANT: Do this first!)
sudo ufw allow 22/tcp comment 'SSH'

# Allow HTTP/HTTPS for web applications
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Allow Dokploy UI (default port 3000)
sudo ufw allow 3000/tcp comment 'Dokploy UI'

# Docker Swarm ports for cluster communication
sudo ufw allow 2377/tcp comment 'Docker Swarm Management'
sudo ufw allow 7946/tcp comment 'Docker Swarm Node Communication'
sudo ufw allow 7946/udp comment 'Docker Swarm Node Communication'
sudo ufw allow 4789/udp comment 'Docker Overlay Network'

# Reload firewall
sudo ufw reload

# Verify status
sudo ufw status numbered
```

### Step 2: Install Dokploy

#### 2.1 Run Installation Script
```bash
# Download and execute Dokploy installation script
curl -sSL https://dokploy.com/install.sh | sh
```

**What This Script Does:**
- Installs Docker Engine if not present
- Installs Docker Compose
- Downloads and starts Dokploy container
- Initializes Docker Swarm (if not already initialized)
- Sets up Traefik reverse proxy
- Configures initial network settings

#### 2.2 Wait for Installation (3-5 minutes)
```bash
# Monitor installation logs
docker logs -f dokploy
```

**Expected Output:**
```
✓ Docker installed successfully
✓ Docker Compose installed
✓ Initializing Docker Swarm
✓ Creating Dokploy network
✓ Starting Dokploy container
✓ Dokploy is running on port 3000
```

#### 2.3 Verify Installation
```bash
# Check Docker Swarm status
docker info | grep -i swarm
# Expected: Swarm: active

# Check Dokploy container
docker ps | grep dokploy
# Should show running container

# Check Swarm manager status
docker node ls
# Should show this node as Leader
```

### Step 3: Initial Dokploy Configuration

#### 3.1 Access Dokploy UI
```bash
# Open browser and navigate to:
http://your-manager-ip:3000
# OR
http://dokploy.yourdomain.com:3000
```

#### 3.2 Complete Setup Wizard

1. **Create Admin Account**
   - Email: your-admin@email.com
   - Password: Use strong password (20+ characters, mixed case, symbols)
   - Confirm password

2. **Server Configuration**
   - Server Name: dokploy-manager
   - Server IP: Your manager node's private IP
   - Keep default settings

3. **Database Selection** (for Dokploy's internal data)
   - PostgreSQL (default and recommended)
   - Location: Local (on manager node)
   - Backup: Enable automatic backups

#### 3.3 Get Docker Swarm Join Token
```bash
# On manager node, get worker join token
docker swarm join-token worker

# Save this output - you'll need it for worker nodes
# Output looks like:
# docker swarm join --token SWMTKN-1-xxxxxxxxxxxxx manager-ip:2377
```

**IMPORTANT:** Save this token securely. You'll use it on each worker node.

---

## Worker Servers Setup

### Step 4: Prepare Worker Nodes

**Repeat these steps for EACH worker node (Worker 1, Worker 2, etc.)**

#### 4.1 Update and Configure Worker Node
```bash
# SSH into worker server
ssh root@worker-1-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y curl wget git nano

# Set hostname
sudo hostnamectl set-hostname dokploy-worker-1
echo "127.0.0.1 dokploy-worker-1" | sudo tee -a /etc/hosts
```

#### 4.1.5 **CRITICAL: Test Network Connectivity**
```bash
# On worker node, test connectivity to manager
ping -c 3 MANAGER_PRIVATE_IP

# On manager node, test connectivity to worker
ping -c 3 WORKER_PRIVATE_IP
```

**⚠️ If ping fails (100% packet loss):**
- **Different subnets detected** (e.g., manager on 192.168.5.x, worker on 192.168.1.x)
- **Solutions:**
  1. **RECOMMENDED:** Move nodes to same subnet (change network configuration)
  2. Configure static routes on both nodes (only works if same gateway)
  3. Use Tailscale/VPN overlay network
  4. Configure router to bridge the networks

**DO NOT PROCEED** until ping succeeds in both directions. Docker Swarm requires bidirectional connectivity.

#### 4.2 Configure Worker Firewall
```bash
# Enable UFW
sudo ufw --force enable

# Allow SSH
sudo ufw allow 22/tcp comment 'SSH'

# Allow HTTP/HTTPS (for applications)
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Docker Swarm communication with manager
sudo ufw allow from MANAGER_NODE_IP to any port 2377 proto tcp comment 'Swarm Manager'
sudo ufw allow from MANAGER_NODE_IP to any port 7946 comment 'Swarm Cluster'
sudo ufw allow from MANAGER_NODE_IP to any port 4789 proto udp comment 'Overlay Network'

# Allow communication between all cluster nodes
# Replace with actual worker IPs
sudo ufw allow from WORKER_2_IP to any port 7946 comment 'Worker 2 Communication'
sudo ufw allow from WORKER_2_IP to any port 4789 proto udp comment 'Worker 2 Overlay'

# Reload firewall
sudo ufw reload
sudo ufw status numbered
```

#### 4.3 Install Docker on Worker
```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
docker --version
sudo docker run hello-world
```

### Step 5: Join Workers to Swarm

#### 5.1 Execute Join Command
```bash
# On worker node, execute the join token command you saved earlier
# Replace with your actual token and manager IP
docker swarm join --token SWMTKN-1-xxxxxxxxxxxxx MANAGER_PRIVATE_IP:2377
```

**Expected Output:**
```
This node joined a swarm as a worker.
```

#### 5.2 Verify from Manager Node
```bash
# SSH back to manager node
ssh root@manager-ip

# List all nodes in the cluster
docker node ls
```

**Expected Output:**
```
ID                            HOSTNAME           STATUS    AVAILABILITY   MANAGER STATUS
abc123def456 *                dokploy-manager    Ready     Active         Leader
ghi789jkl012                  dokploy-worker-1   Ready     Active         
mno345pqr678                  dokploy-worker-2   Ready     Active         
```

#### 5.3 Label Worker Nodes (Optional but Recommended)
```bash
# On manager node, add labels for better deployment control
docker node update --label-add role=worker dokploy-worker-1
docker node update --label-add role=worker dokploy-worker-2
docker node update --label-add environment=production dokploy-worker-1
docker node update --label-add environment=production dokploy-worker-2

# Verify labels
docker node inspect dokploy-worker-1 --pretty | grep Labels -A 5
```

### Step 6: Add Worker Nodes to Dokploy UI

#### 6.1 Navigate to Dokploy Dashboard
1. Go to `http://your-manager-ip:3000`
2. Login with admin credentials
3. Click on **"Servers"** in left sidebar
4. Click **"Add Server"** button

#### 6.2 Register Worker 1
```
Server Name: worker-1
Server IP: [Worker 1 Private IP]
SSH Port: 22
SSH User: root
Authentication: SSH Key (recommended) or Password

For SSH Key:
- Paste your private SSH key
- Or use the key Dokploy generated

Labels (optional):
- environment: production
- role: worker
- zone: us-east-1a
```

#### 6.3 Test Connection
- Click **"Test Connection"**
- Wait for green checkmark: ✓ Connection Successful
- Click **"Save"**

#### 6.4 Repeat for Additional Workers
- Repeat 6.2-6.3 for worker-2, worker-3, etc.

---

## Security Hardening

### Step 7: SSH Hardening (ALL NODES)

#### 7.1 Create Dedicated User (Optional but Recommended)
```bash
# On each node, create a dedicated user for Dokploy
sudo adduser dokploy-admin
sudo usermod -aG sudo dokploy-admin
sudo usermod -aG docker dokploy-admin

# Set up SSH key authentication
sudo mkdir -p /home/dokploy-admin/.ssh
sudo chmod 700 /home/dokploy-admin/.ssh

# Copy your public key
echo "your-public-key-here" | sudo tee /home/dokploy-admin/.ssh/authorized_keys
sudo chmod 600 /home/dokploy-admin/.ssh/authorized_keys
sudo chown -R dokploy-admin:dokploy-admin /home/dokploy-admin/.ssh
```

#### 7.2 Harden SSH Configuration
```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config
```

**Recommended Changes:**
```bash
# Disable root login
PermitRootLogin no

# Disable password authentication (use keys only)
PasswordAuthentication no
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Change default SSH port (optional but recommended)
Port 2222

# Limit authentication attempts
MaxAuthTries 3

# Disconnect idle sessions
ClientAliveInterval 300
ClientAliveCountMax 2

# Allow only specific users
AllowUsers dokploy-admin your-username
```

```bash
# Restart SSH service
sudo systemctl restart sshd

# Test new SSH connection BEFORE closing current session
ssh -p 2222 dokploy-admin@server-ip
```

#### 7.3 Configure Fail2Ban
```bash
# Install fail2ban (if not already)
sudo apt install -y fail2ban

# Create local configuration
sudo nano /etc/fail2ban/jail.local
```

**Add Configuration:**
```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
destemail = your-email@domain.com
sendername = Fail2Ban
action = %(action_mwl)s

[sshd]
enabled = true
port = 2222
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400

[dokploy]
enabled = true
port = 3000
logpath = /var/log/dokploy/access.log
maxretry = 5
```

```bash
# Start and enable fail2ban
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Check status
sudo fail2ban-client status
```

### Step 8: Docker Security

#### 8.1 Enable Docker Content Trust
```bash
# On manager node
export DOCKER_CONTENT_TRUST=1
echo "export DOCKER_CONTENT_TRUST=1" | sudo tee -a /etc/environment
```

#### 8.2 Configure Docker Daemon Security
```bash
# Edit Docker daemon config
sudo nano /etc/docker/daemon.json
```

**Add Security Settings:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "icc": false,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "metrics-addr": "127.0.0.1:9323",
  "experimental": false
}
```

```bash
# Restart Docker
sudo systemctl restart docker
```

#### 8.3 Create Docker Secrets (On Manager)
```bash
# Example: Create database password secret
echo "your-strong-database-password" | docker secret create db_root_password -

# Create API keys
echo "your-api-key-here" | docker secret create api_key -

# List secrets
docker secret ls
```

### Step 9: Network Security

#### 9.1 Create Isolated Overlay Networks
```bash
# On manager node, create encrypted overlay networks
docker network create \
  --driver overlay \
  --subnet 10.20.0.0/24 \
  --opt encrypted \
  dokploy-frontend

docker network create \
  --driver overlay \
  --subnet 10.20.1.0/24 \
  --opt encrypted \
  dokploy-backend

docker network create \
  --driver overlay \
  --subnet 10.20.2.0/24 \
  --opt encrypted \
  dokploy-database

# List networks
docker network ls
```

#### 9.2 Configure Network Policies (Firewall Rules)
```bash
# Example: Allow only frontend to access backend
# Block direct internet access to backend/database networks

# This is typically done at application level with network attachment
# Or using external firewall/security groups
```

### Step 10: Monitoring & Intrusion Detection

#### 10.1 Install and Configure Auditd
```bash
# On all nodes
sudo apt install -y auditd audispd-plugins

# Start service
sudo systemctl start auditd
sudo systemctl enable auditd

# Configure audit rules
sudo nano /etc/audit/rules.d/docker.rules
```

**Add Audit Rules:**
```bash
# Monitor Docker daemon
-w /usr/bin/docker -p wa -k docker
-w /var/lib/docker -p wa -k docker
-w /etc/docker -p wa -k docker
-w /usr/lib/systemd/system/docker.service -p wa -k docker
-w /etc/systemd/system/docker.service -p wa -k docker

# Monitor Dokploy
-w /var/lib/dokploy -p wa -k dokploy
```

```bash
# Reload audit rules
sudo augenrules --load
```

---

## Network Configuration

### Step 11: Configure External Access

#### 11.1 DNS Configuration

**On your DNS provider (Cloudflare, Route53, etc.):**
```
Type    Name                TTL     Value
A       dokploy             300     MANAGER_PUBLIC_IP
A       *.dokploy           300     MANAGER_PUBLIC_IP
CNAME   app1.yourdomain.com 300     dokploy.yourdomain.com
CNAME   app2.yourdomain.com 300     dokploy.yourdomain.com
```

#### 11.2 Port Forwarding (If Behind NAT/Router)

**On your router/firewall:**
```
External Port → Internal IP:Port
80 → MANAGER_PRIVATE_IP:80
443 → MANAGER_PRIVATE_IP:443
3000 → MANAGER_PRIVATE_IP:3000
```

#### 11.3 Configure Traefik in Dokploy

1. **Login to Dokploy UI**
2. **Navigate to Settings → Traefik**
3. **Configure Dashboard:**
   ```
   Enable Dashboard: Yes
   Dashboard Domain: traefik.yourdomain.com
   Basic Auth: Enable (set username/password)
   ```

4. **Configure Let's Encrypt:**
   ```
   Enable SSL: Yes
   Email: admin@yourdomain.com
   Challenge Type: HTTP-01 (or DNS-01 if available)
   Staging: No (use production)
   ```

5. **Save and Apply**

---

## SSL/TLS Configuration

### Step 12: Automatic SSL with Let's Encrypt

#### 12.1 Verify Traefik Configuration
```bash
# On manager node, check Traefik logs
docker logs $(docker ps | grep traefik | awk '{print $1}')
```

#### 12.2 Force HTTPS Redirect
1. Go to Dokploy UI → Settings → Traefik
2. Enable **"Force HTTPS Redirect"**
3. Set **"HSTS Max Age"**: 31536000 (1 year)
4. Enable **"HSTS Preload"**: Yes
5. Save changes

#### 12.3 Test SSL Configuration
```bash
# Test SSL certificate
curl -I https://dokploy.yourdomain.com

# Verify SSL score
# Visit: https://www.ssllabs.com/ssltest/analyze.html?d=dokploy.yourdomain.com
```

**Expected Grade:** A or A+

---

## Application Deployment

### Step 13: Deploy Your First Application

#### 13.1 Using Dokploy UI

1. **Navigate to Projects**
   - Click **"Projects"** in sidebar
   - Click **"Create Project"**
   - Name: production-apps
   - Click **"Create"**

2. **Create Application**
   - Inside project, click **"Create Application"**
   - Application Name: my-web-app
   - Type: Application (not Docker Compose)

3. **Configure Source**
   ```
   Source Type: Git
   Repository: https://github.com/your-username/your-app
   Branch: main
   Build Type: Nixpacks (auto-detect) OR Dockerfile
   ```

4. **Configure Build Settings**
   ```
   Build Command: npm install && npm run build
   Start Command: npm start
   Port: 3000
   ```

5. **Configure Deployment**
   ```
   Deployment Mode: Swarm
   Replicas: 2
   Update Strategy: Rolling
   
   Placement Constraints:
   - node.role == worker
   
   Resource Limits:
   - CPU: 0.5
   - Memory: 512MB
   ```

6. **Configure Domain**
   ```
   Enable Public Access: Yes
   Domain: myapp.yourdomain.com
   Enable SSL: Yes (automatic Let's Encrypt)
   ```

7. **Environment Variables**
   ```
   NODE_ENV=production
   DATABASE_URL=postgresql://user:pass@db:5432/mydb
   API_KEY=[Use Dokploy Secrets]
   ```

8. **Deploy**
   - Click **"Deploy"**
   - Monitor build logs in real-time
   - Wait for deployment completion

#### 13.2 Verify Deployment
```bash
# On manager node, check service
docker service ls
docker service ps my-web-app

# Check which nodes are running the service
docker service ps my-web-app --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
```

#### 13.3 Access Your Application
```
https://myapp.yourdomain.com
```

### Step 14: Deploy Database

#### 14.1 Create Database Service

1. **Navigate to Project → Create Database**
2. **Select Database Type:**
   - PostgreSQL 16
   - MySQL 8.0
   - MongoDB 7.0
   - Redis 7.2

3. **Configure Database (Example: PostgreSQL)**
   ```
   Database Name: production-db
   Version: 16
   
   Credentials:
   Username: dbadmin
   Password: [Generate Strong Password]
   Database Name: myapp_db
   
   Resources:
   Memory: 2GB
   CPU: 1
   Storage: 20GB
   
   Placement:
   Node: dokploy-worker-2 (dedicated database node)
   ```

4. **Configure Backups**
   ```
   Enable Automatic Backups: Yes
   Backup Schedule: Daily at 2:00 AM
   Retention: 7 days
   Destination: S3/Minio/Local
   
   S3 Configuration (if using):
   Bucket: dokploy-backups
   Region: us-east-1
   Access Key: [Your S3 Access Key]
   Secret Key: [Your S3 Secret Key]
   ```

5. **Deploy Database**

#### 14.2 Connect Application to Database
```bash
# Get database connection string from Dokploy UI
# Format: postgresql://dbadmin:password@production-db:5432/myapp_db

# Update application environment variable
DATABASE_URL=postgresql://dbadmin:password@production-db:5432/myapp_db
```

---

## Monitoring & Maintenance

### Step 15: Enable Monitoring

#### 15.1 Configure Dokploy Monitoring

1. **Navigate to Settings → Monitoring**
2. **Enable Prometheus**
   ```
   Enable: Yes
   Retention: 15 days
   ```

3. **Enable Grafana**
   ```
   Enable: Yes
   Domain: monitoring.yourdomain.com
   Admin Password: [Set Strong Password]
   ```

4. **Configure Alerts**
   ```
   Enable Alerts: Yes
   
   Alert Channels:
   - Slack: webhook-url
   - Discord: webhook-url
   - Email: alerts@yourdomain.com
   - Telegram: bot-token and chat-id
   
   Alert Rules:
   - CPU usage > 80% for 5 minutes
   - Memory usage > 90% for 5 minutes
   - Disk space < 10%
   - Service down for 2 minutes
   ```

#### 15.2 Access Monitoring Dashboards

**Grafana:**
```
https://monitoring.yourdomain.com
Default Login: admin / [your-password]
```

**Traefik Dashboard:**
```
https://traefik.yourdomain.com/dashboard/
```

### Step 16: Regular Maintenance Tasks

#### 16.1 Weekly Tasks
```bash
# Update system packages (all nodes)
sudo apt update && sudo apt upgrade -y

# Clean up Docker resources
docker system prune -af --volumes

# Check disk space
df -h

# Review logs for errors
docker service logs --tail 100 my-web-app
```

#### 16.2 Monthly Tasks
```bash
# Rotate logs
sudo logrotate -f /etc/logrotate.conf

# Review and update firewall rules
sudo ufw status numbered

# Check SSL certificate expiration
echo | openssl s_client -servername yourdomain.com -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates

# Backup Dokploy configuration
docker exec dokploy backup-config
```

#### 16.3 Security Audits (Quarterly)
```bash
# Check for security updates
sudo apt list --upgradable

# Scan for vulnerabilities
docker scan $(docker ps --format "{{.Image}}")

# Review user access
docker node ls
cat /etc/passwd | grep /bin/bash

# Check fail2ban logs
sudo fail2ban-client status sshd
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Worker Node Won't Join Swarm

**Symptoms:**
```
Error response from daemon: rpc error: code = DeadlineExceeded desc = context deadline exceeded
OR
Error response from daemon: rpc error: code = Unavailable desc = connection error
```

**Root Cause Analysis:**
```bash
# Test basic connectivity from worker to manager
ping -c 3 MANAGER_IP

# If 100% packet loss, this is a NETWORK ROUTING ISSUE, not a Docker problem
```

**Solutions (in order of likelihood):**

**A. Network Routing Problem (Most Common):**
```bash
# Check if nodes are on different subnets
# On manager:
ip addr show | grep "inet "
# Example output: 192.168.5.252/24

# On worker:
ip addr show | grep "inet "
# Example output: 192.168.1.30/24

# If different subnets (192.168.5.x vs 192.168.1.x), you have routing issues

# SOLUTION 1 (RECOMMENDED): Move to same subnet
# Reconfigure network interface on one machine to match the other's subnet
# Example: Change manager from 192.168.5.252 to 192.168.1.236
# Then reinitialize swarm with new IP:
docker swarm leave --force
docker swarm init --advertise-addr NEW_MANAGER_IP

# SOLUTION 2: Static route (only if same gateway/router)
# On worker:
sudo ip route add MANAGER_SUBNET/24 via GATEWAY_IP dev INTERFACE
# Example: sudo ip route add 192.168.5.0/24 via 192.168.1.1 dev wlo1

# SOLUTION 3: Use Tailscale/VPN
# Install Tailscale on both nodes
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# Use Tailscale IPs for swarm
tailscale ip -4  # Get Tailscale IP
docker swarm init --advertise-addr TAILSCALE_IP
```

**B. Firewall Issues:**
```bash
# 1. Check firewall on manager
sudo ufw status | grep 2377

# 2. Verify manager is listening
sudo netstat -tulpn | grep 2377

# 3. Test connectivity from worker
nc -zv MANAGER_IP 2377
# OR
openssl s_client -connect MANAGER_IP:2377 -showcerts

# 4. Allow worker IP on manager
sudo ufw allow from WORKER_IP to any port 2377 proto tcp
sudo ufw allow from WORKER_IP to any port 7946
sudo ufw allow from WORKER_IP to any port 4789 proto udp
sudo ufw reload
```

**C. Docker/Swarm State Issues:**
```bash
# On worker: Clean Docker swarm state
docker swarm leave --force
sudo systemctl stop docker
sudo rm -rf /var/lib/docker/swarm
sudo systemctl start docker

# On manager: Get fresh join token
sudo systemctl restart docker
sleep 10
docker swarm join-token worker

# On worker: Retry join with fresh token
docker swarm join --token NEW_TOKEN MANAGER_IP:2377
```

**D. Time Synchronization:**
```bash
# Check time on both nodes
timedatectl

# If time differs by more than a few seconds, sync it:
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

#### Issue 2: Application Not Accessible

**Symptoms:**
- 502 Bad Gateway
- Connection timeout

**Solutions:**
```bash
# 1. Check service status
docker service ls
docker service ps my-web-app

# 2. Check Traefik logs
docker logs $(docker ps | grep traefik | awk '{print $1}') --tail 100

# 3. Verify DNS resolution
nslookup myapp.yourdomain.com

# 4. Check application logs
docker service logs my-web-app --tail 100

# 5. Verify port exposure
docker service inspect my-web-app | grep -A 5 "PublishedPort"

# 6. Test internal connectivity
docker exec -it $(docker ps | grep my-web-app | awk '{print $1}') curl localhost:3000
```

#### Issue 3: SSL Certificate Not Generating

**Symptoms:**
- Certificate errors in browser
- Let's Encrypt rate limit errors

**Solutions:**
```bash
# 1. Check Traefik ACME logs
docker logs $(docker ps | grep traefik | awk '{print $1}') 2>&1 | grep -i acme

# 2. Verify DNS is resolving correctly
dig +short myapp.yourdomain.com

# 3. Check if port 80 is reachable
curl -I http://myapp.yourdomain.com

# 4. Check Let's Encrypt rate limits
# Visit: https://crt.sh/?q=yourdomain.com

# 5. Switch to Let's Encrypt staging (if testing)
# Dokploy Settings → Traefik → Use Staging: Yes

# 6. Remove old certificates and retry
docker exec traefik rm /letsencrypt/acme.json
docker service update --force traefik
```

#### Issue 4: High CPU/Memory Usage

**Symptoms:**
- Slow response times
- Services crashing

**Solutions:**
```bash
# 1. Identify resource hogs
docker stats

# 2. Check service resource limits
docker service inspect my-web-app | grep -A 10 "Resources"

# 3. Update resource limits
docker service update --limit-memory 1024M --limit-cpu 1 my-web-app

# 4. Scale horizontally
docker service scale my-web-app=4

# 5. Check for memory leaks in application logs
docker service logs my-web-app | grep -i "memory\|oom"
```

#### Issue 5: Database Connection Issues

**Symptoms:**
- "ECONNREFUSED"
- "Authentication failed"

**Solutions:**
```bash
# 1. Verify database is running
docker service ls | grep postgres

# 2. Check database logs
docker service logs production-db --tail 100

# 3. Test connection from application container
docker exec -it $(docker ps | grep my-web-app | awk '{print $1}') \
  nc -zv production-db 5432

# 4. Verify credentials
# Check Dokploy UI → Databases → production-db → Credentials

# 5. Check network connectivity
docker network inspect dokploy-backend

# 6. Restart database service
docker service update --force production-db
```

#### Issue 6: Cross-Subnet Communication Failure

**Symptoms:**
- Worker on 192.168.1.x cannot join manager on 192.168.5.x
- Ping from worker to manager shows 100% packet loss
- Manager can ping worker, but worker cannot ping manager (or vice versa)
- `openssl s_client -connect MANAGER_IP:2377` times out

**Diagnosis:**
```bash
# On worker:
ping -c 3 MANAGER_IP
traceroute MANAGER_IP
ip route show
ip route get MANAGER_IP

# Check output - if route goes through different gateway, networks are isolated
```

**Solutions:**

**Option 1: Same Subnet (BEST - Simplest & Most Reliable)**
```bash
# Reconfigure one node to match the other's subnet
# Example: Move manager to worker's subnet

# On manager, edit network configuration:
sudo nano /etc/netplan/01-network-manager-all.yaml
# Change IP from 192.168.5.252 to 192.168.1.236 (or available IP)

# Apply changes:
sudo netplan apply

# Verify new IP:
ip addr show

# Reinitialize swarm with new IP:
docker swarm leave --force
docker swarm init --advertise-addr 192.168.1.236
docker swarm join-token worker

# Update Dokploy access:
# New URL: http://192.168.1.236:3000
```

**Option 2: VPN Overlay (BEST - for remote nodes)**
```bash
# Install Tailscale on both nodes:
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Get Tailscale IPs:
tailscale ip -4

# Use Tailscale IPs for swarm:
docker swarm init --advertise-addr <TAILSCALE_IP>
```

**Option 3: Router Configuration (Requires network admin access)**
- Access router at gateway IP
- Add static route: 192.168.5.0/24 ↔ 192.168.1.0/24
- Enable inter-VLAN routing if using VLANs

**Option 4: Static Routes (Only if same physical router)**
```bash
# This usually FAILS if networks use different gateways
# On worker:
sudo ip route add 192.168.5.0/24 via 192.168.1.1 dev wlo1

# Make permanent:
echo "192.168.5.0/24 via 192.168.1.1 dev wlo1" | sudo tee -a /etc/network/if-up.d/routes

# Test:
ping -c 3 MANAGER_IP
```

#### Issue 7: Swarm Node Down

**Symptoms:**
- Node shows as "Down" in docker node ls

**Solutions:**
```bash
# On affected node
# 1. Check Docker service
sudo systemctl status docker

# 2. Restart Docker if needed
sudo systemctl restart docker

# 3. Check system resources
htop
df -h

# 4. Review system logs
sudo journalctl -u docker -n 100

# On manager node
# 5. Check node availability
docker node inspect NODE_ID

# 6. Update node availability
docker node update --availability active NODE_ID

# 7. If permanently failed, remove and re-add
docker node demote NODE_ID
docker node rm NODE_ID
# Then rejoin from worker node
```

---

## Backup and Disaster Recovery

### Step 17: Backup Strategy

#### 17.1 Automated Database Backups (Configured in Step 14)

**Verify Backup Job:**
```bash
# Check backup logs in Dokploy UI
# Projects → Databases → production-db → Backups

# Manual backup trigger
docker exec production-db pg_dump -U dbadmin myapp_db > backup.sql
```

#### 17.2 Backup Dokploy Configuration
```bash
# On manager node
# Backup Dokploy data directory
sudo tar -czf dokploy-backup-$(date +%Y%m%d).tar.gz /var/lib/dokploy

# Backup Docker volumes
docker run --rm -v dokploy-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/dokploy-volumes-$(date +%Y%m%d).tar.gz /data

# Store backups offsite (S3, Backblaze, etc.)
aws s3 cp dokploy-backup-*.tar.gz s3://your-backup-bucket/dokploy/
```

#### 17.3 Disaster Recovery Plan

**Complete Cluster Failure:**
```bash
# 1. Provision new servers
# 2. Install Dokploy on new manager (Step 2)
# 3. Restore Dokploy configuration
sudo tar -xzf dokploy-backup-YYYYMMDD.tar.gz -C /

# 4. Restore Docker volumes
docker run --rm -v dokploy-data:/data -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/dokploy-volumes-YYYYMMDD.tar.gz --strip 1"

# 5. Restart Dokploy
docker restart dokploy

# 6. Join new worker nodes (Step 4-5)
# 7. Restore database from backup
# 8. Redeploy applications
```

---

## Performance Optimization

### Step 18: Optimize Cluster Performance

#### 18.1 Enable Docker BuildKit
```bash
# On manager node
export DOCKER_BUILDKIT=1
echo "export DOCKER_BUILDKIT=1" | sudo tee -a /etc/environment
```

#### 18.2 Configure Docker Registry Cache
```bash
# Deploy registry cache on manager
docker service create \
  --name registry-cache \
  --publish 5000:5000 \
  --mount type=volume,source=registry-cache,target=/var/lib/registry \
  registry:2

# Configure Docker daemon to use cache
sudo nano /etc/docker/daemon.json
```

**Add:**
```json
{
  "registry-mirrors": ["http://localhost:5000"]
}
```

#### 18.3 Optimize Application Placement
```bash
# Pin CPU-intensive apps to high-performance nodes
docker service update \
  --constraint-add "node.labels.cpu==high" \
  my-web-app

# Pin databases to nodes with SSD storage
docker service update \
  --constraint-add "node.labels.storage==ssd" \
  production-db
```

---

## Scaling Guide

### Step 19: Horizontal Scaling

#### 19.1 Scale Application Replicas
```bash
# Via Dokploy UI:
# Project → Application → Settings → Replicas: 4

# Via CLI:
docker service scale my-web-app=4

# Auto-scaling (requires custom setup)
# Monitor CPU and automatically scale between 2-10 replicas
```

#### 19.2 Add More Worker Nodes

**When to Scale:**
- Average CPU > 70%
- Memory usage > 80%
- Request latency increasing

**How to Add:**
1. Provision new physical server
2. Follow Steps 4-6 (Worker Servers Setup)
3. Applications will automatically distribute

---

## Advanced Configuration

### Step 20: Multi-Region Setup (Optional)

For globally distributed applications:

1. **Set up multiple manager nodes** (requires 3+ managers for quorum)
2. **Configure Docker Swarm overlay network** across regions
3. **Use GeoDNS** to route users to nearest region
4. **Replicate databases** across regions (PostgreSQL replication, MongoDB replica sets)

---

## Compliance & Logging

### Step 21: Centralized Logging

#### 21.1 Set Up Log Aggregation
```bash
# Deploy ELK stack or Loki for centralized logs
# Via Dokploy templates: Grafana Loki + Promtail

# Forward Docker logs to external system
sudo nano /etc/docker/daemon.json
```

**Add:**
```json
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "tcp://log-server:514",
    "tag": "{{.Name}}/{{.ID}}"
  }
}
```

---

## Final Checklist

### Production Readiness Checklist

- [ ] Manager node fully configured and accessible
- [ ] All worker nodes joined to swarm successfully
- [ ] Docker Swarm cluster verified (docker node ls shows all nodes)
- [ ] Firewall rules configured on all nodes
- [ ] SSH hardened with key-based auth only
- [ ] Fail2ban configured and running
- [ ] SSL certificates issued and auto-renewal working
- [ ] DNS properly configured and resolving
- [ ] Traefik routing traffic correctly
- [ ] Test application deployed and accessible
- [ ] Database deployed with automatic backups
- [ ] Monitoring enabled (Prometheus + Grafana)
- [ ] Alerts configured and tested
- [ ] Backup strategy implemented and tested
- [ ] Disaster recovery plan documented
- [ ] Security audit completed
- [ ] Documentation updated with your specific IPs/domains
- [ ] Team trained on Dokploy operations

---

## Support and Resources

### Official Documentation
- Dokploy Docs: https://docs.dokploy.com
- GitHub Repository: https://github.com/dokploy/dokploy
- Discord Community: https://discord.gg/2tBnJ3jDJc

### Docker Swarm Resources
- Docker Swarm Documentation: https://docs.docker.com/engine/swarm/
- Docker Security Best Practices: https://docs.docker.com/engine/security/

### Security Resources
- CIS Docker Benchmark: https://www.cisecurity.org/benchmark/docker
- OWASP Container Security: https://owasp.org/www-project-docker-top-10/

---

## Appendix

### A. Port Reference

| Port  | Protocol | Service                    | Open On        |
|-------|----------|----------------------------|----------------|
| 22    | TCP      | SSH                        | All nodes      |
| 80    | TCP      | HTTP                       | Manager        |
| 443   | TCP      | HTTPS                      | Manager        |
| 2377  | TCP      | Docker Swarm Management    | Manager        |
| 3000  | TCP      | Dokploy UI                 | Manager        |
| 4789  | UDP      | Docker Overlay Network     | All nodes      |
| 7946  | TCP/UDP  | Docker Node Communication  | All nodes      |
| 9323  | TCP      | Docker Metrics             | All nodes      |

### B. Sample docker-compose.yml for Dokploy

```yaml
version: '3.8'

services:
  my-web-app:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ./app:/app
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
    ports:
      - "3000:3000"
    command: "npm start"
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.yourdomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      - "traefik.http.services.myapp.loadbalancer.server.port=3000"
```

### C. Useful Commands Reference

```bash
# Swarm Management
docker node ls                          # List all nodes
docker node inspect NODE_ID             # Inspect node details
docker node update --availability drain NODE_ID  # Drain node for maintenance
docker node update --availability active NODE_ID # Reactivate node

# Service Management
docker service ls                       # List all services
docker service ps SERVICE_NAME          # List tasks for service
docker service logs SERVICE_NAME        # View service logs
docker service update SERVICE_NAME      # Update service
docker service scale SERVICE_NAME=5     # Scale service to 5 replicas
docker service rollback SERVICE_NAME    # Rollback to previous version

# Network Management
docker network ls                       # List networks
docker network inspect NETWORK_NAME     # Inspect network details
docker network create --driver overlay NETWORK_NAME  # Create overlay network

# Monitoring
docker stats                            # Real-time container stats
docker system df                        # Docker disk usage
docker system prune -a                  # Clean up unused resources

# Debugging
docker service ps --no-trunc SERVICE_NAME  # Show full error messages
docker inspect CONTAINER_ID             # Inspect container details
docker exec -it CONTAINER_ID sh         # Shell into container
```

---

**Document Version:** 1.0  
**Last Updated:** January 8, 2026  
**Maintained By:** Your DevOps Team  

---

## Notes

This guide assumes:
- You have physical servers or VPS instances ready
- You have a domain name configured
- You have basic Linux command-line knowledge
- All servers are on Ubuntu 22.04+ LTS

Adjust IP addresses, domain names, and configurations according to your specific environment.

**SECURITY WARNING:** Always use strong passwords, enable 2FA where possible, keep systems updated, and regularly audit your security configuration.
