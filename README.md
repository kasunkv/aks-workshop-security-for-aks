# Security for AKS - AKS Workshop 2019 Colombo
Security for AKS Hands-On Session Guidelines for AKS Workshop 2019. Make sure you read the prerequisites section and install all the necessary tools needed to follow along with the hands-on session.

# Content of the Hands-On Session

* [Authenticating Your AKS with ACR to Pulldown Images](https://github.com/kasunkv/aks-workshop-security-for-aks/blob/master/guidelines/authenticating-with-acr.md)
* [Creating AKS Cluster with Azure Active Directory Integration](https://github.com/kasunkv/aks-workshop-security-for-aks/blob/master/guidelines/rbac-for-aks.md)

> ### Note:
> Make sure you have read and completed the **Prerequisites** section to install and configure the tools needed to follow along in the hands-on session. The prerequisites can be found below.


# Prerequisites
Make sure to install the following tools in your development machine prior to the hands-on session.

## 1. Install Git
Make sure you have git installed so you can clone the source code for the hands-on session. You can download Git from the [Official Download Page](https://git-scm.com/downloads)

## 2. Latest Version of Azure CLI
The current latest version of the Azure CLI is 2.0.59 as of writing this guidelines. Download and Install the Azure CLI for your operating system by following the links given below.

* [Install On Windows](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest)
* [Install on MacOS](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest)
* [Install on Linux](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

Run the following commands to verify the installation and the CLI version
```bash
# Check the Azure CLI version
az --version

# Login to Azure Subscription
az login --use-device-code
```

## 3. kubectl - Kubernetes Command-Line Client
You need to install `kubectl` on your local machine to manage your Kubernetes cluster on AKS. It's really simple. Once you have Azure CLI installed run the following command.

```bash
# Install kubectl using Azure CLI
az aks install-cli
```

You can run the following command to connect to your AKS cluster by downloading the credentials and configuring kubectl to use them

```bash
# Download and merge the AKS config with your local kubernetes config
az aks get-credentials --resource-group "<your-resource-group>" --name "<aks-cluster-name>"
```

Once the credentials are downloaded. Run the following command to make sure you have access to the AKS cluster.

```bash
# You should see the available nodes in your AKS cluster
kubectl get nodes
```


# Optional Steps
If you don't want to build the docker image used for the hands-on session. You can download the pre-made docker image from the Docker Hub. Use the following command to download the docker image to your local machine.

```bash
# Pull the kasunkv/aks-security-demo image from docker hub
docker pull kasunkv/aks-security-demo
```

After that you can tag the image for the ACR and push it to Azure

# Using Azure Cloud Shell
If anything goes wrong with your local development environment when you try to use Azure CLI or kubectl commands, you can use the Azure Cloud Shell to execute the same commands and follow along.