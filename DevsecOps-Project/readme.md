# DevSecOps Project: Jenkins CI/CD Pipeline with Docker & Kubernetes Integration

This project demonstrates a complete DevSecOps pipeline. It automates the process of provisioning infrastructure, building and securing an application, and deploying it to a Kubernetes cluster. The pipeline integrates tools like Terraform, Jenkins, SonarQube, Docker, Trivy, and ArgoCD.

## Architecture Overview

The pipeline follows these key steps:
1.  **Infrastructure Provisioning:** Terraform creates an EC2 instance with Jenkins, Docker, SonarQube, and Trivy.
2.  **CI/CD Pipeline (Jenkins):**
    *   Code is checked out from Git.
    *   Static code analysis is performed by **SonarQube** with a quality gate check.
    *   Dependencies are installed.
    *   A filesystem scan is run using **Trivy**.
    *   A Docker image is built and pushed to Docker Hub.
    *   The Docker image is scanned for vulnerabilities using **Trivy**.
    *   The CD pipeline in Jenkins is triggered.
3.  **Continuous Deployment (ArgoCD):**
    *   The triggered CD job sends a webhook to **ArgoCD**.
    *   ArgoCD, which monitors the Git repository (or the Jenkins trigger), syncs the application with the **EKS (Elastic Kubernetes Service)** cluster.
4.  **Monitoring:** The Kubernetes cluster is monitored using a **Prometheus** and **Grafana** stack, installed via **Helm**.

## Prerequisites

Before you begin, ensure you have the following:
*   An **AWS Account** with appropriate permissions to create EC2 instances, IAM roles, and EKS clusters.
*   **AWS CLI** configured on your local machine with your access and secret keys (`aws configure`).
*   **Terraform** installed on your local machine.
*   **Git** installed on your local machine.
*   A **GitHub Account** and a **Docker Hub Account**.

## Step 1: Provision the Jenkins Server with Terraform

We will use Terraform to launch an EC2 instance (t2.large) and install all necessary tools (Jenkins, Docker, SonarQube, Trivy) via a user-data script.

1.  **Create the Terraform Configuration Files:**
    *   **`main.tf`** - Defines the AWS instance and security group.
        ```hcl
        resource "aws_instance" "web" {
          ami                    = "ami-0075013580f6322a1"      # Change for your region
          instance_type          = "t2.large"
          key_name               = "eks-pair"                   # Change to your key pair
          vpc_security_group_ids = [aws_security_group.Jenkins-VM-SG.id]
          user_data              = templatefile("./install.sh", {})

          tags = {
            Name = "Jenkins-SonarQube"
          }

          root_block_device {
            volume_size = 40
          }
        }

        resource "aws_security_group" "Jenkins-VM-SG" {
          name        = "Jenkins-VM-SG"
          description = "Allow TLS inbound traffic"

          ingress = [
            for port in [22, 80, 443, 8080, 9000, 3000] : {
              description      = "inbound rules"
              from_port        = port
              to_port          = port
              protocol         = "tcp"
              cidr_blocks      = ["0.0.0.0/0"]
              ipv6_cidr_blocks = []
              prefix_list_ids  = []
              security_groups  = []
              self             = false
            }
          ]

          egress {
            from_port   = 0
            to_port     = 0
            protocol    = "-1"
            cidr_blocks = ["0.0.0.0/0"]
          }

          tags = {
            Name = "Jenkins-VM-SG"
          }
        }
        ```
    *   **`provider.tf`** - Configures the AWS provider.
        ```hcl
        terraform {
          required_providers {
            aws = {
              source  = "hashicorp/aws"
              version = "~> 5.0"
            }
          }
        }

        provider "aws" {
          region = "us-west-2"     # Change to your desired region
        }
        ```
    *   **`install.sh`** - The user-data script to install tools on the EC2 instance.
        ```bash
        #!/bin/bash
        sudo apt update -y

        # Install Java 17 (required for Jenkins)
        wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
        echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
        sudo apt update -y
        sudo apt install temurin-17-jdk -y

        # Install Jenkins
        curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
        echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
        sudo apt-get update -y
        sudo apt-get install jenkins -y
        sudo systemctl start jenkins
        sudo systemctl enable jenkins

        # Install Docker and run SonarQube
        sudo apt-get update
        sudo apt-get install docker.io -y
        sudo usermod -aG docker ubuntu
        sudo usermod -aG docker jenkins
        sudo chmod 777 /var/run/docker.sock
        docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

        # Install Trivy
        sudo apt-get install wget apt-transport-https gnupg lsb-release -y
        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
        echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
        sudo apt-get update
        sudo apt-get install trivy -y
        ```

