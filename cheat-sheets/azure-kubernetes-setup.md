# Microsoft Azure Kubernetes Setup
> Step by step instructions on how to get up and running with Kubernetes on Microsoft Azure Cloud

## Table of Contents

- [Configuration Details](#configuration-details)
- [Minimum Requirements](#minimum-requirements)
- [Once Off Setups](#once-off-setups)
    - [Login to Azure](#login-to-azure)
    - [Create Resource Group](#create-resource-group)
    - [Create Container Registry](#create-container-registry)
    - [Create Service Principal](#create-service-principal)
    - [Create Kubernetes Cluster Without AD Enabling AKS](#create-kubernetes-cluster-without-ad-enabling-aks)
    - [Add Credentials to KubeConfig](#add-credentials-to-kubeconfig)
    - [Create Cluster Role Binding and Launch Dashboard](#create-cluster-role-binding-and-launch-dashboard)
- [Links](#links)

## Configuration Details

The following are config values you will need to use to create the necessary resources

- ***Resource Group Name***: Used to group together resources that make up your Kubernetes implementation
   - A best practice is to use `camelCase` and prefix the name with `rg` (i.e. rg=Resource Group)
   - e.g. `rgAcmeDev`
- ***Location***: The [location](https://azure.microsoft.com/en-us/global-infrastructure/locations/) of the Azure data center that will host your Kubernetes cluster
- ***Azure Container Registry Name***: This is where container images will be hosted
    - The name must be in `lowercase`
    - A best practice prefix the name with `acr` (i.e. acr=Azure Container Registry)
    - e.g. `acracmedev`
- ***Azure SKU***: The type of SKU to be used for the Kubernetes service
    - Basic
    - Standard
    - Premium
- ***Node Count***: A Number value representing the amount of nodes to be created
- ***Cluster Name***: The name of the Kubernetes cluster you are creating
   - A best practice is to use `camelCase` and prefix the name with `cluster`
   - e.g. `clusterAcmeDev`

## Minimum Requirements

* [Microsoft Azure Account](https://azure.microsoft.com/en-us/free/?WT.mc_id=A261C142F)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Kubernetes CLI - kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

> ***NOTE:*** Unless told otherwise, all instructions happen via a Terminal or Command Prompt

> ***NOTE:*** All values below in curly brackets `{}` are placeholders and need to be replaced with [configuration values](#configuration-details) mentioned above. e.g. `{resourceGroupName}` needs to be replaced with `rgAcmeDev`.

## Once Off Setups

This section focuses on once-off setups. Please note that some of these are `(Optional)`.

### Login to Azure

```bash
az login
```
This takes you to a browser page to complete the authentication. You can close the page once authentication is successful.

### Create Resource Group

```bash
az group create --name {resourceGroupName} --location {locationName}
```

### Create Container Registry

```bash
az acr create --resource-group {resourceGroupName} --name {acrname} --sku {skuType} --location {locationName}
```

### Create Service Principal

```bash
az ad sp create-for-rbac --skip-assignment #NB: Take note of App Id and Secret/Password as they get used in the next few commands

ACR_ID=$(az acr show --name {acrname} --resource-group {resourceGroupName} --query "id" --output tsv)

az role assignment create --assignee {appId} --scope $ACR_ID --role acrpull
```

### Create Kubernetes Cluster Without AD Enabling AKS

```bash
az aks create --resource-group {resourceGroupName} --node-count {nodeCount} --name {clusterName} --location {locationName} --service-principal {appId} --client-secret {secretPassword} --generate-ssh-keys
```

### Add Credentials to KubeConfig

```bash
az aks get-credentials --resource-group {resourceGroupName} --name {clusterName} --overwrite-existing
```

### Create Cluster Role Binding and Launch Dashboard

> ***NOTE***: The process below is a bit wonky, but that's because the shorter version sometimes produces weird results. The process below eliminates the weird results.

1. Create cluster role bindings for Kubernetes Dashboard and Default Namespace:

```bash
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

kubectl create clusterrolebinding default --clusterrole=cluster-admin --serviceaccount=kube-system:default
```

2. Create YAML file for Cluster Admins and Azure Account (e.g. `myFile.yaml`):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
name: cluster-admins
roleRef:
apiGroup: rbac.authorization.k8s.io
kind: ClusterRole
name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
kind: User
name: "john@agilite.io"
```

3. Temporarily get credentials as Admin and deploy YAML file:

```bash
az aks get-credentials --resource-group {resourceGroupName} --name {clusterName} --admin

kubectl apply -f {fileName.yaml} #fileName.yaml refers to the file created in step 2
```

4. Delete local admin credentials

The credentials are stored in the Kube Config file. The easiest way to remove them is via VS Code's [Kubernetes](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) extension.

Once installed, open the Kubernetes tab, right-click on the temporary admin cluster created and select `Delete from kubeconfig`.

5. Get credentials again after deleting admin credentials

```bash
az aks get-credentials --resource-group {resourceGroupName} --name {clusterName}
```

6. Launch Kubernetes Dashboard

```bash
az aks browse --resource-group {resourceGroupName} --name {clusterName}
```

## Links

* [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/)
* [VS Code](https://code.visualstudio.com)