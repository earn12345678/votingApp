# Deploy a 3-tier Microservice Voting App using ArgoCD and Azure DevOps Pipeline

The Docker Example Voting App is a **microservices application** implemented using **Python** and **Node.js**. All components run in separate Docker containers to ensure scalability and isolation. The application consists of the following components:
1. **Voting Frontend (Python / Flask)**: The voting page where users choose and submit their vote.
	
2. **Vote processor Backend (Node.js / Express)**: Handling incoming vote request by receiving votes from the frontend and processing the request, then forwards the vote for temporary storage.
   
3. **Redis Database**: Stores votes temporarily for fast access.
	
4. **Worker (Python)**: Processes votes from Redis and sends the final count to the main database (PostgreSQL).
	
5. **PostgreSQL Database**: It is a permanent storage where stores the final voting results.
	
6. **Results Frontend (Python / Flask)**: The results page displays real-time voting results.

---

## **Stage One: Continuous Integration (CI)**

•	Step 1: Clone and Deploy the App Locally Using Docker-Compose

•	Step 2: Create an Azure DevOps Project and Import the Repository

•	Step 3: Create an Azure Container Registry

•	Step 4: Set Up Self-Hosted Agent for the Pipeline

•	Step 5: Write a CI Pipeline Script for Each Microservice using separate build and push stages

## **Stage Two: Continuous Delivery (CD)**

•	Step 1: Create an Azure Managed Kubernetes Cluster (AKS)

•	Step 2: Install Azure CLI and Set Up AKS for Use

•	Step 3: Install ArgoCD

•	Step 4: Configure ArgoCD

•	Step 5: Write a Bash Script that updates the pipeline image on K8s manifest

•	Step 6: Create an ACR ImagePullSecret on AKS

•	Step 7: Verify the CI/CD process

---

## **Stage One: Continuous Integration (CI)**

### **Step 1: Clone and Deploy the app locally using docker-compose**

Create a VM to test our app locally 

(Make sure you have Docker Desktop and Git installed on your computer)

**a. Create an Azure Linux Ubuntu VM and Login:**

```bash
# Make sure you have
az version
ssh -V

# If Azure CLI is not stalled (Mac):
brew install azure-cli

# Login to Azure
az login
```

A browser will open → sign in to **Microsoft Azure** → return to terminal. 

(If you do not have an Azure account, please create one first)

1. **Set Variables**:

```bash
RESOURCE_GROUP=devops-rg
VM_NAME=devops-vm
LOCATION=southeastasia
USERNAME=azureuser
```

2. **Create a Resource Group**

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

3. **Create an SSH Key**

Check if SSH key exists:

```bash
ls ~/.ssh
```

<img width="650" height="55" alt="image" src="https://github.com/user-attachments/assets/00d27917-13df-4637-b60a-45c01ef20580" />

If **id_rsa** does not exist, create one:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
```

Create **Ubuntu Linux VM**

This command creates **Ubuntu VM**, adds your **SSH key** automatically, and opens **port 22 (SSH)** by default.

```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --admin-username $USERNAME \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --size Standard_B2s \
  --public-ip-sku Standard
```

**Get the Public IP Address by:**

```bash
az vm list-ip-addresses \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --output table
```

**You’ll see something like:**

```bash
VirtualMachine    PublicIP
---------------   ------------
devops-vm         20.xxx.xxx.xxx
```

**SSH into the VM:**

```bash
ssh -i ~/.ssh/id_rsa azureuser@<PUBLIC-IP>
```

If asked “Are you sure you want to continue connecting?”, type “yes”. You are now inside the Azure Ubuntu VM.

**b. Update the VM: run the following command**

```bash
sudo apt-get update
```

**c. Install Docker and Docker-Compose**: To test the app locally on the VM, we need to install Docker on the machine to run the ``` docker-compose up -d ``` command. 

Docker Compose is used to define and manage multi-container applications, including microservices.

```bash
#### 1. Install required packages
sudo apt-get install \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

#### 2. Add Docker Official GPG Key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

#### 3. Set Up the Stable Repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#### 4. Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y

#### 5. Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -oP '"tag_name": "\K(.*)(?=")')/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