2.  **Deploy the Infrastructure:**
    ```bash
    terraform init
    terraform plan
    terraform apply --auto-approve
    ```
    After the deployment, note the public IP of the newly created EC2 instance.

| EC2 Instance Created |
| :---: |
| ![EC2 Instance](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_AePoGw381TYDrMNpcJ_YXg.png) |

| Security Groups Configuration |
| :---: |
| ![Security Groups](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1__KwX5diUOcBB3ySRiCpFPg.png) |

| Jenkins, Docker, Trivy and SonarQube Running |
| :---: |
| ![Services Running](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_lqib_nOj8kVb3AGfYc7VHw.png) |

## Step 2: Configure Jenkins

1.  Access Jenkins by navigating to `http://<your-ec2-public-ip>:8080`. Get the initial admin password from the instance:
    ```bash
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```

| Jenkins Initial Setup |
| :---: |
| ![Jenkins Initial](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_QMfPDvlmFW_Z_H_4iQn6Rg.png) |

| Jenkins Dashboard |
| :---: |
| ![Jenkins Dashboard](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_1vTb5ats6bAt_MiIYZYfzg.png) |

2.  **Install Additional Plugins:**
    *   Navigate to **Manage Jenkins > Plugins > Available Plugins**.
    *   Search for and install: `Docker`, `Docker Pipeline`, `NodeJS`, `SonarQube Scanner`, `Email Extension`.

| Jenkins Plugins Installation |
| :---: |
| ![Jenkins Plugins](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_1t901P4npnt8W0ExJ5HuKg.png) |

| Docker Plugins Installation |
| :---: |
| ![Docker Plugins](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_rpNvIUScq00JFfbUaKjuFg.png) |

3.  **Configure Global Tools:**
    *   Go to **Manage Jenkins > Tools**.

| NodeJS Installation |
| :---: |
| ![NodeJS Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_rCLg0jDvHJzwezfpXCA8cg.png) |

| JDK 17 Configuration |
| :---: |
| ![JDK 17](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_1Ml1mSl_i52eIobpZkp0hA.png) |

| Docker Installation |
| :---: |
| ![Docker Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_HVL3nsHFYNb3yj3Q4GwPpA.png) |

| SonarQube Scanner Installation |
| :---: |
| ![SonarQube Scanner](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_r5PG6qg05oDWs1p-ZtMWkw.png) |

## Step 3: Configure SonarQube and Integrate with Jenkins

1.  Access SonarQube at `http://<your-ec2-public-ip>:9000`. Login with the default credentials (`admin` / `admin`) and change the password.

| SonarQube Initial Screen |
| :---: |
| ![SonarQube Initial](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_T3aIsNAiucCRiXOR2spb3w.png) |

| SonarQube Dashboard |
| :---: |
| ![SonarQube Dashboard](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_Is9LIk6a-GknzN1O7eIn_g.png) |

2.  **Generate a Token in SonarQube:**
    *   Go to **Administration > Security > Users**.
    *   Click on the **Tokens** icon for the `admin` user.
    *   Generate a token (e.g., `jenkins-sonar-token`). Copy it.

| SonarQube Token Generation |
| :---: |
| ![Token Generation](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_dK2rkiXJTLlXzWSMh-6h0A.png) |

| Token Display |
| :---: |
| ![Token](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_vdvVbejzy0zF3NW1SiZTZw.png) |

