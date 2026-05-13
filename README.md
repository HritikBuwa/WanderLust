# 🌍 Wanderlust — Complete DevOps Deployment Guide

> A full MERN stack travel blog deployed with Jenkins CI/CD, SonarQube, Trivy, Docker, and optional DNS via Route 53.  
> **This guide is written so that even a complete beginner can follow it step by step.**

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Required Ports (Open These First!)](#required-ports)
4. [Step 1 — Launch AWS EC2 Instance](#step-1--launch-aws-ec2-instance)
5. [Step 2 — Install Docker & Docker Compose](#step-2--install-docker--docker-compose)
6. [Step 3 — Install Jenkins](#step-3--install-jenkins)
7. [Step 4 — Install SonarQube via Docker](#step-4--install-sonarqube-via-docker)
8. [Step 5 — Install Trivy](#step-5--install-trivy)
9. [Step 6 — Configure Jenkins Plugins](#step-6--configure-jenkins-plugins)
10. [Step 7 — Connect SonarQube to Jenkins](#step-7--connect-sonarqube-to-jenkins)
11. [Step 8 — Configure Jenkins Tools](#step-8--configure-jenkins-tools)
12. [Step 9 — Set Up the Jenkins Pipeline](#step-9--set-up-the-jenkins-pipeline)
13. [Step 10 — Fix .env Files & Load Sample Data](#step-10--fix-env-files--load-sample-data)
14. [Step 11 — Run the Application](#step-11--run-the-application)
15. [Step 12 — (Optional) Connect a Domain via Route 53](#step-12--optional-connect-a-domain-via-route-53)
16. [Step 13 — Enable Auto Trigger from GitHub (SCM)](#step-13--enable-auto-trigger-from-github-scm)
17. [Common Errors & Fixes](#common-errors--fixes)

---

## Project Overview

**Wanderlust** is a MERN (MongoDB, Express, React, Node.js) travel blog app.  
This repo adds a full DevOps pipeline on top of it:

| Tool | Purpose |
|---|---|
| Jenkins | CI/CD automation |
| SonarQube | Code quality & security analysis |
| OWASP Dependency Check | Dependency vulnerability scan |
| Trivy | Docker image & filesystem vulnerability scan |
| Docker + Docker Compose | Containerization |
| AWS EC2 | Cloud server |
| Route 53 + Hostinger | DNS mapping (optional) |

**Services & ports the app runs on:**

| Service | Port |
|---|---|
| Jenkins | 8080 |
| SonarQube | 9000 |
| Frontend (React/Vite) | 5173 |
| Backend (Node/Express) | 5000 |
| MongoDB | 27017 |
| Redis | 6379 (internal only) |

---

## Architecture Diagram

```
Developer pushes code to GitHub
        │
        ▼
  Jenkins (port 8080)
        │
        ├──► OWASP Dependency Check
        ├──► Trivy Filesystem Scan
        ├──► SonarQube Analysis (port 9000)
        ├──► SonarQube Quality Gate
        ├──► Update .env files (Automations scripts)
        ├──► Docker Build (frontend + backend images)
        └──► Docker Push to DockerHub
                    │
                    ▼
         docker-compose up on EC2
         ┌──────────────────────────┐
         │  Frontend  :5173         │
         │  Backend   :5000         │
         │  MongoDB   :27017        │
         │  Redis     :6379         │
         └──────────────────────────┘
```

---

## Required Ports

Before starting, open **all** these ports in your EC2 Security Group (SG).  
> **How to open ports:** EC2 → Security Groups → Inbound Rules → Add Rule

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH into server |
| 8080 | TCP | Jenkins UI |
| 9000 | TCP | SonarQube UI |
| 5173 | TCP | Frontend app |
| 5000 | TCP | Backend API |
| 27017 | TCP | MongoDB (optional, only if connecting externally) |
| 80 | TCP | HTTP (if using domain) |
| 443 | TCP | HTTPS (if using domain) |

---

## Step 1 — Launch AWS EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Configure as follows:

| Setting | Value |
|---|---|
| Name | `wanderlust-server` |
| OS | Ubuntu 22.04 LTS |
| Instance Type | `t2.large` (minimum — needs 2 vCPU, 8 GB RAM for Jenkins + SonarQube) |
| Storage | 25 GB (between 20–30 GB is fine) |
| Key Pair | Create a new `.pem` key and download it |
| Security Group | Allow all ports listed in the table above |

3. Click **Launch Instance** and wait ~2 minutes.
4. SSH into your server:

```bash
# Replace with your actual .pem file path and EC2 public IP
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>
```

---

## Step 2 — Install Docker & Docker Compose

Run these commands **one by one** on your EC2 server:

```bash
# Update package list
sudo apt-get update -y

# Install Docker
sudo apt-get install docker.io -y

# Install Docker Compose
sudo apt-get install docker-compose -y

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to the docker group (so you don't need sudo for docker commands)
sudo usermod -aG docker $USER

# Apply group changes without logging out
newgrp docker

# Test Docker works
docker ps
```

> **Note:** If `docker ps` still says "permission denied", log out and SSH back in. The group change takes effect on the next login.

---

## Step 3 — Install Jenkins

Jenkins requires Java. Install Java first, then Jenkins.

```bash
# Install Java 17
sudo apt-get install fontconfig openjdk-17-jre -y

# Verify Java
java -version
```

Now install Jenkins from the official repository:

```bash
# Add Jenkins GPG key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins apt repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt-get update -y
sudo apt-get install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Add Jenkins user to docker group (so Jenkins can run Docker commands)
sudo usermod -aG docker jenkins

# Restart Jenkins to apply docker group change
sudo systemctl restart jenkins
```

**Access Jenkins:**  
Open your browser → `http://<YOUR_EC2_PUBLIC_IP>:8080`

**Get the initial admin password:**

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy that password, paste it in the browser, and follow the setup wizard. Choose **Install Suggested Plugins** when asked.

---

## Step 4 — Install SonarQube via Docker

Run SonarQube as a Docker container (much simpler than a manual install):

```bash
docker run -itd \
  --name SonarQube-server \
  -p 9000:9000 \
  sonarqube:lts-community
```

> **Important:** Make sure port `9000` is open in your EC2 Security Group.

**Access SonarQube:**  
Open `http://<YOUR_EC2_PUBLIC_IP>:9000`

- **Default Username:** `admin`  
- **Default Password:** `admin`

SonarQube will ask you to change the password on first login. Choose any new password and save it.

---

## Step 5 — Install Trivy

Trivy scans Docker images and filesystems for vulnerabilities.

```bash
# Install dependencies
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

# Add Trivy GPG key
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

# Add Trivy repository
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb \
  $(lsb_release -sc) main" | \
  sudo tee -a /etc/apt/sources.list.d/trivy.list

# Update and install Trivy
sudo apt-get update -y
sudo apt-get install trivy -y

# Verify installation
trivy --version
```

---

## Step 6 — Configure Jenkins Plugins

Go to: **Jenkins → Manage Jenkins → Plugins → Available Plugins**

Search and install each of these (tick the checkbox, then click **Install**):

| Plugin Name | What It Does |
|---|---|
| SonarQube Scanner | Runs SonarQube code analysis from Jenkins |
| Sonar Quality Gates | Fails the build if code quality is below threshold |
| OWASP Dependency-Check | Scans project dependencies for known CVEs |
| Docker | Allows Jenkins to run Docker commands |
| Docker Pipeline | Allows Jenkins to use Docker in pipeline scripts |
| Pipeline: Stage View | Shows a visual stage-by-stage view of your pipeline runs |

After installing, click **Restart Jenkins when no jobs are running**.

---

## Step 7 — Connect SonarQube to Jenkins

This is done in **two parts**: configure SonarQube to notify Jenkins (webhook), and configure Jenkins to talk to SonarQube (token + server URL).

### Part A — Create a Webhook in SonarQube

1. Open SonarQube → `http://<YOUR_EC2_PUBLIC_IP>:9000`
2. Go to **Administration → Configuration → Webhooks**
3. Click **Create**
4. Fill in:
   - **Name:** `jenkins`
   - **URL:** `http://<YOUR_EC2_PUBLIC_IP>:8080/sonarqube-webhook/`
5. Click **Create**

### Part B — Generate a SonarQube Token

1. In SonarQube, click your profile icon (top right) → **My Account**
2. Go to **Security** tab
3. Under **Generate Tokens**, enter a name like `jenkins-token`
4. Click **Generate** and **copy the token immediately** (it won't show again)

### Part C — Add the Token as a Jenkins Credential

1. Go to **Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Fill in:
   - **Kind:** `Secret text`
   - **Secret:** paste the SonarQube token
   - **ID:** `sonar-token`
   - **Description:** `SonarQube Token`
3. Click **Create**

### Part D — Add SonarQube Server in Jenkins

1. Go to **Jenkins → Manage Jenkins → System**
2. Scroll down to **SonarQube servers**
3. Tick **Environment variables** checkbox
4. Click **Add SonarQube**
5. Fill in:
   - **Name:** `Sonar`  *(must match the name used in the Jenkinsfile)*
   - **Server URL:** `http://<YOUR_EC2_PUBLIC_IP>:9000`
   - **Server authentication token:** select `sonar-token` (the credential you just created)
6. Click **Save**

---

## Step 8 — Configure Jenkins Tools

Go to **Jenkins → Manage Jenkins → Tools**

### SonarQube Scanner

1. Scroll to **SonarQube Scanner installations**
2. Click **Add SonarQube Scanner**
3. Fill in:
   - **Name:** `Sonar`  *(must match exactly what's in the Jenkinsfile)*
   - Tick **Install automatically**
4. Click **Save**

### OWASP Dependency-Check

1. Scroll to **Dependency-Check installations**
2. Click **Add Dependency-Check**
3. Fill in:
   - **Name:** `dc`
   - Tick **Install automatically**
   - Click **Add Installer** → select **Install from GitHub releases**
   - **Version:** select the latest stable
4. Click **Save**

---

## Step 9 — Set Up the Jenkins Pipeline

### Option A — Manual Pipeline (Paste Script Directly)

1. Jenkins → **New Item**
2. Enter name: `Wanderlust-CI`
3. Select **Pipeline** → click **OK**
4. Under **General**:
   - Tick **GitHub project**
   - Enter your repository URL (e.g. `https://github.com/YOUR_USERNAME/wanderlust`)
5. Under **Build Triggers**:
   - Tick **GitHub hook trigger for GITScm polling**
6. Under **Pipeline**:
   - Select **Pipeline script**
   - Paste the contents of the `Jenkinsfile` from the root of this repo
7. Click **Save**

### Option B — Pipeline from SCM (Recommended — Auto-reads from Repo)

1. Jenkins → **New Item** → `Wanderlust-CI` → **Pipeline** → **OK**
2. Under **Pipeline**:
   - Select **Pipeline script from SCM**
   - **SCM:** Git
   - **Repository URL:** `https://github.com/YOUR_USERNAME/wanderlust`
   - **Branch:** `*/devops` (or your branch name)
   - **Script Path:** `Jenkinsfile`
3. Click **Save**

> The `Jenkinsfile` at the root of the repo already contains all pipeline stages. Jenkins will read it automatically.

---

## Step 10 — Fix .env Files & Load Sample Data

### Fix the .env.docker Files

The `.env.docker` files contain a hardcoded IP address that must be changed to your EC2 public IP.

**Backend** — edit `backend/.env.docker`:

```bash
# Find your EC2 public IP
curl -s http://169.254.169.254/latest/meta-data/public-ipv4
```

Open `backend/.env.docker` and update it:

```env
MONGODB_URI="mongodb://mongo/wanderlust"
REDIS_URL="redis://redis:6379"
PORT=5000
FRONTEND_URL="http://<YOUR_EC2_PUBLIC_IP>:5173"
ACCESS_COOKIE_MAXAGE=120000
ACCESS_TOKEN_EXPIRES_IN='120s'
REFRESH_COOKIE_MAXAGE=120000
REFRESH_TOKEN_EXPIRES_IN='120s'
JWT_SECRET=70dd8b38486eee723ce2505f6db06f1ee503fde5eb06fc04687191a0ed665f3f98776902d2c89f6b993b1c579a87fedaf584c693a106f7cbf16e8b4e67e9d6df
NODE_ENV=Development
```

> **Important fixes applied here:**
> - `mongo-service` → changed to `mongo` (matches `container_name` in docker-compose.yml)
> - `redis-service` → changed to `redis` (matches `container_name` in docker-compose.yml)
> - `PORT` changed from `8080` to `5000` (matches the exposed port in docker-compose.yml)
> - Replace `<YOUR_EC2_PUBLIC_IP>` with your actual IP

**Frontend** — edit `frontend/.env.docker`:

```env
VITE_API_PATH="http://<YOUR_EC2_PUBLIC_IP>:5000"
```

> Replace `<YOUR_EC2_PUBLIC_IP>` with your actual IP.  
> Port is `5000` (backend API), not `31100` (that is for Kubernetes deployments only).

The Jenkins pipeline runs `Automations/updateBackend.sh` and `Automations/updateFrontend.sh` to update these IPs automatically using the EC2 metadata service — so if you're running through Jenkins, this happens automatically.

### Fix docker-compose.yml

The default `docker-compose.yml` mounts `./backend/data:/data` for MongoDB. This won't seed the database automatically. Use this corrected version:

```yaml
version: "3.8"
services:
  mongodb:
    container_name: mongo
    image: mongo:latest
    volumes:
      - mongo-data:/data/db
    ports:
      - "27017:27017"

  backend:
    container_name: backend
    build: ./backend
    env_file:
      - ./backend/.env.docker
    ports:
      - "5000:5000"
    depends_on:
      - mongodb
      - redis

  frontend:
    container_name: frontend
    build: ./frontend
    env_file:
      - ./frontend/.env.docker
    ports:
      - "5173:5173"
    depends_on:
      - backend

  redis:
    container_name: redis
    restart: unless-stopped
    image: redis:7.0.5-alpine
    expose:
      - 6379
    depends_on:
      - mongodb

volumes:
  mongo-data:
```

> **Fixes applied:**
> - Named volume `mongo-data` instead of binding `./backend/data` (avoids permission errors)
> - Backend now depends on both `mongodb` AND `redis`
> - Frontend now depends on `backend` (proper startup order)

### Load Sample Data into MongoDB

After running `docker-compose up` (Step 11), seed the database with sample travel posts:

```bash
# Import sample posts into MongoDB container
docker exec -i mongo mongoimport \
  --db wanderlust \
  --collection posts \
  --file /data/sample_posts.json \
  --jsonArray
```

If the above doesn't find the file (because we changed the volume mount), copy it in first:

```bash
docker cp backend/data/sample_posts.json mongo:/tmp/sample_posts.json

docker exec mongo mongoimport \
  --db wanderlust \
  --collection posts \
  --file /tmp/sample_posts.json \
  --jsonArray
```

You should see output like:

```
2024-xx-xx connected to: mongodb://localhost/
2024-xx-xx 15 document(s) imported successfully. 0 document(s) failed to import.
```

---

## Step 11 — Run the Application

If you want to run **without Jenkins** (direct Docker Compose):

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/wanderlust.git
cd wanderlust

# Build and start all containers
docker-compose up --build -d

# Check all containers are running
docker ps
```

You should see 4 containers: `mongo`, `redis`, `backend`, `frontend`.

**Access the app:**

| Service | URL |
|---|---|
| Frontend | `http://<YOUR_EC2_PUBLIC_IP>:5173` |
| Backend API | `http://<YOUR_EC2_PUBLIC_IP>:5000` |
| Jenkins | `http://<YOUR_EC2_PUBLIC_IP>:8080` |
| SonarQube | `http://<YOUR_EC2_PUBLIC_IP>:9000` |

**To stop the app:**

```bash
docker-compose down
```

**To view logs of a specific container:**

```bash
docker logs backend
docker logs frontend
docker logs mongo
```

---

## Step 12 — (Optional) Connect a Domain via Route 53

This maps your IP address (e.g. `13.52.243.88`) to a domain name (e.g. `hritikbuwa.online`).

### Part A — Configure Route 53 in AWS

1. Go to **AWS Console → Route 53 → Hosted Zones**
2. Click **Create Hosted Zone**
3. Enter your domain name (e.g. `hritikbuwa.online`)
4. Type: **Public hosted zone**
5. Click **Create hosted zone**
6. AWS will give you **4 Name Servers (NS records)** — copy all 4. They look like:
   ```
   ns-123.awsdns-45.com
   ns-456.awsdns-67.net
   ns-789.awsdns-89.org
   ns-012.awsdns-10.co.uk
   ```

### Part B — Point Hostinger DNS to Route 53

1. Log in to **Hostinger → Domains → Manage → DNS / Nameservers**
2. Select **Custom nameservers**
3. Delete the existing nameservers and add the 4 AWS ones you copied
4. Save — changes propagate in 15 minutes to 48 hours

### Part C — Create an A Record in Route 53

1. Back in Route 53 → your Hosted Zone → click **Create Record**
2. Fill in:
   - **Record name:** leave blank (for root domain) or enter `www`
   - **Record type:** `A`
   - **Value:** `<YOUR_EC2_PUBLIC_IP>`
   - **TTL:** 300
3. Click **Create records**

### Part D — Update .env.docker Files to Use Domain

Once DNS propagates, update your env files to use the domain name instead of IP.

**frontend/.env.docker:**

```env
VITE_API_PATH="http://hritikbuwa.online:5000"
```

**backend/.env.docker:**

```env
FRONTEND_URL="http://hritikbuwa.online:5173"
```

Rebuild and restart containers:

```bash
docker-compose down
docker-compose up --build -d
```

Test in browser: `http://hritikbuwa.online:5173`

---

## Step 13 — Enable Auto Trigger from GitHub (SCM)

This makes Jenkins automatically run your pipeline every time you push code to GitHub.

### Step A — Configure Pipeline as SCM

In Jenkins → your pipeline → **Configure**:
- **Pipeline Definition:** Pipeline script from SCM
- **SCM:** Git
- **Repository URL:** `https://github.com/YOUR_USERNAME/wanderlust`
- **Credentials:** Add your GitHub credentials if it's a private repo
- **Branch:** `*/devops` (or `*/main` — use your actual branch)
- **Script Path:** `Jenkinsfile`

### Step B — Add GitHub Webhook

1. Go to your **GitHub repository → Settings → Webhooks → Add webhook**
2. Fill in:
   - **Payload URL:** `http://<YOUR_EC2_PUBLIC_IP>:8080/github-webhook/`
   - **Content type:** `application/json`
   - **Which events:** Select **Just the push event**
3. Click **Add webhook**

Now every `git push` will automatically trigger the Jenkins pipeline.

---

## Common Errors & Fixes

### ❌ `docker ps` gives "permission denied"

```bash
sudo usermod -aG docker $USER
newgrp docker
# Or log out and log back in via SSH
```

### ❌ Backend container exits immediately

Check logs:
```bash
docker logs backend
```

**Most likely cause:** wrong service names in `backend/.env.docker`.  
Make sure:
```env
MONGODB_URI="mongodb://mongo/wanderlust"   # "mongo" = container_name in docker-compose
REDIS_URL="redis://redis:6379"              # "redis" = container_name in docker-compose
```

### ❌ Frontend shows blank page / can't fetch posts

Check:
```bash
docker logs frontend
```

**Most likely cause:** `VITE_API_PATH` in `frontend/.env.docker` has wrong IP or port.  
It should be:
```env
VITE_API_PATH="http://<YOUR_EC2_PUBLIC_IP>:5000"
```
Note: port `5000`, not `31100` (that port is for Kubernetes only).

### ❌ No posts showing on the website

The database is empty. Run the sample data import:

```bash
docker cp backend/data/sample_posts.json mongo:/tmp/sample_posts.json
docker exec mongo mongoimport --db wanderlust --collection posts --file /tmp/sample_posts.json --jsonArray
```

### ❌ SonarQube shows "Analysis Failed" in Jenkins

- Check the webhook URL in SonarQube ends with `/sonarqube-webhook/` (trailing slash is required)
- Check the SonarQube server name in Jenkins → Manage Jenkins → System is exactly `Sonar`
- Check the SonarQube Scanner tool name in Jenkins → Manage Jenkins → Tools is also exactly `Sonar`

### ❌ Jenkins can't run Docker commands

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### ❌ Port 8080 not accessible from browser

- Check EC2 Security Group inbound rules include port 8080 from `0.0.0.0/0`
- Verify Jenkins is running: `sudo systemctl status jenkins`

### ❌ `mongo-service` or `redis-service` not found (container connection error)

The original `.env.docker` uses `mongo-service` and `redis-service` — these are Kubernetes service names, not Docker Compose service names. In Docker Compose, use the `container_name` values:
- `mongo-service` → `mongo`
- `redis-service` → `redis`

---

## Project Structure (Quick Reference)

```
wanderlust-devops/
├── Jenkinsfile                  # Main CI pipeline (copy to your repo root)
├── docker-compose.yml           # Runs all 4 services together
├── backend/
│   ├── Dockerfile               # Multi-stage Node.js build
│   ├── .env.docker              # Env vars for Docker deployment ← edit this
│   ├── .env.sample              # Template for local development
│   └── data/
│       └── sample_posts.json    # Sample travel posts to seed MongoDB
├── frontend/
│   ├── .env.docker              # Env vars for Docker deployment ← edit this
│   └── .env.sample              # Template for local development
├── Automations/
│   ├── updateBackend.sh         # Auto-updates backend .env.docker with EC2 IP
│   └── updateFrontend.sh        # Auto-updates frontend .env.docker with EC2 IP
└── GitOps/
    └── Jenkinsfile              # CD pipeline (updates Kubernetes manifests)
```

---

## Quick Start Summary (TL;DR)

```bash
# 1. Launch t2.large Ubuntu EC2, open ports: 22, 8080, 9000, 5173, 5000

# 2. SSH in and run:
sudo apt-get update -y
sudo apt-get install docker.io docker-compose -y
sudo usermod -aG docker $USER && newgrp docker

# 3. Install Jenkins (see Step 3)

# 4. Start SonarQube
docker run -itd --name SonarQube-server -p 9000:9000 sonarqube:lts-community

# 5. Install Trivy (see Step 5)

# 6. Edit backend/.env.docker — replace IP, fix service names to "mongo" and "redis"
# 7. Edit frontend/.env.docker — set VITE_API_PATH to http://<IP>:5000

# 8. Start the app
docker-compose up --build -d

# 9. Seed the database
docker cp backend/data/sample_posts.json mongo:/tmp/sample_posts.json
docker exec mongo mongoimport --db wanderlust --collection posts --file /tmp/sample_posts.json --jsonArray

# 10. Open http://<YOUR_EC2_PUBLIC_IP>:5173 🎉
```

---

*Made with ❤️ for the Wanderlust DevOps project. Happy deploying!*
