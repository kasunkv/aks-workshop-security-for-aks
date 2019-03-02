# Authenticating with ACR


## 1. Login to Azure Subscription with Azure CLI
---

```powershell
az login --use-device-code

# If you want to swtich subscriptions use the following command.
az account set -s "<subscription-id>"
```

## 2. Create the Azure Container Registry to push our Docker Images
---

```powershell
az acr create --name "<registry-name>" --resource-group "aks-security-rg" --location "southeastasia" --sku basic
```

## 3. Create the Docker Image of the Application
---

```powershell
docker build --file .\Dockerfile --tag aks-security-demo .
```

### Optional
You can run the docker container locally and test the application. Use the following command

```powershell
docker run -p 5001:80 aks-security-demo
```

## 4. Push Docker Image to Azure Container Registry
---

### a). Tag the Docker Image with the ACS url
```powershell 
docker tag "aks-security-demo" "<your-acr-name>.azurecr.io/aks-security-demo"
```

### b). Login to ACR
```powershell
az acr login --name "<registry-name>"
```

### c). Push the newly tagged docker image to ACR
```powershell
docker push "<your-acr-url>/aks-security-demo"
```

## 5. Create a Deployment on AKS using a YAML File
---

### a). Update the `deploy.yml` file with the Docker Image Location
In the `deploy.yml` file find the containers section and update the `image` property to reflect the image tag we created for ACR. It should look something like this.

```yaml
containers:
- name: aksworkshop
  image: "<your-acr-url>/aks-security-demo"
  imagePullPolicy: IfNotPresent
```

### b). Merge Credentials for AKS with your local Kubernetes Config
```powershell
# View available Kubetnetes Contexts
kubectl config get-contexts

# Download the AKS credentials and set the Kubernetes context in the config
az aks get-credentials --name "<aks-name>" --resource-group "aks-security-rg"

# Make sure the config is updated and new context is set to the AKS
kubectl config get-contexts
```

### c). Create the Kubernetes Deployment using YAML file
```powershell
kubectl create --filename .\deploy.yml

# Check the deployment 
kubectl get deployment

# Check the services
kubectl get service

# Watch the pods
kubectl get pods --watch
```

## 6. Authenticate with ACR to Give Access for AKS to Pull Docker Images
---

### a). Get a Reference to the ACR resource ID
```powershell 
# Switch to powershell
powershell

# View the details about the ACR Instance
az acr show --name "<acr-name>" --resource-group "aks-security-rg"

# Store the ACR ID in a variable
$acrId = az acr show --name "<acr-name>" --resource-group "aks-security-rg" --query "id" --output tsv
```

### b). Assign a role to the Service Principle only to Read Images from ACR
```powershell
az role assignment create --assignee "http://sp-aks-security" --role acrpull --scope $acrId
```