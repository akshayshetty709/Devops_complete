# 🚀 Complete DevOps Engineer Hands-On Project
### AWS + Docker + Kubernetes (EKS) + Jenkins + Harbor + Prometheus + Grafana + Vault + GitHub

---

# TABLE OF CONTENTS

1. [Architecture Overview](#architecture-overview)
2. [Phase 1 — AWS EC2 Instance Setup](#phase-1)
3. [Phase 2 — Install & Configure Docker](#phase-2)
4. [Phase 3 — Install Jenkins](#phase-3)
5. [Phase 4 — Install Harbor Registry](#phase-4)
6. [Phase 5 — Setup EKS Cluster](#phase-5)
7. [Phase 6 — Install HashiCorp Vault](#phase-6)
8. [Phase 7 — Install Prometheus](#phase-7)
9. [Phase 8 — Install Grafana](#phase-8)
10. [Phase 9 — Sample Application & GitHub Setup](#phase-9)
11. [Phase 10 — Jenkins Pipeline (Full CI/CD)](#phase-10)
12. [Phase 11 — End-to-End Verification](#phase-11)

---

# ARCHITECTURE OVERVIEW

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER MACHINE                            │
│                    Git Push → GitHub Repo                           │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ Webhook Trigger
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│               EC2: Jenkins Server (t3.medium)                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Pipeline Stages:                                            │   │
│  │  1. Checkout Code from GitHub                               │   │
│  │  2. Build Docker Image                                      │   │
│  │  3. Scan Image (Trivy)                                      │   │
│  │  4. Push to Harbor Registry                                 │   │
│  │  5. Fetch Secrets from Vault                                │   │
│  │  6. Deploy to EKS                                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
└──────┬──────────────┬──────────────────────────┬────────────────────┘
       │              │                          │
       ▼              ▼                          ▼
┌──────────┐  ┌───────────────┐      ┌──────────────────────┐
│ EC2:     │  │ EC2: Harbor   │      │ EC2: Vault Server    │
│ EKS Node │  │ Registry      │      │ (Secrets Storage)    │
│ (Worker) │  │ (Docker imgs) │      └──────────────────────┘
└──────────┘  └───────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   EKS Cluster (Kubernetes)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐    │
│  │  App Pods   │  │  Services   │  │  Ingress Controller     │    │
│  │  (Node.js)  │  │  (ClusterIP │  │  (Load Balancer)        │    │
│  └─────────────┘  │  /NodePort) │  └─────────────────────────┘    │
│                   └─────────────┘                                   │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼ (metrics scrape)
┌──────────────────────┐      ┌──────────────────────┐
│ EC2: Prometheus      │─────▶│ EC2: Grafana          │
│ (Metrics Collection) │      │ (Dashboards & Alerts) │
└──────────────────────┘      └──────────────────────┘
```

## EC2 Instances We'll Create:

| Server          | Instance Type | OS           | Purpose                          |
|-----------------|---------------|--------------|----------------------------------|
| Jenkins Server  | t3.medium     | Ubuntu 22.04 | CI/CD Pipelines                  |
| Harbor Registry | t3.medium     | Ubuntu 22.04 | Docker Image Registry            |
| Vault Server    | t3.small      | Ubuntu 22.04 | Secrets Management               |
| Prometheus      | t3.small      | Ubuntu 22.04 | Metrics Collection               |
| Grafana         | t3.small      | Ubuntu 22.04 | Dashboards & Visualization       |
| EKS Nodes       | t3.medium x2  | Managed      | Kubernetes Worker Nodes          |

---

# PHASE 1 — AWS EC2 INSTANCE SETUP {#phase-1}

## 1.1 Architecture Explanation

An EC2 (Elastic Compute Cloud) instance is a virtual server in AWS. Think of it as renting a computer in Amazon's data center. We create these manually through the AWS Console — no automation yet — so you understand every piece before we automate later.

**Key Concepts:**
- **AMI** (Amazon Machine Image) = The OS template (like an ISO file)
- **Instance Type** = CPU + RAM spec (t3.medium = 2 vCPU, 4GB RAM)
- **Security Group** = Cloud firewall controlling inbound/outbound traffic
- **Key Pair** = SSH authentication (private key on your machine, public key on EC2)
- **VPC** = Your private network inside AWS
- **Elastic IP** = Static public IP that doesn't change on restart

## 1.2 Create a Key Pair First

```
AWS Console → EC2 → Key Pairs → Create Key Pair
Name: devops-project-key
Type: RSA
Format: .pem
```

Save the `.pem` file. Then on your LOCAL machine:

```bash
# Move key to SSH directory
mv ~/Downloads/devops-project-key.pem ~/.ssh/

# Restrict permissions (REQUIRED — SSH refuses keys with open permissions)
chmod 400 ~/.ssh/devops-project-key.pem

# Verify permissions
ls -la ~/.ssh/devops-project-key.pem
```

**Expected output:**
```
-r-------- 1 youruser yourgroup 1678 Jun 04 10:00 /home/youruser/.ssh/devops-project-key.pem
```

## 1.3 Create Security Groups

### Security Group: Jenkins-SG

```
AWS Console → EC2 → Security Groups → Create Security Group

Name: Jenkins-SG
Description: Security group for Jenkins CI/CD server
VPC: (select your default VPC)

Inbound Rules:
┌──────────┬──────────┬───────────┬─────────────────────────────┐
│ Type     │ Protocol │ Port      │ Source                      │
├──────────┼──────────┼───────────┼─────────────────────────────┤
│ SSH      │ TCP      │ 22        │ My IP (your home/office IP) │
│ Custom   │ TCP      │ 8080      │ 0.0.0.0/0 (Jenkins Web UI)  │
│ Custom   │ TCP      │ 50000     │ 0.0.0.0/0 (Jenkins agents)  │
└──────────┴──────────┴───────────┴─────────────────────────────┘
```

### Security Group: Harbor-SG

```
Inbound Rules:
┌──────────┬──────────┬───────────┬─────────────────────────────┐
│ Type     │ Protocol │ Port      │ Source                      │
├──────────┼──────────┼───────────┼─────────────────────────────┤
│ SSH      │ TCP      │ 22        │ My IP                       │
│ HTTPS    │ TCP      │ 443       │ 0.0.0.0/0                   │
│ HTTP     │ TCP      │ 80        │ 0.0.0.0/0                   │
└──────────┴──────────┴───────────┴─────────────────────────────┘
```

### Security Group: Monitoring-SG (for Prometheus + Grafana)

```
Inbound Rules:
┌──────────┬──────────┬───────────┬─────────────────────────────┐
│ Type     │ Protocol │ Port      │ Source                      │
├──────────┼──────────┼───────────┼─────────────────────────────┤
│ SSH      │ TCP      │ 22        │ My IP                       │
│ Custom   │ TCP      │ 9090      │ 0.0.0.0/0 (Prometheus UI)   │
│ Custom   │ TCP      │ 3000      │ 0.0.0.0/0 (Grafana UI)      │
│ Custom   │ TCP      │ 9100      │ 0.0.0.0/0 (Node Exporter)   │
└──────────┴──────────┴───────────┴─────────────────────────────┘
```

### Security Group: Vault-SG

```
Inbound Rules:
┌──────────┬──────────┬───────────┬─────────────────────────────┐
│ Type     │ Protocol │ Port      │ Source                      │
├──────────┼──────────┼───────────┼─────────────────────────────┤
│ SSH      │ TCP      │ 22        │ My IP                       │
│ Custom   │ TCP      │ 8200      │ Jenkins-SG (source = SG ID) │
│ Custom   │ TCP      │ 8200      │ My IP                       │
└──────────┴──────────┴───────────┴─────────────────────────────┘
```

## 1.4 Launch EC2 Instances

### Launch Jenkins Server

```
AWS Console → EC2 → Instances → Launch Instance

Name: Jenkins-Server
AMI: Ubuntu Server 22.04 LTS (ami-0c7217cdde317cfec in us-east-1)
Instance Type: t3.medium
Key Pair: devops-project-key
Network: Default VPC
Security Group: Jenkins-SG

Storage:
  Root Volume: 30 GB gp3

Advanced Details → User Data (paste this to auto-update on boot):
#!/bin/bash
apt-get update -y
hostnamectl set-hostname jenkins-server
```

**Repeat for all servers** with appropriate names and security groups:
- Harbor-Server → t3.medium → Harbor-SG → 30GB
- Vault-Server → t3.small → Vault-SG → 20GB
- Prometheus-Server → t3.small → Monitoring-SG → 20GB
- Grafana-Server → t3.small → Monitoring-SG → 20GB

## 1.5 Assign Elastic IPs

```
AWS Console → EC2 → Elastic IPs → Allocate Elastic IP → Associate

Associate each to:
- Jenkins-Server
- Harbor-Server
- Vault-Server
- Prometheus-Server
- Grafana-Server
```

**Why?** Without Elastic IP, the public IP changes every time the EC2 restarts, breaking all your configurations.

## 1.6 SSH into Your Instances

```bash
# SSH to Jenkins server (replace with YOUR Elastic IP)
ssh -i ~/.ssh/devops-project-key.pem ubuntu@<JENKINS_ELASTIC_IP>

# First time you'll see:
# The authenticity of host '1.2.3.4' can't be established.
# Type 'yes' to continue
```

**Expected output after login:**
```
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-1034-aws x86_64)

  System information as of Wed Jun  4 10:00:00 UTC 2026

  System load:  0.0               Processes:             107
  Usage of /:   5.4% of 29.01GB   Users logged in:       0
  Memory usage: 12%               IPv4 address for eth0: 172.31.x.x

ubuntu@jenkins-server:~$
```

## 1.7 Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Permission denied (publickey)` | Wrong key or permissions | Run `chmod 400 key.pem` |
| `Connection timed out` | Security group port 22 not open | Check SG inbound rules |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | Key permissions too open | Run `chmod 400 key.pem` |
| `Host key verification failed` | IP reused for new instance | Run `ssh-keygen -R <IP>` |

## 1.8 Interview Questions — EC2

**Q: What is the difference between a Security Group and a NACL?**
A: Security Groups are stateful (return traffic allowed automatically), operate at instance level, and allow only ALLOW rules. NACLs (Network Access Control Lists) are stateless (must allow both directions), operate at subnet level, and support both ALLOW and DENY rules.

**Q: What is an AMI?**
A: An Amazon Machine Image is a pre-configured template that contains the OS, software, and configuration needed to launch an EC2 instance. It's like a snapshot you can launch as many times as needed.

**Q: What is the difference between t3.micro and t3.medium?**
A: t3.micro has 1 vCPU and 1GB RAM, suitable for very light workloads. t3.medium has 2 vCPU and 4GB RAM, better for running services like Jenkins. Both use burstable CPU credits.

**Q: What happens to data on an EC2 instance when it's stopped?**
A: EBS (Elastic Block Store) volumes persist after stop. The public IP changes unless you use Elastic IP. Instance store (ephemeral) volumes are LOST. Data in RAM is lost.

---

# PHASE 2 — INSTALL & CONFIGURE DOCKER {#phase-2}

## 2.1 Architecture Explanation

Docker is a containerization platform. A **container** is a lightweight, isolated environment that packages your application with all its dependencies. Unlike a VM, containers share the host OS kernel.

```
WITHOUT DOCKER:                    WITH DOCKER:
┌─────────────────┐               ┌─────────────────────────────┐
│  Your App       │               │  Container A  │ Container B  │
│  (needs Node 14)│               │  Node 14      │  Python 3.9  │
│                 │               │  App v1       │  App v2      │
│  CONFLICTS with │               ├───────────────┴──────────────┤
│  Another App    │               │         Docker Engine        │
│  (needs Node 18)│               ├─────────────────────────────┤
└─────────────────┘               │    Ubuntu 22.04 (Host OS)   │
                                  └─────────────────────────────┘
```

**Key Docker Concepts:**
- **Image** = Blueprint/template (like a class in OOP)
- **Container** = Running instance of an image (like an object)
- **Dockerfile** = Instructions to build an image
- **Registry** = Storage for images (Harbor, Docker Hub)
- **Docker Compose** = Tool to run multi-container apps

## 2.2 Install Docker on ALL EC2 Instances

**SSH into each server and run this script:**

```bash
# ==============================================================
# DOCKER INSTALLATION SCRIPT
# Run this on: Jenkins, Harbor, Vault, Prometheus, Grafana servers
# ==============================================================

# Step 1: Update package index
sudo apt-get update -y

# Step 2: Install required dependencies
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https

# Step 3: Add Docker's official GPG key (verifies packages are genuine)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Step 4: Add Docker repository to apt sources
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: Update package index again (now includes Docker repo)
sudo apt-get update -y

# Step 6: Install Docker Engine, CLI, and plugins
sudo apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Step 7: Start Docker and enable it to start on boot
sudo systemctl start docker
sudo systemctl enable docker

# Step 8: Add ubuntu user to docker group (avoids needing sudo)
sudo usermod -aG docker ubuntu

# Step 9: Apply group change without logging out
newgrp docker
```

## 2.3 Verify Docker Installation

```bash
# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Run the hello-world test container
docker run hello-world

# Check running containers
docker ps

# Check all containers (including stopped)
docker ps -a

# Check Docker system info
docker info
```

**Expected output of `docker --version`:**
```
Docker version 26.1.4, build 5650f9b
```

**Expected output of `docker run hello-world`:**
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**Expected output of `sudo systemctl status docker`:**
```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled)
     Active: active (running) since Wed 2026-06-04 10:05:00 UTC; 1min ago
```

## 2.4 Docker Key Commands Reference

```bash
# IMAGE COMMANDS
docker images                    # List all local images
docker pull nginx:latest         # Download image from registry
docker build -t myapp:v1 .       # Build image from Dockerfile in current dir
docker rmi nginx:latest          # Remove an image
docker image prune               # Remove all unused images

# CONTAINER COMMANDS
docker run -d -p 8080:80 nginx          # Run nginx, detached, port 8080→80
docker run -it ubuntu:22.04 bash        # Interactive terminal
docker stop <container_id>              # Stop a running container
docker start <container_id>             # Start a stopped container
docker rm <container_id>                # Remove a stopped container
docker logs <container_id>              # View container logs
docker logs -f <container_id>           # Follow/tail container logs
docker exec -it <container_id> bash     # Shell into running container
docker inspect <container_id>           # Detailed container info (JSON)

# SYSTEM COMMANDS
docker system df                 # Show disk usage
docker system prune -a           # Remove everything unused (CAREFUL!)
docker stats                     # Live resource usage per container

# VOLUME COMMANDS
docker volume create mydata      # Create a persistent volume
docker volume ls                 # List volumes
docker run -v mydata:/data nginx # Mount volume into container

# NETWORK COMMANDS
docker network ls                # List networks
docker network create mynet      # Create custom network
docker network inspect mynet     # Inspect network details
```

## 2.5 Write Your First Dockerfile (Practice)

```bash
# Create a test directory
mkdir ~/docker-test && cd ~/docker-test

# Create a simple Node.js app
cat > app.js << 'EOF'
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from Docker Container!\n');
});
server.listen(3000, () => console.log('Server running on port 3000'));
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
# Base image — start FROM an official Node.js image
FROM node:18-alpine

# Set working directory inside container
WORKDIR /app

# Copy app files into the container
COPY app.js .

# Expose port 3000 (documentation only — must still -p flag when running)
EXPOSE 3000

# Command to run when container starts
CMD ["node", "app.js"]
EOF

# Build the image
docker build -t my-node-app:v1 .

# Run it
docker run -d -p 3000:3000 --name test-app my-node-app:v1

# Test it
curl http://localhost:3000

# Clean up
docker stop test-app && docker rm test-app
```

**Expected output of curl:**
```
Hello from Docker Container!
```

## 2.6 Common Docker Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `permission denied while trying to connect to Docker daemon` | User not in docker group | `sudo usermod -aG docker $USER && newgrp docker` |
| `port is already allocated` | Port in use by another process | Use different port: `-p 8081:80` |
| `Cannot connect to Docker daemon` | Docker not running | `sudo systemctl start docker` |
| `no space left on device` | Disk full from images | `docker system prune -a` |
| `COPY failed: no such file or directory` | File not found during build | Check file paths in Dockerfile |

## 2.7 Interview Questions — Docker

**Q: What is the difference between CMD and ENTRYPOINT in a Dockerfile?**
A: ENTRYPOINT defines the main executable that always runs and cannot be overridden (only appended to). CMD provides default arguments that can be overridden at runtime. Together: `ENTRYPOINT ["node"]` + `CMD ["app.js"]` → runs `node app.js`, but you can override with `docker run myimage server.js`.

**Q: What is Docker layer caching and why does it matter?**
A: Each Dockerfile instruction creates a layer. Docker caches layers and reuses them if the instruction and context haven't changed. This speeds up builds. Best practice: put things that change rarely (like `apt install`) before things that change often (like `COPY . .`).

**Q: Difference between COPY and ADD in Dockerfile?**
A: COPY simply copies files. ADD can also extract tar archives and fetch URLs. Best practice: use COPY unless you specifically need ADD's extra features.

**Q: How do containers communicate with each other?**
A: Containers on the same Docker network can communicate by container name (Docker's built-in DNS). Containers on different networks cannot communicate unless connected. Use `docker network create` and `--network` flag.

---

# PHASE 3 — INSTALL JENKINS {#phase-3}

## 3.1 Architecture Explanation

Jenkins is an open-source automation server. It automates the process of building, testing, and deploying software — this is called **CI/CD** (Continuous Integration / Continuous Delivery).

```
JENKINS CI/CD FLOW:
                                                          
 Developer        GitHub         Jenkins          Production
    │                │               │                │
    │  git push      │               │                │
    ├──────────────▶ │               │                │
    │                │   Webhook     │                │
    │                ├─────────────▶ │                │
    │                │               │ 1. Pull code   │
    │                │               │ 2. Run tests   │
    │                │               │ 3. Build image │
    │                │               │ 4. Push Harbor │
    │                │               │ 5. Deploy EKS  │
    │                │               ├──────────────▶ │
    │                │               │                │
```

**Key Jenkins Concepts:**
- **Job/Pipeline** = A configured automation task
- **Stage** = A step in the pipeline (Build, Test, Deploy)
- **Jenkinsfile** = Pipeline defined as code (stored in Git)
- **Agent** = The machine that runs the pipeline
- **Plugin** = Extension to add features (Docker, Kubernetes, Vault...)
- **Credential** = Securely stored secrets (passwords, keys)
- **Webhook** = GitHub notifies Jenkins automatically on push

## 3.2 Install Jenkins on Jenkins-Server

```bash
# SSH into Jenkins server
ssh -i ~/.ssh/devops-project-key.pem ubuntu@<JENKINS_ELASTIC_IP>

# ============================================================
# JENKINS INSTALLATION
# ============================================================

# Install Java (Jenkins requires Java 17+)
sudo apt-get update -y
sudo apt-get install -y fontconfig openjdk-17-jre

# Verify Java installation
java -version

# Add Jenkins repository key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt-get update -y
sudo apt-get install -y jenkins

# Start Jenkins and enable on boot
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins
```

**Expected output of `java -version`:**
```
openjdk version "17.0.9" 2023-10-17
OpenJDK Runtime Environment (build 17.0.9+9-Ubuntu-122.04)
OpenJDK 64-Bit Server VM (build 17.0.9+9, mixed mode, sharing)
```

**Expected output of `systemctl status jenkins`:**
```
● jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/lib/systemd/system/jenkins.service; enabled)
     Active: active (running) since Wed 2026-06-04 10:15:00 UTC
```

## 3.3 Initial Jenkins Setup

```bash
# Get the initial admin password (you NEED this for first login)
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Expected output:**
```
a1b2c3d4e5f6789012345678901234ab
```

Now in your browser:
```
http://<JENKINS_ELASTIC_IP>:8080
```

Follow the wizard:
1. Paste the initial admin password
2. Click "Install Suggested Plugins" (wait ~5 minutes)
3. Create your admin user
4. Set Jenkins URL to `http://<JENKINS_ELASTIC_IP>:8080`

## 3.4 Install Required Jenkins Plugins

```
Jenkins Dashboard → Manage Jenkins → Plugins → Available Plugins

Search and install (check each, then "Install"):
✅ Docker Pipeline
✅ Docker Commons
✅ Kubernetes
✅ Kubernetes CLI
✅ HashiCorp Vault
✅ Pipeline: Stage View
✅ Blue Ocean
✅ GitHub Integration
✅ Credentials Binding
✅ Git
✅ Pipeline Utility Steps
✅ Workspace Cleanup
```

After installing, restart Jenkins:
```bash
sudo systemctl restart jenkins
```

## 3.5 Give Jenkins Access to Docker

```bash
# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Restart Jenkins to apply group membership
sudo systemctl restart jenkins

# Verify
sudo -u jenkins docker ps
```

## 3.6 Install Additional Tools on Jenkins Server

```bash
# Install kubectl (to deploy to Kubernetes)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install AWS CLI (to interact with EKS)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install -y unzip
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Install Trivy (image security scanner)
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
    sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb \
    $(lsb_release -sc) main | \
    sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
trivy --version
```

## 3.7 Configure AWS Credentials on Jenkins Server

```bash
# On the Jenkins server, configure AWS credentials
# (so kubectl/aws cli can talk to EKS later)
aws configure
# Enter:
# AWS Access Key ID: (from IAM → Users → Security Credentials)
# AWS Secret Access Key: (same place)
# Default region: us-east-1
# Default output format: json
```

**How to get AWS credentials:**
```
AWS Console → IAM → Users → Your User → Security credentials →
Create access key → Choose "CLI" → Download CSV
```

## 3.8 Common Jenkins Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Can't access port 8080 | Security group missing rule | Add port 8080 to Jenkins-SG |
| `java.io.IOException: Cannot run program "docker"` | Jenkins can't find Docker | Add jenkins to docker group, restart |
| Plugin install fails | Network or proxy issue | Try again, or install manually via .hpi file |
| `initialAdminPassword` file not found | Jenkins not running | `sudo systemctl start jenkins` |

## 3.9 Interview Questions — Jenkins

**Q: What is the difference between Declarative and Scripted Pipeline?**
A: Declarative pipeline uses a structured syntax with predefined sections (pipeline, stages, steps) — easier to read and write, has built-in validation. Scripted pipeline uses Groovy code — more flexible and powerful but complex. For new projects, always use Declarative.

**Q: What is a Jenkins agent and why do you use multiple agents?**
A: An agent is the machine that executes pipeline steps. Using multiple agents allows parallel execution, different OS/environment requirements, and keeps the master node clean. You can have static agents (always on) or dynamic agents (spin up on demand in Docker/Kubernetes).

**Q: How do you store secrets in Jenkins?**
A: Using Jenkins Credentials Manager (Manage Jenkins → Credentials). Never hardcode passwords in Jenkinsfiles. Use `withCredentials` binding in pipelines: `withCredentials([usernamePassword(credentialsId: 'harbor-creds', ...)]) { ... }`

**Q: What is a webhook and how does it work with Jenkins?**
A: A webhook is an HTTP callback. GitHub sends a POST request to your Jenkins URL whenever code is pushed. Jenkins receives it and triggers the configured pipeline automatically, instead of Jenkins polling GitHub every X minutes.

---

# PHASE 4 — INSTALL HARBOR REGISTRY {#phase-4}

## 4.1 Architecture Explanation

Harbor is an enterprise-grade Docker image registry. It's like a private Docker Hub that you host yourself.

```
WHY HARBOR OVER DOCKER HUB?
┌─────────────────────────────────────────────────────┐
│  Docker Hub (Public)     │  Harbor (Self-hosted)    │
│  - Public by default     │  - Private by default    │
│  - Rate limited pulls    │  - No rate limits        │
│  - No vulnerability scan │  - Built-in Trivy scan   │
│  - No access control     │  - RBAC (role-based)     │
│  - Images leave company  │  - Images stay on-prem   │
└─────────────────────────────────────────────────────┘
```

**Harbor Components:**
- **Core** = API server and UI
- **Registry** = Actual Docker registry (distribution)
- **Database** = PostgreSQL (stores metadata)
- **Redis** = Cache
- **Trivy** = Built-in vulnerability scanner
- **Nginx** = Reverse proxy (handles HTTPS)

## 4.2 Install Docker Compose on Harbor Server

```bash
# SSH into Harbor server
ssh -i ~/.ssh/devops-project-key.pem ubuntu@<HARBOR_ELASTIC_IP>

# Docker Compose is already installed via docker-compose-plugin
# Verify:
docker compose version

# Expected output:
# Docker Compose version v2.27.0
```

## 4.3 Download and Install Harbor

```bash
# Download Harbor installer (check latest version at github.com/goharbor/harbor)
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz

# Extract
tar xzf harbor-offline-installer-v2.10.0.tgz

# Move to /opt
sudo mv harbor /opt/harbor
cd /opt/harbor

# List files
ls -la
```

**Expected output:**
```
-rw-r--r-- 1 root root 1094 harbor.yml.tmpl
-rwxr-xr-x 1 root root 2397 install.sh
-rwxr-xr-x 1 root root 3510 prepare
drwxr-xr-x 3 root root 4096 common
```

## 4.4 Configure Harbor

```bash
# Copy the template config
sudo cp harbor.yml.tmpl harbor.yml

# Edit the config
sudo nano harbor.yml
```

**Edit these key sections in harbor.yml:**

```yaml
# Replace with your Harbor server's Elastic IP
hostname: <HARBOR_ELASTIC_IP>

# For now, use HTTP (we'll note HTTPS setup)
# Comment out the https section:
# https:
#   port: 443
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path

# Change the http port
http:
  port: 80

# Set a strong admin password
harbor_admin_password: Harbor12345!

# Database password
database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

# Data directory (where images are stored)
data_volume: /data/harbor
```

```bash
# Create data directory
sudo mkdir -p /data/harbor

# Run the prepare script (generates config files)
sudo ./prepare

# Run the installer
sudo ./install.sh --with-trivy
```

**Expected output (last lines):**
```
[Step 5]: starting Harbor ...
[+] Running 10/10
 ✔ Container harbor-log       Started
 ✔ Container registry         Started
 ✔ Container harbor-db        Started
 ✔ Container redis             Started
 ✔ Container harbor-core      Started
 ✔ Container harbor-portal    Started
 ✔ Container harbor-jobservice Started
 ✔ Container nginx             Started
✔ ----Harbor has been installed and started successfully.----
```

## 4.5 Access Harbor UI

```
Browser: http://<HARBOR_ELASTIC_IP>
Username: admin
Password: Harbor12345! (what you set in harbor.yml)
```

## 4.6 Create a Harbor Project

```
Harbor UI → Projects → New Project

Project Name: devops-project
Access Level: Private
Storage Limit: -1 (unlimited)

Click OK
```

This creates a project namespace. Your images will be pushed as:
`<HARBOR_IP>/devops-project/myapp:v1`

## 4.7 Configure Docker to Trust Harbor (on Jenkins Server)

By default, Docker only trusts registries with valid SSL certificates. Since we're using HTTP, we need to mark Harbor as an insecure registry.

```bash
# On the JENKINS server:
sudo nano /etc/docker/daemon.json
```

Add:
```json
{
  "insecure-registries": ["<HARBOR_ELASTIC_IP>:80"]
}
```

```bash
# Restart Docker to apply
sudo systemctl restart docker

# Test login to Harbor from Jenkins server
docker login <HARBOR_ELASTIC_IP>:80
# Username: admin
# Password: Harbor12345!

# Expected output:
# WARNING! Your password will be stored unencrypted in /root/.docker/config.json
# Login Succeeded
```

## 4.8 Make Harbor Start Automatically After Reboot

```bash
# On Harbor server, create a systemd service
sudo nano /etc/systemd/system/harbor.service
```

```ini
[Unit]
Description=Harbor Container Registry
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/harbor
ExecStart=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f /opt/harbor/docker-compose.yml down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable harbor
```

## 4.9 Interview Questions — Harbor

**Q: What is image scanning in Harbor and why is it important?**
A: Harbor uses Trivy (or Clair) to scan Docker images for known CVEs (Common Vulnerabilities and Exposures) in OS packages and application dependencies. In a DevSecOps pipeline, you can configure Harbor to block deployment of images with Critical or High vulnerabilities — this is called "shift left" security.

**Q: What is a Harbor Project and how does RBAC work?**
A: A Harbor Project is a namespace for images. RBAC (Role-Based Access Control) in Harbor has roles: Guest (read-only), Developer (push/pull), Master (manage repos), Project Admin (full control). You assign users to roles per project.

**Q: How does Harbor handle image replication?**
A: Harbor supports replication rules to sync images between registries (Harbor-to-Harbor, Harbor-to-DockerHub, etc.) either on push (immediate) or on schedule. This is useful for DR (Disaster Recovery) or geo-distributed teams.

---

# PHASE 5 — SETUP EKS CLUSTER {#phase-5}

## 5.1 Architecture Explanation

EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes service. AWS manages the control plane (master nodes) — you only manage worker nodes.

```
EKS ARCHITECTURE:
┌──────────────────────────────────────────────────────────────┐
│                    AWS MANAGED                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              CONTROL PLANE                          │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │    │
│  │  │ API      │  │  etcd    │  │  Scheduler /     │  │    │
│  │  │ Server   │  │ (state)  │  │  Controller Mgr  │  │    │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
         │ kubectl commands reach here
         ▼
┌──────────────────────────────────────────────────────────────┐
│                    YOUR VPC (Worker Nodes)                   │
│  ┌────────────────────┐   ┌────────────────────┐            │
│  │  EC2: Worker Node 1│   │  EC2: Worker Node 2│            │
│  │  ┌──────┐ ┌──────┐ │   │  ┌──────┐ ┌──────┐ │            │
│  │  │ Pod  │ │ Pod  │ │   │  │ Pod  │ │ Pod  │ │            │
│  │  │(app) │ │(app) │ │   │  │(app) │ │(app) │ │            │
│  │  └──────┘ └──────┘ │   │  └──────┘ └──────┘ │            │
│  │  kubelet + kube-  │   │  kubelet + kube-   │            │
│  │  proxy            │   │  proxy             │            │
│  └────────────────────┘   └────────────────────┘            │
└──────────────────────────────────────────────────────────────┘
```

**Kubernetes Key Concepts (CRITICAL to understand):**
- **Pod** = Smallest deployable unit; runs 1+ containers
- **Deployment** = Manages Pods, handles rolling updates, replicas
- **Service** = Stable network endpoint for Pods (Pods change IPs, Services don't)
- **Namespace** = Virtual cluster within a cluster
- **ConfigMap** = Non-secret configuration data
- **Secret** = Sensitive data (base64 encoded)
- **Ingress** = HTTP routing rules (like a reverse proxy)
- **Node** = Physical/virtual machine running Pods
- **Control Plane** = The "brain" — schedules, manages state
- **etcd** = Key-value store; the cluster's source of truth
- **kubelet** = Agent on each node; talks to API server

## 5.2 Install eksctl (EKS Management Tool)

```bash
# On your LOCAL machine AND the Jenkins server:

# Install eksctl
curl --silent --location \
    "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | \
    tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

# Verify
eksctl version
```

**Expected output:**
```
0.180.0
```

## 5.3 Create IAM Role for EKS

Before creating the cluster, ensure your AWS user has the right permissions:

```
AWS Console → IAM → Users → Your User → Add Permissions

Attach these policies:
✅ AmazonEKSClusterPolicy
✅ AmazonEKSWorkerNodePolicy
✅ AmazonEC2ContainerRegistryReadOnly
✅ AmazonEKS_CNI_Policy
✅ AmazonEC2FullAccess
✅ IAMFullAccess
✅ CloudFormationFullAccess
```

## 5.4 Create the EKS Cluster

```bash
# This command creates:
# - EKS Control Plane
# - 2 worker nodes (t3.medium)
# - VPC, subnets, security groups
# - kubeconfig on your machine

eksctl create cluster \
  --name devops-cluster \
  --region us-east-1 \
  --nodegroup-name devops-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

# This takes 15-20 minutes. You'll see CloudFormation stacks being created.
```

**Expected output (final lines):**
```
2026-06-04 10:30:00 [ℹ]  building cluster stack "eksctl-devops-cluster-cluster"
2026-06-04 10:40:00 [ℹ]  building nodegroup stack "eksctl-devops-cluster-nodegroup-devops-nodes"
2026-06-04 10:45:00 [✔]  EKS cluster "devops-cluster" in "us-east-1" region is ready
```

## 5.5 Configure kubectl to Connect to EKS

```bash
# Update kubeconfig (tells kubectl where the cluster is)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name devops-cluster

# Verify connection
kubectl get nodes

# Verify cluster info
kubectl cluster-info
```

**Expected output of `kubectl get nodes`:**
```
NAME                            STATUS   ROLES    AGE   VERSION
ip-192-168-1-10.ec2.internal    Ready    <none>   5m    v1.29.1-eks-...
ip-192-168-1-11.ec2.internal    Ready    <none>   5m    v1.29.1-eks-...
```

## 5.6 Create Kubernetes Namespace for Our App

```bash
# Create a dedicated namespace for our app
kubectl create namespace devops-app

# Verify
kubectl get namespaces
```

**Expected output:**
```
NAME              STATUS   AGE
default           Active   10m
devops-app        Active   5s
kube-node-lease   Active   10m
kube-public       Active   10m
kube-system       Active   10m
```

## 5.7 Create Kubernetes Secret for Harbor Registry

```bash
# Kubernetes needs credentials to pull images from private Harbor registry
kubectl create secret docker-registry harbor-registry-secret \
  --docker-server=<HARBOR_ELASTIC_IP>:80 \
  --docker-username=admin \
  --docker-password=Harbor12345! \
  --docker-email=admin@example.com \
  --namespace=devops-app

# Verify
kubectl get secrets -n devops-app
```

## 5.8 Essential kubectl Commands Reference

```bash
# CLUSTER INFO
kubectl cluster-info                        # Cluster endpoints
kubectl get nodes                           # All nodes
kubectl describe node <node-name>           # Detailed node info
kubectl top nodes                           # CPU/Memory usage

# PODS
kubectl get pods -n devops-app              # Pods in namespace
kubectl get pods -A                         # All pods, all namespaces
kubectl describe pod <pod-name> -n devops-app   # Detailed pod info
kubectl logs <pod-name> -n devops-app       # Pod logs
kubectl logs -f <pod-name> -n devops-app    # Follow pod logs
kubectl exec -it <pod-name> -n devops-app -- bash  # Shell into pod
kubectl delete pod <pod-name> -n devops-app # Delete pod (Deployment recreates it)

# DEPLOYMENTS
kubectl get deployments -n devops-app       # List deployments
kubectl rollout status deployment/myapp -n devops-app  # Deployment status
kubectl rollout history deployment/myapp -n devops-app # Rollout history
kubectl rollout undo deployment/myapp -n devops-app    # Rollback!

# SERVICES
kubectl get services -n devops-app          # List services
kubectl describe service myapp -n devops-app # Service details

# APPLY MANIFESTS
kubectl apply -f deployment.yaml            # Create/update resources
kubectl delete -f deployment.yaml           # Delete resources
kubectl apply -f ./k8s/                     # Apply all files in directory
```

## 5.9 Interview Questions — Kubernetes / EKS

**Q: What is the difference between a Deployment and a StatefulSet?**
A: Deployment is for stateless applications — Pods are interchangeable, can be killed/recreated anywhere. StatefulSet is for stateful applications (databases) — Pods have stable network identities, stable storage, and ordered deployment/termination (pod-0, pod-1...).

**Q: What happens when a Pod crashes?**
A: If the Pod is managed by a Deployment, the ReplicaSet controller detects it's below the desired replica count and schedules a new Pod. The Scheduler places it on an available node. This is Kubernetes' self-healing capability.

**Q: What is the difference between ClusterIP, NodePort, and LoadBalancer services?**
A: ClusterIP (default) = only accessible within the cluster. NodePort = exposes on a static port on every node's IP (30000-32767), accessible from outside. LoadBalancer = provisions a cloud load balancer (AWS ALB/NLB), gets a public IP. Use ClusterIP for internal services, LoadBalancer for production public access.

**Q: What is etcd and what happens if it goes down?**
A: etcd is the distributed key-value store that is the cluster's "source of truth" — storing all cluster state, config, and secrets. If etcd goes down, the cluster can't make scheduling decisions or accept new API requests. Existing running workloads continue but can't be modified. This is why etcd is always run with 3 or 5 replicas (odd number for quorum).

**Q: What is the difference between EKS and self-managed Kubernetes?**
A: With EKS, AWS manages the control plane (high availability, patches, upgrades). You pay ~$0.10/hour per cluster. With self-managed, you run everything yourself — more control but huge operational overhead (etcd backups, master node HA, security patching, upgrades).

---

# PHASE 6 — INSTALL HASHICORP VAULT {#phase-6}

## 6.1 Architecture Explanation

HashiCorp Vault is a secrets management tool. Instead of hardcoding passwords in code or config files, applications fetch secrets from Vault dynamically.

```
WITHOUT VAULT (BAD):                WITH VAULT (GOOD):
                                    
Jenkinsfile:                        Jenkinsfile:
  DB_PASS="super_secret"              withVault(path:'secret/db') {
  HARBOR_TOKEN="abc123"                 // secrets fetched at runtime
                                      }
Problems:                           Benefits:
- Secret in Git history            - Secrets never in code
- Everyone can see it              - Access controlled per app/role
- Can't rotate easily              - Full audit log
- One breach = all secrets gone    - Auto-rotation possible
                                   - TTL (secrets expire)
```

**Vault Key Concepts:**
- **Secret Engine** = Plugin that manages a category of secrets (KV, database, AWS, PKI)
- **KV Engine** = Simple key-value storage for secrets
- **Policy** = Rules defining what paths an entity can access
- **Token** = Authentication credential (like a session key)
- **AppRole** = Machine-to-machine authentication (Jenkins authenticates this way)
- **Seal/Unseal** = Vault starts "sealed" (encrypted) and must be "unsealed" with keys

## 6.2 Install Vault on Vault Server

```bash
# SSH into Vault server
ssh -i ~/.ssh/devops-project-key.pem ubuntu@<VAULT_ELASTIC_IP>

# Add HashiCorp repository
wget -O- https://apt.releases.hashicorp.com/gpg | \
    sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update
sudo apt-get install -y vault

# Verify
vault version
```

**Expected output:**
```
Vault v1.16.2 (build date: ...)
```

## 6.3 Configure Vault

```bash
# Create Vault configuration directory
sudo mkdir -p /etc/vault.d
sudo mkdir -p /opt/vault/data

# Create Vault config file
sudo nano /etc/vault.d/vault.hcl
```

```hcl
# /etc/vault.d/vault.hcl

# Storage backend — where Vault stores its encrypted data
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"
}

# Listener — where Vault listens for connections
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_disable   = "true"   # Disable TLS for lab (NEVER in production!)
}

# UI enabled
ui = true

# Cluster address
cluster_addr  = "http://<VAULT_ELASTIC_IP>:8201"
api_addr      = "http://<VAULT_ELASTIC_IP>:8200"
```

## 6.4 Create Vault Systemd Service

```bash
sudo nano /etc/systemd/system/vault.service
```

```ini
[Unit]
Description=HashiCorp Vault - A tool for managing secrets
Documentation=https://www.vaultproject.io/docs
After=network-online.target
Wants=network-online.target

[Service]
User=vault
Group=vault
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# Create vault user
sudo useradd --system --home /etc/vault.d --shell /bin/false vault

# Set ownership
sudo chown -R vault:vault /opt/vault
sudo chown -R vault:vault /etc/vault.d

# Start Vault
sudo systemctl daemon-reload
sudo systemctl start vault
sudo systemctl enable vault
sudo systemctl status vault
```

## 6.5 Initialize and Unseal Vault

```bash
# Set Vault address
export VAULT_ADDR='http://<VAULT_ELASTIC_IP>:8200'
echo 'export VAULT_ADDR="http://<VAULT_ELASTIC_IP>:8200"' >> ~/.bashrc

# Initialize Vault (run ONCE — generates master keys)
vault operator init

# ⚠️  SAVE THIS OUTPUT — YOU CANNOT GET IT AGAIN ⚠️
```

**Expected output:**
```
Unseal Key 1: abc123def456...
Unseal Key 2: xyz789uvw012...
Unseal Key 3: mno345pqr678...
Unseal Key 4: stu901vwx234...
Unseal Key 5: yza567bcd890...

Initial Root Token: hvs.CAESIB...

Vault initialized with 5 key shares and a threshold of 3.
```

**SAVE THESE KEYS SECURELY.** In production, distribute unseal keys to different people.

```bash
# Unseal Vault (need 3 of 5 keys — run this 3 times with different keys)
vault operator unseal   # Paste Unseal Key 1
vault operator unseal   # Paste Unseal Key 2
vault operator unseal   # Paste Unseal Key 3

# Login with root token
vault login hvs.CAESIB...

# Check status
vault status
```

**Expected output of `vault status`:**
```
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            5
Threshold               3
Version                 1.16.2
Storage Type            raft
Cluster Name            vault-cluster-abc123
HA Enabled              true
HA Cluster              https://127.0.0.1:8201
HA Mode                 active
```

## 6.6 Configure Vault for Jenkins (Store CI/CD Secrets)

```bash
# Enable the KV (Key-Value) secrets engine at path "secret/"
vault secrets enable -path=secret kv-v2

# Store Harbor credentials
vault kv put secret/harbor \
  username="admin" \
  password="Harbor12345!" \
  url="http://<HARBOR_ELASTIC_IP>:80"

# Store other app secrets
vault kv put secret/myapp \
  db_password="mydbpassword123" \
  api_key="myapikey456" \
  jwt_secret="myjwtsecret789"

# Verify secrets were stored
vault kv get secret/harbor
vault kv get secret/myapp
```

**Expected output of `vault kv get secret/harbor`:**
```
====== Secret Path ======
secret/data/harbor

======= Metadata =======
Key              Value
---              -----
created_time     2026-06-04T10:50:00.000000000Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    Harbor12345!
url         http://x.x.x.x:80
username    admin
```

## 6.7 Create AppRole for Jenkins Authentication

```bash
# Enable AppRole auth method
vault auth enable approle

# Create a policy for Jenkins (what Jenkins is allowed to do)
vault policy write jenkins-policy - << 'EOF'
# Allow Jenkins to read Harbor secrets
path "secret/data/harbor" {
  capabilities = ["read"]
}

# Allow Jenkins to read app secrets
path "secret/data/myapp" {
  capabilities = ["read"]
}
EOF

# Create an AppRole for Jenkins
vault write auth/approle/role/jenkins-role \
  token_policies="jenkins-policy" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=720h

# Get the Role ID (not secret, can be shared)
vault read auth/approle/role/jenkins-role/role-id

# Get the Secret ID (treat like a password!)
vault write -f auth/approle/role/jenkins-role/secret-id
```

**Save the Role ID and Secret ID — you'll need them in Jenkins.**

## 6.8 Configure Vault in Jenkins

```
Jenkins → Manage Jenkins → Credentials → System → Global → Add Credentials

Kind: Vault App Role Credential
Scope: Global
Role ID: (paste Role ID from above)
Secret ID: (paste Secret ID from above)
ID: vault-approle-credentials
Description: HashiCorp Vault AppRole for Jenkins
```

```
Jenkins → Manage Jenkins → System → HashiCorp Vault Plugin

Vault URL: http://<VAULT_ELASTIC_IP>:8200
Vault Credential: vault-approle-credentials (just created above)
```

## 6.9 Interview Questions — Vault

**Q: What is the difference between Vault seal and Vault lock?**
A: "Sealed" means Vault's encryption key is in memory but the master key is gone — no secrets can be read. You must unseal with Shamir key shares. Locking (revoking a token) just ends a session. After a restart, Vault is sealed again and must be manually unsealed.

**Q: What is the Shamir Secret Sharing scheme in Vault?**
A: Vault splits the master key into N shares (default 5) where any K shares (default 3 = threshold) can reconstruct it. This means no single person has full control. In enterprises, distribute keys to different executives/security officers.

**Q: What is the difference between KV v1 and KV v2?**
A: KV v2 adds versioning — every write creates a new version, old versions are preserved (up to 10 by default). You can roll back to previous versions. KV v1 has no versioning; writes overwrite previous data. KV v2 also has metadata tracking.

**Q: How would you handle Vault high availability in production?**
A: Run Vault with Integrated Storage (Raft) with 3 or 5 nodes. One node is active (handles all requests), others are standby (can be promoted on failure). Auto-unseal with AWS KMS means Vault automatically unseals after restarts without manual key entry.

---

# PHASE 7 — INSTALL PROMETHEUS {#phase-7}

## 7.1 Architecture Explanation

Prometheus is an open-source monitoring and alerting system. It scrapes (pulls) metrics from targets at regular intervals.

```
PROMETHEUS ARCHITECTURE:

  Application Pods         Node Exporter        Jenkins
  (with /metrics)          (system metrics)     (with plugin)
        │                        │                   │
        └───────────────────────┬┘                   │
                                │ HTTP scrape         │
                                ▼ every 15s           │
                    ┌──────────────────────────┐      │
                    │    PROMETHEUS SERVER      │◄─────┘
                    │  ┌────────────────────┐  │
                    │  │  Time-Series DB    │  │
                    │  │  (TSDB)            │  │
                    │  └────────────────────┘  │
                    │  ┌────────────────────┐  │
                    │  │  Alert Manager     │  │
                    │  │  (PagerDuty/Slack) │  │
                    │  └────────────────────┘  │
                    └──────────────┬───────────┘
                                   │
                                   ▼ PromQL queries
                         ┌──────────────────┐
                         │    GRAFANA       │
                         │  (Dashboards)    │
                         └──────────────────┘
```

**Key Prometheus Concepts:**
- **Metrics** = Numerical data points over time (CPU %, request count, memory)
- **Scrape** = Prometheus fetching metrics from a target endpoint
- **Target** = An endpoint Prometheus monitors (e.g., `:9090/metrics`)
- **PromQL** = Prometheus Query Language (for querying time-series data)
- **Exporter** = An agent that exposes metrics in Prometheus format
  - Node Exporter = System/OS metrics (CPU, RAM, disk, network)
  - cAdvisor = Container metrics
  - kube-state-metrics = Kubernetes object metrics
- **Alert Rule** = PromQL expression that triggers an alert

**Four Metric Types:**
- **Counter** = Only goes up (total requests, errors) — use `rate()` to get per-second
- **Gauge** = Goes up and down (current memory, queue size)
- **Histogram** = Samples observations and counts in buckets (request latency)
- **Summary** = Similar to histogram, pre-calculates quantiles

## 7.2 Install Prometheus

```bash
# SSH into Prometheus server
ssh -i ~/.ssh/devops-project-key.pem ubuntu@<PROMETHEUS_ELASTIC_IP>

# Create prometheus user (security best practice — run as non-root)
sudo useradd --no-create-home --shell /bin/false prometheus

# Create directories
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus

# Download Prometheus (check latest version at prometheus.io/download)
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.51.0/prometheus-2.51.0.linux-amd64.tar.gz

# Extract
tar xzf prometheus-2.51.0.linux-amd64.tar.gz
cd prometheus-2.51.0.linux-amd64

# Copy binaries
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

# Copy config files
sudo cp -r consoles /etc/prometheus/
sudo cp -r console_libraries /etc/prometheus/

# Set ownership
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

# Verify
prometheus --version
```

**Expected output:**
```
prometheus, version 2.51.0 (branch: HEAD, ...)
```

## 7.3 Configure Prometheus

```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval:     15s   # How often to scrape targets
  evaluation_interval: 15s  # How often to evaluate alert rules

# Rule files for alerts
rule_files:
  - "/etc/prometheus/rules/*.yml"

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: []  # Add alertmanager later

# Scrape configurations
scrape_configs:

  # Prometheus monitors itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Jenkins metrics
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<JENKINS_ELASTIC_IP>:8080']

  # Harbor metrics
  - job_name: 'harbor'
    static_configs:
      - targets: ['<HARBOR_ELASTIC_IP>:9090']

  # Node Exporter - system metrics for all EC2s
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - '<JENKINS_ELASTIC_IP>:9100'
          - '<HARBOR_ELASTIC_IP>:9100'
          - '<VAULT_ELASTIC_IP>:9100'
          - '<PROMETHEUS_ELASTIC_IP>:9100'
          - '<GRAFANA_ELASTIC_IP>:9100'
        labels:
          environment: 'devops-lab'

  # Kubernetes metrics via kube-state-metrics
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

## 7.4 Create Prometheus Systemd Service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus Monitoring System
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090 \
    --storage.tsdb.retention.time=15d \
    --web.enable-lifecycle
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# Start Prometheus
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus

# Verify config is valid
promtool check config /etc/prometheus/prometheus.yml
```

**Expected promtool output:**
```
Checking /etc/prometheus/prometheus.yml
  SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

## 7.5 Install Node Exporter on ALL Servers

Run this on every EC2 instance you want to monitor:

```bash
# Download Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Create user
sudo useradd --no-create-home --shell /bin/false node_exporter

# Create service
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

# Test — should return lots of metrics
curl http://localhost:9100/metrics | head -20
```

## 7.6 Access Prometheus UI

```
Browser: http://<PROMETHEUS_ELASTIC_IP>:9090

Explore:
- Status → Targets (see all scrape targets)
- Graph (run PromQL queries)
- Alerts
```

**Test PromQL queries:**
```promql
# CPU usage across all nodes
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage percentage
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# HTTP requests per second (Jenkins)
rate(http_requests_total[5m])
```

## 7.7 Interview Questions — Prometheus

**Q: What is the difference between push and pull monitoring? Why does Prometheus use pull?**
A: Push = agents send metrics to the monitoring server. Pull (scrape) = monitoring server fetches metrics from agents. Prometheus uses pull because: easier to detect when a target goes down (no scrape = alert), no need to configure each agent with the server's address, server controls scrape rate, and easier debugging.

**Q: How does Prometheus store data?**
A: Prometheus uses a custom time-series database (TSDB). Data is stored in blocks on disk, indexed by metric name and labels. Recent data (last 2 hours) is kept in memory. Old data is compacted into larger blocks. Default retention is 15 days.

**Q: What is a PromQL rate() function and when do you use it?**
A: `rate(counter[5m])` calculates the per-second average rate of increase of a counter over the last 5 minutes. You use it with Counter metrics because counters only go up — rate() gives you the actual change rate. For example, `rate(http_requests_total[5m])` gives requests per second.

**Q: What is the difference between recording rules and alert rules?**
A: Recording rules pre-compute expensive PromQL queries and store the result as new metrics — improving query performance. Alert rules evaluate PromQL expressions and fire alerts when they're true for a duration. Both are defined in rule files.

---

# PHASE 8 — INSTALL GRAFANA {#phase-8}

## 8.1 Architecture Explanation

Grafana is a visualization and analytics platform. It queries Prometheus (and other data sources) and displays beautiful dashboards.

```
GRAFANA DATA FLOW:

Prometheus ──────────────▶ Grafana
(Data Source)              │
                           │  Dashboards show:
                           ├─ CPU/RAM/Disk charts
                           ├─ Application metrics
                           ├─ Kubernetes metrics
                           ├─ CI/CD pipeline stats
                           └─ Custom alerts (email/Slack)
```

## 8.2 Install Grafana on Grafana Server

```bash
# SSH into Grafana server
ssh -i ~/.ssh/devops-project-key.pem ubuntu@<GRAFANA_ELASTIC_IP>

# Install prerequisites
sudo apt-get install -y software-properties-common

# Add Grafana repository
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"

# Install Grafana
sudo apt-get update
sudo apt-get install -y grafana

# Start and enable
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

**Expected status output:**
```
● grafana-server.service - Grafana instance
     Active: active (running) since ...
```

## 8.3 Access and Configure Grafana

```
Browser: http://<GRAFANA_ELASTIC_IP>:3000

Default credentials:
Username: admin
Password: admin
(You'll be prompted to change password on first login)
```

### Add Prometheus as Data Source

```
Grafana → Connections → Data sources → Add data source → Prometheus

HTTP URL: http://<PROMETHEUS_ELASTIC_IP>:9090
Name: Prometheus-DevOps
Default: ✅ (toggle on)

Click "Save & test"
Expected: ✅ "Successfully queried the Prometheus API."
```

## 8.4 Import Pre-built Dashboards

Grafana has a community library of pre-built dashboards. Import these:

```
Grafana → Dashboards → New → Import

Dashboard 1 — Node Exporter Full
  Dashboard ID: 1860
  Click "Load"
  Select Prometheus data source
  Click "Import"

Dashboard 2 — Jenkins Performance
  Dashboard ID: 9964
  Click "Load" → Import

Dashboard 3 — Docker Container Metrics
  Dashboard ID: 893
  Click "Load" → Import

Dashboard 4 — Kubernetes Cluster
  Dashboard ID: 6417
  Click "Load" → Import
```

## 8.5 Create a Custom Dashboard

```
Grafana → Dashboards → New → New Dashboard → Add visualization

Panel 1: CPU Usage
  Query: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  Title: CPU Usage %
  Visualization: Time series
  Unit: Percent (0-100)

Panel 2: Memory Usage
  Query: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
  Title: Memory Usage %
  Visualization: Gauge
  Min: 0, Max: 100

Panel 3: Disk Usage
  Query: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
  Title: Disk Usage %
  Visualization: Bar chart

Save dashboard as: "DevOps Infrastructure Overview"
```

## 8.6 Configure Alerts in Grafana

```
Grafana → Alerting → Alert rules → New alert rule

Rule name: High CPU Alert
Query:
  A: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

Conditions:
  WHEN A IS ABOVE 80 FOR 5m

Labels:
  severity: warning
  team: devops

Annotations:
  Summary: CPU usage is above 80%
  Description: Instance {{ $labels.instance }} CPU is {{ $values.A }}%

Folder: DevOps Alerts
Evaluation group: infrastructure
```

## 8.7 Interview Questions — Grafana

**Q: What is the difference between Grafana and Kibana?**
A: Grafana is primarily for time-series metrics visualization (Prometheus, InfluxDB) and infrastructure monitoring. Kibana is part of the ELK stack and is primarily for log analysis and full-text search (Elasticsearch). Modern Grafana supports logs via Loki, making it a more complete platform.

**Q: What is Grafana Loki and how does it relate to Prometheus?**
A: Loki is Grafana's log aggregation system, inspired by Prometheus but for logs instead of metrics. Where Prometheus scrapes metrics, Promtail (Loki's agent) collects logs and pushes to Loki. In Grafana, you can correlate metrics and logs on the same dashboard using the same label scheme.

**Q: What is the difference between a panel query and an alert query in Grafana?**
A: Panel queries return time-series data for visualization. Alert queries evaluate to a single number (scalar) at a point in time and compare to a threshold. The same PromQL can be used for both, but alert queries need to be reduced to a single value using functions like `avg()`, `max()`, `last()`.

---

# PHASE 9 — SAMPLE APPLICATION & GITHUB SETUP {#phase-9}

## 9.1 Create a Sample Node.js Application

On your LOCAL machine (or Jenkins server):

```bash
mkdir devops-sample-app && cd devops-sample-app
```

### app.js

```javascript
// app.js — Simple Node.js web application
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;
const APP_VERSION = process.env.APP_VERSION || '1.0.0';

app.get('/', (req, res) => {
  res.json({
    message: 'Welcome to DevOps Sample App!',
    version: APP_VERSION,
    environment: process.env.NODE_ENV || 'development',
    timestamp: new Date().toISOString(),
    hostname: require('os').hostname()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy', uptime: process.uptime() });
});

app.get('/metrics', (req, res) => {
  // Expose basic metrics for Prometheus
  const metrics = [
    '# HELP app_requests_total Total number of requests',
    '# TYPE app_requests_total counter',
    `app_requests_total{path="/"} ${requestCount}`,
    '# HELP app_uptime_seconds Application uptime in seconds',
    '# TYPE app_uptime_seconds gauge',
    `app_uptime_seconds ${process.uptime()}`
  ].join('\n');
  
  res.set('Content-Type', 'text/plain');
  res.send(metrics);
});

let requestCount = 0;
app.use((req, res, next) => { requestCount++; next(); });

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Version: ${APP_VERSION}`);
});
```

### package.json

```json
{
  "name": "devops-sample-app",
  "version": "1.0.0",
  "description": "Sample app for DevOps project",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo 'Tests passed' && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

### Dockerfile

```dockerfile
# ---- Build Stage ----
FROM node:18-alpine AS builder

WORKDIR /app

# Copy dependency files first (layer caching optimization)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# ---- Production Stage ----
FROM node:18-alpine

# Security: run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy from builder stage
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "app.js"]
```

### .dockerignore

```
node_modules
npm-debug.log
.git
.gitignore
README.md
*.test.js
```

### .gitignore

```
node_modules/
.env
*.log
.DS_Store
```

## 9.2 Create Kubernetes Manifests

```bash
mkdir k8s && cd k8s
```

### k8s/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-app
  labels:
    app: devops-sample-app
```

### k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-sample-app
  namespace: devops-app
  labels:
    app: devops-sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-sample-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # Zero downtime: never remove a pod before new one is ready
      maxSurge: 1            # Can temporarily have 1 extra pod during update
  template:
    metadata:
      labels:
        app: devops-sample-app
      annotations:
        prometheus.io/scrape: "true"   # Tell Prometheus to scrape this pod
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      imagePullSecrets:
        - name: harbor-registry-secret  # Secret we created earlier
      containers:
        - name: devops-sample-app
          image: HARBOR_IMAGE_PLACEHOLDER  # Jenkins replaces this
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: PORT
              value: "3000"
            - name: NODE_ENV
              value: "production"
            - name: APP_VERSION
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['version']
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
```

### k8s/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: devops-sample-app-service
  namespace: devops-app
  labels:
    app: devops-sample-app
spec:
  type: LoadBalancer      # Creates an AWS ALB with public IP
  selector:
    app: devops-sample-app
  ports:
    - name: http
      protocol: TCP
      port: 80             # External port
      targetPort: 3000     # Container port
```

## 9.3 Push to GitHub

```bash
# Back in project root
cd ~/devops-sample-app

# Initialize git
git init
git add .
git commit -m "Initial commit: DevOps sample application"

# Create GitHub repo:
# Go to github.com → New Repository
# Name: devops-sample-app
# Private: ✅
# Don't initialize (we already have code)

# Add remote and push
git remote add origin https://github.com/<YOUR_USERNAME>/devops-sample-app.git
git branch -M main
git push -u origin main
```

## 9.4 Setup GitHub Webhook for Jenkins

```
GitHub → Your Repo → Settings → Webhooks → Add webhook

Payload URL: http://<JENKINS_ELASTIC_IP>:8080/github-webhook/
Content type: application/json
Secret: (leave blank for now)
Which events: Just the push event
Active: ✅

Click "Add webhook"
```

GitHub will send a ping and you should see a green ✅ if Jenkins is reachable.

---

# PHASE 10 — JENKINS PIPELINE (FULL CI/CD) {#phase-10}

## 10.1 Architecture Explanation

The Jenkinsfile defines our entire CI/CD pipeline as code. This means:
- Pipeline is version-controlled in Git
- Anyone can review/change the pipeline with a PR
- Pipeline is reproducible

```
PIPELINE STAGES:

  [1. Checkout]
  Pull code from GitHub
       │
       ▼
  [2. Install Dependencies]
  npm install
       │
       ▼
  [3. Run Tests]
  npm test
       │
       ▼
  [4. Build Docker Image]
  docker build -t devops-sample-app:${BUILD_NUMBER}
       │
       ▼
  [5. Security Scan]
  trivy image (scan for CVEs)
       │
       ▼
  [6. Push to Harbor]
  docker push harbor_ip/devops-project/app:${BUILD_NUMBER}
       │
       ▼
  [7. Deploy to Kubernetes]
  kubectl set image deployment/...
       │
       ▼
  [8. Verify Deployment]
  kubectl rollout status
       │
       ▼
  [9. Post (Notify)]
  Cleanup + Slack/Email notification
```

## 10.2 Add Jenkins Credentials

```
Jenkins → Manage Jenkins → Credentials → Global → Add Credential

Credential 1: Harbor Registry
  Kind: Username with password
  Username: admin
  Password: Harbor12345!
  ID: harbor-credentials
  Description: Harbor Registry Credentials

Credential 2: Kubernetes Config
  Kind: Secret file
  File: (upload your ~/.kube/config file)
  ID: kubeconfig
  Description: EKS Kubernetes Config

Credential 3: GitHub
  Kind: Username with password (or SSH key)
  Username: your-github-username
  Password: your-github-personal-access-token
  ID: github-credentials
```

**Get GitHub PAT:**
```
GitHub → Settings → Developer Settings →
Personal Access Tokens → Tokens (classic) → Generate new token
Scopes: repo, admin:repo_hook
```

## 10.3 The Complete Jenkinsfile

Create this file in your project root as `Jenkinsfile`:

```groovy
pipeline {
    // Run on any available agent (Jenkins worker)
    agent any

    // Environment variables available throughout pipeline
    environment {
        // Harbor Registry settings
        HARBOR_URL        = '<HARBOR_ELASTIC_IP>:80'
        HARBOR_PROJECT    = 'devops-project'
        IMAGE_NAME        = 'devops-sample-app'
        IMAGE_TAG         = "${BUILD_NUMBER}"
        FULL_IMAGE        = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"

        // Kubernetes settings
        K8S_NAMESPACE     = 'devops-app'
        K8S_DEPLOYMENT    = 'devops-sample-app'

        // Vault settings
        VAULT_ADDR        = 'http://<VAULT_ELASTIC_IP>:8200'
    }

    // Pipeline options
    options {
        timestamps()                    // Add timestamps to console output
        timeout(time: 30, unit: 'MINUTES') // Fail if pipeline takes > 30 min
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keep last 10 builds
        disableConcurrentBuilds()       // Don't run multiple builds simultaneously
    }

    stages {

        // ========================================================
        // STAGE 1: Checkout code from GitHub
        // ========================================================
        stage('Checkout') {
            steps {
                echo '📥 Checking out source code from GitHub...'
                checkout scm   // 'scm' = the repo configured in Jenkins job

                // Show what commit we're building
                sh 'git log --oneline -5'
                sh 'git status'
            }
        }

        // ========================================================
        // STAGE 2: Install Dependencies
        // ========================================================
        stage('Install Dependencies') {
            steps {
                echo '📦 Installing Node.js dependencies...'
                sh 'npm ci'   // 'ci' = clean install, respects package-lock.json

                // Show what was installed
                sh 'npm list --depth=0'
            }
        }

        // ========================================================
        // STAGE 3: Run Tests
        // ========================================================
        stage('Test') {
            steps {
                echo '🧪 Running tests...'
                sh 'npm test'
            }
            post {
                failure {
                    echo '❌ Tests FAILED — stopping pipeline'
                }
            }
        }

        // ========================================================
        // STAGE 4: Fetch Secrets from Vault
        // ========================================================
        stage('Fetch Secrets from Vault') {
            steps {
                echo '🔐 Fetching secrets from HashiCorp Vault...'

                withVault([
                    configuration: [
                        vaultUrl: "${VAULT_ADDR}",
                        vaultCredentialId: 'vault-approle-credentials',
                        engineVersion: 2
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/harbor',
                            engineVersion: 2,
                            secretValues: [
                                [envVar: 'HARBOR_USERNAME', vaultKey: 'username'],
                                [envVar: 'HARBOR_PASSWORD', vaultKey: 'password']
                            ]
                        ],
                        [
                            path: 'secret/myapp',
                            engineVersion: 2,
                            secretValues: [
                                [envVar: 'DB_PASSWORD', vaultKey: 'db_password'],
                                [envVar: 'API_KEY', vaultKey: 'api_key']
                            ]
                        ]
                    ]
                ]) {
                    // Verify we got the secrets (print lengths, never values!)
                    sh 'echo "Harbor user length: ${#HARBOR_USERNAME}"'
                    sh 'echo "Secrets fetched successfully from Vault ✅"'
                }
            }
        }

        // ========================================================
        // STAGE 5: Build Docker Image
        // ========================================================
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image: ${FULL_IMAGE}"

                sh """
                    docker build \
                        --build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                        --build-arg VCS_REF=\$(git rev-parse --short HEAD) \
                        --build-arg VERSION=${IMAGE_TAG} \
                        -t ${FULL_IMAGE} \
                        -t ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest \
                        .
                """

                // Show built images
                sh "docker images | grep ${IMAGE_NAME}"
            }
        }

        // ========================================================
        // STAGE 6: Security Scan with Trivy
        // ========================================================
        stage('Security Scan') {
            steps {
                echo '🔍 Scanning Docker image for vulnerabilities...'

                sh """
                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        ${FULL_IMAGE}
                """

                // Fail the build if CRITICAL vulnerabilities found
                // Change --exit-code to 1 to enforce in production:
                // --exit-code 1 means: fail if vulnerabilities found
            }
        }

        // ========================================================
        // STAGE 7: Push to Harbor Registry
        // ========================================================
        stage('Push to Harbor') {
            steps {
                echo "📤 Pushing image to Harbor Registry..."

                withVault([
                    configuration: [
                        vaultUrl: "${VAULT_ADDR}",
                        vaultCredentialId: 'vault-approle-credentials',
                        engineVersion: 2
                    ],
                    vaultSecrets: [
                        [
                            path: 'secret/harbor',
                            engineVersion: 2,
                            secretValues: [
                                [envVar: 'HARBOR_USERNAME', vaultKey: 'username'],
                                [envVar: 'HARBOR_PASSWORD', vaultKey: 'password']
                            ]
                        ]
                    ]
                ]) {
                    sh """
                        echo \$HARBOR_PASSWORD | docker login ${HARBOR_URL} \
                            -u \$HARBOR_USERNAME --password-stdin

                        docker push ${FULL_IMAGE}
                        docker push ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest

                        echo "✅ Image pushed successfully!"
                    """
                }
            }
            post {
                always {
                    // Always logout from registry
                    sh "docker logout ${HARBOR_URL}"
                }
            }
        }

        // ========================================================
        // STAGE 8: Deploy to Kubernetes
        // ========================================================
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes cluster..."

                withKubeConfig([credentialsId: 'kubeconfig']) {

                    // Update the deployment with new image
                    sh """
                        kubectl set image deployment/${K8S_DEPLOYMENT} \
                            ${K8S_DEPLOYMENT}=${FULL_IMAGE} \
                            -n ${K8S_NAMESPACE}
                    """

                    // Add annotation to record deployment details
                    sh """
                        kubectl annotate deployment/${K8S_DEPLOYMENT} \
                            kubernetes.io/change-cause="Build #${BUILD_NUMBER} - Commit: \$(git rev-parse --short HEAD)" \
                            -n ${K8S_NAMESPACE} \
                            --overwrite
                    """

                    // Wait for rollout to complete (timeout 5 minutes)
                    sh """
                        kubectl rollout status deployment/${K8S_DEPLOYMENT} \
                            -n ${K8S_NAMESPACE} \
                            --timeout=300s
                    """

                    // Show final pod status
                    sh "kubectl get pods -n ${K8S_NAMESPACE}"

                    // Show service endpoint
                    sh "kubectl get svc -n ${K8S_NAMESPACE}"
                }
            }
            post {
                failure {
                    echo '❌ Deployment FAILED — rolling back...'
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh """
                            kubectl rollout undo deployment/${K8S_DEPLOYMENT} \
                                -n ${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }

        // ========================================================
        // STAGE 9: Verify Deployment
        // ========================================================
        stage('Verify Deployment') {
            steps {
                echo '✅ Verifying deployment health...'

                withKubeConfig([credentialsId: 'kubeconfig']) {
                    // Get the LoadBalancer IP/hostname
                    script {
                        def svcOutput = sh(
                            script: "kubectl get svc devops-sample-app-service -n ${K8S_NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                            returnStdout: true
                        ).trim()

                        if (svcOutput) {
                            echo "App available at: http://${svcOutput}"

                            // Wait for DNS and test health endpoint
                            sh """
                                sleep 30
                                curl -f http://${svcOutput}/health || echo "Health check not reachable yet (DNS may take a moment)"
                            """
                        } else {
                            echo "LoadBalancer IP not yet assigned — check AWS console"
                        }
                    }

                    // Show all resources in namespace
                    sh "kubectl get all -n ${K8S_NAMESPACE}"
                }
            }
        }

    } // end stages

    // ============================================================
    // POST ACTIONS — Run after all stages regardless of result
    // ============================================================
    post {

        always {
            echo '🧹 Cleaning up...'

            // Clean up Docker images to save disk space
            sh "docker rmi ${FULL_IMAGE} || true"
            sh "docker rmi ${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:latest || true"
            sh "docker system prune -f || true"

            // Clean workspace
            cleanWs()
        }

        success {
            echo """
            ╔═══════════════════════════════════════════╗
            ║   ✅ PIPELINE SUCCEEDED!                  ║
            ║   Build: #${BUILD_NUMBER}                 ║
            ║   Image: ${FULL_IMAGE}                    ║
            ╚═══════════════════════════════════════════╝
            """
        }

        failure {
            echo """
            ╔═══════════════════════════════════════════╗
            ║   ❌ PIPELINE FAILED!                     ║
            ║   Build: #${BUILD_NUMBER}                 ║
            ║   Check console output for details        ║
            ╚═══════════════════════════════════════════╝
            """
        }

    } // end post

} // end pipeline
```

## 10.4 Create Jenkins Pipeline Job

```
Jenkins Dashboard → New Item

Name: devops-sample-app-pipeline
Type: Pipeline
OK

Configuration:
  General:
    ✅ GitHub project
    URL: https://github.com/<YOUR_USERNAME>/devops-sample-app

  Build Triggers:
    ✅ GitHub hook trigger for GITScm polling

  Pipeline:
    Definition: Pipeline script from SCM
    SCM: Git
    Repository URL: https://github.com/<YOUR_USERNAME>/devops-sample-app.git
    Credentials: github-credentials
    Branch: */main
    Script Path: Jenkinsfile

Save
```

## 10.5 First Deployment — Manual

```
Jenkins → devops-sample-app-pipeline → Build Now
```

Click on the build number → Console Output to watch it run.

## 10.6 Apply Kubernetes Manifests First Time

Before the pipeline can deploy, apply the base manifests:

```bash
# On Jenkins server
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Verify
kubectl get all -n devops-app
```

## 10.7 Push Code to Trigger Webhook

```bash
# Make a small change to test webhook
echo "# Updated" >> README.md
git add .
git commit -m "Test: trigger Jenkins webhook"
git push origin main

# Watch Jenkins dashboard — build should start automatically within seconds
```

## 10.8 Interview Questions — Jenkins Pipeline / CI/CD

**Q: What is the difference between CI and CD?**
A: CI (Continuous Integration) = automatically building and testing code on every commit — catching integration issues early. CD can mean Continuous Delivery (code is always deployable, but humans approve releases) or Continuous Deployment (every commit automatically deploys to production with no manual approval). Most companies do CI + Continuous Delivery.

**Q: What is a multi-branch pipeline in Jenkins?**
A: Jenkins automatically discovers and creates pipelines for every branch and PR in a repository. Each branch gets its own pipeline, build history, and configuration. Great for feature branch workflows — you can test PRs before merging.

**Q: What is the `withCredentials` block and why is it important?**
A: `withCredentials` securely injects credentials into the pipeline environment only for the duration of that block. Outside the block, the variable is masked (shows ****). This prevents secrets from appearing in logs, environment dumps, or being passed to subprocesses unnecessarily.

**Q: How would you implement zero-downtime deployments in Kubernetes?**
A: Using a Rolling Update strategy with `maxUnavailable: 0` and `maxSurge: 1`. This means Kubernetes adds one new pod (new version), waits for its readiness probe to pass, then removes one old pod. Repeat until all pods are updated. Combined with proper `readinessProbe` and `livenessProbe`, users never experience downtime.

---

# PHASE 11 — END-TO-END VERIFICATION {#phase-11}

## 11.1 Complete Health Check Checklist

Run these verification commands to confirm everything is working:

### EC2 Instances Health

```bash
# Check all your EC2 instance statuses in AWS Console
# OR use AWS CLI:
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[Tags[?Key==`Name`]|[0].Value,State.Name,PublicIpAddress]' \
  --output table
```

### Docker Health

```bash
# On each server:
docker ps -a
docker system df
docker info | grep -E "Containers|Images|Server Version"
```

### Jenkins Health

```bash
# Check Jenkins is running
sudo systemctl status jenkins

# Check Jenkins logs for errors
sudo journalctl -u jenkins -n 50

# Test Jenkins API
curl -s http://localhost:8080/api/json | python3 -m json.tool
```

### Harbor Health

```bash
# SSH into Harbor server
cd /opt/harbor
docker compose ps

# All should show "Up" status:
# harbor-core       running
# harbor-db         running
# harbor-jobservice running
# harbor-log        running
# harbor-portal     running
# nginx             running
# redis             running
# registry          running
# registryctl       running
```

### EKS / Kubernetes Health

```bash
# Node status
kubectl get nodes -o wide

# All system pods should be Running
kubectl get pods -n kube-system

# App pods
kubectl get pods -n devops-app -o wide

# Check pod logs
kubectl logs -l app=devops-sample-app -n devops-app --tail=20

# Test the application
APP_URL=$(kubectl get svc devops-sample-app-service -n devops-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$APP_URL/health
curl http://$APP_URL/
```

**Expected output of app curl:**
```json
{
  "message": "Welcome to DevOps Sample App!",
  "version": "1.0.0",
  "environment": "production",
  "timestamp": "2026-06-04T10:00:00.000Z",
  "hostname": "devops-sample-app-abc123-xyz"
}
```

### Vault Health

```bash
export VAULT_ADDR='http://<VAULT_ELASTIC_IP>:8200'
vault status
vault kv get secret/harbor

# Should show sealed: false, initialized: true
```

### Prometheus Health

```bash
# Check targets are UP
curl http://<PROMETHEUS_ELASTIC_IP>:9090/api/v1/targets | \
  python3 -m json.tool | grep '"health"'

# Should see "up" for all configured targets
```

### Grafana Health

```bash
curl http://<GRAFANA_ELASTIC_IP>:3000/api/health
# Expected: {"commit":"...","database":"ok","version":"..."}
```

## 11.2 End-to-End Pipeline Test

```bash
# 1. Make a code change
cd devops-sample-app
echo "Updated in phase 11 test" >> README.md

# 2. Commit and push
git add .
git commit -m "Phase 11: End-to-end verification test"
git push origin main

# 3. Watch Jenkins build (check http://<JENKINS_IP>:8080)
# Build should be triggered automatically via webhook

# 4. Watch the pipeline execute all stages:
#    ✅ Checkout
#    ✅ Install Dependencies
#    ✅ Test
#    ✅ Fetch Secrets from Vault
#    ✅ Build Docker Image
#    ✅ Security Scan
#    ✅ Push to Harbor
#    ✅ Deploy to Kubernetes
#    ✅ Verify Deployment

# 5. Verify Harbor has the new image
# Browser: http://<HARBOR_IP> → devops-project → devops-sample-app
# You should see new tag: <BUILD_NUMBER>

# 6. Verify Kubernetes has the new version
kubectl get pods -n devops-app
kubectl describe deployment devops-sample-app -n devops-app | grep Image

# 7. Verify Prometheus is scraping
# Browser: http://<PROMETHEUS_IP>:9090/targets

# 8. Verify Grafana dashboard shows metrics
# Browser: http://<GRAFANA_IP>:3000
```

## 11.3 Rollback Test

```bash
# Test that rollback works if something goes wrong

# View rollout history
kubectl rollout history deployment/devops-sample-app -n devops-app

# Rollback to previous version
kubectl rollout undo deployment/devops-sample-app -n devops-app

# Or rollback to specific revision
kubectl rollout undo deployment/devops-sample-app \
  -n devops-app \
  --to-revision=2

# Watch pods roll back
kubectl rollout status deployment/devops-sample-app -n devops-app

# Verify app works after rollback
curl http://$APP_URL/health
```

## 11.4 Monitoring Verification

```promql
-- Run these PromQL queries in Prometheus to verify monitoring:

-- Check Jenkins is being scraped
up{job="jenkins"}

-- Check Node Exporter for all instances
up{job="node-exporter"}

-- CPU usage of Kubernetes nodes
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

-- Memory pressure
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

-- If harbor metrics work
up{job="harbor"}
```

## 11.5 Final Architecture Diagram (Verified)

```
Your Push to GitHub
         │
         ▼ Webhook
Jenkins Pipeline                               Harbor Registry
  [Checkout] ─────────────────────────────────────────────────▶ [devops-project/]
  [npm test]                                                        [app:1] ✅
  [Vault fetch secrets] ◀─── HashiCorp Vault                       [app:2] ✅
  [docker build]              (harbor creds)                        [app:3] ✅
  [trivy scan]
  [docker push] ──────────────────────────────────────────────────▶ Harbor
  [kubectl set image] ─────────────────────▶ EKS Cluster
                                              [Pod v1 → v2] ✅
                                              [Service] ✅
                                              [LoadBalancer] ✅
                                                   │
                                                   ▼
Prometheus ─────────────────────────────────── Metrics
  [scraping all targets] ✅
         │
         ▼
Grafana ─────────────────────────────────────── Dashboards
  [Infrastructure] ✅
  [Jenkins] ✅
  [Kubernetes] ✅
```

---

# APPENDIX: QUICK REFERENCE CHEAT SHEET

## Git Commands
```bash
git clone <url>              # Clone a repo
git add .                    # Stage all changes
git commit -m "message"      # Commit
git push origin main         # Push to GitHub
git pull                     # Pull latest changes
git log --oneline            # Show commit history
git branch -a                # List all branches
git checkout -b feature/xyz  # Create and switch to new branch
```

## Docker Quick Reference
```bash
docker build -t name:tag .                    # Build image
docker run -d -p 8080:80 --name myapp nginx   # Run container
docker exec -it myapp bash                    # Shell into container
docker logs -f myapp                          # Follow logs
docker stop myapp && docker rm myapp          # Stop and remove
docker system prune -a                        # Clean everything
```

## Kubernetes Quick Reference
```bash
kubectl get pods -A                           # All pods
kubectl describe pod <name> -n <ns>           # Pod details
kubectl logs <pod> -n <ns> -f                 # Follow logs
kubectl exec -it <pod> -n <ns> -- bash        # Shell
kubectl apply -f manifest.yaml                # Apply config
kubectl delete -f manifest.yaml               # Delete resources
kubectl rollout undo deployment/<name>        # Rollback
kubectl scale deployment/<name> --replicas=3  # Scale
kubectl get events -n <ns> --sort-by='.lastTimestamp'  # Events
```

## Vault Quick Reference
```bash
vault status                                  # Check seal status
vault kv put secret/mykey value=hello         # Store secret
vault kv get secret/mykey                     # Read secret
vault kv list secret/                         # List secrets
vault token create -policy=mypolicy          # Create token
vault audit list                              # List audit devices
```

## AWS CLI Quick Reference
```bash
aws ec2 describe-instances                    # List instances
aws ec2 start-instances --instance-ids i-xxx  # Start instance
aws ec2 stop-instances --instance-ids i-xxx   # Stop instance
aws eks update-kubeconfig --name <cluster>    # Update kubeconfig
aws ecr get-login-password | docker login ... # ECR login
```

---

# COST ESTIMATE (Monthly Approximate)

| Resource                     | Type        | Monthly Cost |
|------------------------------|-------------|--------------|
| Jenkins Server (t3.medium)   | EC2         | ~$30         |
| Harbor Server (t3.medium)    | EC2         | ~$30         |
| Vault Server (t3.small)      | EC2         | ~$15         |
| Prometheus Server (t3.small) | EC2         | ~$15         |
| Grafana Server (t3.small)    | EC2         | ~$15         |
| EKS Control Plane            | EKS         | ~$72         |
| EKS Worker Nodes (2x t3.med) | EC2         | ~$60         |
| ELB (Load Balancer)          | ALB         | ~$16         |
| EBS Storage (200GB total)    | EBS gp3     | ~$16         |
| Data Transfer                | Various     | ~$5          |
| **TOTAL**                    |             | **~$274/mo** |

**💡 SAVE MONEY:** Stop EC2 instances when not in use. Use `aws ec2 stop-instances` for all servers at end of day. Delete the EKS cluster with `eksctl delete cluster --name devops-cluster` when done practicing.

---

*Guide created for: Complete DevOps Engineer Hands-On Project*  
*Date: June 2026 | Version: 1.0*
