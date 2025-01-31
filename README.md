# Deploy Reddit App to Amazon Elastic Kubernetes Service (EKS) using ArgoCD and monitor its performance

## 🛠️ Steps to Implement

### **Step 1: Setup Jenkins Server**

1. **Cd to jenkins sevrer folder:**

   ```bash
   cd Reddit-Project/Jenkins-Server-TF/
   ```

2. **Modify Backend.tf:**
   - Create an S3 bucket and a DynamoDB table.
  
3. **Install Terraform and AWS CLI:**

   ```bash
   # Install Terraform
   sudo apt install wget -y
   wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install terraform

   # Install AWS CLI 
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   sudo apt-get install unzip -y
   unzip awscliv2.zip
   sudo ./aws/install

   # Configure AWS CLI
   aws configure
   ```

   Provide your AWS Access Key ID, Secret Access Key, region name.

4. **Run Terraform Commands:**

   ```bash
   terraform init
   terraform validate
   terraform plan -var-file=variables.tfvars
   terraform apply -var-file=variables.tfvars --auto-approve
   ```

   - This will create an instance on AWS.

5. **Access Jenkins:**

   - Copy the public IP of the instance and access Jenkins on your favorite browser:

     ```
     <public_ip>:8080
     ```
  
6. **Get Jenkins Password:**

   - Connect to the instance and retrieve the password.
  
7. **Create Jenkins User:**

   - (Optional) Create a user if you don’t want to keep the default password.

8. **Install Required Plugins:**

   - Navigate to **Manage Jenkins → Plugins → Available Plugins** and install the following plugins without restarting:
     1. Eclipse Temurin Installer
     2. SonarQube Scanner
     3. NodeJs Plugin
     4. Docker Plugins (Docker, Docker commons, Docker pipeline, Docker API, Docker Build step)
     5. Owasp Dependency Check
     6. Terraform
     7. AWS Credentials
     8. Pipeline: AWS Steps
     9. Prometheus Metrics Plugin

9. **Access SonarQube Console:**

    ```
    <public_ip>:9000
    ```

    - Both Username and Password are "admin". Update the password and configure as needed.

10. **Create and Configure Credentials:**
    
    - Navigate to **Manage Jenkins → Credentials → Global** and create credentials for AWS, GitHub, and Docker.

---

### **Step 2: Create EKS Cluster with Jenkins Pipeline**

1. **Create a New Jenkins Pipeline:**
   
   - Click on **New Item**, give a name, select **Pipeline**, and click **OK**.

2. **Configure Pipeline:**

   - Navigate to the Pipeline section, provide the GitHub URL of your project, and specify the credentials and the path to the Jenkinsfile. "Jenkinsfile-EKS-Terraform"

3. **Build the Pipeline:**

   - Click **Apply** and then **Build**. This will create an EKS cluster.

---

### **Step 3: Create Jenkins Job to Build and Push the Image**

1. **Create a New Jenkins Job:**

   - Click on **New Item**, give a name, select **Pipeline**, and click **OK**.

2. **Configure Jenkins Job:**

   - In the pipeline section:
     - Choose **Script from SCM**.
     - Set up **Git** with your GitHub credentials.
     - Set the branch as `main` and the pipeline path as `Jenkins-Pipeline-Code/Jenkinsfile-Reddit`.

3. **Build the Pipeline:**

   - Before building, create a GitHub token as secret text with the ID `githubcred` to update the built image in the deployment.yml file.

4. **Check Scanning Results:**

   - View Trivy scan results, SonarQube analysis, and Dependency Checker outputs.

5. **Update Deployment File:**

   - The deployment file will be updated with the tag of the Jenkins build number.

---

### **Step 4: Configure EKS and ArgoCD**

1. **Update EKS Cluster Config:**

   ```bash
   aws eks update-kubeconfig --name Reddit-EKS-Cluster
   ```

2. **Install ArgoCD:**

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
   ```

3. **Expose ArgoCD Server:**

   ```bash
   kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
   ```

4. **Retrieve ArgoCD Server Info:**

   ```bash
   export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
   export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
   ```

5. **Access ArgoCD Console:**

   - Login using the DNS name and credentials.

6. **Create ArgoCD Application:**
   - Click on **Create App**, edit the YAML, and replace `repoURL` with your GitHub project URL:

   ```yaml
   project: default
   source:
     repoURL: 'https://github.com/uniquesreedhar/Reddit-Project.git'
     path: K8s/
     targetRevision: HEAD
   destination:
     server: 'https://kubernetes.default.svc'
     namespace: default
   syncPolicy:
     automated:
       prune: true
       selfHeal: true
   ```

7. **Deploy and Sync:**

    - Deploy and sync your Reddit application with EKS.

---

## 🔍 Monitoring

### **Implement Prometheus and Grafana:**

1. **Deploy Prometheus and Grafana:**

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/uniquesreedhar/Reddit-Project/main/Prometheus/
   ```

2. **Expose Prometheus and Grafana:**

   ```bash
   kubectl expose pod <prometheus-pod> --port=8080 --target-port=9090 --name=prometheus-lb --type=LoadBalancer
   kubectl expose pod <grafana-pod> --port=8081 --target-port=3000 --name=grafana-lb --type=LoadBalancer
   ```

3. **Access Grafana:**

   - Copy the public IP and access it through `<public_ip>:8081`.

4. **Login to Grafana:**

   - Default credentials: `admin/admin`.

5. **Add Prometheus Data Source in Grafana:**

   - Navigate to **Add Data Source → Prometheus**.
   - Set up and start monitoring.

