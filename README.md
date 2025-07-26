# Deploy an Amazon Clone on AWS Cloud using Jenkins, SonarQube, Docker, and Trivy for code quality and vulnerability scanning.

<img width="1587" height="808" alt="image" src="https://github.com/user-attachments/assets/f698229a-8d8d-404a-b7ab-b8050d0f6b78" />

# Architecture

## flowchart TD

    A[Developer] -->|Push Code| B[GitHub Repository]
    B -->|Webhook| C[Jenkins Pipeline]
    C --> D[SonarQube - Code Analysis]
    C --> E[OWASP Dependency Check]
    C --> F[Trivy Scan]
    C --> G[Docker Build & Push to Docker Hub]
    G --> H[AWS EC2 Deployment - Container Run]

## Project Overview

### This project demonstrates:

+ Automated CI/CD pipeline with Jenkins

+ Static code analysis and quality gates using SonarQube  

+ Containerization with Docker and publishing to Docker Hub  

+ Security scanning using Trivy and OWASP Dependency Check  

+ Deployment of the application on AWS EC2

### Prerequisites

+ AWS account with permissions to create VPC, Subnet, Security Groups, EC2

+ Installed CLI tools: Git, SSH, Docker, Trivy

+ Jenkins server with required plugins (SonarQube, Docker, NodeJS, OWASP)

+ SonarQube instance running (via Docker or standalone)

## Phase 1: Initial Setup and AWS Environment Deployment

### Step 1: Networking and Compute Environment

