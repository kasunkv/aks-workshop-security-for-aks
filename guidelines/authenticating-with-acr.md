# Authenticating with ACR


## 1. Login to Azure Subscription with Azure CLI
```bash
az login --use-device-code

# If you want to swtich subscriptions use the following command.
az acount set -s "<subscription-id>"
```

## 2. Create a Resource Group for the Resources
```bash
az group create --name "aks-security-rg" --location "southeastasia"
```

## 3. Create the Azure Container Registry to push our Docker Images
```bash
az acr create --name "<registry-name>" --resource-group "aks-security-rg" --location "southeastasia" --sku basic
```

## 4. Create the Azure Kubernetes Service to Run the Containers

### a). Create the Service Principle for AKS
```bash
az ad sp create-for-rbac --name "sp-aks-security" --skip-assignment

# this will give a json object containing the appId and the password. Make note of these 2 values
# we need this for the next step
```

### b). Create the Azure Kubernetes Service
```bash
az aks create --name "<aks-name>" --resource-group "aks-security-rg" --node-count 1 --generate-ssh-keys  --service-principal "<sp-app-id>" --client-secret "<sp-app-password>"
```

## 5. Create the Docker Image of the Application
```bash
docker build --file .\Dockerfile --tag aksworkshop:v1 .
```

## 6. Push Docker Image to Azure Container Registry

### a). Tag the Docker Image with the ACS url
```bash 
docker tag "aksworkshop:v1" "<your-acr-url>/aksworkshop:v1"
```

### b). Login to ACR
```bash
az acr login --name "<registry-name>"
```

### c). Push the newly tagged docker image to ACR
```bash
docker push "<your-acr-url>/aksworkshop:v1"
```

## 7. Create a Deployment on AKS using a YAML File

### a). Update the `deploy.yml` file with the Docker Image Location
In the `deploy.yml` file find the containers section and update the `image` property to reflect the image tag we created for ACR. It should look something like this.

```yaml
containers:
- name: aksworkshop
  image: "<your-acr-url>/aksworkshop:v1"
  imagePullPolicy: IfNotPresent
```

### b). Merge Credentials for AKS with your local Kubernetes Config
```bash
# View available Kubetnetes Contexts
kubectl config get-contexts

# Download the AKS credentials and set the Kubernetes context in the config
az aks get-credentials --name "<aks-name>" --resource-group "aks-security-rg"

# Make sure the config is updated and new context is set to the AKS
kubectl config get-contexts
```

### c). Create the Kubernetes Deployment using YAML file
```bash
kubectl create --filename .\deploy.yml

# Check the deployment 
kubectl get deployment

# Check the services
kubectl get service

# Watch the pods
kubectl get pods --watch
```

## 8. Authenticate with ACR to Give Access for AKS to Pull Docker Images

### a). Get a referece to the ACR resource ID
```bash 
# Switch to powershell
powershell

# View the details about the ACR Instance
az acr show --name "<acr-name>" --resource-group "aks-security-rg"

# Store the ACR ID in a variable
$acrId = az acr show --name "<acr-name>" --resource-group "aks-security-rg" --query "id" --output tsv
```

### b). Assign a role to the Service Principle only to Read Images from ACR
```bash
az role assignment create --assignee "http://sp-aks-security" --role acrpull --scope $acrId
```