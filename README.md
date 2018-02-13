## Introduction

This article documents the process of installing a Kubernetes Cluster in Azure using AKS.

### Login to your Azure Subscription
```
$ az login
```

### Enabling AKS preview for your Azure subscription
```
$ az provider register -n Microsoft.ContainerService
```

### Create a new Resource Group
```
$ az group create --name TestAKS --location westeurope
```

### Create AKS cluster
```
$ az aks create --resource-group TestAKS --name TestAKSCluster --node-count 1 --generate-ssh-keys --kubernetes-version 1.8.7
```

### Install kubectl on your local computer
```
$ az aks install-cli
```

### Configure kubectl to connect to your Kubernetes cluster,
```
$ az aks get-credentials --resource-group TestAKS --name TestAKSCluster
```

### Verify the connection to your cluster
```
$ kubectl get nodes
```

### Upgrade to a newer Kubernetes version
```
$ az aks get-versions --name TestAKSCluster --resource-group TestAKS --output table
```

```
$ az aks upgrade --name TestAKSCluster --resource-group TestAKS --kubernetes-version 1.8.7
```

### Create a secret for pulling the image from a private registry

If you want to be able to pull an image from a private Docker registry, you need to create a secret for it.

```
$ kubectl create secret docker-registry <your-secret-name> --docker-server=<your-docker-server> --docker-username=<your-username> --docker-password=<your-password> --docker-email=<your-email>
```

### Deploy

You are now ready to deploy to your Kubernetes cluster

### Start Kubernetes Dashboard
```
$ az aks browse --resource-group TestAKS --name TestAKSCluster
```

### To get the Public IP address of a deployed service
```
$ kubectl get service homeautomationweb
```

### Scale the cluster
```
$ az aks scale --name myAKSCluster --resource-group myResourceGroup --node-count 1
```