| Tokens Generated |
| :---: |
| ![Tokens Generated](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_OLid5I4hslyJdxnorDRRtw.png) |

3.  **Add the Token to Jenkins Credentials:**
    *   In Jenkins, go to **Manage Jenkins > Credentials > System > Global credentials (unrestricted) > Add Credentials**.
    *   **Kind:** `Secret text`
    *   **Secret:** Paste the token you copied from SonarQube.
    *   **ID:** `SonarQube-Token` (or a memorable name).

| Jenkins Credentials Setup |
| :---: |
| ![Jenkins Credentials](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1__daezp9Tcz4m_QwBZ4byiA.png) |

4.  **Configure SonarQube Server in Jenkins:**
    *   Go to **Manage Jenkins > System**.
    *   Find the **SonarQube servers** section.
    *   Check **"Environment variables"**.
    *   Click **Add SonarQube**.
    *   **Name:** `SonarQube-Server`
    *   **Server URL:** `http://<private-ip-of-ec2-instance>:9000` (Use the private IP for internal communication).
    *   **Server authentication token:** Select the credential you just added (`SonarQube-Token`).

| Jenkins SonarQube Server Configuration |
| :---: |
| ![SonarQube Config](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_nD0FrI6K-mQCi5SpZzXJsw.png) |

5.  **Create a Quality Gate in SonarQube:**
    *   Go to **Quality Gates** and click **Create**.

| Quality Gates Creation |
| :---: |
| ![Quality Gates](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_hG5DOJN4j0EW3AX5xICfXA.png) |

| Quality Gates Created |
| :---: |
| ![Quality Gates Created](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_m7cJjEQqf6ZweBdpdmBhlA.png) |

6.  **Create a Webhook in SonarQube:**
    *   Go to **Administration > Configuration > Webhooks**.
    *   Click **Create**.
    *   **Name:** `Jenkins-Webhook`
    *   **URL:** `http://<private-ip-of-ec2-instance>:8080/sonarqube-webhook/`

| Webhook Configuration |
| :---: |
| ![Webhook](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_puHra26s4Y8Nd2lh_ZZ21g.png) |

| Webhook URL Setup |
| :---: |
| ![Webhook URL](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_S4tX8iGnCYySm1_FdRut8Q.png) |

| Webhook Established |
| :---: |
| ![Webhook Established](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_Hseku5NIs1m3_HBiT01-0A.png) |

## Step 4: Create the CI/CD Pipeline in Jenkins

1.  **Add GitHub and Docker Hub Credentials to Jenkins:**
    *   Go to **Manage Jenkins > Credentials > System > Global credentials**.
    *   **For GitHub:** Add a **Username with password** credential (ID: `github`). Use your GitHub username and a personal access token (classic) with `repo` scope as the password.
    *   **For Docker Hub:** Add a **Username with password** credential (ID: `dockerhub`). Use your Docker Hub username and password.

| GitHub Credentials Setup |
| :---: |
| ![GitHub Credentials](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_Qlj6kibFaEK9BWB-cSu8fw.png) |

| Credentials Established |
| :---: |
| ![Credentials Established](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_pxfPhWf7VuaGjDtLEPuHmw.png) |

2.  **Create a SonarQube Project:**
    *   In SonarQube, go to **Administration > Projects > Create Project**.

| Create Project in SonarQube |
| :---: |
| ![Create Project](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_LpNcBuAYBYCQDX9tXimR1Q.png) |

| Create Project Step |
| :---: |
| ![Create Project Step](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_N3hAGlK1N85pvWmWA3x-Hw.png) |

| Continue Project Creation |
| :---: |
| ![Continue](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_C1y651jdkI6wbGb93YuKvQ.png) |

| Linux Platform Selection |
| :---: |
| ![Linux Platform](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_b0XIa8XirZLmJsdvLtDiQw.png) |

