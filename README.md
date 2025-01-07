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