#### 6. Manage Docker as a Non-Root User and refresh it
sudo usermod -aG docker $USER
newgrp docker

#### 7. Check versions
sudo docker --version
docker-compose --version

#### 8. Check you can docker command without sudo
docker ps
```

**d. Clone the Repo:**

Clone or fork the App > ``` cd ``` into the repository > run the ``` docker-compose up -d ``` command

```bash
git clone <https://github.com/xxxxx/<example-voting-app>.git>
```

```bash
ls
```

```bash
cd <example-voting-app>
```

Run the command to start all docker containers simultaneously

```bash
docker-compose up -d
```

The output shows that all containers are running

```bash
docker ps
```

**e. From the output, we can see the port number mapped to the app container**

If it is port 5000, run ``` curl http://localhost:5000 ``` to view the app in the terminal.

**f. Copy the VM public address and type into the browser ``` http://vm-public-ip>:5000 ``` to view the App.**

<img width="850" height="400" alt="image" src="https://github.com/user-attachments/assets/a53ade23-6ca8-41c7-b405-43f106f7c5b5" />


### **Step 2: Create an Azure DevOps Project and Import the repository **

a. Access **Azure DevOps** by navigating to **https://dev.azure.com** and signing in. If this is your first time, you may need to sign up and create an organization. You may choose any organization name.

b. **Create a new project** named ``` votingApp ``` > set the visibility to **Private** > click **Create Project**

c. **Import the Git Repo to Azure Repo**: Repos > Files > In Repository type leave it as **Git** > Paste your **Git clone URL** > Click **Import**

Then, it will show the entire repository.
<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/4bf7c63c-d252-4dce-809e-1e3e73185740" />


### **Step 3: Azure Container Registry (ACR)**

```bash
# Set variables (use same resource group as your VM)
ACR_NAME=<your-acr-name>
RESOURCE_GROUP=<your-resource-group>

# Create the ACR
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic

# Verify creation
az acr list --output table

# Get the login server
az acr show --name $ACR_NAME --query loginServer --output tsv

# Authenticate Docker with ACR
az acr login --name $ACR_NAME
```

#### Step 4: Self-Hosted Agent

1. In Azure DevOps, go to **Project Settings → Agent Pools → Add Pool**
   - Pool type: **Self-hosted**
   - Enable: **Grant access permissions to all pipelines**
2. Click the pool → **New Agent → Linux**
3. Download and extract the agent:

```bash
mkdir myagent && cd myagent
curl -O <agent-download-url>
tar zxvf vsts-agent-linux-x64-*.tar.gz
```

4. Configure and run:

```bash
./config.sh   # Enter DevOps URL, PAT, and agent pool name
./run.sh      # Keep this running during pipeline execution
```

#### Step 5: CI Pipeline (azure-pipelines.yml)

```yaml
trigger:
  paths:
    include:
      - vote   # Change to 'result' or 'worker' for other services

pool:
  name: <your-pool-name>

variables:
  dockerRegistryServiceConnection: '<service-connection-id>'
  imageRepository: '<your-image-name>'
  containerRegistry: '<your-acr>.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/vote/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
  - stage: Build
    displayName: Build the Voting App
    jobs:
      - job: Build
        steps:
          - task: Docker@2
            inputs:
              containerRegistry: '<your-acr-name>'
              repository: 'votingapp/vote'
              command: 'build'
              Dockerfile: 'vote/Dockerfile'

  - stage: Push
    displayName: Push the Voting App
    jobs:
      - job: Push
        steps:
          - task: Docker@2
            inputs:
              containerRegistry: '<your-acr-name>'
              repository: 'votingapp/vote'
              command: 'push'
```

> Repeat for `result` and `worker` microservices — update `displayName`, `repository`, and `Dockerfile` fields accordingly.

---

### Stage 2 — CD Setup

#### Step 1: Provision AKS Cluster

```bash
# Check available resource groups
az group list -o table

# Create the AKS cluster
az aks create --resource-group <ResourceGroup> --name <ClusterName> --node-count 2 --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group <ResourceGroup> --name <ClusterName>

# Verify connection
kubectl get nodes
```

