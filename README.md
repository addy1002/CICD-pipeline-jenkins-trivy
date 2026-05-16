# 🚀 Insure-Me DevSecOps CI/CD Pipeline

A complete end-to-end CI/CD pipeline with security scanning, built on AWS.

> **Replace all `YOUR_*` placeholders** with your actual values before using.

---

## 🧰 Tools Used

| Tool | What it does |
|------|-------------|
| Jenkins | Runs the CI/CD pipeline |
| SonarQube | Scans code quality |
| Docker | Containerizes the app |
| Trivy | Scans Docker image for vulnerabilities |
| AWS EC2 | Hosts everything |
| AWS S3 | Stores build artifacts |
| DockerHub | Stores Docker images |
| Maven | Builds the Java project |

---

## 🏗️ Pipeline Flow

```
GitHub → Jenkins → Maven Build → SonarQube → Quality Gate
       → S3 Upload → Docker Build → DockerHub → Trivy Scan → Deploy
```

---

## ☁️ Step 1 — Launch EC2 Instance

**Go to AWS Console → EC2 → Launch Instance**

| Setting | Value |
|---------|-------|
| OS | Ubuntu 22.04 |
| Instance Type | `m7i-flex.large` (2 vCPU, 8 GB RAM) |
| Storage | 20 GB |
| Key Pair | Create or use existing |

> ⚠️ Don't use `t2.micro` — SonarQube alone needs ~2 GB RAM. Free tier won't work here.

**Open these ports in Security Group (Inbound Rules):**

| Port | Purpose |
|------|---------|
| 22 | SSH |
| 8080 | Jenkins |
| 9000 | SonarQube |
| 8089 | Your App |

---

## ☕ Step 2 — Install Java & Jenkins

SSH into your EC2 instance, then run:

```bash
# Update packages
sudo apt update

# Install Java
sudo apt install fontconfig openjdk-21-jre -y
java -version   # should print version 21.x
```

```bash
# Add Jenkins repo
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

**Access Jenkins in browser:**
```
http://YOUR_EC2_PUBLIC_IP:8080
```

**Get the initial admin password:**
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste it in the browser → Install Suggested Plugins → Create your admin account.

---

## 🐳 Step 3 — Install Docker

```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

# Allow Jenkins and Ubuntu user to use Docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock

# Restart both services
sudo systemctl restart docker
sudo systemctl restart jenkins

# Verify
docker --version
docker ps
```

---

## 🔍 Step 4 — Run SonarQube (via Docker)

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

**Access SonarQube:**
```
http://YOUR_EC2_PUBLIC_IP:9000
```

**Default login:** `admin` / `admin` → It will ask you to change the password.

**Generate a token for Jenkins:**
- Go to: My Account → Security → Generate Token
- Name it `jenkins-token`, copy the value — you'll need it later.

---

## 📦 Step 5 — Install AWS CLI

```bash
sudo snap install aws-cli --classic
aws --version   # verify
```

---

## 🔌 Step 6 — Install Jenkins Plugins

In Jenkins → **Manage Jenkins → Plugins → Available**

Search and install these:

- ✅ Pipeline Stage View
- ✅ Maven Integration
- ✅ SonarQube Scanner
- ✅ AWS Credentials
- ✅ S3 publisher
- ✅ Docker
- ✅ Docker Pipeline

Restart Jenkins after installing.

---

## 🛠️ Step 7 — Configure Jenkins Tools

Go to: **Manage Jenkins → Tools**

| Tool | Name to use | How to add |
|------|-------------|-----------|
| Maven | `maven` | Add Maven → Install automatically → pick latest |
| SonarQube Scanner | `sonar-scanner` | Add SonarQube Scanner → Install automatically |
| Docker | `docker` | Add Docker → Install automatically |

Click **Save**.

---

## 🔑 Step 8 — Add Jenkins Credentials

Go to: **Manage Jenkins → Credentials → System → Global → Add Credentials**

Add these 3 credentials:

**1. DockerHub**
| Field | Value |
|-------|-------|
| Kind | Username with password |
| Username | YOUR_DOCKERHUB_USERNAME |
| Password | YOUR_DOCKERHUB_PASSWORD |
| ID | `docker-cred` |

**2. AWS**
| Field | Value |
|-------|-------|
| Kind | AWS Credentials |
| Access Key ID | YOUR_AWS_ACCESS_KEY |
| Secret Access Key | YOUR_AWS_SECRET_KEY |
| ID | `aws-cred` |

**3. SonarQube Token**
| Field | Value |
|-------|-------|
| Kind | Secret text |
| Secret | (paste the token from Step 4) |
| ID | `sonar-cred` |

---

## 🔗 Step 9 — Connect SonarQube to Jenkins

Go to: **Manage Jenkins → System**

Scroll to **SonarQube servers** → Add SonarQube:

| Field | Value |
|-------|-------|
| Name | `sonar-server` |
| Server URL | `http://YOUR_EC2_PUBLIC_IP:9000` |
| Auth token | select `sonar-cred` |

