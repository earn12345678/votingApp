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

### **Step 3: Create an Azure Container Registry**

For this project, I will use Azure Container Registry (ACR) to store and push images. I will **create the ACR** using the **command line** for simplicity.

a. **Set variables**: Use the same resource group as your VM.

```bash
RESOURCE_GROUP=devops-rg
LOCATION=southeastasia
ACR_NAME=devopsacr$RANDOM
```
b. **Create the ACR**

```bash
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME \
  --sku Basic \
  --location $LOCATION \
  --admin-enabled true
```

c. **Verify ACR Creation**: You should see your registry name listed.

```bash
az acr list \
  --resource-group $RESOURCE_GROUP \
  --output table
```

Or you can create your ACR from **Azure Portal** > search for **container registries** > click **create container registries** > choose the **resource group**, **ACR name**, **pricing plan**, and click **review and create**, then click **create**.

**d. Get ACR Login Server**

```bash
az acr show \
  --name $ACR_NAME \
  --query loginServer \
  --output tsv
```

**e. Login to ACR from Your Machine / VM**

```bash
az acr login --name $ACR_NAME
```

If this succeeds, Docker can now push images to ACR.

### **Step 4. Set up Self-Hosted Agent for the Pipeline**

To save resources, I will reuse the same VM used for the local app deployment.

a. Go to **Azure DevOps pipeline** page > click **project setting** at the bottom left of the project page > click **agent pools** > On the right, click **add pools** > for **Pool to link: New** > for **Pool type: Self-hosted** > create a name for your agent > for **Pipeline permissions**, check the box “Grant access permissions to all pipelines” > click **create**.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/02d0dfd6-c911-4663-b0a4-8689b28c8ac0" />

b. Click on the created **agent pool** > on the right, click **New agent** button > In the pop up page, select **Linux**.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/c655f620-2090-46ad-945d-ef138db1d9c2" />

1. Click **COPY** beside Download button: Type Curl -O + **COPY**

<img width="850" height="400" alt="image" src="https://github.com/user-attachments/assets/0871735c-6356-46d8-83dc-84a3f6f70474" />

Then ``` ls ```, If it looks like the image below, then it is correct.

<img width="700" height="350" alt="image" src="https://github.com/user-attachments/assets/6ddfb586-954c-4716-bac9-4495ad1f2e38" />

2. **Create the agent**:

2.1 Create a **new folder** and **navigate into it** using the following command:

```bash
mkdir myagent
cd myagent
```

2.2 Paste ``` tar zxvf ~/Downloads/vsts-agent-linux-x64-4.268.0.tar.gz ```

Then ``` ls ```, you’ll see as the following:

<img width="468" height="50" alt="image" src="https://github.com/user-attachments/assets/37edcc78-dbca-4b50-ae85-409b24cf9df2" />

3. **Configure the agent**: ``` ./config.sh ```

Copy and paste your **Azure DevOps URL link** > Create and paste your **PAT (Personal Access Token)** > Type in the **name of the Agent pool**.

<img width="1080" height="650" alt="image" src="https://github.com/user-attachments/assets/3940c4ab-5235-46d2-924b-cfac5fe878f8" />

4. **Run the agent**: ``` ./run.sh ```

<img width="400" height="160" alt="image" src="https://github.com/user-attachments/assets/fbc8a9b1-224c-4f36-bb98-1a5d3a9c4314" />

If the ``` ./run.sh ``` command is not run, the agent will be offline.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/13df7d14-0967-4844-ae1d-b286f1a5df14" />


### **Step 5: Write a CI pipeline Script for each of the Microservices using separate build and push stages**

Go to **Azure Pipelines** > click **Create Pipeline** > select **GitHub** > select **Only select repositories** > click **Approve & Install** > select the **account**.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/9f887951-9cac-4193-b38c-bd47f8066479" />

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/f31bc471-761c-463d-b0d6-6e120546fb0d" />

Choose **Docker** (the second one) > select an **Azure subscription**.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/e8f796d7-5795-411f-ae37-bcd81af23af0" />

