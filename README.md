# Configuration Management With Helm

## Overview

This project focuses on introducing the fundamentals of Helm charts and their integration with Jenkins to design and implement a simplified CI/CD pipeline. The primary objective is to automate the deployment of a basic web application, enhancing hands-on experience and understanding of Helm charts and Jenkins in the DevOps domain.

## Goals and Objectives

1. **Introduce Helm Charts**: Understand the basics of Helm charts and their significance in Kubernetes-based deployments.
2. **CI/CD Pipeline**: Design a pipeline using Jenkins that leverages Helm for deployment automation.
3. **Hands-On Experience**: Gain practical knowledge by automating the deployment of a sample web application.

## Project Approach

The first step involves provisioning infrastructure for Jenkins using Terraform. To ensure consistency and automation, the Terraform configuration pre-installs essential resources such as Docker and Minikube on the server. Below are the Terraform configurations and scripts used in this phase:

---

## Terraform Configuration

### Main Configuration (`main.tf`)
```hcl
terraform {
  backend "s3" {
    bucket         = "helm-terraform"
    key            = "terraform/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "helm-terraform"
    encrypt        = true
  }
}

resource "tls_private_key" "helm" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "helm_keypair" {
  key_name   = "helmkey"
  public_key = tls_private_key.helm.public_key_openssh
}

resource "local_file" "tf_key" {
  content  = tls_private_key.helm.private_key_pem
  filename = "helmkey.pem"
}

resource "aws_eip" "helm_eip" {
  domain = "vpc"
}

resource "aws_security_group" "helm_sg" {
  name_prefix = "helm-sg"
  description = "Security group for helm instances"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "helm_instance" {
  ami                         = "ami-066a7fbea5161f451"
  instance_type               = "t2.medium"
  key_name                    = aws_key_pair.helm_keypair.key_name
  vpc_security_group_ids      = [aws_security_group.helm_sg.id]
  associate_public_ip_address = true
  user_data = file("${path.module}/userdata.sh")
  tags = {
    Name = "helm-instance2"
  }
}

resource "aws_eip_association" "helm_eip_association" {
  instance_id   = aws_instance.helm_instance.id
  allocation_id = aws_eip.helm_eip.id
}
```

### Outputs Configuration (`outputs.tf`)
```hcl
output "public_ip" {
  value = aws_instance.helm_instance.public_ip
}

output "instance_public_dns" {
  description = "Public DNS of the EC2 instance"
  value       = aws_instance.helm_instance.public_dns
}

output "ssh_connection" {
  description = "Example SSH command to connect to the instance"
  value       = "ssh -i \"${local_file.tf_key.filename}\" ec2-user@${aws_instance.helm_instance.public_dns}"
}
```

### Provider Configuration (`provider.tf`)
```hcl
provider "aws" {
  region = "us-west-2"
}
```

### User Data Script (`userdata.sh`)
```bash
#!/bin/bash

sudo bash -c 'exec > >(tee /var/log/userdata.log | logger -t user-data) 2>&1'
set -e

if [ -f /etc/instance-setup-complete ]; then
    echo "Setup already complete. Skipping."
    exit 0
fi

sudo touch /etc/instance-setup-complete
sudo yum update -y

# Install Java and Jenkins
if ! command -v java &> /dev/null; then
    sudo dnf install java-17-amazon-corretto -y
fi

if ! command -v jenkins &> /dev/null; then
    sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    sudo yum upgrade
    sudo yum install jenkins -y
    sudo systemctl enable jenkins
    sudo systemctl start jenkins
fi

# Install Docker
if ! command -v docker &> /dev/null; then
    sudo yum install -y docker
    sudo systemctl enable docker
    sudo systemctl start docker
    sudo usermod -aG docker ec2-user
fi

# Install kubectl and Minikube
if ! command -v kubectl &> /dev/null; then
    curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.27.4/bin/linux/amd64/kubectl
    sudo install kubectl /usr/local/bin/kubectl
fi

if ! command -v minikube &> /dev/null; then
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    minikube start
fi

echo "Setup complete!"
```