3.  **Create a New Pipeline Job in Jenkins:**
    *   Click **New Item**.
    *   Enter a name (e.g., `Reddit-Clone-CI`), select **Pipeline**, and click OK.
    *   In the job configuration, under the **Pipeline** section:
        *   **Definition:** `Pipeline script from SCM`
        *   **SCM:** `Git`
        *   **Repository URL:** `https://github.com/GeraldAkenji/reddit-clone.git` (or your forked repo).
        *   **Script Path:** `Jenkinsfile`

| Pipeline Configuration |
| :---: |
| ![Pipeline Config](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_CjB4Iq7kelJoJpNDCWWa4Q.png) |

4.  **Define the `Jenkinsfile` in your Repository:**
    Create a file named `Jenkinsfile` in the root of your GitHub repository with the following content. **Make sure to replace the placeholder values** like your Docker Hub username and the correct URL for your CD job.

    ```groovy
    pipeline {
        agent any
        tools {
            jdk 'jdk17'
            nodejs 'node16'
        }
        environment {
            SCANNER_HOME = tool 'sonar-scanner'
            APP_NAME = "reddit-clone-pipeline"
            RELEASE = "1.0.0"
            DOCKER_USER = "your-dockerhub-username"   // CHANGE THIS
            DOCKER_PASS = 'dockerhub'                 // ID of Docker Hub credentials in Jenkins
            IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
            IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
            JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN") // API token for triggering CD job
        }
        stages {
            stage('Clean Workspace') {
                steps {
                    cleanWs()
                }
            }
            stage('Checkout from Git') {
                steps {
                    git branch: 'main', url: 'https://github.com/GeraldAkenji/reddit-clone.git'
                }
            }
            stage("SonarQube Analysis") {
                steps {
                    withSonarQubeEnv('SonarQube-Server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Reddit-Clone-Cl \
                        -Dsonar.projectKey=Reddit-Clone-Cl'''
                    }
                }
            }
            stage("Quality Gate") {
                steps {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-Token'
                    }
                }
            }
            stage('Install Dependencies') {
                steps {
                    sh "npm install"
                }
            }
            stage('TRIVY FS SCAN') {
                steps {
                    sh "trivy fs . > trivyfs.txt"
                }
            }
            stage("Build & Push Docker Image") {
                steps {
                    script {
                        docker.withRegistry('', DOCKER_PASS) {
                            docker_image = docker.build "${IMAGE_NAME}"
                        }
                        docker.withRegistry('', DOCKER_PASS) {
                            docker_image.push("${IMAGE_TAG}")
                            docker_image.push('latest')
                        }
                    }
                }
            }
            stage("Trivy Image Scan") {
                steps {
                    script {
                        sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${DOCKER_USER}/reddit-clone-pipeline:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt'
                    }
                }
            }
            stage('Cleanup Artifacts') {
                steps {
                    script {
                        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker rmi ${IMAGE_NAME}:latest"
                    }
                }
            }
            stage("Trigger CD Pipeline") {
                steps {
                    script {
                        sh "curl -v -k --user your-jenkins-username:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'http://<jenkins-server-dns-or-ip>:8080/job/Reddit-Clone-CD/buildWithParameters?token=gitops-token'" // CHANGE URL AND USERNAME
                    }
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
                    to: 'your-email@gmail.com', // CHANGE EMAIL
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            }
        }
    }
    ```

| Jenkins CI Pipeline Execution |
| :---: |
| ![Jenkins Pipeline](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1__KFDS37tvYlyjV6sE5uESg.png) |

| SonarQube Quality Gates Results |
| :---: |
| ![Quality Gates Results](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_5dYvIGECHXqbhyeluA1Xvw.png) |

## Step 5: Set Up Email Notifications in Jenkins

1.  **Configure Email Server:**
    *   Go to **Manage Jenkins > System**.
    *   Find the **E-mail Notification** section.
    *   **SMTP server:** `smtp.gmail.com` (if using Gmail).
    *   Click **Advanced**.
    *   Check **"Use SSL"**.
    *   **Port:** `465`.
    *   Provide a **Username** (your Gmail address) and **Password** (your Gmail app password - you must generate this in your Google account settings, as 2FA is required).
    *   Check **"Test configuration by sending test e-mail"** and enter a recipient email to verify.