From the drop down, select your ACR name and image name

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/23ddf53e-8b1d-485b-b5dc-df96a0d63305" />

Create the pipeline scripts: We will start with the ``` vote ``` microservice. 

Edit **azure-pipelines.yml** to be like this example:

```yaml
trigger:
  paths:
    include:
      - vote   # Change to 'result' or 'worker' for other services

pool:
  name: <your-pool-name>

resources:
- repo: self

variables:
  dockerRegistryServiceConnection: '<service-connection-id>'
  imageRepository: '<your-image-name>'
  containerRegistry: '<your-acr>.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/vote/Dockerfile'
  tag: '$(Build.BuildId)'

# Agent VM image name
  vmImageName: '<your-vm-image-name>'

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

Wait until show **green** mark on build and push. 

(In this process, please make sure that your **agent pool** is **online** = ``` ./run.sh ``` is still running on the terminal)

<img width="650" height="200" alt="image" src="https://github.com/user-attachments/assets/4739d04e-8224-4cf3-962b-de483d70d3ef" />

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/b4a36d4b-1ff8-4e5d-8142-2663aba0680e" />

You also can see the result on the terminal (the last line).

<img width="650" height="200" alt="image" src="https://github.com/user-attachments/assets/7474271b-075f-4215-afc7-978eeb7caa39" />

Repeat for ``` result ``` and ``` worker ``` microservices — update `displayName`, `repository`, and `Dockerfile` fields accordingly.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/521bc37e-9a5f-4982-a619-11dad59cbed9" />

Go to **Azure Portal** > You can view  all the images pushed into it.

<img width="500" height="200" alt="image" src="https://github.com/user-attachments/assets/328fcc0b-1a2a-4b02-b9e2-dccf5e1671e1" />

---

## **Stage Two: Continuous Delivery (CD)**

### **Step 1: Create an Azure Managed Kubernetes Cluster (AKS)**

**Note:** use this command to check what are the name of your resource group 

```bash
az group list -o table
```

Create AKS Cluster on terminal (It might shows an error about **node-vm-size**, you can change to the another one) or you can create by accessing **Azure Portal**, then click **“create”** on Azure Kubernetes Service). Wait until finish creating.

```bash
az aks create |
	--resource-group <RESOURCE_GROUP> \
	--name <AKS_CLUSTER_NAME> \
	--node-count 1 l
	-node-vm-size Standard_B2s \
	--enable-managed-identity \
	--generate-ssh-keys |
	--attach-acı < ACR_NAME>
```

### **Step 2: Install Azure CLI and Set Up AKS for use**

```bash
# Install Azure CLI
echo "Installing Azure CLI..."
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

```bash
# Login to Azure
az login
```

```bash
# For MacOS
brew update 
```

```bash
# It will appear with a URL and login authentication code.
# Copy the URL into a browser and paste the code into the login page to finish installing the AZ CLI.
az login --use-device-code 
```

```bash
# For MacOS
brew install kubectl 
```

```bash
# Check that it is successfully installed or not
kubectl version –client 
```

```bash
# Verify the connection
kubectl get nodes 
```

Get AKS credentials (Replace <ResourceGroupName> and <AKSClusterName> with your actual resource group and AKS cluster names)

```bash
az aks get-credentials \
	--resource-group SRESOURCE_GROUP \
	--name SAKS_NAME \
	--overwrite-existing
```

### **Step 3: Install ArgoCD**

**Script Overview:**
•	**Step 1:** Installs ArgoCD in a namespace named argocd.
•	**Step 2:** Verifies that all ArgoCD services and pods are fully up and running.
•	**Step 3:** Fetches the default administrator password for ArgoCD.
•	**Step 4:** Publishes the Argo CD server using a NodePort service so it can be accessed through a browser.
•	**Step 5:** Optionally installs the Argo CD command-line tool and authenticates using the initial admin credentials.


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

# Ensure all ArgoCD pods are running before proceeding
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=600s

