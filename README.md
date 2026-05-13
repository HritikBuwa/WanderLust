<div align="center">

# 🌍 Wanderlust DevOps

**A full-stack MERN travel blog with a production-grade CI/CD pipeline**

[![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](http://jenkins-url:8080)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)](https://www.sonarsource.com/)
[![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![MongoDB](https://img.shields.io/badge/MongoDB-Database-47A248?style=for-the-badge&logo=mongodb&logoColor=white)](https://www.mongodb.com/)

![Preview](https://github.com/krishnaacharyaa/wanderlust/assets/116620586/17ba9da6-225f-481d-87c0-5d5a010a9538)

</div>

---

## 🗺️ What Is This?

Wanderlust is a **MERN stack travel blog** wrapped in a complete DevOps pipeline. Push code → Jenkins automatically tests, scans, builds, and deploys it. Everything runs in Docker containers on a single AWS EC2 instance.

```
GitHub Push → Jenkins CI → SonarQube + OWASP + Trivy → Docker Build → Deploy
```

---

## 🏗️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React + Vite (port `5173`) |
| Backend | Node.js + Express (port `5000`) |
| Database | MongoDB (port `27017`) |
| Cache | Redis (internal) |
| CI/CD | Jenkins (port `8080`) |
| Code Quality | SonarQube (port `9000`) |
| Security Scan | OWASP Dependency-Check + Trivy |
| Infrastructure | AWS EC2 — Ubuntu 22.04, t2.large |
| Containers | Docker + Docker Compose |
| DNS (optional) | AWS Route 53 + Hostinger |

---

## ⚡ Quick Start (5 steps)

> **Prerequisites:** An AWS account. That's it.

### 1️⃣ Launch EC2 Instance

| Setting | Value |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Type | `t2.large` |
| Storage | 25 GB |
| Open ports | `22, 5000, 5173, 8080, 9000` |

### 2️⃣ Install Docker

```bash
sudo apt-get update -y
sudo apt-get install docker.io docker-compose -y
sudo usermod -aG docker $USER
newgrp docker
```

### 3️⃣ Clone & Configure

```bash
git clone https://github.com/YOUR_USERNAME/wanderlust.git
cd wanderlust
```

Edit **`backend/.env.docker`** — replace the IP and fix service names:

```env
MONGODB_URI="mongodb://mongo/wanderlust"
REDIS_URL="redis://redis:6379"
PORT=5000
FRONTEND_URL="http://<YOUR_EC2_IP>:5173"
JWT_SECRET=your_secret_here
NODE_ENV=Development
```

Edit **`frontend/.env.docker`**:

```env
VITE_API_PATH="http://<YOUR_EC2_IP>:5000"
```

### 4️⃣ Launch

```bash
docker-compose up --build -d
docker ps   # you should see 4 containers running
```

### 5️⃣ Seed the Database

```bash
docker cp backend/data/sample_posts.json mongo:/tmp/sample_posts.json

docker exec mongo mongoimport \
  --db wanderlust --collection posts \
  --file /tmp/sample_posts.json --jsonArray
```

🎉 Open `http://<YOUR_EC2_IP>:5173` — your app is live!

---

## 🔁 Full CI/CD Pipeline Setup

### Install Jenkins

```bash
# Java (required for Jenkins)
sudo apt-get install fontconfig openjdk-17-jre -y

# Jenkins official repo
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update -y && sudo apt-get install jenkins -y
sudo systemctl start jenkins

# Let Jenkins run Docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Get the initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then open `http://<YOUR_EC2_IP>:8080` and complete the setup wizard.

---

### Install SonarQube

```bash
docker run -itd \
  --name SonarQube-server \
  -p 9000:9000 \
  sonarqube:lts-community
```

Open `http://<YOUR_EC2_IP>:9000` → login: `admin` / `admin` → change password when prompted.

---

### Install Trivy

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee -a /etc/apt/sources.list.d/trivy.list

sudo apt-get update -y && sudo apt-get install trivy -y
```

---

### Install Jenkins Plugins

Go to **Manage Jenkins → Plugins → Available** and install:

- ✅ SonarQube Scanner
- ✅ Sonar Quality Gates
- ✅ OWASP Dependency-Check
- ✅ Docker
- ✅ Docker Pipeline
- ✅ Pipeline: Stage View

---

### Connect SonarQube ↔ Jenkins

**Step 1 — SonarQube webhook** (so SonarQube notifies Jenkins):

> SonarQube → Administration → Configuration → Webhooks → Create
> - Name: `jenkins`
> - URL: `http://<YOUR_EC2_IP>:8080/sonarqube-webhook/`

**Step 2 — Generate token in SonarQube:**

> Profile icon → My Account → Security → Generate Token → Copy it

**Step 3 — Add token to Jenkins credentials:**

> Jenkins → Manage Jenkins → Credentials → Add
> - Kind: `Secret text`
> - Secret: *paste token*
> - ID: `sonar-token`

**Step 4 — Add SonarQube server in Jenkins:**

> Manage Jenkins → System → SonarQube servers → Add
> - Name: `Sonar`
> - URL: `http://<YOUR_EC2_IP>:9000`
> - Token: select `sonar-token`

**Step 5 — Configure tools:**

> Manage Jenkins → Tools
> - **SonarQube Scanner** → Add → Name: `Sonar` → Install automatically ✅
> - **Dependency-Check** → Add → Name: `dc` → Install from GitHub releases → Latest stable

---

### Create the Pipeline

**Option A — Script in Jenkins UI:**

1. New Item → `Wanderlust-CI` → Pipeline
2. General: tick **GitHub project** → paste your repo URL
3. Build Triggers: tick **GitHub hook trigger for GITScm polling**
4. Pipeline: select **Pipeline script** → paste `Jenkinsfile` content

**Option B — Auto-read from repo (recommended):**

1. New Item → `Wanderlust-CI` → Pipeline
2. Pipeline: select **Pipeline script from SCM**
   - SCM: `Git`
   - Repo URL: `https://github.com/YOUR_USERNAME/wanderlust`
   - Branch: `*/devops` (or your branch)
   - Script Path: `Jenkinsfile`

---

## 🌐 Custom Domain (Optional)

Connect your domain (e.g. `yourdomain.com`) to your EC2 IP using Route 53 + Hostinger.

**1. Create a Hosted Zone in Route 53:**
> Route 53 → Hosted Zones → Create → enter your domain → Public hosted zone

**2. Copy the 4 NS records AWS gives you.**

**3. In Hostinger:**
> Domains → DNS/Nameservers → Custom nameservers → paste the 4 AWS nameservers

**4. Add an A record in Route 53:**
> Create Record → Type: `A` → Value: `<YOUR_EC2_IP>` → Save

**5. Update your `.env.docker` files:**

```env
# frontend/.env.docker
VITE_API_PATH="http://yourdomain.com:5000"

# backend/.env.docker
FRONTEND_URL="http://yourdomain.com:5173"
```

DNS propagates in 15 min – 48 hours. Then: `http://yourdomain.com:5173` 🎉

---

## 🤖 Auto-Trigger on Git Push (SCM Webhook)

Make every `git push` automatically trigger Jenkins:

**1. GitHub → your repo → Settings → Webhooks → Add webhook:**
- Payload URL: `http://<YOUR_EC2_IP>:8080/github-webhook/`
- Content type: `application/json`
- Events: **Just the push event**

**2. In Jenkins pipeline config:**
- Change **Pipeline script** → **Pipeline script from SCM**
- Set repo URL, branch, and script path as shown above

Now every push runs the full pipeline automatically. ✅

---

## 🗂️ Project Structure

```
wanderlust/
├── Jenkinsfile                  ← CI pipeline definition
├── docker-compose.yml           ← spins up all 4 services
├── backend/
│   ├── Dockerfile
│   ├── .env.docker              ← ⚠️ edit this with your IP
│   ├── .env.sample              ← template for local dev
│   └── data/
│       └── sample_posts.json   ← seed data for MongoDB
├── frontend/
│   ├── .env.docker              ← ⚠️ edit this with your IP
│   └── .env.sample
├── Automations/
│   ├── updateBackend.sh         ← auto-sets IP in backend env
│   └── updateFrontend.sh        ← auto-sets IP in frontend env
└── GitOps/
    └── Jenkinsfile              ← CD pipeline (Kubernetes)
```

---

## 🐛 Common Issues & Fixes

<details>
<summary><b>docker ps says "permission denied"</b></summary>

```bash
sudo usermod -aG docker $USER
newgrp docker
# or log out and SSH back in
```
</details>

<details>
<summary><b>Backend container keeps restarting</b></summary>

Check `backend/.env.docker`. The service names must match `container_name` in `docker-compose.yml`:
```env
MONGODB_URI="mongodb://mongo/wanderlust"   # ✅ not "mongo-service"
REDIS_URL="redis://redis:6379"              # ✅ not "redis-service"
PORT=5000                                   # ✅ not 8080
```
</details>

<details>
<summary><b>Frontend shows blank page / no posts</b></summary>

1. Check `frontend/.env.docker`:
   ```env
   VITE_API_PATH="http://<YOUR_EC2_IP>:5000"   # ✅ port 5000, not 31100
   ```
2. Run the sample data import (Step 5 in Quick Start above)
</details>

<details>
<summary><b>Jenkins can't run Docker commands</b></summary>

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
</details>

<details>
<summary><b>SonarQube analysis fails in Jenkins</b></summary>

- Webhook URL in SonarQube must end with `/sonarqube-webhook/` (trailing slash required)
- Tool name in Jenkins → Tools must be exactly `Sonar`
- Server name in Jenkins → System must also be exactly `Sonar`
</details>

---

## 📄 License

[MIT](LICENSE)

---

<div align="center">

Made with ❤️ · [Report a Bug](../../issues) · [Request a Feature](../../issues)

</div>