| Email Notification Setup |
| :---: |
| ![Email Setup](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_apNjBELo1KsOFZ2fEQWu2w.png) |

| SMTP Port Configuration |
| :---: |
| ![SMTP Config](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_gdYb7nteIBt3FRmrGJfvcw.png) |

## Step 6: Create an Amazon EKS Cluster

These commands are run on your **Jenkins EC2 instance**.

1.  **Install `kubectl`:**
    ```bash
    sudo apt update
    sudo apt install curl -y
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    kubectl version --client
    ```

| kubectl Installation |
| :---: |
| ![kubectl Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_tpFD1ioCaYdcAP0jgvMbIw.png) |

2.  **Install AWS CLI:**
    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    sudo apt install unzip -y
    unzip awscliv2.zip
    sudo ./aws/install
    aws --version
    ```

| AWS CLI Installation |
| :---: |
| ![AWS CLI Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_EFWldd1jlxfFe4Vo1plfXQ.png) |

3.  **Install `eksctl`:**
    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    eksctl version
    ```

| eksctl Installation |
| :---: |
| ![eksctl Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_soqLddXtdtUGvGHcTgXwhg.png) |

4.  **Attach IAM Role to EC2:**
    *   In the AWS Console, create an IAM role for EC2 with the `AdministratorAccess` policy (for simplicity in this demo).
    *   Attach this role to your Jenkins EC2 instance (Actions > Security > Modify IAM role).

| IAM Role Creation |
| :---: |
| ![IAM Role](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_MaJ61XfplFYvaFqQCiFxvg.png) |

| IAM Role Created |
| :---: |
| ![IAM Role Created](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_SduuFXvUM6r4uJaKNzBokA.png) |

| Modify IAM Role Action |
| :---: |
| ![Modify IAM Role Action](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_5KfFHdSFkrliwX_JDTvnzg.png) |

| Modify IAM Role |
| :---: |
| ![Modify IAM Role](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_H5GO4mNLn4NX9IwKEZvhmQ.png) |

5.  **Create the EKS Cluster:**
    ```bash
    eksctl create cluster --name virtualtechbox-cluster \
    --region us-west-2 \
    --node-type t2.small \
    --nodes 3
    ```
    This process will take 10-15 minutes.

| EKS Cluster Creation |
| :---: |
| ![EKS Cluster](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_2W7y2jiwNadNXwlGLPB31A.png) |

| Cluster Creation with Nodes |
| :---: |
| ![Cluster Nodes](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_PDjSxxtw0EboMKXyicAbHw.png) |

6.  **Verify the Cluster:**
    ```bash
    kubectl get nodes
    ```

| Get Nodes Command |
| :---: |
| ![Get Nodes](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_rfv7pW9KPZs7JRvdJ2aFIA.png) |

## Step 7: Set Up Monitoring with Prometheus and Grafana

1.  **Install Helm:**
    ```bash
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    helm version
    ```

2.  **Add Helm Repositories and Install Prometheus Stack:**
    ```bash
    helm repo add stable https://charts.helm.sh/stable
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    kubectl create namespace prometheus

    helm install stable prometheus-community/kube-prometheus-stack -n prometheus
    ```

| Helm Install |
| :---: |
| ![Helm Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_T5bMhRfJayVUOw0icI5QNQ.png) |

3.  **Verify Installation:**
    ```bash
    kubectl get pods -n prometheus
    kubectl get svc -n prometheus
    ```

| Services Check |
| :---: |
| ![Services Check](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_KkuHLNM2-vNO7OsaFbNVRg.png) |

4.  **Expose Prometheus and Grafana:**
    *   **Prometheus:** Change its service to `LoadBalancer`.
        ```bash
        kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
        # Change type: ClusterIP to type: LoadBalancer
        # Also change the 'port' and 'targetPort' to 9090 if they are different.
        ```