VPC resource map
![VPC ressource map](https://github.com/user-attachments/assets/fb8ee1bc-af95-47d7-a56f-7d41dde2ecf4)

Create a VPC: Isolate resources in AWS.

* Create a Public Subnet: Allow external access (web servers).

* Attach Internet Gateway: Enable internet connectivity.

* Configure Security Groups: Allow HTTP/HTTPS, SSH, TCP 9000/8080/3000.

* Launch EC2 Instances: Deploy Ubuntu 22.04 server and connect via SSH.

### Step 2: Update and Upgrade Ubuntu

    sudo apt-get update -y && sudo apt-get upgrade -y


### Step 3: Install Prerequisite Packages

    sudo apt-get install apt-transport-https gnupg lsb-release -y


## Phase 2: Security Setup

### Step 1: Install Trivy

    sudo apt-get install wget gnupg
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
    echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install trivy
    trivy --version


### Step 2: Install and Run SonarQube

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

Access at http://EC2-PUBLIC-IP:9000 (default credentials: admin/admin).

<img width="834" height="370" alt="image" src="https://github.com/user-attachments/assets/c1e6471e-b46f-44eb-81e3-2cdf9e8c1aaa" />


## Phase 3: CI/CD Setup

### Step 1: Install Jenkins and Java

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

Access Jenkins at http://EC2-PUBLIC-IP:8080.

Install Jenkins Plugins and Configure Tools

Required Plugins

Install these plugins via:

Dashboard → Manage Jenkins → Plugins → Available plugins

| Plugin Name | Purpose |
| --- | --- |
| Eclipse Temurin Installer | Java JDK management |
|SonarQube Scanner	|SonarQube integration|
|NodeJS Plugin |	Node.js/npm tooling |
| OWASP Dependency-Check	| Vulnerability scanning |
| Docker Pipeline	| Docker build/push in pipelines |
| Docker Build Step	| Docker container operations |

Note: Restart Jenkins after plugin installation if prompted.

Configure Tools

<img width="1795" height="904" alt="image" src="https://github.com/user-attachments/assets/367c12fe-d5fb-47a2-90f5-34a3f7a7205c" />

Go to:
Manage Jenkins → Tools → Tool Installations
1. JDK (Java Development Kit)

    Name: jdk21

    Install from: Eclipse Temurin

    Version: 21.x

    Install automatically (checked)

2. NodeJS

    Name: nodejs

    Version: 20.x (LTS recommended)

    Global npm packages: (Leave empty)

3. SonarQube Scanner

    Name: sonar-scanner

    Install automatically (checked)

    Version: Latest stable

4. OWASP Dependency-Check

    Name: DP-Check

    Version: 8.4.0 (or latest)

    Installation directory: /var/lib/jenkins/tools/dependency-check

5. Docker

    Name: docker

    Install from:

    Unix socket path:
   
### Step 2: Configure Credentials and SonarQube Integration

<img width="1100" height="876" alt="image" src="https://github.com/user-attachments/assets/77e7a748-825f-40d2-bd52-ad19b056d750" />


1. Add Docker Hub Credentials

    In Jenkins, go to:
    Dashboard → Manage Jenkins → Credentials → System → Global credentials → Add Credentials

    Select:

     Kind: Username and password

     Scope: Global

     Username: Your Docker Hub username

     Password: Your Docker Hub password/access token

     ID: docker (must match your pipeline)

     Description: Optional

2. Add SonarQube Token

    Generate Token in SonarQube:

     Log in to SonarQube → User Icon (Top Right) → My Account → Security

     Generate a new token (e.g., jenkins-token) and copy it.

    Store in Jenkins:

     Go to Credentials → Add Credentials

     Select:

     Kind: Secret text

     Secret: Paste the SonarQube token

     ID: sonar-scanner (must match pipeline)

3. Configure SonarQube Server in Jenkins

    Go to:
    Manage Jenkins → Configure System

    Under SonarQube servers:

     Add SonarQube → Name: sonar-server

       Server URL: http://<SONARQUBE_SERVER_IP>:9000

     Server authentication token: Select sonar-scanner (from 2. )
   
   <img width="1198" height="794" alt="image" src="https://github.com/user-attachments/assets/55154492-1b4a-4a40-b5eb-9b032208d519" />


5. Create Webhook in SonarQube

    In SonarQube:
    Administration → Configuration → Webhooks → Create

    Add:

     Name: Jenkins

       URL: http://<JENKINS_SERVER_IP>/sonarqube-webhook/

   <img width="1392" height="360" alt="image" src="https://github.com/user-attachments/assets/0180a11a-0183-487b-81a4-eb5d7a2a72c4" />

After configuring the environment and intalling the different tools and plugins, we can now create the Jenkins pipeline to automate tasks like code checkout, code analysis, vulnerability scans, and deployment. Here's how to set it up:

Go to Jenkins Dashboard:

Click on New Item.
Name your project (e.g., "Amazon-Test") and select Pipeline as the project type.
Click OK.

Define Pipeline Syntax:

In the Pipeline section, select Pipeline script and add the following pipeline code:

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

Do not forget to replace elstefanovic with your actual DockerHub username where necessary in the pipeline.

### Pipeline Stages Overview

1. Clean Workspace

    + Removes previous build files to ensure a clean working environment

    + Prevents conflicts from residual artifacts

2. Checkout from Git

    + Clones source code from the specified repository

    Example: https://github.com/ElStefanovic/Amazon-FE.git (main branch)

3. SonarQube Analysis

    + Performs static code analysis using sonar-scanner

    + Evaluates code quality against predefined standards

    Requires:

    Project Key (Amazon)

    Project Name (Amazon)

4. Quality Gate

    + Validates SonarQube analysis results

    Fails pipeline if quality thresholds aren’t met

5. Install Dependencies

    + Runs npm install to set up Node.js dependencies

    + Ensures all required packages are available

6. OWASP FS Scan

    + Security scan using OWASP Dependency-Check

    + Identifies vulnerabilities in third-party dependencies

    + Generates report: dependency-check-report.xml

7. Trivy FS Scan

    + Filesystem security scan with Trivy

    + Detects misconfigurations/exposed secrets

    + Outputs results to trivyfs.txt

8. Docker Build & Push

    + Builds Docker image from Dockerfile

    + Tags image (e.g., youruser/amazon:latest)

    + Pushes to Docker Hub (requires credentials)

9. Trivy Image Scan

    + Security scans the built Docker image

    + Checks for CVEs in container layers

    + Logs results to trivyimage.txt

10. Deploy to Container

    + Launches container from the image

    + Maps host port 3000 → container port 3000

    Accessible at: http://<server-ip>:3000

### Post-Setup Instructions

  Trigger the Pipeline:

  In Jenkins Dashboard → Select your pipeline → Build Now

  Monitor Execution:

  View real-time logs via Console Output

  Check stage results in Pipeline Overview

  Access the Application:

  After successful deployment, open:
  http://<your-server-ip>:3000

### Key Benefits

✅ Automated Quality Checks – SonarQube + Quality Gate

✅ Security Scans – OWASP + Trivy (FS/Image)

✅ Consistent Deployment – Dockerized build/push

✅ End-to-End Visibility – Centralized Jenkins logs

Note: Replace <your-server-ip> with your actual server IP/DNS. Ensure Docker Hub credentials and SonarQube tokens are pre-configured in Jenkins.

For troubleshooting, verify:

+ Docker permissions (/var/run/docker.sock)

+ Network access to Docker Hub/SonarQube

+ Tool paths in Jenkins Global Tool Configuration


![deployement OK](https://github.com/user-attachments/assets/cd775773-181f-4f07-bd15-5b34ab974449)


## Phase 4: Cleanup

Terminate AWS EC2 instances and delete resources when not in use.
