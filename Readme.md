<div align="center">

# 🌍 Wanderlust — Kubernetes Deployment

**MERN travel blog · Jenkins CI/CD · SonarQube · Trivy · AWS EKS / kubeadm**

[![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](http://jenkins-url:8080)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-Images-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://hub.docker.com/)
[![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)](https://www.sonarsource.com/)
[![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/)

![Preview](https://github.com/krishnaacharyaa/wanderlust/assets/116620586/17ba9da6-225f-481d-87c0-5d5a010a9538)

</div>

---

## 🗺️ What Is This?

Wanderlust is a **MERN stack travel blog** deployed on Kubernetes with a two-pipeline Jenkins CI/CD setup:

- **CI Pipeline** — clones code, runs SonarQube analysis, OWASP scan, Trivy scan, builds Docker images, pushes to DockerHub, then triggers the CD pipeline
- **CD Pipeline** — updates the image tags in Kubernetes manifests, commits them back to GitHub, and Kubernetes pulls the new images automatically

```
Git Push ──► Jenkins CI
              │
              ├── SonarQube Analysis
              ├── OWASP Dependency Check
              ├── Trivy FS Scan
              ├── Docker Build + Push to DockerHub
              └── Trigger Jenkins CD
                      │
                      ├── Update kubernetes/backend.yaml image tag
                      ├── Update kubernetes/frontend.yaml image tag
                      └── Git push → Kubernetes redeploys
```

---

## 🏗️ Architecture

```
                        ┌─────────────────────────────────┐
                        │         Kubernetes Cluster        │
                        │                                   │
  User ──► NodePort     │  ┌─────────────┐                 │
           :31000  ─────┼─►│  Frontend   │                 │
                        │  │  Pod :5173  │                 │
  User ──► NodePort     │  └──────┬──────┘                 │
           :31100  ─────┼─►│  Backend    │                 │
                        │  │  Pod :8080  │                 │
                        │  └──────┬──────┘                 │
                        │         │                         │
                        │  ┌──────┴──────┐  ┌───────────┐  │
                        │  │   MongoDB   │  │   Redis   │  │
                        │  │  Pod :27017 │  │ Pod :6379 │  │
                        │  └──────┬──────┘  └───────────┘  │
                        │         │                         │
                        │  ┌──────┴──────┐                 │
                        │  │  PV / PVC   │ 5Gi storage     │
                        │  └─────────────┘                 │
                        └─────────────────────────────────┘
```

| Service | Type | Port | NodePort |
|---|---|---|---|
| Frontend | NodePort | 5173 | **31000** |
| Backend | NodePort | 8080 | **31100** |
| MongoDB | ClusterIP | 27017 | internal only |
| Redis | ClusterIP | 6379 | internal only |

---

## 🔁 Jenkins Pipelines

There are **two pipelines**: CI (build & scan) and CD (deploy to Kubernetes).

### CI Pipeline — `Jenkinsfile` (root of repo)

```groovy
pipeline {
    agent any

    environment {
        SONAR_HOME        = tool 'Sonar'            // Jenkins → Tools → SonarQube Scanner
        DOCKERHUB_USER    = 'your-dockerhub-username'
        IMAGE_TAG         = "${BUILD_NUMBER}"        // unique tag per build
    }

    stages {

        stage('Clone Code from GitHub') {
            steps {
                git url: 'https://github.com/YOUR_USERNAME/wanderlust.git',
                    branch: 'devops'
            }
        }

        stage('SonarQube Quality Analysis') {
            steps {
                withSonarQubeEnv('Sonar') {          // Jenkins → System → SonarQube server name
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

        stage('Update .env for Kubernetes') {
            parallel {
                stage('Backend env') {
                    steps {
                        dir('Automations') {
                            sh 'bash updateBackend.sh'
                        }
                    }
                }
                stage('Frontend env') {
                    steps {
                        dir('Automations') {
                            sh 'bash updateFrontend.sh'
                        }
                    }
                }
            }
        }

        stage('Docker Build Images') {
            steps {
                withDockerRegistry(credentialsId: 'dockerhub-cred', url: '') {
                    dir('backend') {
                        sh "docker build -t ${DOCKERHUB_USER}/wanderlust-backend:${IMAGE_TAG} ."
                    }
                    dir('frontend') {
                        sh "docker build -t ${DOCKERHUB_USER}/wanderlust-frontend:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-backend-image.html ${DOCKERHUB_USER}/wanderlust-backend:${IMAGE_TAG}"
                sh "trivy image --format table -o trivy-frontend-image.html ${DOCKERHUB_USER}/wanderlust-frontend:${IMAGE_TAG}"
            }
        }

        stage('Docker Push to DockerHub') {
            steps {
                withDockerRegistry(credentialsId: 'dockerhub-cred', url: '') {
                    sh "docker push ${DOCKERHUB_USER}/wanderlust-backend:${IMAGE_TAG}"
                    sh "docker push ${DOCKERHUB_USER}/wanderlust-frontend:${IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-fs-report.html, trivy-backend-image.html, trivy-frontend-image.html',
                             fingerprint: true
            archiveArtifacts artifacts: '**/dependency-check-report.xml',
                             fingerprint: true
        }
        success {
            // Trigger CD pipeline and pass the image tag
            build job: 'Wanderlust-CD',
                  parameters: [
                      string(name: 'BACKEND_DOCKER_TAG',  value: "${IMAGE_TAG}"),
                      string(name: 'FRONTEND_DOCKER_TAG', value: "${IMAGE_TAG}")
                  ]
        }
        failure {
            echo 'CI Pipeline failed. CD pipeline will NOT be triggered.'
        }
    }
}
```

---

### CD Pipeline — `GitOps/Jenkinsfile`

> This pipeline is triggered automatically by the CI pipeline above.  
> It updates the image tags in `kubernetes/*.yaml` and pushes to GitHub.

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Frontend image tag from CI')
        string(name: 'BACKEND_DOCKER_TAG',  defaultValue: '', description: 'Backend image tag from CI')
    }

    stages {

        stage('Workspace Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('Clone Code from GitHub') {
            steps {
                git url: 'https://github.com/YOUR_USERNAME/wanderlust.git',
                    branch: 'devops'
            }
        }

        stage('Verify Docker Image Tags') {
            steps {
                sh """
                    echo "Backend  tag : ${params.BACKEND_DOCKER_TAG}"
                    echo "Frontend tag : ${params.FRONTEND_DOCKER_TAG}"
                """
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                dir('kubernetes') {
                    sh """
                        sed -i 's|wanderlust-backend:.*|wanderlust-backend:${params.BACKEND_DOCKER_TAG}|g'  backend.yaml
                        sed -i 's|wanderlust-frontend:.*|wanderlust-frontend:${params.FRONTEND_DOCKER_TAG}|g' frontend.yaml
                    """
                }
            }
        }

        stage('Push Updated Manifests to GitHub') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'Github-cred', gitToolName: 'Default')]) {
                    sh '''
                        git config user.email "jenkins@wanderlust.com"
                        git config user.name  "Jenkins CD"
                        git add kubernetes/backend.yaml kubernetes/frontend.yaml
                        git commit -m "CD: update image tags to build ${BUILD_NUMBER}" || echo "Nothing to commit"
                        git push https://github.com/YOUR_USERNAME/wanderlust.git devops
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Manifests updated. Kubernetes will pull the new images shortly.'
        }
        failure {
            echo 'CD Pipeline failed. Check logs above.'
        }
    }
}
```

### CI/CD Stage Summary

| Pipeline | Stage | What it does |
|---|---|---|
| CI | Clone Code | Pulls latest code from `devops` branch |
| CI | SonarQube Analysis | Static code quality scan |
| CI | OWASP Dependency Check | Scans packages for known CVEs |
| CI | Sonar Quality Gate | Waits for quality threshold result |
| CI | Trivy FS Scan | Scans source filesystem for vulnerabilities |
| CI | Update .env | Injects current EC2 IP into `.env.docker` files |
| CI | Docker Build | Builds backend + frontend images |
| CI | Trivy Image Scan | Scans the built Docker images |
| CI | Docker Push | Pushes images to DockerHub |
| CD | Update Manifests | Edits `kubernetes/*.yaml` with new image tags |
| CD | Git Push | Commits manifest changes back to repo |

---

## ⚡ Quick Start

### 1️⃣ Launch EC2 Instances

You need **2 instances** for a simple kubeadm cluster (or 1 if using Minikube/K3s for testing):

| Instance | Role | Type | Storage | Open Ports |
|---|---|---|---|---|
| `wanderlust-master` | Control Plane | `t2.medium` | 20 GB | `22, 6443, 2379-2380, 10250-10252` |
| `wanderlust-worker` | Worker Node | `t2.large` | 25 GB | `22, 10250, 31000, 31100, 9000, 8080` |

> Run **Jenkins and SonarQube on the worker node** — the control plane should stay light.

---

### 2️⃣ Install Docker on All Nodes

Run this on **both master and worker**:

```bash
sudo apt-get update -y
sudo apt-get install docker.io -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

---

### 3️⃣ Set Up Kubernetes Cluster (kubeadm)

**On both master and worker — disable swap and install kubeadm:**

```bash
# Disable swap (Kubernetes requirement)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

# Required sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**On master only — initialize the cluster:**

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Set up kubectl for your user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI (pod networking)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**On worker only — join the cluster:**

```bash
# Copy the kubeadm join command printed by kubeadm init above, e.g.:
sudo kubeadm join <MASTER_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Verify from master:
```bash
kubectl get nodes   # both nodes should show Ready
```

---

### 4️⃣ Install Jenkins (on Worker Node)

```bash
sudo apt-get install fontconfig openjdk-17-jre -y

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update -y && sudo apt-get install jenkins -y
sudo systemctl start jenkins && sudo systemctl enable jenkins

# Allow Jenkins to run Docker and kubectl
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Get admin password → open `http://<WORKER_IP>:8080`:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Give Jenkins access to kubectl (so it can apply manifests):
```bash
sudo cp /etc/kubernetes/admin.conf /var/lib/jenkins/.kube/config
sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config
```

---

### 5️⃣ Install SonarQube (on Worker Node)

```bash
docker run -itd \
  --name SonarQube-server \
  -p 9000:9000 \
  sonarqube:lts-community
```

Open `http://<WORKER_IP>:9000` → login `admin` / `admin` → change password.

---

### 6️⃣ Install Trivy (on Worker Node)

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

### Plugins to Install

> Manage Jenkins → Plugins → Available Plugins

- ✅ SonarQube Scanner
- ✅ Sonar Quality Gates
- ✅ OWASP Dependency-Check
- ✅ Docker
- ✅ Docker Pipeline
- ✅ Pipeline: Stage View
- ✅ Kubernetes CLI *(optional — for `kubectl apply` from pipeline)*

### Credentials to Add

> Manage Jenkins → Credentials → System → Global → Add Credentials

| ID | Kind | What it stores |
|---|---|---|
| `sonar-token` | Secret text | SonarQube user token |
| `dockerhub-cred` | Username/Password | DockerHub login |
| `Github-cred` | Username/Password | GitHub username + Personal Access Token |

### Connect SonarQube ↔ Jenkins

**1. SonarQube Webhook:**
> SonarQube → Administration → Configuration → Webhooks → Create
> - Name: `jenkins`
> - URL: `http://<WORKER_IP>:8080/sonarqube-webhook/` ← trailing slash required

**2. Generate SonarQube token:**
> Profile → My Account → Security → Generate Token → copy it → add as `sonar-token` credential in Jenkins

**3. Register SonarQube server in Jenkins:**
> Manage Jenkins → System → SonarQube servers → Add
> - Name: `Sonar` ← must match `withSonarQubeEnv('Sonar')` and `tool 'Sonar'` in Jenkinsfile
> - URL: `http://<WORKER_IP>:9000`
> - Token: `sonar-token`

**4. Configure Tools:**
> Manage Jenkins → Tools

| Tool | Name | How |
|---|---|---|
| SonarQube Scanner | `Sonar` | Install automatically |
| Dependency-Check | `dc` | GitHub releases → Latest stable |

---

## 📦 Configure the App

### Fix `.env.docker` Files

> These are used **during the Docker build** (copied into the image as `.env`).  
> The Automation scripts update the IP automatically when run via Jenkins.

**`backend/.env.docker`** — update manually if not using the Automation scripts:

```env
MONGODB_URI="mongodb://mongo-service/wanderlust"
REDIS_URL="redis://redis-service:6379"
PORT=8080
FRONTEND_URL="http://<WORKER_NODE_IP>:31000"
ACCESS_COOKIE_MAXAGE=120000
ACCESS_TOKEN_EXPIRES_IN='120s'
REFRESH_COOKIE_MAXAGE=120000
REFRESH_TOKEN_EXPIRES_IN='120s'
JWT_SECRET=your_long_random_secret_here
NODE_ENV=Production
```

**`frontend/.env.docker`:**

```env
VITE_API_PATH="http://<WORKER_NODE_IP>:31100"
```

> ℹ️ In Kubernetes, services communicate via **Service names** (`mongo-service`, `redis-service`) defined in the manifest — this is different from Docker Compose where you use `container_name`.

---

### Seed the Database (First Time Only)

After the app is deployed, run this from any node that can reach the cluster:

```bash
# Find the mongo pod name
kubectl get pods -n wanderlust

# Copy and import sample data
kubectl cp backend/data/sample_posts.json wanderlust/<mongo-pod-name>:/tmp/sample_posts.json

kubectl exec -n wanderlust <mongo-pod-name> -- \
  mongoimport --db wanderlust --collection posts \
  --file /tmp/sample_posts.json --jsonArray
```

Expected: `15 document(s) imported successfully.`

---

## ☸️ Deploy to Kubernetes

### 1. Create Namespace

```bash
kubectl create namespace wanderlust
```

### 2. Apply Manifests (in order)

```bash
# Storage first
kubectl apply -f kubernetes/persistentVolume.yaml
kubectl apply -f kubernetes/persistentVolumeClaim.yaml

# Databases
kubectl apply -f kubernetes/mongodb.yaml
kubectl apply -f kubernetes/redis.yaml

# Application
kubectl apply -f kubernetes/backend.yaml
kubectl apply -f kubernetes/frontend.yaml
```

### 3. Verify Everything is Running

```bash
kubectl get all -n wanderlust
```

Expected output:

```
NAME                                      READY   STATUS    RESTARTS
pod/backend-deployment-xxx                1/1     Running   0
pod/frontend-deployment-xxx               1/1     Running   0
pod/mongo-deployment-xxx                  1/1     Running   0
pod/redis-deployment-xxx                  1/1     Running   0

NAME                       TYPE        CLUSTER-IP     PORT(S)
service/backend-service    NodePort    10.x.x.x       8080:31100/TCP
service/frontend-service   NodePort    10.x.x.x       5173:31000/TCP
service/mongo-service      ClusterIP   10.x.x.x       27017/TCP
service/redis-service      ClusterIP   10.x.x.x       6379/TCP
```

### 4. Access the App

| Service | URL |
|---|---|
| 🌐 Frontend | `http://<WORKER_NODE_IP>:31000` |
| ⚙️ Backend API | `http://<WORKER_NODE_IP>:31100` |
| 🔧 Jenkins | `http://<WORKER_NODE_IP>:8080` |
| 📊 SonarQube | `http://<WORKER_NODE_IP>:9000` |

---

## 🚀 Create Jenkins Pipelines

### CI Pipeline

1. New Item → `Wanderlust-CI` → Pipeline → OK
2. General → tick `GitHub project` → paste your repo URL
3. Build Triggers → tick `GitHub hook trigger for GITScm polling`
4. Pipeline → `Pipeline script from SCM`
   - SCM: `Git` · URL: `https://github.com/YOUR_USERNAME/wanderlust` · Branch: `*/devops`
   - Script Path: `Jenkinsfile`
5. Save

### CD Pipeline

1. New Item → `Wanderlust-CD` → Pipeline → OK
2. General → tick `This project is parameterized`
   - Add String Parameter → Name: `BACKEND_DOCKER_TAG` · Default: (blank)
   - Add String Parameter → Name: `FRONTEND_DOCKER_TAG` · Default: (blank)
3. Pipeline → `Pipeline script from SCM`
   - SCM: `Git` · URL: your repo · Branch: `*/devops`
   - Script Path: `GitOps/Jenkinsfile`
4. Save

> The CI pipeline calls `build job: 'Wanderlust-CD'` in its `post { success }` block — so the CD pipeline triggers automatically after a successful CI run. You never need to trigger CD manually.

### Auto-Trigger on Git Push

> GitHub → your repo → Settings → Webhooks → Add webhook
> - Payload URL: `http://<WORKER_IP>:8080/github-webhook/`
> - Content type: `application/json`
> - Events: Just the push event → Save

---

## 📂 Project Structure

```
wanderlust/
├── Jenkinsfile                    ← CI pipeline (build, scan, push)
├── GitOps/
│   └── Jenkinsfile                ← CD pipeline (update k8s manifests)
├── kubernetes/
│   ├── backend.yaml               ← Deployment + NodePort :31100
│   ├── frontend.yaml              ← Deployment + NodePort :31000
│   ├── mongodb.yaml               ← Deployment + ClusterIP + PVC
│   ├── redis.yaml                 ← Deployment + ClusterIP
│   ├── persistentVolume.yaml      ← 5Gi hostPath PV
│   └── persistentVolumeClaim.yaml ← 5Gi PVC bound to PV
├── backend/
│   ├── Dockerfile
│   ├── .env.docker                ← ⚠️ uses mongo-service / redis-service
│   └── data/sample_posts.json    ← seed data
├── frontend/
│   └── .env.docker                ← ⚠️ VITE_API_PATH points to :31100
└── Automations/
    ├── updateBackend.sh           ← updates FRONTEND_URL with current IP
    └── updateFrontend.sh          ← updates VITE_API_PATH with current IP
```

---

## 🔑 Key Differences: Docker Compose vs Kubernetes

| | Docker Compose | Kubernetes |
|---|---|---|
| Service discovery | `container_name` (e.g. `mongo`) | Service name (e.g. `mongo-service`) |
| Backend port | `5000` | `8080` (containerPort in manifest) |
| Frontend access | `:5173` direct | NodePort `:31000` |
| Backend access | `:5000` direct | NodePort `:31100` |
| Scaling | Manual `--scale` | `kubectl scale` or HPA |
| Self-healing | ❌ | ✅ automatic pod restart |
| Rolling updates | ❌ | ✅ zero-downtime by default |

---

## 🐛 Troubleshooting

<details>
<summary><b>Pods stuck in Pending state</b></summary>

```bash
kubectl describe pod <pod-name> -n wanderlust
```

Usually means the PersistentVolume is not bound. Check:
```bash
kubectl get pv,pvc -n wanderlust
```
Make sure the `storageClassName: ""` in the PVC matches the PV (empty string = no storage class, uses the PV directly).
</details>

<details>
<summary><b>Backend pod CrashLoopBackOff</b></summary>

```bash
kubectl logs <backend-pod-name> -n wanderlust
```

Check that `backend/.env.docker` uses Kubernetes service names:
```env
MONGODB_URI="mongodb://mongo-service/wanderlust"   # ✅
REDIS_URL="redis://redis-service:6379"              # ✅
PORT=8080                                           # ✅ (containerPort in backend.yaml)
```
</details>

<details>
<summary><b>Frontend can't reach backend API</b></summary>

`frontend/.env.docker` must point to the NodePort, not the internal port:
```env
VITE_API_PATH="http://<WORKER_NODE_IP>:31100"   # ✅ NodePort of backend-service
```
After changing, rebuild and push the image, then update the image tag in `kubernetes/frontend.yaml`.
</details>

<details>
<summary><b>kubectl not found in Jenkins</b></summary>

```bash
sudo cp /etc/kubernetes/admin.conf /var/lib/jenkins/.kube/config
sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config
# also make sure kubectl is installed on the Jenkins node
which kubectl
```
</details>

<details>
<summary><b>Jenkins can't push to DockerHub</b></summary>

Add DockerHub credentials in Jenkins:  
Manage Jenkins → Credentials → Add → Username/Password  
ID: `dockerhub-cred` · Username: your DockerHub ID · Password: your DockerHub access token
</details>

<details>
<summary><b>CD pipeline can't push to GitHub</b></summary>

The `Github-cred` credential must use a **GitHub Personal Access Token** (not your password).  
Generate one at: GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)  
Scopes needed: `repo` (full)
</details>

<details>
<summary><b>Nodes showing NotReady</b></summary>

```bash
kubectl get nodes
kubectl describe node <node-name>
```

Most common causes:
- CNI not installed → apply Flannel manifest again
- Swap not disabled → `sudo swapoff -a`
- Docker not running → `sudo systemctl start docker`
</details>

---

## 📄 License

[MIT](LICENSE)

---

<div align="center">

Made with ❤️ · [Report a Bug](../../issues) · [Request a Feature](../../issues)

</div>