| Prometheus Services |
| :---: |
| ![Prometheus Services](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_Z9eC5UzrmdrliA9auXEZtQ.png) |

| Prometheus DNS |
| :---: |
| ![Prometheus DNS](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_KroEVTGXRbkpW6Y9fq3qEQ.png) |

| Prometheus Targets |
| :---: |
| ![Prometheus Targets](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_bUgMyOeK4GiXy5h7inO_Kg.png) |

    *   **Grafana:** Change its service to `LoadBalancer`.
        ```bash
        kubectl edit svc stable-grafana -n prometheus
        # Change type: ClusterIP to type: LoadBalancer
        ```

| Grafana Edit |
| :---: |
| ![Grafana Edit](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_SKK8YnjpP5C7Bu6HijH5FQ.png) |

| Grafana Services |
| :---: |
| ![Grafana Services](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_QehvPdpfUn4L8jVxhx9-Gw.png) |

5.  **Get the Load Balancer URLs:**
    ```bash
    kubectl get svc -n prometheus
    ```
    *   Access Prometheus using the `stable-kube-prometheus-sta-prometheus` LB URL on port `9090`.
    *   Access Grafana using the `stable-grafana` LB URL.

| Grafana Login |
| :---: |
| ![Grafana Login](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1__PFgsJ6hBlM6tkXaZSAS6g.png) |

6.  **Get Grafana Login Password:**
    ```bash
    kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```
    The username is `admin`.

| Grafana Dashboard |
| :---: |
| ![Grafana Dashboard](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_r0oEQN7FnI2d6Zuhe6VNzA.png) |

| Grafana Dashboard |
| :---: |
| ![Grafana Dashboard](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_hvXhzVDmBZNGAUHVQgT0Wg.png) |

**Import the dashboard for the kubernetes cluster.**

| Import Dashboard |
| :---: |
| ![Import Dashboard](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_mwozTHaRwpZpOPz9dRmF7w.png) |

**v) Import dashboard — 15760 — Load — Select Prometheus & Click Import.**

| Dashboard 15760 |
| :---: |
| ![Dashboard 15760](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_CzL2FVFh-9rAsX3YXY5sfg.png) |

**Imported Dashboard**

| Imported Dashboard |
| :---: |
| ![Imported Dashboard](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_hB84JpTVgR8r_BRUkAUGVQ.png) |

**CPU Usage**

| CPU Usage |
| :---: |
| ![CPU Usage](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_muSS0QK_cU9w_ksKl64aVw.png) |

**Memory Utilization**

| Memory Utilization |
| :---: |
| ![Memory Utilization](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_h27CLySVb1uNus0vzhURqQ.png) |

**iv) Import more dashboard — 12740 — Load — Select Prometheus & Click Import.**

| Dashboard 12740 |
| :---: |
| ![Dashboard 12740](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_LnjRmIDQKQzAMbhORzBlCA.png) |

| Dashboard 12740 Details |
| :---: |
| ![Dashboard 12740 Details](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_nKYELYba1HAdwQJ1vjEJqg.png) |

## Step 8: ArgoCD Installation on Kubernetes Cluster and Add EKS Cluster to ArgoCD

ArgoCD is an open source platform that streamlines the deployment and management of applications on Kubernetes clusters.

**1) First, create a namespace**
```bash
kubectl create namespace argocd
```

**2) Next, let's apply the yaml configuration files for ArgoCD**
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

| ArgoCD Installation |
| :---: |
| ![ArgoCD Install](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_rMAnZV6d9BMXsDkOevocuQ.png) |

**3) Now we can view the pods created in the ArgoCD namespace.**
```bash
kubectl get pods -n argocd
```

