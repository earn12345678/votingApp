# Deploy a 3-tier Microservice Voting App using ArgoCD and Azure DevOps Pipeline

The Docker Example Voting App is a microservices application implemented using Python and Node.js. All components run in separate Docker containers to ensure scalability and isolation. The application consists of the following components:

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CI — Azure DevOps                        │
│  GitHub Repo ──► Azure Pipelines ──► Azure Container Registry  │
└───────────────────────────────┬─────────────────────────────────┘
                                │  Image tag update (Bash script)
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CD — ArgoCD (GitOps)                         │
│     Azure Repo (k8s manifests) ──► ArgoCD ──► AKS Cluster      │
└─────────────────────────────────────────────────────────────────┘
```

### Application Components

| Service | Technology | Description |
|---|---|---|
| **Voting Frontend** | Python / Flask | UI for casting votes |
| **Vote Processor** | Node.js / Express | Handles and forwards vote requests |
| **Redis** | Redis | Temporary vote buffer |
| **Worker** | Python | Transfers votes from Redis to PostgreSQL |
| **PostgreSQL** | PostgreSQL | Persistent storage for final results |
| **Results Frontend** | Python / Flask | Real-time results display |

---

## 🔁 Pipeline Stages

### Stage 1 — Continuous Integration (CI)

| Step | Description |
|---|---|
| 1 | Clone and deploy the app locally via Docker Compose |
| 2 | Create an Azure DevOps project and import the repository |
| 3 | Provision an Azure Container Registry (ACR) |
| 4 | Configure a self-hosted Azure DevOps agent |
| 5 | Write CI pipeline scripts (build & push) for each microservice |

### Stage 2 — Continuous Delivery (CD)

| Step | Description |
|---|---|
| 1 | Provision an Azure Managed Kubernetes Cluster (AKS) |
| 2 | Install Azure CLI and connect to AKS |
| 3 | Install ArgoCD on the cluster |
| 4 | Configure ArgoCD to sync with the Azure repository |
| 5 | Write a Bash script to update Kubernetes manifests on new image push |
| 6 | Create an ACR `imagePullSecret` on AKS |
| 7 | Verify the full CI/CD process end-to-end |

---

## 🚀 Getting Started

### Prerequisites

- Azure account with active subscription
- Azure CLI installed
- Docker Desktop & Git installed
- kubectl installed

---

### Stage 1 — CI Setup

#### Step 1: Local Deployment with Docker Compose

```bash
# Provision an Azure Ubuntu Linux VM
az vm create ...

# SSH into the VM
ssh -i ~/.ssh/id_rsa azureuser@<vm-public-ip>

# Update the system
sudo apt-get update

# Install Docker and Docker Compose
# (Follow Docker's official installation guide for Ubuntu)

# Clone the repository
git clone <your-repo-url>
cd <repo-folder>

# Start all services
docker-compose up -d

# Verify locally
curl http://localhost:5000
# Or open in browser: http://<vm-public-ip>:5000
```

#### Step 2: Azure DevOps Project Setup

1. Navigate to [https://dev.azure.com](https://dev.azure.com) and sign in
2. Create a new project named `votingApp` (set visibility to **Private**)
3. Go to **Repos → Files → Import Repository**
4. Paste your Git clone URL and click **Import**

#### Step 3: Azure Container Registry (ACR)

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
