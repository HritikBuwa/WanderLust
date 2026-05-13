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

Wanderlust is a **MERN stack travel blog** with a complete DevOps CI/CD pipeline.  
Every `git push` triggers Jenkins — which scans, tests, and deploys the app automatically in Docker.

```
Git Push → Clone → SonarQube Analysis → Quality Gate
       → OWASP Dependency Check → Trivy FS Scan → Docker Compose Deploy
```

---

## 🏗️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React + Vite · port `5173` |
| Backend | Node.js + Express · port `5000` |
| Database | MongoDB · port `27017` |
| Cache | Redis (internal only) |
| CI/CD | Jenkins · port `8080` |
| Code Quality | SonarQube · port `9000` |
| Security | OWASP Dependency-Check + Trivy |
| Infrastructure | AWS EC2 — Ubuntu 22.04, t2.large, 25 GB |
| Containers | Docker + Docker Compose |
| DNS (optional) | AWS Route 53 + Hostinger |

---

## 🔁 CI/CD Pipeline

Copy this `Jenkinsfile` to the **root of your repository**.

```groovy
pipeline {
    agent any

    environment {
        SONAR_HOME = tool 'Sonar'   // must match Jenkins → Tools → SonarQube Scanner name
    }

    stages {

        stage('Clone Code from GitHub') {
            steps {
                git url: 'https://github.com/krishnaacharyaa/wanderlust.git',
                    branch: 'devops'
            }
        }

        stage('SonarQube Quality Analysis') {
            steps {
                withSonarQubeEnv('Sonar') {   // must match Jenkins → System → SonarQube server name
                    sh """
                        ${SONAR_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=wanderlust \
                        -Dsonar.projectKey=wanderlust
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML',
                                odcInstallation: 'dc'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Sonar Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('Deploy using Docker Compose') {
            steps {
                sh 'docker-compose down --remove-orphans || true'
                sh 'docker-compose up -d --build'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-fs-report.html', fingerprint: true
            archiveArtifacts artifacts: '**/dependency-check-report.xml', fingerprint: true
        }
        success {
            echo 'Pipeline completed. App is live!'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
    }
}
```

### What Each Stage Does

| # | Stage | Purpose |
|---|---|---|
| 1 | Clone Code | Pulls latest code from the `devops` branch |
| 2 | SonarQube Analysis | Static code quality scan — bugs, smells, duplications |
| 3 | OWASP Dependency Check | Scans dependencies for known CVEs, publishes XML report |
| 4 | Sonar Quality Gate | Waits for SonarQube result; warns if below threshold |
| 5 | Trivy FS Scan | Scans filesystem for vulnerabilities, saves HTML report |
| 6 | Docker Compose Deploy | Stops old containers, rebuilds and starts all 4 services |

> **Reports** — Trivy HTML and OWASP XML are archived automatically after every build. Find them under **Build → Artifacts** in Jenkins.

---

## ⚡ Quick Start

> You only need an AWS account. Everything else is installed below.

### 1️⃣ Launch EC2 Instance

| Setting | Value |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Instance Type | `t2.large` (Jenkins + SonarQube need 8 GB RAM) |
| Storage | 25 GB |
| Open inbound ports | `22, 5000, 5173, 8080, 9000` |

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<YOUR_EC2_IP>
```

### 2️⃣ Install Docker

```bash
sudo apt-get update -y
sudo apt-get install docker.io docker-compose -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
docker ps   # empty table = working fine
```

### 3️⃣ Install Jenkins

```bash
sudo apt-get install fontconfig openjdk-17-jre -y

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update -y && sudo apt-get install jenkins -y
sudo systemctl start jenkins && sudo systemctl enable jenkins

sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Get your admin password and open `http://<YOUR_EC2_IP>:8080`:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Choose **Install Suggested Plugins** in the setup wizard.

### 4️⃣ Install SonarQube

