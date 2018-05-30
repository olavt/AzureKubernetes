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
$ az aks create --resource-group TestAKS --name TestAKSCluster --node-count 1 --node-vm-size Standard_B2s --generate-ssh-keys --kubernetes-version 1.9.6
```

The above command will create a 1-node cluster using the VM-size "Standard_B1s". To see what VM-sizes are available in a given location issue the following command:

```
$ az vm list-sizes -l westeurope 
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
$ az aks get-versions --output table --location westeurope
```

```
$ az aks upgrade --name TestAKSCluster --resource-group TestAKS --kubernetes-version 1.9.6
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
$ az aks scale --name TestAKSCluster --resource-group TestAKS --node-count 2
```

## Configure HTTPS Ingress

Run these commands from an Azure CLI (where Helm is installed by default)

Make sure you have configured Kubectl as described earlier to allow access to your AKS cluster.

### Initialize Helm

```
$ helm init
```

### Update Helm repositories

```
$ helm repo update
```

### Install the NGINX ingress controller

```
$ helm install stable/nginx-ingress --namespace kube-system
```

During the installation, an Azure public IP address is created for the ingress controller. To get the public IP address, use the kubectl get service command. It may take some time for the IP address to be assigned to the service.

Check the assigned public IP address:

```
$ kubectl get service -l app=nginx-ingress --namespace kube-system
NAME                                       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
eager-crab-nginx-ingress-controller        LoadBalancer   10.0.182.160   13.82.238.45   80:30920/TCP,443:30426/TCP   20m
eager-crab-nginx-ingress-default-backend   ClusterIP      10.0.255.77    <none>         80/TCP                       20m
```

### Configure DNS Name

Run the followin script to configure the DNS name
```
#!/bin/bash

# Public IP address
IP="<Your external IP address>"

# Name to associate with public IP address
DNSNAME="test-aks-ingress"

# Get resource group and public ip name
RESOURCEGROUP=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[resourceGroup]" --output tsv)
PIPNAME=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[name]" --output tsv)

# Update public ip address with dns name
az network public-ip update --resource-group $RESOURCEGROUP --name  $PIPNAME --dns-name $DNSNAME
```

If needed, run the following command to retrieve the FQDN. Update the IP address value with that of your ingress controller.
```
$ az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '52.224.125.195')].[dnsSettings.fqdn]" --output tsv
```

### Install KUBE-LEGO

The NGINX ingress controller supports TLS termination. While there are several ways to retrieve and configure certificates for HTTPS, this document demonstrates using [KUBE-LEGO](https://github.com/jetstack/kube-lego), which provides automatic [Let's Encrypt](https://letsencrypt.org/) certificate generation and management functionality.
To install the KUBE-LEGO controller, use the following Helm install command. Update the email address with your email-address.

```
helm install stable/kube-lego \
  --set config.LEGO_EMAIL=user@contoso.com \
  --set config.LEGO_URL=https://acme-v01.api.letsencrypt.org/directory
```

### Create an ingress route

This step assumes that you already have deployed some service (web application) to test with.

Create a test-ingress.yaml file with the following content:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - <YourDNSName>.eastus.cloudapp.azure.com
    secretName: tls-secret
  rules:
  - host: <YourDNSName>.eastus.cloudapp.azure.com
    http:
      paths:
      - path: /
        backend:
          serviceName: <YourTestService>
          servicePort: 80
```

Create the ingress resource with the kubectl apply command:

```
kubectl apply -f test-ingress.yaml
```

Now you should be able to browse to https://<YourDNSName>.eastus.cloudapp.azure.com