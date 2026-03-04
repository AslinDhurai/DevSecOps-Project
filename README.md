<div align="center">
  <img src="./public/assets/DevSecOps.png" alt="DevSecOps Banner" width="100%" height="100%">
  <br>
  <a href="http://netflix-clone-with-tmdb-using-react-mui.vercel.app/">
    <img src="./public/assets/netflix-logo.png" alt="Netflix Logo" width="100" height="32">
  </a>
</div>

<br />

<div align="center">
  <img src="./public/assets/home-page.png" alt="Home Page" width="100%" height="100%">
  <p align="center">Home Page</p>
</div>

# Deploy Netflix Clone on Cloud using Jenkins (DevSecOps Project)


## Video Tutorial

Refer to the official project walkthrough/documentation for a full video-based setup.

---

## Phase 1: Initial Setup and Deployment

### 1) Launch EC2 (Ubuntu 22.04)

- Provision an EC2 instance with Ubuntu 22.04.
- Connect via SSH.

### 2) Clone the Repository

```bash
git clone https://github.com/AslinDhurai/DevSecOps-Project.git
```

### 3) Install Docker and Run the App Container

Install Docker:

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

Build and run container:

```bash
docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest

# cleanup
docker stop <containerid>
docker rmi -f netflix
```

If the app shows an API error, generate and pass your TMDB key.

### 4) Get TMDB API Key

1. Login/Register at TMDB.
2. Go to Profile → Settings → API.
3. Create API key and submit details.
4. Rebuild image with key:

```bash
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```

---

## Phase 2: Security

### 1) Install SonarQube and Trivy

Run SonarQube:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access SonarQube:

- `http://<public-ip>:9000`
- Default credentials: `admin / admin`

Install Trivy:

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

Scan image:

```bash
trivy image <imageid>
```

### 2) Integrate SonarQube

- Connect SonarQube in Jenkins.
- Configure project analysis and quality gate.

---

## Phase 3: CI/CD Setup (Jenkins)

### 1) Install Jenkins (with Java)

```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Access Jenkins:

- `http://<public-ip>:8080`

### 2) Install Required Jenkins Plugins

Go to **Manage Jenkins → Plugins → Available Plugins** and install:

1. Eclipse Temurin Installer
2. SonarQube Scanner
3. NodeJS Plugin
4. Email Extension Plugin
5. OWASP Dependency-Check
6. Docker-related plugins:
   - Docker
   - Docker Commons
   - Docker Pipeline
   - Docker API
   - docker-build-step

### 3) Configure Global Tools

Go to **Manage Jenkins → Tools**:

- Install JDK 17
- Install NodeJS 16
- Install Sonar Scanner tool
- Configure OWASP Dependency-Check as `DP-Check`

### 4) Add Credentials

Go to **Manage Jenkins → Credentials**:

- Add Sonar token as Secret Text (e.g. `Sonar-token`)
- Add DockerHub credentials (ID used in pipeline: `docker`)

### 5) Example Jenkins Pipeline

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
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/AslinDhurai/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
}
```

### 6) Extended Pipeline (Security + Docker + Deploy)

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/AslinDhurai/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                       sh "docker tag netflix aslindhurai/netflix:latest "
                       sh "docker push aslindhurai/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image aslindhurai/netflix:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 aslindhurai/netflix:latest'
            }
        }
    }
}
```

If Docker login fails in Jenkins agent:

```bash
sudo su
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## Phase 4: Monitoring (New EC2 Machine)

**Note:** Set up monitoring on a **separate/new EC2 instance** (Ubuntu 22.04) instead of the Jenkins machine.

### 1) Install Prometheus

Create user and download:

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```

Extract and place files:

```bash
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

Create service file:

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Use:

```ini
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
```

Enable and start:

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Access:

- `http://<your-server-ip>:9090`

### 2) Install Node Exporter

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

Create service file:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Use:

```ini
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
```

Enable and start:

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

### 3) Configure Prometheus Scrape Jobs

Update `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:<your-jenkins-port>']
```

Validate and reload:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

Targets page:

- `http://<your-prometheus-ip>:9090/targets`

### 4) Install Grafana

Install dependencies and Grafana:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
```

Enable and start:

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Access Grafana:

- `http://<your-server-ip>:3000`
- Default login: `admin / admin` (change password on first login)

Add Prometheus datasource:

- URL: `http://localhost:9090`
- Click **Save & Test**

Import dashboard:

- Use dashboard ID `1860` (Node Exporter full)

---

## Phase 5: Notification

### Implement Notification Services

- Configure Jenkins email notifications (or equivalent notification channel).

---

## Phase 6: Kubernetes

### 1) Create Kubernetes Cluster with Nodegroups

Set up your cluster and node groups for scalable deployment.

### 2) Monitor Kubernetes with Prometheus

Install Node Exporter in cluster using Helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus-node-exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
```

Add app scrape job in `prometheus.yml`:

```yaml
- job_name: 'Netflix'
  metrics_path: '/metrics'
  static_configs:
    - targets: ['node1Ip:9100']
```

Reload/restart Prometheus after update.

### 3) Deploy with ArgoCD

1. Install ArgoCD (EKS workshop guide can be followed).
2. Configure your GitHub repo as source.
3. Create ArgoCD app with:
   - `name`
   - `destination`
   - `project`
   - `source` (repo URL, revision, path)
   - `syncPolicy` (auto sync, prune, self-heal)
4. Access app:
   - Ensure port `30007` is open in security group.
   - Open `http://<node-ip>:30007`

---

## Phase 7: Cleanup

### Cleanup AWS Resources

- Terminate EC2 instances that are no longer required.

---

## Notes

- This README intentionally keeps the same cloud-first flow and command style.
- If your setup differs (usernames, IPs, credentials IDs), replace placeholders accordingly.
- You can also use `pipeline.txt` from this repository directly in Jenkins.