### Variables Configuration (`variables.tf`)
```hcl
variable "instance_state" {
  description = "State of the helm instance"
}
```

---

### Next Steps

1. **Test and Validate the Terraform Script:**
   - `terraform init`
     ![running terraform init](https://github.com/user-attachments/assets/cbb4bea4-e74f-4d0d-8b4e-64be50778ab3)
   - `terraform validate`
     ![image](https://github.com/user-attachments/assets/e4d8e549-aaa6-4f78-a910-658267f8aeb8)
   - `terraform plan`
     ![image](https://github.com/user-attachments/assets/6a1e784e-0907-4124-86a0-e2ca2ff2d509)
   - `terraform apply`
     ![image](https://github.com/user-attachments/assets/1fcaa0a0-73f4-4423-bb17-052b599015d0)

2. **Confirm Services Are Running:**
   - SSH into the server to check services.
   - To check Jenkins:
     - Run `systemctl status jenkins`
       ![image](https://github.com/user-attachments/assets/d610b308-e9c0-40bb-8b9b-b74b55bbb9a0)
   - To check Docker:
     - Run `systemctl status docker` and `docker ps` to see containers.

3. **Start Minikube:**
   - Run `minikube start`.
   - **Why Minikube?** Minikube is a lightweight Kubernetes implementation ideal for local development and testing. Other options for Kubernetes setups include:
     - Kubernetes on cloud platforms (e.g., Amazon EKS, Google GKE, Azure AKS)
     - Kind (Kubernetes in Docker)
     - K3s (lightweight Kubernetes for production environments)
   - Minikube was chosen for its simplicity and ease of use in this project.
     ![image](https://github.com/user-attachments/assets/b66bf3f4-e7c7-437e-b0de-ffb9ec250847)



2. Access the Jenkins instance to configure pipelines.
3. Begin Helm chart creation for the sample web application.
4. Integrate Helm into the Jenkins pipeline.





























# Configuration-Management-With-Helm
---
In this  project, I will embark on a journey to introduce the basics of Helm charts and their integration with Jenkins. As a DevOps Engineer, my goal is to design and implement a simplified CI/CD pipeline using Jenkins, with a primary focus on Helm charts. The objective is to automate the deployment of a basic web application, promoting my understanding and hands-on experience as I explore the field.
---
### Demonstration:

Step-by-step demonstration of the CI/CD pipeline with Helm Integration.

## Jenkins Server Setup

### Objective: Configure Jenkins server for a CI/CD pipeline automation.

**Steps:**

1. Install Jenkins on a dedicated server (with detailed explanations)

2. Set up necessary plugins (Git, Helm, etc.) with simple configurations.

3. Configure Jenkins with basic security measures.
---

# **Integrating Helm with Jenkins**

In this project, I will be using Amazon Linux. You can find the official documentation for installing Jenkins on AWS here: [Tutorial for Installing Jenkins on AWS](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/).

---

## **1. Jenkins Server Setup**

### **Step 1: Install Jenkins on a Dedicated Server**

#### **Prerequisites:**
1. An Amazon Linux server instance.
2. Java Development Kit (JDK) installed (Jenkins requires Java 11 or 17).

---

#### **Instructions:**

##### **1. Update the System Packages**
Update the system to ensure all packages are up-to-date:
```bash
sudo yum update -y
```

##### **2. Install Java**
Ensure Java is installed. If not, install OpenJDK:
```bash
sudo amazon-linux-extras enable java-openjdk11
sudo yum install java-11-openjdk -y
```
Verify the installation:
```bash
java -version
```

##### **3. Add the Jenkins Repository**
Add the official Jenkins repository:
```bash
sudo curl -fsSL https://pkg.jenkins.io/redhat-stable/jenkins.io.key | sudo tee /etc/pki/rpm-gpg/RPM-GPG-KEY-jenkins
sudo sh -c 'echo "[jenkins]
name=Jenkins-stable
baseurl=https://pkg.jenkins.io/redhat-stable
gpgcheck=1
gpgkey=https://pkg.jenkins.io/redhat-stable/jenkins.io.key
enabled=1" > /etc/yum.repos.d/jenkins.repo'
```

##### **4. Install Jenkins**
Update the package list and install Jenkins:
```bash
sudo yum install jenkins -y
```

##### **5. Start and Enable Jenkins**
Start the Jenkins service and ensure it starts on boot:
```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

##### **6. Open Firewall for Jenkins**
By default, Jenkins runs on port 8080. Allow it through the firewall:
```bash
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
sudo firewall-cmd --reload
```

##### **7. Access Jenkins**
1. Open your browser and navigate to `http://<server-ip>:8080`.
2. Obtain the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Enter the password on the Jenkins setup page to proceed with the initial configuration.

##### **8. Complete Initial Setup**
1. Choose "Install suggested plugins" on the plugin setup page.
2. Create an admin user and complete the setup wizard.

---

## **2. Installing Jenkins Plugins**

Ensure the following Jenkins plugins are installed:

- **Git Plugin**: For cloning repositories.
- **Pipeline Plugin**: For creating Jenkins pipelines.
- **Kubernetes CLI Plugin**: For running `kubectl` commands.
- **Helm Plugin**: For executing Helm commands (optional, as CLI commands can also be used).

Refer to Jenkins plugin documentation if needed.

---

## **3. Configuring Jenkins for Kubernetes and Helm**

### **Step 1: Set Up Kubernetes Credentials in Jenkins**

1. Navigate to **Manage Jenkins > Credentials > System > Global credentials**.
2. Add a new credential:
   - **Kind**: Kubernetes configuration (kubeconfig).
   - **Scope**: Global.
   - **ID**: `kubeconfig` (or any name you'll use in the pipeline).
   - **File**: Upload your kubeconfig file.

### **Step 2: Verify Helm Availability in Jenkins**

Ensure Helm is accessible in Jenkins pipelines:

```bash
helm version
```

If Helm is not installed, install it on the Jenkins server:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## **4. Creating the Jenkins Pipeline**

### **Pipeline Script Overview**

The Jenkins pipeline will:
1. Clone the Helm chart repository.
2. Run Helm commands to install or upgrade the release.

### **Sample Jenkinsfile**

```groovy
pipeline {
    agent any
    environment {
        KUBECONFIG_CREDENTIAL_ID = 'kubeconfig'
    }
    stages {
        stage('Clone Repository') {
            steps {
                git 'https://github.com/your-username/your-helm-chart-repo.git'
            }
        }
        stage('Helm Deployment') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CREDENTIAL_ID, variable: 'KUBECONFIG')]) {
                    sh '''
                    export KUBECONFIG=$KUBECONFIG
                    helm upgrade --install my-release ./my-web-app \
                      --set replicaCount=2,image.tag="1.23.1"
                    '''
                }
            }
        }
    }
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
```

---

## **5. Running the Pipeline**

1. Add the pipeline to Jenkins:
   - Create a new Jenkins job and select **Pipeline**.
   - Paste the pipeline script or link the **Jenkinsfile** from a Git repository.

2. Run the pipeline:
   - Trigger a build and monitor the console output.

---

## **6. Validating the Deployment**

- Verify the Kubernetes resources:
  ```bash
  kubectl get all
  ```
- Confirm the application is accessible:
  - If the service type is **NodePort**, use `<node-ip>:<node-port>` to access the application.

---

## **7. Automating Rollbacks**

In case of deployment failure, Helm's rollback feature can be included in the pipeline:

```groovy
stage('Rollback') {
    steps {
        script {
            try {
                // Your deployment steps
            } catch (Exception e) {
                sh 'helm rollback my-release <revision>'
                error('Rollback executed due to failure.')
            }
        }
    }
}
```

To view revisions:

```bash
helm history my-release
```

To roll back to a specific revision:

```bash
helm rollback my-release <revision>
```

---

## **8. Clean Up Resources**

When you're done, uninstall the Helm release to remove the deployment:

```bash
helm uninstall my-release
```

---
