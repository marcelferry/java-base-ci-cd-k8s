DevFest Nantes 2017 presentation: [Pipeline de déploiement continu dans Kubernetes (PDF, French)](https://1drv.ms/b/s!Aq8u6oBZD1GUjvtjs-al76Od4VudqA).

# How to install Jenkins on Kubernetes on Azure with ACS
This document will walk you through the installation of an Azure Container Service cluster with Kubernetes as the orchestrator, and then the installation of Jenkins as a service in the cluster.

## Prerequisites
You will need a Bash terminal on Linux or macOS or Bash on [Windows Subsystem for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/about "Windows Subsystem for Linux Documentation").

Then you need an SSH Key. We assume you have one in $HOME/.ssh and we use the public key only: $HOME/.ssh/id_rsa.pub, so you will be able to login to the cluster using your key.

And if not already done, install [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest "Install Azure CLI 2.0"), login and select the subscription:
```
az login
az account set --subscription <subscription name>
```

## Initialize some variables
Some variables will simplify the following steps (values shown here are an example):
```
LOCATION=westeurope         # Azure region (full list: `az account list-locations`)
RG_NAME=demo-kube           # Resource group name
ACS_NAME=demo-cluster       # ACS cluster name
ACR_NAME=demoacr$RANDOM     # Azure Container Registry name (must be unique)
EMAIL=xxx@xxx.com           # Some components require a valid email
```

## Install and configure Kubernetes
### Install the cluster
The installation of the cluster itself is pretty straightforward: create a resource group and then the cluster in it.
```
# Install the Kubernetes cluster
az group create -n $RG_NAME -l $LOCATION
az acs create -n $ACS_NAME -g $RG_NAME -l $LOCATION --ssh-key-value $HOME/.ssh/id_rsa.pub --agent-count 2 -t Kubernetes
```

Get the master0 FQDN in a variable, so that we can SSH to it later:
```
MASTER0=$(az acs show -g $RG_NAME -n $ACS_NAME --query masterProfile.fqdn -o tsv)
```

Try the SSH connection:
```
ssh azureuser@$MASTER0
```
Exit the SSH session.

### Kubernetes CLI
You need to install the `kubectl` command on your system and to store the right credentials in the kubectl config file (in $HOME/.kube).

On Linux, Azure CLI provides for this nicely:
```
# Install Kubernetes client
sudo az acs kubernetes install-cli
```

On a Mac with Homebrew:
```
# Install Kubernetes client (Mac with Homebrew:)
brew install kubectl
```

More installation options [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

You can then get the credentials for your cluster in your `$HOME/.kube/config` file using this Azure CLI command:
```
# Configure Kubernetes client
az acs kubernetes get-credentials -n $ACS_NAME -g $RG_NAME
```
Note: if this fails, this may be due to a bug in Azure CLI 2.0.19 (specifically in acs 2.0.17). You may just have to get a newer version of Azure CLI.

You may test your Kubernetes CLI installation with these simple commands:
```
# Client config
kubectl config view
# Cluster info
kubectl cluster-info
kubectl get nodes
```

### Kubernetes console
You can access the Kubernetes web console this way:
- In a terminal, run `kubectl proxy` and keep it running.
- In a web browser, open http://127.0.0.1:8001/ui.
When you are done, just stop the proxy with Ctrl+C.

## Install Azure Container Registry

Install a new Azure Container Registry in the resource group to store the Docker images in a private registry close to the cluster.
```
az acr create -n $ACR_NAME -g $RG_NAME --sku Managed_Standard --admin-enabled true
```

Get the id, server and credentials for the registry in variables:
```
ACR_ID=$(az acr show -n $ACR_NAME --query id -o tsv)
ACR_LOGINSERVER=$(az acr show -n $ACR_NAME --query loginServer -o tsv)
ACR_USERNAME=$(az acr credential show -n $ACR_NAME --query username -o tsv)
ACR_PASSWORD=$(az acr credential show -n $ACR_NAME --query passwords[0].value -o tsv)
```

## Install Helm
[Helm](https://helm.sh/) is the package manager for Kubernetes. There is just an executable to store somewhere in your PATH.

On a Mac:
```
# install Helm (Mac with Homebrew):
brew install kubernetes-helm
```
On Linux:
```
# install helm (Linux)
wget https://kubernetes-helm.storage.googleapis.com/helm-v2.6.2-linux-amd64.tar.gz
tar -xzvf helm-v2.6.2-linux-amd64.tar.gz
mv linux-amd64/helm $HOME/bin/
```
More installation options [here](https://docs.helm.sh/using_helm/#installing-helm).

To configure Helm, you need to upgrade the server component (Tiller) and initialize the client:
```
ssh azureuser@$MASTER0 sudo sed -i s/'2.5.1'/'2.6.2'/g /etc/kubernetes/addons/kube-tiller-deployment.yaml
helm init --upgrade
helm list
```
The `helm list` command should not return anything as no helm package has been installed yet.

## Install Kube-lego chart
[kube-lego](https://github.com/jetstack/kube-lego) automatically requests certificates for Kubernetes Ingress resources from Let's Encrypt.
```
helm install stable/kube-lego --set config.LEGO_EMAIL=$EMAIL,config.LEGO_URL=https://acme-v01.api.letsencrypt.org/directory
```

## Install Nginx ingress chart
[nginx-ingress](https://github.com/kubernetes/ingress-nginx) is an Ingress controller that uses ConfigMap to store the nginx configuration.
```
helm install stable/nginx-ingress
```

Watch the `*-nginx-ingress-controller` service for its external IP to be available:
```
kubectl --namespace default get services -o wide
```
Once available, assign this IP address to a wildcard DNS name such as *.kube.yourdomain.com.

For instance, if the DNS zone for yourdomain.com contains this:
```
*.kube 10800 IN A <external IP address>
```
Any DNS request to anyname.kube.yourdomain.com will return the external IP address.

## Install Jenkins

It is time to install Jenkins on the cluster, using its Helm chart named `stable/jenkins`. You can find it using `helm search jenkins` or on the https://kubeapps.com website.

I used the `jenkins-values.yaml` file from https://github.com/lachie83/croc-hunter. Doing the same, you just have to replace the DNS domain name with yours. In the following command, replace `kube.yourdomain.com` with your own real domain:
```
# Install Jenkins
curl -O https://raw.githubusercontent.com/lachie83/croc-hunter/master/jenkins-values.yaml
sed -i s/'acs.az.estrado.io'/'kube.yourdomain.com'/g ./jenkins-values.yaml
helm --namespace jenkins --name jenkins -f ./jenkins-values.yaml install stable/jenkins
```
(replace acs.yourdomain.com with your actual domain)

## Docker Registry credentials

Add the Docker Registry (ACR) password in Jenkins Secrets.

In the Jenkins portal:
```
# Credentials
# ==> Global credentials (unrestricted)
# <adding some credentials>
#   Kind:        Username with password
#   Scope:       Global
#   Username:    copy and paste the value of $ACR_USERNAME
#   Password:    copy and paste the value of $ACR_PASSWORD
#   ID:          acr_creds
#   Description: ACR credentials
```

Add the Docker Registry password in the Kubernetes Secrets:
```
kubectl create secret docker-registry docker-secret --docker-server=$ACR_LOGINSERVER --docker-username=$ACR_USERNAME --docker-password="$ACR_PASSWORD" --docker-email=$EMAIL
```

_This is a work in progress. More on the full application workflow later._

Sources :
* https://gist.github.com/everett-toews/ed56adcfd525ce65b178d2e5a5eb06aa
* https://github.com/lachie83/croc-hunter/blob/master/DEMO.md
*
https://gist.github.com/pascals-msft/6b2af5f56dacbc01ffd14fd3ddd96025
*
https://docs.microsoft.com/pt-br/azure/aks/jenkins-continuous-deployment
