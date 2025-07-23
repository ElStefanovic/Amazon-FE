Amazon-FE



Deploy a Netflix Clone on AWS Cloud using Jenkins, SonarQube, Docker, and Trivy for code quality and vulnerability scanning.
Architecture

flowchart TD
    A[Developer] -->|Push Code| B[GitHub Repository]
    B -->|Webhook| C[Jenkins Pipeline]
    C --> D[SonarQube - Code Analysis]
    C --> E[OWASP Dependency Check]
    C --> F[Trivy Scan]
    C --> G[Docker Build & Push to Docker Hub]
    G --> H[AWS EC2 Deployment - Container Run]

Project Overview

This project demonstrates:

    Automated CI/CD pipeline with Jenkins

    Static code analysis and quality gates using SonarQube

    Containerization with Docker and publishing to Docker Hub

    Security scanning using Trivy and OWASP Dependency Check

    Deployment of the application on AWS EC2

Prerequisites

    AWS account with permissions to create VPC, Subnet, Security Groups, EC2

    Installed CLI tools: Git, SSH, Docker, Trivy

    Jenkins server with required plugins (SonarQube, Docker, NodeJS, OWASP)

    SonarQube instance running (via Docker or standalone)

Phase 1: Initial Setup and AWS Environment Deployment
Step 1: Networking and Compute Environment

VPC resource map

    Create a VPC: Isolate resources in AWS.

    Create a Public Subnet: Allow external access (web servers).

    Attach Internet Gateway: Enable internet connectivity.

    Configure Security Groups: Allow HTTP/HTTPS, SSH, TCP 9000/8080/3000.

    Launch EC2 Instances: Deploy Ubuntu 22.04 server and connect via SSH.

Step 2: Update and Upgrade Ubuntu

sudo apt-get update -y && sudo apt-get upgrade -y

Step 3: Install Prerequisite Packages

sudo apt-get install apt-transport-https gnupg lsb-release -y

Phase 2: Security Setup
Step 4: Install Trivy

sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
trivy --version

Step 5: Install and Run SonarQube
Install Docker and Configure

sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock

Add Jenkins to Docker Group

sudo su
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

Run SonarQube via Docker

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

Access at http://<EC2-PUBLIC-IP>:9000 (default credentials: admin/admin).
Phase 3: CI/CD Setup
Step 6: Install Jenkins and Java

Follow Jenkins Docs:

sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version

Install Jenkins

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

Access Jenkins at http://<EC2-PUBLIC-IP>:8080.
Install Jenkins Plugins

Install:

    Eclipse Temurin Installer

    SonarQube Scanner

    NodeJS Plugin

    OWASP Dependency-Check

    Docker, Docker Pipeline, Docker Build Step

Configure Tools in Jenkins

Go to Manage Jenkins → Tools and add:

    JDK: jdk21

    NodeJS: nodejs

    Sonar Scanner: sonar-scanner

    Dependency Check: DP-Check

    Docker: docker

Step 7: Add Credentials
Docker Hub

    Add username/password → ID: docker

SonarQube

    Generate token in SonarQube → Add as Secret Text → ID: sonar-scanner

Configure SonarQube in Jenkins

    Go to Manage Jenkins → Configure System

    Add:

        Name: sonar-server

        Server URL: http://<sonarqube-server>:9000

        Auth Token: sonar-scanner

Create Webhook in SonarQube

    Go to Administration → Configuration → Webhooks

    Add:

        Name: Jenkins

        URL: http://<jenkins-server>/sonarqube-webhook/

Jenkins Pipeline

pipeline {
    agent any
    tools {
        jdk 'jdk21'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps { git branch: 'main', url: 'https://github.com/ElStefanovic/Amazon-FE.git' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Amazon \
                    -Dsonar.projectKey=Amazon '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-scanner'
                }
            }
        }
        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }
        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t amazon .'
                        sh 'docker tag amazon elstefanovic/amazon:latest'
                        sh 'docker push elstefanovic/amazon:latest'
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps { sh 'trivy image --timeout 10m elstefanovic/amazon:latest > trivyimage.txt' }
        }
        stage('Deploy to Container') {
            steps { sh 'docker run -d --name amazon -p 3000:3000 elstefanovic/amazon:latest' }
        }
    }
}

Cleanup

Terminate AWS EC2 instances and delete resources when not in use.