Click **Save**.

---

## 🔔 Step 10 — Set Up SonarQube Webhook

In SonarQube → **Administration → Configuration → Webhooks → Create**

| Field | Value |
|-------|-------|
| Name | jenkins |
| URL | `http://YOUR_EC2_PUBLIC_IP:8080/sonarqube-webhook/` |

Click **Create**.

---

## 🪣 Step 11 — Create S3 Bucket

Go to AWS Console → S3 → Create Bucket

- Name: `YOUR_BUCKET_NAME` (must be globally unique)
- Region: `ap-south-1` (or your preferred region)
- Block Public Access: keep enabled
- Click **Create bucket**

---

## 🛡️ Step 12 — Install Trivy

```bash
# Install dependencies
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

# Add GPG key
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

# Add repo
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
https://aquasecurity.github.io/trivy-repo/deb generic main" | \
sudo tee -a /etc/apt/sources.list.d/trivy.list

# Install
sudo apt-get update
sudo apt-get install trivy -y

# Verify
trivy --version
```

---

## 🔨 Step 13 — Create Jenkins Pipeline Job

In Jenkins → **New Item**

- Name: `insure-me-pipeline`
- Type: **Pipeline**
- Click OK

In the job config, scroll to **Pipeline** section → paste the Jenkinsfile below.

---

## 📄 Jenkinsfile

> **Replace `YOUR_DOCKERHUB_USERNAME` and `YOUR_BUCKET_NAME` before running.**

```groovy
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        S3_BUCKET      = "YOUR_BUCKET_NAME"
        REGION         = "ap-south-1"
        warFile        = "target/Insurance-0.0.1-SNAPSHOT.jar"
        DOCKER_IMAGE   = "YOUR_DOCKERHUB_USERNAME/insureme"
    }

    stages {

        stage('Pull Code') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/mukundDeo9325/Project-InsureMe1.git']]
                )
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=InsureMe \
                        -Dsonar.projectName=InsureMe \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target/classes
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-cred'
                }
            }
        }

        stage('Upload to S3') {
            steps {
                withCredentials([aws(
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                    credentialsId: 'aws-cred'
                )]) {
                    sh 'aws s3 cp ${warFile} s3://${S3_BUCKET}/Artifacts/ --region ${REGION}'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'dockerHubUser',
                    passwordVariable: 'dockerHubPassword'
                )]) {
                    sh "docker login -u ${dockerHubUser} -p ${dockerHubPassword}"
                    sh 'docker push ${DOCKER_IMAGE}'
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh 'trivy image ${DOCKER_IMAGE}:latest > trivy-report.txt'
            }
        }

        stage('Deploy Container') {
            steps {
                sh 'docker run -itd --name insure-me -p 8089:8081 ${DOCKER_IMAGE}'
            }
        }
    }
}
```

---

## ▶️ Step 14 — Run the Pipeline

In Jenkins → open your job → click **Build Now**

The pipeline will:
1. Pull code from GitHub
2. Build with Maven
3. Scan with SonarQube
4. Wait for Quality Gate result
5. Upload `.jar` to S3
6. Build Docker image
7. Push to DockerHub
8. Run Trivy scan → saves `trivy-report.txt`
9. Deploy container

---

## 🌐 Step 15 — Access Your App

```
http://YOUR_EC2_PUBLIC_IP:8089
```

---

## 📸 Screenshots

> Replace these with your own screenshots after completing the project.

| Step | Screenshot |
|------|-----------|
| EC2 Instance | ![EC2](screenshots/ec2-instance.png) |
| Jenkins Setup | ![Jenkins](screenshots/jenkins-setup.png) |
| SonarQube Result | ![SonarQube](screenshots/sonarqube-result.png) |
| Pipeline Success | ![Pipeline](screenshots/pipeline-success.png) |
| App Running | ![App](screenshots/app-running.png) |

---

## ⚠️ Common Issues & Fixes

| Problem | Fix |
|---------|-----|
| Jenkins can't run Docker commands | Run `sudo chmod 777 /var/run/docker.sock` and restart Jenkins |
| SonarQube container keeps restarting | Not enough RAM — use at least 4 GB, ideally 8 GB |
| S3 upload fails | Check AWS credentials and IAM permissions (needs `s3:PutObject`) |
| Trivy scan fails | Run `sudo apt-get update && sudo apt-get install trivy -y` again |
| Pipeline fails at Quality Gate | Check if SonarQube webhook URL is correct (must end with `/sonarqube-webhook/`) |

---

## 👨‍💻 Author

**YOUR_NAME**  
B.Tech Computer Science Engineering  
DevOps | Cloud | CI/CD | AWS | Docker | Jenkins
