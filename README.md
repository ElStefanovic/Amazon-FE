# Amazon-FE

Deploy Netflix Clone on Cloud using jenkins, SonarQube, Docker and Trivy for code quality and vulnerability scanning.

Phase 1: Initial Setup and AWS environment Deployment

 Step 1: Set up the Networking and Compute Environment
 ![VPC ressource map](https://github.com/user-attachments/assets/d81ca1a8-2019-4b19-a0de-c22888593b03)

    Create a VPC (Virtual Private Cloud):
    Define a custom VPC to isolate resources within your AWS account.

    Create a Public Subnet:
    Configure a subnet that allows public access to resources (e.g., web servers).

    Attach an Internet Gateway (IGW):
    Enable outbound and inbound internet connectivity for the public subnet.

    Configure Security Groups:
    Define security group rules to control inbound/outbound traffic (e.g., allow HTTP/HTTPS and SSH, TCP 9000, TCP 8080, TCP 3000).

    Launch EC2 Instances:
    Deploy EC2 instances (Ubuntu 22.04) within the public subnet and associate them with the appropriate security groups.

    Connect to the instance using SSH.

Step 2: Updating and Upgrading the Ubuntu server :

     sudo apt-get update -y && Sudo apt-get uprade -y

Step 3: Installing required packages to install docker : 

     sudo apt-get install apt-transport-https gnupg lsb-release -y
     
     This command installs tools to enable HTTPS repositories, manage GPG keys, and identify the Linux distribution version.

Phase 2: Security
        
  Step 4: Setting Up Trivy for vulnerability scanning and testing the installation :

      Add repository setting to /etc/apt/sources.list.d.
      
      sudo apt-get install wget gnupg
      wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
      echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
      sudo apt-get update
      sudo apt-get install trivy
      trivy --version

  Step 5: Installation of sonarqube : 
    
    Set up Docker on the EC2 instance:
    
    sudo apt-get install docker.io -y
    sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
    newgrp docker
    sudo chmod 777 /var/run/docker.sock

    Do no forget to add jenkins in the docker group :
    sudo su
    sudo usermod -aG docker jenkins
    sudo systemctl restart jenkins

    Run sonarQube in Docker : 
    
    docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
    
    To access: 
    publicIP:9000 (by default username & password is admin)



    Integrate SonarQube and Configure:
        Integrate SonarQube with your CI/CD pipeline.
        Configure SonarQube to analyze code for quality and security issues.

Phase 3: CI/CD Setup

Step 6: Intalling Jenkins an Java (following jenkins officials docs: https://www.jenkins.io/doc/book/installing/linux/)

      Installing Java and check the installation with the command below :

      sudo apt update
      sudo apt install fontconfig openjdk-21-jre
      java -version

      
      intalling Jenkins :

      sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
      https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
      echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
      https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
      /etc/apt/sources.list.d/jenkins.list > /dev/null
      sudo apt-get update
      sudo apt-get install jenkins -y

      Verify the jenkins installation : 

      sudo systemctl enable jenkins      
      sudo systemctl start jenkins
      sudo systemctl status jenkins
      
      Access Jenkins in a web browser using the public IP of your EC2 instance.

      publicIp:8080

      Install Necessary Plugins in Jenkins:

      Goto Manage Jenkins →Plugins → Available Plugins →

        Install below plugins :

        1 Eclipse Temurin Installer (Install without restart)
        2 SonarQube Scanner (Install without restart)
        3 NodeJs Plugin (Install Without restart)
        4 OWASP Dependency-Check
        5 Docker, Docker pipeline, Docker build step

      Install required tools in Jenkins:
        
        Goto Manage Jenkins →Tools →

        Installation of JDK (name: "jdk21", Version:"21.0.7+6", Install from adoptium.net ) 
        Installation of NodeJs ( name: "nodejs", version:"24.4.1", install from nodejs.org
        Installation of sonarQube ( name: "sonar-scanner", version:"7.2.0.5079",Install from Maven Central  )
        Installation of Dependency-Check ( name:"DP-Check", Version: "12.1.3",  Install from github.com)
        Installation of Docker ( name:"docker", - )

    Step 7: Adding credentials for sonarquebe and docker hub in jenkins :
    
      Add Docker Hub Credentials

        Go to Jenkins Dashboard → Manage Jenkins → Credentials → System → Global credentials (unrestricted).

        Click Add Credentials.
        Select Username with password.
    Enter:
        Username: Your Docker Hub username
        Password: Your Docker Hub password or personal access token
        ID: docker (important – match with credentialsId: 'docker' in your pipeline)
        Description: Docker Hub credentials (optional)
    Click Save.

    Add SonarQube Token Credentials

        Generate a token in SonarQube UI:
        Go to My Account → Security → Generate Tokens.
        In Jenkins: Manage Jenkins → Credentials → System → Global credentials (unrestricted).

    Click Add Credentials.
        Select Secret text.
        Enter:
        Secret: Paste the SonarQube token
        ID: sonar-scanner (use this ID in pipeline credentialsId: 'sonar-scanner')
        Description: SonarQube token (optional)
        Save.

    Configure SonarQube Server in Jenkins

        Go to Manage Jenkins → Configure System.
        Scroll to SonarQube servers.
        Click Add SonarQube:
        Name: Sonar-server
        Server URL: http://<your-sonarqube-server>:9000
        Server authentication token: Select the credential you added (sonar-scanner).
        Save.

    Create a Jenkins webhook :
    Configure the Webhook in SonarQube

    Log in to SonarQube as an administrator.
    Go to Administration → Configuration → Webhooks.
    Click Create.
    Enter:
        Name: Jenkins
        URL: http://<jenkins-server>/sonarqube-webhook/
        Description: Webhook for Jenkins Quality Gate
        Save.

    Configure Jenkins SonarQube Plugin:
        In Jenkins, go to Manage Jenkins → Configure System.
        Scroll to SonarQube servers:
        Add your SonarQube server URL and token.
        Enable the option Enable webhook for SonarQube if available.

Configure CI/CD Pipeline in Jenkins:

  pipeline{
    agent any
    tools{
        jdk 'jdk21'
        nodejs 'nodejs'
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
                git branch: 'main', url: 'https://github.com/ElStefanovic/Amazon-FE.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Amazon \
                    -Dsonar.projectKey=Amazon '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-scanner'
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
               script {
                         withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                         sh "docker build -t amazon ."
                         sh "docker tag amazon elstefanovic/amazon:latest "
                         sh "docker push elstefanovic/amazon:latest "
                         }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image --timeout 10m elstefanovic/amazon:latest > trivyimage.txt"
            }
        }
       stage('Deploy to container'){
            steps{
                sh 'docker run -d --name amazon -p 3000:3000 elstefanovic/amazon:latest'
            }
        }
    }
}        


Do not forget to replace elstefanovic with your actual DockerHub username where necessary in the pipeline.

Step 8: Cleanup

    Cleanup AWS EC2 Instances:
        Terminate AWS EC2 instances that are no longer needed.

