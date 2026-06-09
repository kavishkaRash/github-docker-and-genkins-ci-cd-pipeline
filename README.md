# CI/CD Pipeline with GitHub, Docker, Jenkins & AWS EC2

A fully automated CI/CD pipeline that builds, pushes, and deploys a Node.js application to AWS EC2 using Jenkins and Docker — triggered automatically on every GitHub push.

---

## Overview

This project demonstrates a real-world CI/CD pipeline. When a developer pushes code to GitHub, a webhook automatically triggers Jenkins. Jenkins then builds a Docker image, pushes it to Docker Hub, and deploys the latest container to an AWS EC2 instance via SSH — all without any manual intervention.

---

## How It Works

1. Developer pushes code to the GitHub repository
2. GitHub Webhook triggers Jenkins automatically via ngrok
3. Jenkins builds a Docker image tagged with the build number
4. Jenkins pushes the image to Docker Hub
5. Jenkins SSH's into the AWS EC2 instance
6. EC2 pulls the latest image, stops the old container, and starts the new one
7. The application is live at `http://13.60.57.12:3000`

---

## Project Structure

```
github-docker-and-jenkins-ci-cd-pipeline/
├── nodeapp/
│   ├── index.js          
│   ├── package.json      
│   └── ...
├── Dockerfile            
├── Jenkinsfile           
├── .gitignore
└── README.md
```

---

## Prerequisites

- Node.js 18+
- Docker Desktop
- Jenkins 2.555+ (with Java 17)
- AWS Account with EC2 access
- Docker Hub Account
- ngrok Account (for webhook tunnel)

---

## Setup Guide

### 1. Clone the Repository

```bash
git clone https://github.com/kavishkaRash/github-docker-and-genkins-ci-cd-pipeline.git
cd github-docker-and-genkins-ci-cd-pipeline
```

### 2. Install Jenkins on Mac

```bash
brew install openjdk@17
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

brew install jenkins-lts
brew services start jenkins-lts
```

Open Jenkins at `http://localhost:8080` and get the initial password:

```bash
cat ~/.jenkins/secrets/initialAdminPassword
```

### 3. Install Required Jenkins Plugins

- Pipeline
- Git
- SSH Agent
- GitHub Integration

### 4. Launch AWS EC2 Instance

- AMI: Ubuntu 22.04 LTS
- Instance Type: t2.micro (Free Tier)
- Security Group: Allow port 22 (SSH) and port 3000 (App)

### 5. Install Docker on EC2

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### 6. Setup ngrok

```bash
brew install ngrok
ngrok config add-authtoken <YOUR_AUTH_TOKEN>
ngrok http 8080
```

### 7. Add GitHub Webhook

Go to your GitHub repository Settings → Webhooks → Add webhook:

- Payload URL: `https://reaffirm-animator-retouch.ngrok-free.dev/github-webhook/`
- Content type: `application/json`
- Event: Just the push event

### 8. Configure Jenkins Pipeline

- Go to Jenkins Dashboard → New Item → Pipeline
- Under Triggers, check **GitHub hook trigger for GITScm polling**
- Paste the Jenkinsfile contents under Pipeline Script
- Save

---

## Jenkins Credentials

Add the following under Manage Jenkins → Credentials → Global:

| ID | Type | Description |
|----|------|-------------|
| `dockerHub-password` | Secret text | Docker Hub Access Token |
| `ec2-ssh-key` | SSH Username with private key | EC2 PEM file |

---

## Pipeline Stages

| Stage | Description |
|-------|-------------|
| SCM Checkout | Clones the latest code from GitHub |
| Build Docker Image | Builds a linux/amd64 Docker image |
| Login to Docker Hub | Authenticates with Docker Hub |
| Push Image | Pushes the image tagged with build number |
| Deploy to EC2 | SSH into EC2 and runs the new container |
| Post Actions | Docker logout and cleanup |

---

## Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        PATH = "/usr/local/bin:${PATH}"
        EC2_HOST = 'YOUR_EC2_PUBLIC_IP'
        EC2_USER = 'ubuntu'
        IMAGE_NAME = 'YOUR_DOCKERHUB_USERNAME/YOUR_IMAGE_NAME'
    }

    stages {
        stage('SCM Checkout') {
            steps {
                retry(3) {
                    git branch: 'main', url: 'YOUR_GITHUB_REPO_URL'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build --platform linux/amd64 -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerHub-password', variable: 'dockerHubPass')]) {
                    sh "docker login -u YOUR_DOCKERHUB_USERNAME -p ${dockerHubPass}"
                }
            }
        }

        stage('Push Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                            docker pull ${IMAGE_NAME}:${BUILD_NUMBER}
                            docker stop nodejs-app 2>/dev/null || true
                            docker rm nodejs-app 2>/dev/null || true
                            docker run -d --name nodejs-app -p 3000:3000 ${IMAGE_NAME}:${BUILD_NUMBER}
                        '
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}
```

---

## Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /usr/src/app
COPY nodeapp/* /
RUN npm install
EXPOSE 3000
CMD ["node", "index.js"]
```

---

## Live Application

After a successful pipeline run, the application is accessible at:

```
http://<EC2-PUBLIC-IP>:3000
```

---

## Tech Stack

- Node.js 18 / Express.js
- Docker / Docker Hub
- Jenkins
- AWS EC2 (Ubuntu)
- GitHub / GitHub Webhooks
- ngrok

---

## Author

D. Kavishka Rashen
- GitHub: [@kavishkaRash](https://github.com/kavishkaRash)
- Docker Hub: [rushferz](https://hub.docker.com/u/rushferz)

---