```bash
docker run -itd \
  --name SonarQube-server \
  -p 9000:9000 \
  sonarqube:lts-community
```

Open `http://<YOUR_EC2_IP>:9000` → login `admin` / `admin` → change password.

### 5️⃣ Install Trivy

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee -a /etc/apt/sources.list.d/trivy.list

sudo apt-get update -y && sudo apt-get install trivy -y
trivy --version
```

---

## ⚙️ Jenkins Configuration

### Plugins

> Manage Jenkins → Plugins → Available Plugins → install all 6:

- ✅ SonarQube Scanner
- ✅ Sonar Quality Gates
- ✅ OWASP Dependency-Check
- ✅ Docker
- ✅ Docker Pipeline
- ✅ Pipeline: Stage View

Restart after installing.

### Connect SonarQube ↔ Jenkins

**1. SonarQube Webhook** — so SonarQube can call Jenkins back after analysis:
> SonarQube → Administration → Configuration → Webhooks → Create
> - Name: `jenkins`
> - URL: `http://<YOUR_EC2_IP>:8080/sonarqube-webhook/` ← trailing slash is required

**2. Generate a token in SonarQube:**
> Profile icon → My Account → Security → Generate Token → copy it

**3. Add token to Jenkins credentials:**
> Manage Jenkins → Credentials → System → Global → Add Credentials
> - Kind: `Secret text` · Secret: paste token · ID: `sonar-token`

**4. Register SonarQube server in Jenkins:**
> Manage Jenkins → System → SonarQube servers → Add SonarQube
> - Name: `Sonar` ← must match `withSonarQubeEnv('Sonar')` in Jenkinsfile
> - URL: `http://<YOUR_EC2_IP>:9000`
> - Token: `sonar-token`

**5. Configure Tools:**
> Manage Jenkins → Tools

| Tool | Name | How to install |
|---|---|---|
| SonarQube Scanner | `Sonar` ← must match `tool 'Sonar'` in Jenkinsfile | Install automatically |
| Dependency-Check | `dc` ← must match `odcInstallation: 'dc'` in Jenkinsfile | GitHub releases → Latest stable |

---

## 📦 Configure the App

### Fix `.env.docker` Files

**`backend/.env.docker`** — replace `<YOUR_EC2_IP>` with your actual IP:
```env
MONGODB_URI="mongodb://mongo/wanderlust"
REDIS_URL="redis://redis:6379"
PORT=5000
FRONTEND_URL="http://<YOUR_EC2_IP>:5173"
ACCESS_COOKIE_MAXAGE=120000
ACCESS_TOKEN_EXPIRES_IN='120s'
REFRESH_COOKIE_MAXAGE=120000
REFRESH_TOKEN_EXPIRES_IN='120s'
JWT_SECRET=your_long_random_secret_here
NODE_ENV=Development
```

**`frontend/.env.docker`:**
```env
VITE_API_PATH="http://<YOUR_EC2_IP>:5000"
```

> ⚠️ The original files ship with `mongo-service` / `redis-service` — those are Kubernetes names.  
> For Docker Compose use `mongo` and `redis` (the `container_name` values in `docker-compose.yml`).

### Seed the Database (first time only)

```bash
docker cp backend/data/sample_posts.json mongo:/tmp/sample_posts.json

docker exec mongo mongoimport \
  --db wanderlust \
  --collection posts \
  --file /tmp/sample_posts.json \
  --jsonArray
```

Expected: `15 document(s) imported successfully.`

---

## 🚀 Create the Jenkins Pipeline

1. New Item → name: `Wanderlust-CI` → Pipeline → OK
2. **General** → tick `GitHub project` → paste your repo URL
3. **Build Triggers** → tick `GitHub hook trigger for GITScm polling`
4. **Pipeline** — choose one:

**Quick (paste script):**
- Select `Pipeline script` → paste the Jenkinsfile from above

