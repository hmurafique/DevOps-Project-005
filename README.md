# DevOps Project 005 — DevSecOps Netflix Clone CI/CD with Monitoring

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-red?style=flat&logo=jenkins)
![Docker](https://img.shields.io/badge/Docker-Containerization-blue?style=flat&logo=docker)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?style=flat&logo=kubernetes)
![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?style=flat&logo=sonarqube)
![Trivy](https://img.shields.io/badge/Trivy-Security%20Scan-1904DA?style=flat)
![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-E6522C?style=flat&logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-Dashboard-F46800?style=flat&logo=grafana)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=flat&logo=amazonaws)

A complete DevSecOps pipeline that builds, secures, and deploys a Netflix Clone application on Kubernetes, with full monitoring using Prometheus and Grafana.

---

## Architecture Overview

```
GitHub (Netflix Clone Source Code)
        ↓
Jenkins (CI/CD Pipeline) — EC2 t2.large
        ↓
SonarQube (Code Quality Analysis) — Docker Container
        ↓
OWASP Dependency Check + Trivy (Security Scanning)
        ↓
Docker Build → Push to DockerHub
        ↓
Kubernetes Cluster (Master + Worker) — EC2 t2.medium
        ↓
Netflix App Live on NodePort 30007
        ↓
Prometheus + Grafana (Monitoring) — EC2 t2.medium
```

---

## Tools and Technologies

| Tool | Purpose |
|---|---|
| Jenkins | CI/CD Pipeline automation |
| SonarQube | Static code quality analysis |
| Trivy | Docker image vulnerability scanning |
| OWASP Dependency Check | Application dependency security scan |
| Docker | Containerization |
| DockerHub | Container image registry |
| Kubernetes (kubeadm) | Container orchestration |
| Prometheus | Metrics collection |
| Grafana | Monitoring dashboards |
| Node Exporter | System metrics exporter |
| TMDB API | Netflix clone movie data |
| AWS EC2 | Cloud infrastructure |

---

## Infrastructure

| Server | Instance Type | Purpose |
|---|---|---|
| netflix-server | t2.large | Jenkins + Docker + SonarQube + Trivy |
| k8s-master | t2.medium | Kubernetes Control Plane |
| k8s-worker | t2.medium | Kubernetes Worker Node |
| monitoring-server | t2.medium | Prometheus + Grafana + Node Exporter |

---

## Prerequisites

- AWS account with EC2 access
- GitHub account
- DockerHub account
- TMDB API key (themoviedb.org)
- Gmail account with App Password enabled

---

## Step-by-Step Implementation

### Step 1 — Launch Netflix Server EC2

Launch an EC2 instance with the following configuration:

- **AMI:** Ubuntu 22.04 LTS
- **Instance Type:** t2.large
- **Storage:** 30 GB

Open these ports in the Security Group:

| Port | Purpose |
|---|---|
| 22 | SSH |
| 8080 | Jenkins |
| 9000 | SonarQube |
| 8081 | Netflix App (Docker) |
| 3000 | Grafana |
| 9090 | Prometheus |

---

### Step 2 — Install Java 21

Jenkins latest version requires Java 21.

```bash
sudo apt-get update -y
sudo apt install openjdk-21-jre -y
java --version
```

---

### Step 3 — Install Jenkins

```bash
# Download Jenkins WAR file
wget https://get.jenkins.io/war-stable/latest/jenkins.war

# Create Jenkins home directory
mkdir -p /var/jenkins_home

# Create systemd service
cat > /etc/systemd/system/jenkins.service << 'EOF'
[Unit]
Description=Jenkins Server
After=network.target

[Service]
Environment="JENKINS_HOME=/var/jenkins_home"
ExecStart=/usr/bin/java -jar /home/ubuntu/jenkins.war --httpPort=8080
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

# Start Jenkins
systemctl daemon-reload
systemctl enable jenkins
systemctl start jenkins
```

Get the initial admin password:

```bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

Open Jenkins at `http://<netflix-server-ip>:8080` and complete the setup wizard.

---

### Step 4 — Install Docker and Trivy

```bash
# Install Docker
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock

# Install Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

# Verify
docker --version
trivy --version
```

---

### Step 5 — Run SonarQube Container

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Open SonarQube at `http://<netflix-server-ip>:9000`

- Default login: `admin / admin`
- Go to **Administration → Security → Users → Administrator → Tokens**
- Generate a token named `jenkins-sonar-token` and copy it

---

### Step 6 — Get TMDB API Key

1. Go to [themoviedb.org](https://www.themoviedb.org)
2. Create a free account
3. Go to **Settings → API → Create**
4. Select **Developer** and fill in the form
5. Copy the **API Key (v3 auth)**

---

### Step 7 — Configure Jenkins

#### Install Plugins

Go to **Manage Jenkins → Plugins → Available plugins** and install:

- Eclipse Temurin Installer
- SonarQube Scanner
- NodeJs Plugin
- OWASP Dependency-Check
- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- docker-build-step
- Prometheus metrics
- Email Extension Plugin
- SSH Agent

Tick **Restart Jenkins when installation is complete**.

#### Configure Tools

Go to **Manage Jenkins → Tools** and configure:

**JDK:**
- Name: `jdk17`
- Install from adoptium.net
- Version: `jdk-17.0.8.1+1`

**NodeJS:**
- Name: `node16`
- Version: `NodeJS 16.20.0`

**SonarQube Scanner:**
- Name: `sonar-scanner`
- Install automatically

**Docker:**
- Name: `docker`
- Install automatically

**Dependency-Check:**
- Name: `DP-Check`
- Install from github.com

#### Add Credentials

Go to **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

**SonarQube Token:**
- Kind: Secret text
- Secret: paste your SonarQube token
- ID: `Sonar-token`

**DockerHub:**
- Kind: Username with password
- Username: your DockerHub username
- Password: your DockerHub password
- ID: `docker`

**Gmail:**
- Kind: Username with password
- Username: your Gmail address
- Password: your Gmail App Password (16 digits)
- ID: `mail-cred`

#### Configure SonarQube Server

Go to **Manage Jenkins → System → SonarQube servers → Add SonarQube**:

- Name: `sonar-server`
- Server URL: `http://<netflix-server-ip>:9000`
- Token: select `Sonar-token`

#### Configure Email Notification

Go to **Manage Jenkins → System → Extended E-mail Notification**:

- SMTP server: `smtp.gmail.com`
- SMTP Port: `465`
- Credentials: `mail-cred`
- Use SSL: ✅

#### Add SonarQube Webhook

In SonarQube go to **Administration → Configuration → Webhooks → Create**:

- Name: `jenkins`
- URL: `http://<netflix-server-ip>:8080/sonarqube-webhook/`

---

### Step 8 — Create Jenkins Pipeline

Go to **Jenkins → New Item → Pipeline** and name it `Netflix`.

Use this pipeline script (replace `YOUR_TMDB_API_KEY` and `YOUR_DOCKERHUB_USERNAME`):

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install --legacy-peer-deps'
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build and Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build --build-arg TMDB_V3_API_KEY=YOUR_TMDB_API_KEY -t netflix .'
                        sh 'docker tag netflix YOUR_DOCKERHUB_USERNAME/netflix:latest'
                        sh 'docker push YOUR_DOCKERHUB_USERNAME/netflix:latest'
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image YOUR_DOCKERHUB_USERNAME/netflix:latest > trivyimage.txt'
            }
        }
        stage('Deploy to Container') {
            steps {
                sh 'docker stop netflix || true && docker rm netflix || true'
                sh 'docker run -d --name netflix -p 8081:80 YOUR_DOCKERHUB_USERNAME/netflix:latest'
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'your-email@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```

Click **Build Now**. The pipeline will complete all stages successfully.

Access the Netflix app at `http://<netflix-server-ip>:8081`

---

### Step 9 — Set Up Kubernetes Cluster

#### Launch Two EC2 Instances

| Name | Instance Type | Purpose |
|---|---|---|
| k8s-master | t2.medium | Control Plane |
| k8s-worker | t2.medium | Worker Node |

Open these ports on both instances:

| Port | Purpose |
|---|---|
| 22 | SSH |
| 6443 | Kubernetes API Server |
| 80, 443 | Application access |
| 30000-32767 | NodePort services |

#### Install Kubernetes on Both Nodes

Run these commands on **both master and worker**:

```bash
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install containerd
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install Kubernetes packages
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### Initialize Master Node

Run on **master only**:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel network plugin
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Check nodes
kubectl get nodes
```

#### Join Worker Node

Copy the `kubeadm join` command from the master init output and run it on the **worker**:

```bash
sudo kubeadm join <master-private-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --ignore-preflight-errors=all
```

Verify on master:

```bash
kubectl get nodes
# Both nodes should show Ready status
```

---

### Step 10 — Deploy Netflix on Kubernetes

Run on the **master node**:

```bash
mkdir -p ~/netflix-k8s
cd ~/netflix-k8s

# Create Deployment
cat > deployment.yml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netflix-app
  labels:
    app: netflix
spec:
  replicas: 2
  selector:
    matchLabels:
      app: netflix
  template:
    metadata:
      labels:
        app: netflix
    spec:
      containers:
      - name: netflix
        image: YOUR_DOCKERHUB_USERNAME/netflix:latest
        ports:
        - containerPort: 80
EOF

# Create Service
cat > service.yml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: netflix-service
spec:
  selector:
    app: netflix
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30007
EOF

# Deploy
kubectl apply -f deployment.yml
kubectl apply -f service.yml

# Verify
kubectl get pods
kubectl get svc
```

Access the Netflix app at `http://<k8s-worker-ip>:30007`

---

### Step 11 — Set Up Monitoring Server

Launch a new EC2 instance:

- **Name:** monitoring-server
- **Instance Type:** t2.medium
- **AMI:** Ubuntu 22.04 LTS
- **Storage:** 20 GB

Open these ports:

| Port | Purpose |
|---|---|
| 22 | SSH |
| 9090 | Prometheus |
| 3000 | Grafana |
| 9100 | Node Exporter |

---

### Step 12 — Install Prometheus

```bash
# Create prometheus user
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Download and extract
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz

# Setup directories
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.47.1.linux-amd64/
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

# Create systemd service
cat > /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

---

### Step 13 — Install Node Exporter

```bash
# Create user
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Download and install
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

---

### Step 14 — Configure Prometheus Targets

```bash
cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<netflix-server-ip>:8080']
EOF

# Validate and restart
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

Open Prometheus at `http://<monitoring-server-ip>:9090` → Status → Targets to verify all targets are UP.

---

### Step 15 — Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common

sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update -y
sudo apt-get install -y grafana

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Open Grafana at `http://<monitoring-server-ip>:3000`

- Default login: `admin / admin`
- Set a new password when prompted

#### Add Prometheus Data Source

Go to **Connections → Data Sources → Add data source → Prometheus**:

- URL: `http://<monitoring-server-ip>:9090`
- Click **Save & Test** — should show "Successfully queried the Prometheus API"

#### Import Dashboards

Go to **Dashboards → New → Import**:

| Dashboard | ID |
|---|---|
| Node Exporter Full | `1860` |
| Jenkins Performance and Health Overview | `9964` |

---

## Final Result

| Component | URL | Status |
|---|---|---|
| Jenkins | `http://<netflix-server-ip>:8080` | ✅ |
| SonarQube | `http://<netflix-server-ip>:9000` | ✅ |
| Netflix on Docker | `http://<netflix-server-ip>:8081` | ✅ |
| Netflix on Kubernetes | `http://<k8s-worker-ip>:30007` | ✅ |
| Prometheus | `http://<monitoring-server-ip>:9090` | ✅ |
| Grafana | `http://<monitoring-server-ip>:3000` | ✅ |

---

## Common Issues and Fixes

**Jenkins fails to start — Java version error:**
```bash
apt install openjdk-21-jre -y
update-alternatives --set java /usr/lib/jvm/java-21-openjdk-amd64/bin/java
systemctl restart jenkins
```

**Jenkins GPG key error during apt install:**
Use the WAR file method instead of apt — download `jenkins.war` directly and run with Java.

**Docker image pull error in Kubernetes:**
Use a verified public image. In this project `public.ecr.aws/l6m2t8p7/docker-2048:latest` was used as a fallback.

**Maven .m2 permission error:**
```bash
mkdir -p /home/ubuntu/.m2/repository
chmod -R 777 /home/ubuntu/.m2
chown -R ubuntu:ubuntu /home/ubuntu/.m2
```

**Grafana GPG key error:**
```bash
sudo rm /etc/apt/sources.list.d/grafana.list
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

---

## Cleanup

To avoid AWS charges, stop or terminate instances when not in use:

```bash
# Delete Kubernetes resources first
kubectl delete svc netflix-service
kubectl delete deployment netflix-app

# Stop Docker container
docker stop netflix && docker rm netflix
```

Then stop EC2 instances from AWS Console.

---

## What I Learned

- Building a complete DevSecOps pipeline from scratch
- Integrating security scanning (Trivy, OWASP) into CI/CD
- Setting up a Kubernetes cluster manually using kubeadm
- Deploying containerized applications on Kubernetes with NodePort
- Monitoring infrastructure and CI/CD pipelines using Prometheus and Grafana
- Configuring Jenkins with SonarQube for code quality gates
- Managing Docker images with DockerHub registry
