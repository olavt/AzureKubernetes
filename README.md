## Introduction

This article documents the process of installing a Kubernetes Cluster in Azure using AKS.

### Prerequsites

- Azure CLI needs to be installed on the local computer in order to issue az command locally. Alternatively you can use the Cloud Shell from the Azure Portal.

### Login to your Azure Account
```
$ az login
```

### List the Azure Subscriptions for your account

```
$ az account list --output table
```

### Select the right subscription

```
$ az account set --subscription "<Subscription Id>"
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
$ az aks create --resource-group TestAKS --name TestAKSCluster --node-count 1 --node-vm-size Standard_B2s --generate-ssh-keys --kubernetes-version 1.12.5
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
If you re-run the process, you may need to add the --overwrite-existing option. 

### Verify the connection to your cluster
```
$ kubectl get nodes
```

### Upgrade to a newer Kubernetes version

To list the available Kubernetes versions in a specific region:
```
$ az aks get-versions --output table --location westeurope
```

To perform an upgrade:
```
$ az aks upgrade --name TestAKSCluster --resource-group TestAKS --kubernetes-version 1.12.6
```

### Create a secret for pulling the image from a private registry

If you want to be able to pull an image from a private Docker registry, you need to create a secret for it.

```
$ kubectl create secret docker-registry <your-secret-name> --docker-server=<your-docker-server> --docker-username=<your-username> --docker-password=<your-password> --docker-email=<your-email>
```

### Deploy

You are now ready to deploy to your Kubernetes cluster. This is typically done using kubectl with yaml files describing the resources to deploy.

### Configuring the AKS cluster for running the Kubernetes Dashboard

By default the AKS cluster uses RBAC, a ClusterRoleBinding must be created before you can correctly access the dashboard. By default, the Kubernetes dashboard is deployed with minimal read access and displays RBAC access errors. The Kubernetes dashboard does not currently support user-provided credentials to determine the level of access, rather it uses the roles granted to the service account. A cluster administrator can choose to grant additional access to the kubernetes-dashboard service account, however this can be a vector for privilege escalation. You can also integrate Azure Active Directory authentication to provide a more granular level of access.
To create a binding, use the kubectl create clusterrolebinding command as shown in the following example.

Warning
This sample binding does not apply any additional authentication components and may lead to insecure use. The Kubernetes dashboard is open to anyone with access to the URL. Do not expose the Kubernetes dashboard publicly.
For more information on using the different authentication methods, see the Kubernetes dashboard wiki on access controls.

```
$ kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```

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

### Configure your AKS cluster for Helm

Make sure you read and follow the steps outlined in this article: [Install applications with Helm in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm)

Run these commands from an Azure CLI (where Helm is installed by default)

Make sure you have configured Kubectl as described earlier to allow access to your AKS cluster.

### Create a service account for Tiller

Create a file named helm-rbac.yaml and copy in the following YAML:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

Run it:
```
$ kubectl apply -f helm-rbac.yaml
```

### Initialize Helm

```
$ helm init --service-account tiller
```

### Update Helm repositories

```
$ helm repo update
```

## Configure HTTPS Ingress using Nginx

### Install Nginx Manually

Create the mandatory Nginx resources, use kubectl apply and the -f flag to specify the manifest file hosted on GitHub:
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

Create the LoadBalancer Service, kubectl apply a manifest file containing the Service definition:
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
```

In some cases you may need to change some of the Nginx configuration parameters. One example would be to support ASP.NET Core 2.2 applications using Azure AD Authentication. With default configuration you may get a "502 Bad Gateway" after authentication redirects back to the application.

To modify Nginx configuration parameteres:

Create a file named nginx-configmap.yaml with the content found in the first ConfigMap section https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml:
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

Add configuration values like this:
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  proxy_buffer_size: "16k"
```

Apply the configuration changes:
```
$ kubectl apply -f nginx-configmap.yaml
```

Check the assigned public IP address:

```
$ kubectl get service --namespace ingress-nginx
```

### Install the NGINX ingress controller using Helm

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
DNSNAME="<YourDNSName>"

# Get the resource-id of the public ip
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

# Update public ip address with DNS name
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME
```

If needed, run the following command to retrieve the FQDN. Update the IP address value with that of your ingress controller.
```
$ az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[dnsSettings.fqdn]" --output tsv
```

### Install cert-manager (loosely based upon the work of kube-lego)

Run the following script to install cert-manager:
```
kubectl label namespace kube-system certmanager.k8s.io/disable-validation=true

kubectl apply \
    -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/00-crds.yaml

helm install stable/cert-manager \
    --namespace kube-system \
    --set ingressShim.defaultIssuerName=letsencrypt-staging \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --version v0.6.0
```

### Create a CA cluster issuer

Before certificates can be issued, cert-manager requires an Issuer or ClusterIssuer resource. These Kubernetes resources are identical in functionality, however Issuer works in a single namespace, and ClusterIssuer works across all namespaces. For more information, see the cert-manager issuer documentation.
Create a cluster issuer, such as cluster-issuer.yaml, using the following example manifest.

Note! Update the email address with a valid address from your organization:
```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: <Your email address>
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}
```

Create CA Cluster Issues resource with the kubectl apply command:

```
$ kubectl apply -f cluster-issuer.yaml
```

### Create an ingress route (for Nginx)

This step assumes that you already have deployed some service (web application) to test with.

Create a test-ingress.yaml file with the following content:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    certmanager.k8s.io/cluster-issuer: letsencrypt-staging
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - <YourDNSName>.westeurope.cloudapp.azure.com
    secretName: tls-secret
  rules:
  - host: <YourDNSName>.westeurope.cloudapp.azure.com
    http:
      paths:
      - path: /
        backend:
          serviceName: <YourTestService>
          servicePort: 80
```

Create the ingress resource with the kubectl apply command:

```
$ kubectl apply -f test-ingress.yaml
```

Now you should be able to browse to https://<YourDNSName>.westeurope.cloudapp.azure.com

### Create an ingress route (for Træefik)

This step assumes that you already have deployed some service (web application) to test with.

Create a test-ingress.yaml file with the following content:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    certmanager.k8s.io/cluster-issuer: letsencrypt-staging
spec:
  tls:
  - hosts:
    - <YourDNSName>.westeurope.cloudapp.azure.com
    secretName: tls-secret
  rules:
  - host: <YourDNSName>.westeurope.cloudapp.azure.com
    http:
      paths:
      - path: /
        backend:
          serviceName: <YourTestService>
          servicePort: 80
```

Create the ingress resource with the kubectl apply command:

```
$ kubectl apply -f test-ingress.yaml
```

Now you should be able to browse to https://<YourDNSName>.westeurope.cloudapp.azure.com


To check the status of the generated certificate for TSL issue this command:
```
$ kubectl get certificate
```

To drill into details if there are issues:
```
kubectl describe certificate
```
