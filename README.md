# 🚀 DevOps CI/CD Pipeline — Java App on Kubernetes

A production-grade CI/CD pipeline that automates the full software delivery lifecycle — from code commit to Kubernetes deployment — using Jenkins, Docker, SonarQube, Nexus, Trivy, Prometheus, and Grafana.

---

## 📐 Architecture Overview

![](/diagram-cicd.png)

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| **Jenkins** | CI/CD pipeline orchestration |
| **Maven** | Build automation & dependency management |
| **SonarQube** | Static code analysis & quality gates |
| **Trivy** | Vulnerability scanning (filesystem + images) |
| **Nexus Repository** | Artifact storage & versioning |
| **Docker** | Containerization |
| **Kubernetes** | Container orchestration & deployment |
| **Prometheus** | Metrics collection & alerting |
| **Grafana** | Monitoring dashboards |
| **AWS EC2** | Cloud infrastructure |

---

## ☁️ Infrastructure Setup (AWS)

### EC2 Instances Required

| Instance | Role | Min Type |
|----------|------|----------|
| `k8s-master` | Kubernetes Master Node | t2.medium |
| `k8s-worker-1` | Kubernetes Worker Node | t2.medium |
| `k8s-worker-2` | Kubernetes Worker Node | t2.medium |
| `sonarqube-server` | SonarQube Analysis | t2.medium |
| `nexus-server` | Artifact Repository | t2.medium |
| `jenkins-server` | CI/CD Orchestration | t2.medium |
| `monitoring-server` | Prometheus + Grafana | t2.medium |

> **Security Group:** Open port `6443` to allow worker nodes to join the cluster.

---

## ⚙️ Installation Guides

### 1. Kubernetes Cluster (via Kubeadm)

**Run on ALL nodes (Master + Workers):**

```bash
# Disable swap
sudo swapoff -a

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**Install CRI-O Runtime:**

```bash
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/crio:/prerelease:/main/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] \
  https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y && sudo apt-get install -y cri-o
sudo systemctl enable crio --now
```

**Install Kubernetes packages:**

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*" jq
sudo systemctl enable --now kubelet
```

**Run ONLY on Master Node:**

```bash
sudo kubeadm config images pull
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico network plugin
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Generate join command for workers
kubeadm token create --print-join-command
```

**Run on ALL Worker Nodes:**

```bash
sudo kubeadm reset pre-flight checks
sudo <paste-join-command-from-master> --v=5
```

**Verify the cluster:**

```bash
kubectl get nodes
```

---

### 2. Jenkins

```bash
# install_jenkins.sh
sudo apt install openjdk-17-jre-headless -y

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins -y
```

> Jenkins runs on port **8080** by default.

**Also install kubectl on the Jenkins server:**

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

---

### 3. Docker

```bash
# install_docker.sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

### 4. Nexus Repository

```bash
# Run Nexus as a Docker container
docker run -d --name nexus -p 8081:8081 sonatype/nexus3:latest
```

> Access Nexus at `http://<instance-IP>:8081`

**Get the initial admin password:**

```bash
docker ps                                      # Get container ID
docker exec -it <container-id> /bin/bash
cd sonatype-work/nexus3
cat admin.password
exit
```

---

### 5. SonarQube

```bash
# Run SonarQube as a Docker container
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

> Access SonarQube at `http://<instance-IP>:9000`  
> Default credentials: `admin` / `admin`

---

## 🔗 Private GitHub Repository Setup

```bash
# 1. Clone your private repository
git clone <repository-URL>

# 2. Add your code and stage changes
git add .
git commit -m "Initial commit"

# 3. Push (use Personal Access Token as password when prompted)
git push -u origin main
```

> Generate a **Personal Access Token** from GitHub → Settings → Developer Settings → Personal Access Tokens.

---

## 🔌 Jenkins Plugins to Install

Go to **Manage Jenkins → Manage Plugins → Available** and install:

| Plugin | Purpose |
|--------|---------|
| Eclipse Temurin Installer | Auto-install JDK |
| Pipeline Maven Integration | Use Maven in pipelines |
| Config File Provider | Centralized config management |
| SonarQube Scanner | Code quality integration |
| Kubernetes CLI | Run kubectl from Jenkins |
| Kubernetes | Run Jenkins agents as K8s pods |
| Docker | Docker build & registry integration |
| Docker Pipeline Step | Docker steps in Jenkinsfile |

---

## 🧪 Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/your-username/your-repo.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('Trivy - Filesystem Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BoardGame \
                        -Dsonar.projectKey=BoardGame \
                        -Dsonar.java.binaries=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings',
                          jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t your-dockerhub/boardgame:latest ."
                    }
                }
            }
        }

        stage('Trivy - Docker Image Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "trivy image --format table -o trivy-image-report.html your-dockerhub/boardgame:latest"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push your-dockerhub/boardgame:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-cred',
                               clusterName: 'kubernetes',
                               namespace: 'webapps',
                               serverUrl: 'https://<master-node-IP>:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                    sh "kubectl get pods -n webapps"
                }
            }
        }

    }

    post {
        always {
            script {
                def status = currentBuild.result ?: 'UNKNOWN'
                emailext(
                    subject: "${env.JOB_NAME} - Build ${env.BUILD_NUMBER} - ${status}",
                    body: """
                        Job: ${env.JOB_NAME}
                        Build: ${env.BUILD_NUMBER}
                        Status: ${status}
                        Check console output for details.
                    """,
                    to: 'your-email@example.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
```

---

## 📊 Monitoring Setup

### Prometheus

```bash
# Download from https://prometheus.io/download/
# Extract and run
./prometheus &
# Runs on port 9090 → http://<instance-IP>:9090

# Run Blackbox Exporter
./blackbox_exporter &
# Runs on port 9115
```

**Edit `prometheus.yaml`** to add Blackbox scrape config:

```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://prometheus.io
          - https://prometheus.io
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <monitoring-instance-IP>:9115
```

### Grafana

```bash
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_10.4.2_amd64.deb
sudo dpkg -i grafana-enterprise_10.4.2_amd64.deb
sudo systemctl start grafana-server
# Runs on port 3000 → http://<instance-IP>:3000
```

**Connect Grafana to Prometheus:**
1. Go to **Grafana → Data Sources → Add Data Source**
2. Select **Prometheus**
3. Enter: `http://<monitoring-instance-IP>:9090`
4. Import a community dashboard (e.g., Dashboard ID `7587` for Blackbox Exporter)

---

## 📁 Project Structure

```
├── src/                        # Java application source
├── Dockerfile                  # Docker image definition
├── deployment-service.yaml     # Kubernetes deployment manifest
├── pom.xml                     # Maven build configuration
└── Jenkinsfile                 # CI/CD pipeline definition
```

---

## 📚 References

- [Jenkins Docs](https://www.jenkins.io/doc/)
- [Maven Docs](https://maven.apache.org/guides/index.html)
- [SonarQube Docs](https://docs.sonarqube.org/latest/)
- [Trivy Docs](https://github.com/aquasecurity/trivy)
- [Nexus Docs](https://help.sonatype.com/repomanager3)
- [Docker Docs](https://docs.docker.com/)
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [Prometheus Docs](https://prometheus.io/docs/)
- [Grafana Docs](https://grafana.com/docs/)