# Retrieve Argo CD initial admin password
kubectl -n argocd get secret <ARGOCD_ADMIN_SECRET> \
-o jsonpath="{.data.password}" | base64 -d
```

Access the ArgoCD UI at: `http://<node-external-ip>:<nodeport>`

**a. Set up Port Rule for ArgoCD**

```bash
kubectl get svc -n argocd
```

Since ArgoCD is exposed to the internet through a NodePort service. We can access it in a browser using the VM’s IP address and the NodePort assigned to the HTTP (port 80) service, which is 32662 for my cluster, as shown below.”

<img width="468" height="150" alt="image" src="https://github.com/user-attachments/assets/98ad9d35-e23e-4934-ba3e-fcc2080518ae" />

Then, use the following command to get the **EXTERNAL-IP**.

```bash
kubectl get nodes -o wide
```

Enter the IP address and Port number in a web browser. 

Example: http://xx.xxx.xx.xxx:xxxxx

You will see the argoCD sign-in landing page.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/b17efa61-1c1a-4c59-a493-edfc25a2dcc4" />

> **Note:** Ensure the NodePort is allowed through your Azure VMSS inbound security rules.

Login to the argoCD using the following:

```bash
username: admin
password: <your decoded argocd password>
```

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/0988d79a-da5f-4941-9368-2599f856411c" />

Go to the Azure Portal and search for VMSS (Virtual Machine Scale Set) > Select Network settings > click Create port rule > From the dropdown menu, choose Inbound rule > add the required port number.

<img width="164" height="222" alt="image" src="https://github.com/user-attachments/assets/66418ab4-7fdc-4b8a-9faf-d6bdeb7bbf97" />


### **Step 4: Configure ArgoCD**

To connect to our **Azure repository** where the **Kubernetes manifest files** are stored. ArgoCD will deploy these manifests to our Kubernetes cluster and **continuously monitor** the Azure repository for changes. **Any updates detected** in the repository will automatically be synchronized and applied to the cluster. So, we have to connect ArgoCD to both the **Azure repository and our AKS cluster**.

**a. Connect to the Azure repository:**

Copy the HTTPS URL of your Azure repository and paste it into the **URL** field.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/c358542a-24e2-4f9d-8f8c-012e0a4cc7e6" />

**b. Next, create a Personal Access Token (PAT):**

Go to your **Azure repository** and click **User settings** > From the dropdown menu, select **Personal access tokens** > click **New Token** > Give the token a name and assign the required permissions (read access is recommended). > copy the newly created PAT and store it securely. 

(Make sure you copy the token now, the token is not stored and you will not be able to see it again)

**c. Connect Argo CD to the Azure repository:**

On the Argo CD page, go to **Settings** and click + Connect Repo > Choose Via HTTPS as the connection method > select **default** for the project > Paste the **Azure repository URL** > edit the repository settings > enter your **Personal Access Token (PAT)** > click **Connect to test** and confirm that the connection is successful.

For Example: https://<INSERT-PERSONAL-ACCESS-TOKEN-HERE>@dev.azure.con/<project-name>/votingApp/_git/votingApp 

Alternatively, you can enter the PAT in the Password field.

Check the **connection status**

<img width="468" height="152" alt="image" src="https://github.com/user-attachments/assets/5093f4e8-842d-4348-8d50-556e8f5cf9d9" />

**d. Connect Argo CD to Azure Kubernetes Service (AKS):**

On the Argo CD page, click **New Application**. Provide an Application Name > select **default** for the project > Set the Sync Policy to Automatic > For the Repository URL, choose the Azure repository you added earlier, it will appear automatically > In the Path field, enter k8s-specifications > set the Namespace to default > Create the application and wait for all Kubernetes manifest files to be deployed, with the pods reaching a running state.

The k8s-specifications directory is the folder in our Azure repository that contains all Kubernetes manifest files. 

Argo CD will deploy the resources from this folder and continuously monitor it for any changes

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/2267cab2-815b-4afc-b09c-9937fe8e3a3a" />

From the Argo CD dashboard, we can confirm that **all pods are running**.

