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
---

4. **Access the Jenkins Instance to Configure Pipelines:**

##### **7. Access Jenkins**

1. Open your browser and navigate to `http://<server-ip>:8080`.

   ![image](https://github.com/user-attachments/assets/01ccb67e-58ab-470e-8a81-afc8d5bf0d44)

2. Obtain the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
   ![image](https://github.com/user-attachments/assets/38624c19-8d35-4793-817a-e4d9e049a181)

3. Enter the password on the Jenkins setup page to proceed with the initial configuration.

##### **8. Complete Initial Setup**

1. Choose "Install suggested plugins" on the plugin setup page.

   ![image](https://github.com/user-attachments/assets/5b95c3ba-3dbd-4c0e-8ed2-dbbe7b3d57c0)

2. Create an admin user and complete the setup wizard.

---

## **Installing Jenkins Plugins**

Ensure the following Jenkins plugins are installed:
- **Git Plugin**: For cloning repositories.
- **Pipeline Plugin**: For creating Jenkins pipelines.
- **Kubernetes Plugin**: For running `kubectl` commands.

To install plugins:
1. Click on "Manage Jenkins".
2. Select "Plugins" and then "Available Plugins".

Refer to Jenkins plugin documentation if needed.

---

## Installing Helm on the Server

1. For Linux:
   ```bash
   curl -L https://get.helm.sh/helm-v3.5.0-linux-amd64.tar.gz -o helm.tar.gz
   ```
   ![image](https://github.com/user-attachments/assets/3f424122-f566-4df8-ac8d-8347a521e7a8)

2. Extract the archive:
   ```bash
   tar -zxvf helm.tar.gz
   ```

3. Move the binary (use `sudo` if necessary):
   ```bash
   mv linux-amd64/helm /usr/local/bin/helm
   ```
   ![image](https://github.com/user-attachments/assets/da1f81a1-ac22-4c24-956a-b3f3e4826ba5)

4. Verify the installation:
   ```bash
   helm version
   ```

5. Clean up:
   ```bash
   rm helm.tar.gz && rm -r *-amd64
   ```

---

## Helm Chart Creation for the Sample Web Application

1. Create a directory for the Helm chart:
   ```bash
   mkdir helm-web-app
   cd helm-web-app
   ```

2. Create a new chart:
   ```bash
   helm create webapp
   ```
   ![image](https://github.com/user-attachments/assets/377d8d37-3a97-4ba4-bcbd-27cf0add0439)

3. Initialize Git (if not already done):
   ```bash
   git init
   git add .
   git commit -m "Initial Helm webapp chart"
   ```

4. Push to a remote repository:
   ```bash
   git remote add origin <REMOTE_REPOSITORY_URL>
   git push -u origin master
   ```
## Running a Simple Web Application with Helm

### Step 1: Modify `values.yaml`
Edit the `values.yaml` file to define the desired configuration:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: stable
  pullPolicy: IfNotPresent
```

- **Explanation**: Setting `replicaCount` to 2 ensures two instances (replicas) of the application run simultaneously for better availability and scalability.

Save the changes.

### Step 2: Customize `templates/deployment.yaml`
Open the `templates/deployment.yaml` file and make the following adjustments:

1. Remove or comment out the following line:

   ```yaml
   {{- toYaml .Values.resources | nindent 12 }}
   ```

   ![image](https://github.com/user-attachments/assets/86aff9b4-16b2-4466-ad06-685a063b6686)

2. Add a simple resource request and limit under `spec.template.spec.containers.resources` to manage Kubernetes resources efficiently:

   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "100m"
     limits:
       memory: "256Mi"
       cpu: "200m"
   ```

   - **Explanation**: These configurations help Kubernetes allocate resources efficiently, ensuring that the application runs optimally without overloading the nodes.

   ![image](https://github.com/user-attachments/assets/c46a9990-e686-4b05-98ad-18aacfd46270)

### Step 3: Push Changes to Git Repository

```bash
git add .
git commit -m "Customized Helm chart"
git push
```
### **Deploy the Application**

1. Use Helm to deploy the application:
   ```bash
   helm install my-webapp ./webapp
   ```

2. Check the deployment status:
   ```bash
   kubectl get deployments
   ```

3. Visit the application URL. To retrieve the application URL, execute the following commands:

   ```bash
   export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=webapp,app.kubernetes.io/instance=my-webapp" -o jsonpath="{.items[0].metadata.name}")

   export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")

   kubectl --namespace default port-forward $POD_NAME 8081:$CONTAINER_PORT
   ```

4. Access your application by visiting:
   ```
   HTTP://127.0.0.1:8081
   ```

   ![image](https://github.com/user-attachments/assets/006fa63d-b657-4a00-b085-632b3614fb6c)
 


## Integrating Helm into the Jenkins Pipeline

### **4. Create Jenkins Pipeline**
Follow these steps to create a Jenkins job with Git integration:

1. Open Jenkins and create a new pipeline job.
2. Add your Git repository URL under the "Source Code Management" section.
3. Configure the pipeline script as shown below:

### **Pipeline Script Overview**
```groovy

  pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/Fumnanya92/Configuration-Management-With-Helm.git']])
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
## The next step is to create your webhook

### Creating a Webhook for Jenkins Integration
#### **Step 1: Add a Webhook to Your GitHub Repository**
1. **Log in to GitHub**:
   - Navigate to your repository.

2. **Go to Repository Settings**:
   - Click the **Settings** tab in your GitHub repository.

3. **Navigate to Webhooks**:
   - On the left sidebar, select **Webhooks**.
   - Click **Add webhook**.

4. **Set up the webhook**:
   - **Payload URL**:
     Enter your Jenkins GitHub webhook endpoint. For example:
     ```
     http://44.242.22.27:8080/github-webhook/
     ```
   - **Content Type**:
     Choose `application/json`.
   - **Secret**:
     (Optional) Add a secret for additional security. This will need to match the secret configured in Jenkins.
   - **Events**:
     Select **Just the push event** or any other events relevant to your pipeline.

5. **Save Webhook**:
   - Click **Add webhook** to save the configuration.
![image](https://github.com/user-attachments/assets/add4ef43-3fec-425c-ae86-f542e2df8089)


---

#### **Step 2: Test the Webhook**
1. **Make a Change**:
   - Push a change to your GitHub repository (e.g., update the `README.md` file).

2. **Verify Jenkins Trigger**:
   - Open Jenkins.
   - Confirm that the pipeline job was triggered automatically by the webhook.

  ---

The next step is connecting Kubernetes to Jenkins with config file
Configure Jenkins Credentials:

In Jenkins, go to Manage Jenkins > Manage Credentials.
Add a new Secret Text credential:
ID: kubeconfig
Secret: Contents of your ~/.kube/config file. This file contains the configuration and credentials for kubectl to access your Minikube cluster.
Configure Jenkins Pipeline:

In your Jenkins pipeline script, ensure you have a stage that uses the kubectl command to deploy your application.

```groovy
pipeline {
    agent any

    environment {
        KUBECONFIG_PATH = '/path/to/kubeconfig' // Specify a valid path
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/Fumnanya92/Configuration-Management-With-Helm.git']])
            }
        }

        stage('Setup Kubeconfig') {
            environment {
                KUBECONFIG = credentials('jenkins-kubeconfig') // Ensure this matches the ID of the uploaded secret file
            }
            steps {
                withCredentials([string(credentialsId: 'kubernetes', variable: 'KUBECONFIG_CONTENT')]) {
                    sh '''
                        # Write the kubeconfig content to a file
                        echo "$KUBECONFIG_CONTENT" > $KUBECONFIG_PATH

                        # Set secure permissions for the file
                        chmod 600 $KUBECONFIG_PATH

                        # Verify kubeconfig
                        echo "Kubeconfig file created at: $KUBECONFIG_PATH"
                    '''
                }
            }
        }

        stage('Run Kubectl Commands') {
            steps {
                withEnv(["KUBECONFIG=$KUBECONFIG_PATH"]) {
                    sh '''
                        # Check the cluster information
                        kubectl cluster-info

                        # List all pods in all namespaces
                        kubectl get pods --all-namespaces
                    '''
                }
            }
        }

        stage('Deploy with Helm') {
            steps {
                script {
                    sh '/usr/local/bin/helm upgrade --install my-webapp ./webapp --namespace default'
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