**Recommended (auto-reads from repo):**
- Select `Pipeline script from SCM`
- SCM: `Git` · Repo URL: your GitHub URL · Branch: `*/devops`
- Script Path: `Jenkinsfile`

5. Save → **Build Now** to run your first pipeline.

---

## 🤖 Auto-Trigger on Push

Add a GitHub webhook so every `git push` triggers Jenkins automatically.

> GitHub → your repo → Settings → Webhooks → Add webhook
> - Payload URL: `http://<YOUR_EC2_IP>:8080/github-webhook/`
> - Content type: `application/json`
> - Events: **Just the push event**
> - Save

---

## 🌐 Custom Domain (Optional)

Map `yourdomain.com` to your EC2 IP using Route 53 + Hostinger.

1. Route 53 → Hosted Zones → Create Hosted Zone → enter domain → Public
2. Copy the 4 AWS nameservers Route 53 gives you
3. Hostinger → Domains → DNS/Nameservers → Custom → paste the 4 NS records
4. Route 53 → Create Record → Type `A` → value: your EC2 IP → Save
5. Update env files:
   ```env
   # frontend/.env.docker
   VITE_API_PATH="http://yourdomain.com:5000"

   # backend/.env.docker
   FRONTEND_URL="http://yourdomain.com:5173"
   ```
6. `docker-compose up -d --build`

DNS takes 15 min – 48 hrs to propagate.

---

## 📂 Project Structure

```
wanderlust/
├── Jenkinsfile                  ← CI/CD pipeline (6 stages)
├── docker-compose.yml           ← frontend · backend · mongo · redis
├── backend/
│   ├── Dockerfile               ← multi-stage Node.js build
│   ├── .env.docker              ← ⚠️ set your IP here
│   ├── .env.sample              ← template for local dev
│   └── data/
│       └── sample_posts.json   ← 15 sample travel posts for MongoDB
├── frontend/
│   ├── .env.docker              ← ⚠️ set your IP here
│   └── .env.sample
└── Automations/
    ├── updateBackend.sh         ← auto-injects EC2 IP into backend env
    └── updateFrontend.sh        ← auto-injects EC2 IP into frontend env
```

---

## 🐛 Troubleshooting

<details>
<summary><b>docker ps → "permission denied"</b></summary>

```bash
sudo usermod -aG docker $USER && newgrp docker
```
Or log out and SSH back in — the group change needs a new session.
</details>

<details>
<summary><b>Backend container keeps restarting</b></summary>

```bash
docker logs backend
```
Fix `backend/.env.docker` — service names must match `container_name` in `docker-compose.yml`:
```env
MONGODB_URI="mongodb://mongo/wanderlust"   # not mongo-service
REDIS_URL="redis://redis:6379"              # not redis-service
PORT=5000                                   # not 8080
```
</details>

<details>
<summary><b>Frontend loads but no posts appear</b></summary>

Database is empty. Run the seed command in the Configure section above.  
Also check `frontend/.env.docker` — `VITE_API_PATH` must point to port `5000`, not `31100`.
</details>

<details>
<summary><b>Jenkins pipeline fails at SonarQube stage</b></summary>

- Tool name in **Manage Jenkins → Tools** must be exactly `Sonar` (capital S)
- Server name in **Manage Jenkins → System** must also be exactly `Sonar`
- Both must match what's in the Jenkinsfile: `tool 'Sonar'` and `withSonarQubeEnv('Sonar')`
</details>

<details>
<summary><b>Sonar Quality Gate hangs for 2 minutes then times out</b></summary>

The SonarQube webhook URL is wrong or missing. In SonarQube:  
Administration → Configuration → Webhooks → check URL ends with `/sonarqube-webhook/` (trailing slash required).
</details>

<details>
<summary><b>Jenkins can't run docker-compose</b></summary>

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
</details>

---

## 📄 License

[MIT](LICENSE)

---

<div align="center">

Made with ❤️ · [Report a Bug](../../issues) · [Request a Feature](../../issues)

</div>