#### Step 2: Install Azure CLI & Connect to AKS

```bash
# Install Azure CLI (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install kubectl (macOS)
brew install kubectl

# Login to Azure
az login --use-device-code

# Verify kubectl connection
kubectl version --client
kubectl get nodes
```

#### Step 3: Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose via NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode

# Get NodePort and external IP
kubectl get svc -n argocd
kubectl get nodes -o wide
```

Access the ArgoCD UI at: `http://<node-external-ip>:<nodeport>`

> **Note:** Ensure the NodePort is allowed through your Azure VMSS inbound security rules.

#### Step 4: Configure ArgoCD

**Connect the Azure Repository:**
1. In ArgoCD UI, go to **Settings → Repositories → Connect Repo**
2. Method: **Via HTTPS**
3. Enter your Azure Repo URL and Personal Access Token (PAT)

```
https://<PAT>@dev.azure.com/<org>/votingApp/_git/votingApp
```

**Create an ArgoCD Application:**
1. Click **New Application**
2. Set **Sync Policy** to **Automatic**
3. Set **Repository URL** to your connected Azure repo
4. Set **Path** to `k8s-specifications`
5. Set **Namespace** to `default`

#### Step 5: Kubernetes Manifest Update Script

This Bash script bridges CI and CD by automatically updating the image tag in Kubernetes manifests whenever a new image is pushed to ACR.

**Script parameters:**
- `$1` — Deployment file prefix (e.g., `vote`)
- `$2` — Image name
- `$3` — New image tag (Build ID)

**Add an Update stage to the pipeline:**

```yaml
- stage: Update_K8s
  displayName: Update K8s Manifest
  dependsOn: Push
  jobs:
    - job: Update
      steps:
        - checkout: self
          persistCredentials: true
        - task: Bash@3
          displayName: Update deployment image
          inputs:
            targetType: 'inline'
            workingDirectory: $(Build.SourcesDirectory)
            script: |
              chmod +x updateK8sManifests.sh
              ./updateK8sManifests.sh vote $(imageRepository) $(Build.BuildId)
```

#### Step 6: ACR ImagePullSecret

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=<your-acr>.azurecr.io \
  --docker-username=<acr-username> \
  --docker-password=<acr-password> \
  --namespace=default
```

Then add the following to your Kubernetes deployment manifests:

```yaml
spec:
  imagePullSecrets:
    - name: acr-secret
```

#### Step 7: Verify the CI/CD Pipeline

Make a change to the application (e.g., update vote options in the frontend):

```bash
nano vote/app.py   # Edit vote options (e.g., "love / hate")
git add .
git commit -m "feat: update vote options"
git push
```

Azure DevOps will trigger the pipeline automatically → build, push, and update the K8s manifest → ArgoCD detects the change and deploys the new version to AKS.

---

## 🛠️ Tech Stack

| Category | Technology |
|---|---|
| **CI Platform** | Azure DevOps Pipelines |
| **Container Registry** | Azure Container Registry (ACR) |
| **Container Runtime** | Docker / Docker Compose |
| **Orchestration** | Azure Kubernetes Service (AKS) |
| **GitOps / CD** | ArgoCD |
| **IaC / Scripting** | Azure CLI, Bash |
| **Cloud Provider** | Microsoft Azure |

---

## 📁 Repository Structure

```
├── vote/                    # Voting frontend (Python/Flask)
│   ├── Dockerfile
│   └── ...
├── result/                  # Results frontend (Python/Flask)
│   ├── Dockerfile
│   └── ...
├── worker/                  # Vote worker (Python)
│   ├── Dockerfile
│   └── ...
├── k8s-specifications/      # Kubernetes manifests (monitored by ArgoCD)
│   ├── vote-deployment.yaml
│   ├── result-deployment.yaml
│   ├── worker-deployment.yaml
│   └── ...
├── updateK8sManifests.sh    # CI/CD bridge script
├── docker-compose.yml       # Local development stack
└── azure-pipelines.yml      # Azure DevOps CI pipeline definition
```

---

## 📄 License

This project is intended for educational and demonstration purposes.