**4) To interact with the API Server we need to deploy the CLI:**
```bash
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

**5) Expose argocd-server**

| ArgoCD Expose |
| :---: |
| ![ArgoCD Expose](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_wuLZ8GySGM7rsNXFtrj_Jg.png) |

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

**6) Wait about 2 minutes for the LoadBalancer creation**
```bash
kubectl get svc -n argocd
```

| ArgoCD Services |
| :---: |
| ![ArgoCD Services](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_ZmUa4kgmVXrnEV6GnVpvqg.png) |

**Copy the DNS name and open in a new browser**

| ArgoCD Login Screen |
| :---: |
| ![ArgoCD Login](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_0TJybz3ss-Uq-l99FRXjzg.png) |

**7) Get password, decode it, and login to ArgoCD on Browser. Go to user info and change the password**
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
echo WXVpLUg2LWxoWjRkSHFmSA== | base64 --decode
```

| Password Decode |
| :---: |
| ![Password Decode](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_gZgfNUjrNq2ek-GlQUsKcA.png) |

| ArgoCD Settings |
| :---: |
| ![ArgoCD Settings](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_JWzL5vzKuRcah8UiL_x_Ig.png) |

**8) Login to ArgoCD from CLI**
```bash
argocd login a2255bb2bb33f438d9addf8840d294c5-785887595.us-west-2.elb.amazonaws.com --username admin
```

| ArgoCD CLI Login |
| :---: |
| ![CLI Login](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_qYHXhutQM3h4MGWyosxyZw.png) |

**9) Check available clusters in ArgoCD**
```bash
argocd cluster list
```

| Cluster List |
| :---: |
| ![Cluster List](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_Hvi_1QPGem73oE2Lh5rhMA.png) |

**10) Below command will show the EKS cluster details**
```bash
kubectl config get-contexts
```

**11) Add above EKS cluster to ArgoCD with below command**
```bash
argocd cluster add i-08b9d0ff0409f48e7@virtualtechbox-cluster.us-west-2.eksctl.io --name virtualtechbox-eks-cluster
```

| Add Cluster |
| :---: |
| ![Add Cluster](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_cLybTtXvP-O5wdX8w3j6lA.png) |

**12) Now if you give command `argocd cluster list` you will get both the clusters EKS & ArgoCD (in-cluster). This can be verified at ArgoCD Dashboard.**

## Step 9: Configure ArgoCD to Deploy Pods on EKS Cluster and Automate ArgoCD Deployment using GitHub Repository

**Settings**

| ArgoCD Settings |
| :---: |
| ![Settings](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_PhZXUVGZTOCfYQ32MRe62Q.png) |

**Enter credentials**

| Enter Credentials |
| :---: |
| ![Enter Credentials](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_JvI2FHrh0UDNEQgnsa01vA.png) |

**Successfully connected**

| Successfully Connected |
| :---: |
| ![Connected](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1__okYja306hyQQ07mSLECEw.png) |

**Back to Jenkins and build again to automate the deployment.**

| Jenkins Build |
| :---: |
| ![Jenkins Build](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_Gxz2Gl6VDvlscvobhtUI6A.png) |

**Application deployed to ArgoCD**

| Application Deployed |
| :---: |
| ![Application Deployed](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_1F9PBqNrN7kjkrNr7CqCFQ.png) |

**Finally the application is up and running perfectly through ArgoCD**

| Application Running |
| :---: |
| ![Application Running](https://github.com/mohammad00irfan/DevOps-Projects/raw/main/DevsecOps-Project/images/1_UbB_K3ezbsi46ZDkSg1w0w.png) |

## Conclusion

In this DevSecOps project, we successfully implemented a robust CI/CD pipeline using Jenkins, integrated with Docker and Kubernetes to streamline application delivery. The project involved automating the entire build, test, and deployment processes, ensuring security was embedded at each stage. By leveraging Docker for containerization and Kubernetes for orchestration, we enhanced scalability and reliability, allowing the infrastructure to dynamically adapt to application demands.

The integration of security checks, such as vulnerability scans and automated compliance audits, directly into the pipeline ensured a secure software delivery lifecycle. The end result was a fully automated, secure, and efficient CI/CD pipeline, reducing deployment time, improving scalability, and increasing the overall security posture of the application infrastructure.