<img width="160" height="240" alt="image" src="https://github.com/user-attachments/assets/e3ac47ed-d85d-46e7-8ead-8a73ba59c487" />

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/7d65d9ac-8a3e-45bf-b80c-05c18051f324" />

You can also check through the terminal to confirm the running pods by using the following command:

```bash
kubectl get pods -n default
```

### **Step 5: Write a Script that updates the pipeline image on Kubernetes manifest**

This is a key step in our CI/CD pipeline. During the CD stage, Argo CD monitors Kubernetes manifest files in the Azure repository and deploys changes to the Azure Kubernetes Service (AKS) cluster. However, Argo CD does not directly track image updates in Azure Container Registry (ACR).

To bridge the CI and CD stages, we use a Bash script that detects new images pushed to ACR and updates the corresponding Kubernetes manifests in the Azure repository. Argo CD then automatically applies these changes to the cluster.

a. Add the script to the vote-service folder in the repository.

<img width="1250" height="800" alt="image" src="https://github.com/user-attachments/assets/5776dd65-19bd-4660-a8ee-e0a7e7b31268" />

This Bash script automates the process of cloning a Git repository, updating the Docker image tag in a Kubernetes deployment, committing the changes, and pushing them back to the repository. It accepts three parameters:

**Script parameters:**
- `$1` — Deployment file prefix (e.g., `vote`)
- `$2` — Image name
- `$3` — New image tag (Build ID)

The script detects newly built images in the container registry by monitoring image tags. Since each CI build generates a new image tag, the script identifies the latest tag and updates the Kubernetes manifest accordingly.

Once this process is complete, the Kubernetes deployment manifest is updated with the new image and tag. **Argo CD** then detects the change and deploys the updated version to the **Azure Kubernetes Service (AKS)** cluster. As a result, a new version of the application is deployed automatically whenever a change is made.

Specifically, this process updates the **image** field in the ``` vote-deployment.yaml ``` file.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: vote
  name: vote
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
      - image: gabvotingappacr.azurecr.io/votingapp/vote:18 ##tag changes anytime a new image is built
        name: vote
        ports:
        - containerPort: 80
          name: vote
```

b. Add a update stage to the ``` vote-service ``` pipeline

```yaml
# Add the script below to the vote-service pipeline 
# =========================
# STAGE 3: UPDATE K8S MANIFEST
# =========================
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

Now that the shell script has been integrated into the CI process and the CI/CD pipeline is fully set up, we can proceed to test the deployment.

<img width="600" height="200" alt="image" src="https://github.com/user-attachments/assets/bcbd8abf-cd04-430b-87d7-4748ab1d2de2" />

### **Step 6: Create an ACR imagePullSecret on AKS**

Add imagePullSecrets and its name at last of the yaml file.

```yaml
spec:
  imagePullSecrets:
    - name: <acr-secret-name>
```

<img width="468" height="200" alt="image" src="https://github.com/user-attachments/assets/dc883a92-0609-4139-b98c-5cb8f793a0c9" />

View the app using ``` http://<nodeip-address>:port number ```

### **Step 7: Verify the CI/CD Pipeline**

Make a change to the application (e.g., update vote options in the frontend)

I use nano to for simplicity, then commit the changes to the repository:

<img width="650" height="400" alt="image" src="https://github.com/user-attachments/assets/1a3f259a-8ae9-449b-a8ca-217648d97ab7" />

```bash
nano vote/app.py   # Edit vote options (e.g., "love / hate")
git add .
git commit -m "Change vote options"
git push
```
Then, you can see the change via the web page.

Azure DevOps will trigger the pipeline automatically → build, push, and update the K8s manifest → ArgoCD detects the change and deploys the new version to AKS.

---

## Tech Stack

| Category | Technology |
|---|---|
| **CI Platform** | Azure DevOps Pipelines |
| **Container Registry** | Azure Container Registry (ACR) |
| **Container Runtime** | Docker / Docker Compose |
| **Orchestration** | Azure Kubernetes Service (AKS) |
| **GitOps / CD** | ArgoCD |
| **IaC / Scripting** | Azure CLI, Bash |
| **Cloud Provider** | Microsoft Azure |
