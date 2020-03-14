# mcw-aks
Microsoft Cloud-Native Workshop

[https://github.com/microsoft/MCW-Cloud-native-applications](https://github.com/microsoft/MCW-Cloud-native-applications)

[https://github.com/SpektraSystems/Microsoft-Cloud-Workshop/blob/master/cloud-native-applications/Hol%20step-by-step-developer-edition.md](https://github.com/SpektraSystems/Microsoft-Cloud-Workshop/blob/master/cloud-native-applications/Hol%20step-by-step-developer-edition.md)

## Pre-requisites

You need an Azure subscription. If you do not have one, to get started quickly go to [https://my.visualstudio.com](https://my.visualstudio.com).
You can also get free or charge subscription from [https://azure.microsoft.com/en-us/pricing/member-offers/credit-for-visual-studio-subscribers](https://azure.microsoft.com/en-us/pricing/member-offers/credit-for-visual-studio-subscribers), no Credit Card needed.

For MS employees, ask help from the proctor to create your own MS internal subscription. 
 
Check your subscription at : [https://account.azure.com/subscriptions](https://account.azure.com/subscriptions) 
then go to [https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade](https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade ) --> remove filter "Show only subscriptions selected in the global subscriptions filter" to see it.

#### Azure Cloud Shell

You can use the Azure Cloud Shell accessible at <https://shell.azure.com> once you login with an Azure subscription.

#### Uploading and editing files in Azure Cloud Shell

- You can use `vim <file you want to edit>` in Azure Cloud Shell to open the built-in text editor.
- You can upload files to the Azure Cloud Shell by dragging and dropping them
- You can also do a `curl -o filename.ext https://file-url/filename.ext` to download a file from the internet.

### Naming conventions
See also [See also https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/considerations/naming-and-tagging](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/considerations/naming-and-tagging)

### You can use any tool to run SSH & AZ CLI
```sh
# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest
# ex: Windows Subsystem for Linux (WSL), git bash for windows, Putty

##### git bash for windows based on Ubuntu
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
sudo apt-get update
sudo apt-get install ca-certificates curl apt-transport-https lsb-release gnupg

AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | 
    sudo tee /etc/apt/sources.list.d/azure-cli.list

sudo apt-get update
sudo apt-get install azure-cli

# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest#update
sudo apt-get update && sudo apt-get install --only-upgrade -y azure-cli

az login

```

### Tools

```sh
# https://kubernetes.io/docs/reference/kubectl/cheatsheet/
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc 
alias k=kubectl
complete -F __start_kubectl k
```

Optionnaly : If you want to run PowerShell
```sh
alias kn='kubectl config set-context --current --namespace '
#If you run kubectl in PowerShell ISE , you can also define aliases :
function k([Parameter(ValueFromRemainingArguments = $true)]$params) { & kubectl $params }
function kubectl([Parameter(ValueFromRemainingArguments = $true)]$params) { Write-Output "> kubectl $(@($params | ForEach-Object {$_}) -join ' ')"; & kubectl.exe $params; }
function k([Parameter(ValueFromRemainingArguments = $true)]$params) { Write-Output "> k $(@($params | ForEach-Object {$_}) -join ' ')"; & kubectl.exe $params; }
```

### Azure subscription

You can use the Azure Cloud Shell accessible [https://portal.azure.com](https://portal.azure.com) once you login with an Azure subscription. 
The Azure Cloud Shell has the Azure CLI pre-installed and configured to connect to your Azure subscription as well as `kubectl` and `helm`.
```sh
az --version
az account list 
az account show 
# az extension remove --name aks-preview
# az extension add --name aks-preview
```

Please use your username and password to login to <https://portal.azure.com>.

Also please authenticate your Azure CLI by running the command below on your machine and following the instructions.

```sh
# /!\ In CloudShell, the default subscription is not always the one you thought ...
subName="set here the name of your subscription"

subName=$(az account list --query "[?name=='${subName}'].{name:name}"  --output tsv)
echo "subscription Name :" $subName 
subId=$(az account list --query "[?name=='${subName}'].{id:id}"  --output tsv)
echo "subscription ID :" $subId

az account set --subscription $subId
az account show 
#az login
```

## Build pre-requisites 

### Set-up environment variables

<span style="color:red">/!\ IMPORTANT </span> : your **appName** & **cluster_name** values MUST BE UNIQUE
```sh

# az account list-locations : francecentral | northeurope | westeurope
location=francecentral 
echo "location is : " $location 

target_namespace="staging"
echo "Target namespace:" $target_namespace

appName="fabmedical" 
echo "appName is : " $appName 

cluster_name="aks-${appName}-${target_namespace}-101" #aks-<App Name>-<Environment>-<###>
echo "Cluster name:" $cluster_name

# target : version 1.15.7
version=$(az aks get-versions -l $location --query 'orchestrators[-4].orchestratorVersion' -o tsv) 
echo "version is :" $version 


```

Note: The here under variables are built based on the varibales defined above, you should not need to modify them, just run this snippet

```sh

# Storage account name must be between 3 and 24 characters in length and use numbers and lower-case letters only
storage_name="stfr""${appName,,}"
echo "Storage name:" $storage_name

git_url="https://github.com/microsoft/MCW-Cloud-native-applications.git"
echo "Project git repo URL : " $git_url 

rg_name="rg-${appName}-${location}" 
echo "RG name:" $rg_name 

acr_registry_name="acr-${appName,,}"
echo "ACR registry Name :" $acr_registry_name

ssh_passphrase="<your secret>"

```

### Create an SSH key

[https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-4-create-an-ssh-key](https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-4-create-an-ssh-key)

```sh
mkdir .ssh
ssh-keygen -t RSA -b 2048 -C admin@fabmedical -N $ssh_passphrase -f .ssh/fabmedical
cat .ssh/fabmedical.pub
```

### Create Service Principal
```sh
# https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal
# https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az-ad-sp-create-for-rbac
# https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest
# As of Azure CLI 2.0.68, the --password parameter to create a service principal with a user-defined password is no longer supported to prevent the accidental use of weak passwords.
sp_password=$(az ad sp create-for-rbac --name $appName --role contributor --query password --output tsv)
echo $sp_password > spp.txt
echo "Service Principal Password saved to ./spp.txt. IMPORTANT Keep your password ..." 
# sp_password=`cat spp.txt`
#sp_id=$(az ad sp list --all --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
sp_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{appId:appId}" --output tsv)
echo "Service Principal ID:" $sp_id 
echo $sp_id > spid.txt
# sp_id=`cat spid.txt`
az ad sp show --id $sp_id

sp_obj_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{objectId:objectId}" --output tsv)
echo "Service Principal Object ID:" $sp_obj_id
echo $sp_obj_id > sp_obj_id.txt
# sp_obj_id=`cat sp_obj_id.txt`

sp_tenant_id=$(az ad sp list --show-mine --query "[?appDisplayName=='${appName}'].{appOwnerTenantId:appOwnerTenantId}" --output tsv)
echo "Service Principal Tenant ID:" $sp_tenant_id
echo $sp_tenant_id > sp_tenant_id.txt
# sp_tenant_id=`cat sp_tenant_id.txt`


```
### Create RG
```sh
az group create --name $rg_name --location $location
```

### Download Starter Files
[https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-2-download-starter-files](https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-2-download-starter-files)

```sh
git clone https://github.com/microsoft/MCW-Cloud-native-applications.git
rm -rf MCW-Cloud-native-applications/.git


```

### Deploy ARM Template

[https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-4-create-an-ssh-key](https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-4-create-an-ssh-key)

```sh
cd MCW-Cloud-native-applications/Hands-on\ lab/arm/
code azuredeploy.parameters.json

az group deployment create --resource-group $rg_name --template-file azuredeploy.json --parameters azuredeploy.parameters.json

```

###  Setup Azure DevOps project

[https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-7-setup-azure-devops-project](https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-7-setup-azure-devops-project)

```sh
cd ~/MCW-Cloud-native-applications/Hands-on\ lab/lab-files/developer/
ll
```

```sh
cd ~/MCW-Cloud-native-applications/Hands-on\ lab/lab-files/infrastructure/
ll
```

### Azure devOps

[https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-7-setup-azure-devops-project](https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-7-setup-azure-devops-project)


Open a new browser tab to visit [Azure DevOps](https://dev.azure.com) and log into your account.

```sh
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git config --global credential.helper cache

cd content-web
git init
git add .
git commit -m "Initial Commit"

git remote add origin https://ezyakaeagle442@dev.azure.com/ezyakaeagle442/fabmedical/_git/content-web
git push -u origin --all

cd ../content-api
git init
git add .
git commit -m "Initial Commit"
git remote add origin https://ezyakaeagle442@dev.azure.com/ezyakaeagle442/fabmedical/_git/content-api
git push -u origin --all

cd ../content-init
git init
git add .
git commit -m "Initial Commit"
git remote add origin https://ezyakaeagle442@dev.azure.com/ezyakaeagle442/fabmedical/_git/content-init
git push -u origin --all


```

### Connect securely to the build agent

[https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-8-connect-securely-to-the-build-agent](https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/Before%20the%20HOL%20-%20Cloud-native%20applications.md#task-8-connect-securely-to-the-build-agent)

```sh
az vm show -d -g $rg_name -n fabmedical-PIN --query publicIps -o tsv
# BUILDAGENTIP=40.89.181.14
# BUILDAGENTUSERNAME=adminfabmedical
# PRIVATEKEYNAME="~/.ssh/fabmedical"
chmod go-r ~/.ssh/fabmedical*
ssh -i $PRIVATEKEYNAME $BUILDAGENTUSERNAME@$BUILDAGENTIP

sudo apt-get update && sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get install curl python-software-properties
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get update && sudo apt-get install -y docker-ce nodejs mongodb-clients

sudo apt-get upgrade

sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

docker version
nodejs --version
npm -version

sudo npm install -g @angular/cli
sudo usermod -aG docker $USER

# For the user permission changes to take effect, exit the SSH session by typing 'exit', then press <Enter>. Reconnect to the build agent VM using SSH as you did in the previous task.

sudo git config --global user.email "you@example.com"
sudo git config --global user.name "Your Name"

sudo chown -R $USER:$(id -gn $USER) /home/adminfabmedical/.config
git config --global credential.helper cache
sudo chown -R $USER:$(id -gn $USER) /home/adminfabmedical/.config

git clone https://ezyakaeagle442@dev.azure.com/ezyakaeagle442/fabmedical/_git/content-web
git clone https://ezyakaeagle442@dev.azure.com/ezyakaeagle442/fabmedical/_git/content-api
git clone https://ezyakaeagle442@dev.azure.com/ezyakaeagle442/fabmedical/_git/content-init

```

### task 5

```sh
#https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Cloud-native%20applications%20-%20Developer%20edition.md#task-5-run-a-containerized-application

docker container run --name api --net fabmedical -p 3001:3001 content-api
docker container rm api
docker container run --name api --net fabmedical -p 3001:3001 -e MONGODB_CONNECTION=mongodb://mongo:27017/contentdb -d content-api
docker container ls

docker container run --name web --net fabmedical -P -d content-web
docker container ls
curl http://localhost:32768/speakers.html

docker container stop web
docker container rm web
docker container ls -a
cat app.js

vim Dockerfile
docker image build -t content-web .
docker container run --name web --net fabmedical -P -d -e CONTENT_API_URL=http://api:3001 content-web
docker container ls
curl http://localhost:32769/speakers.html

docker container stop web
docker container rm web
docker container run --name web --net fabmedical -p 3000:3000 -d -e CONTENT_API_URL=http://api:3001 content-web

curl http://localhost:3000/speakers.html

git add .
git commit -m "Setup Environment Variables"
git push
```

### task 7 

```sh

#https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Cloud-native%20applications%20-%20Developer%20edition.md#task-7-run-several-containers-with-docker-compose

docker container stop web && docker container rm web
docker container stop api && docker container rm api
docker container stop mongo && docker container rm mongo

cd ~
vim docker-compose.yml
<i>
docker-compose -f docker-compose.yml -p fabmedical up -d
mkdir data
vim docker-compose.yml
vim docker-compose.init.yml
docker-compose -f docker-compose.yml -p fabmedical down
docker-compose -f docker-compose.yml -f docker-compose.init.yml -p fabmedical up -d
ls ./data/

```

### Task 8: Push images to Azure Container Registry

```sh

# https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Cloud-native%20applications%20-%20Developer%20edition.md#task-8-push-images-to-azure-container-registry

docker login fabmedicalPIN.azurecr.io -u fabmedicalPIN -p <Your secret Pass>

docker image tag content-web fabmedicalPIN.azurecr.io/content-web
docker image tag content-api fabmedicalPIN.azurecr.io/content-api
docker image push fabmedicalPIN.azurecr.io/content-web
docker image push fabmedicalPIN.azurecr.io/content-api

docker image tag fabmedicalPIN.azurecr.io/content-web:latest fabmedicalPIN.azurecr.io/content-web:v1
docker image tag fabmedicalPIN.azurecr.io/content-api:latest fabmedicalPIN.azurecr.io/content-api:v1
docker image ls
docker image push fabmedicalPIN.azurecr.io/content-web
docker image push fabmedicalPIN.azurecr.io/content-api
docker image pull fabmedicalPIN.azurecr.io/content-web
docker image pull fabmedicalPIN.azurecr.io/content-web:v1

```

### Task 9: Setup CI Pipeline to Push Images

```sh
# https://github.com/microsoft/MCW-Cloud-native-applications/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Cloud-native%20applications%20-%20Developer%20edition.md#task-9-setup-ci-pipeline-to-push-images

cd ~/content-web
vim azure-pipelines.yml

git add azure-pipelines.yml
git commit -m "Added pipeline YAML"
git push

```

Exercise 2: Deploy the solution to Azure Kubernetes Service
### Task 1: Tunnel into the Azure Kubernetes Service cluster

### Connect to AKS Cluster

```sh

az aks get-credentials --resource-group $rg_name  --name fabmedical-pin
az aks install-cli
kubectl cluster-info
#kubectl config view

# Get the id of the service principal configured for AKS
CLIENT_ID=$(az aks show --resource-group $rg_name --name $cluster_name --query "servicePrincipalProfile.clientId" --output tsv)
echo "CLIENT_ID:" $CLIENT_ID 
```

### Create Namespaces
```sh
kubectl create namespace development
kubectl label namespace/development purpose=development

kubectl create namespace staging
kubectl label namespace/staging purpose=staging

kubectl create namespace production
kubectl label namespace/production purpose=production
```

### Optionnal Play: what resources are in your cluster

```sh
kubectl get nodes
kubectl describe namespace production

# https://docs.microsoft.com/en-us/azure/aks/availability-zones#verify-node-distribution-across-zones
kubectl describe nodes | grep -e "Name:" -e "failure-domain.beta.kubernetes.io/zone"

kubectl get pods
kubectl top node
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false

kubectl get roles --all-namespaces
kubectl get serviceaccounts --all-namespaces
kubectl get rolebindings --all-namespaces
kubectl get ingresses  --all-namespaces
```

### Play with Kubernetes dashboard
```sh
# https://docs.microsoft.com/en-us/azure/aks/kubernetes-dashboard
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
az extension add --name aks-preview
az aks browse --resource-group $rg_name --name $cluster_name

```

### HELM Setup


Helm version 3 does not come with any repositories predefined, so you’ll need [initialize the stable chart repository](https://v3.helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository)

```sh
kubectl create namespace ingress

# # https://helm.sh/docs/intro/install/
wget https://get.helm.sh/helm-v3.1.1-linux-amd64.tar.gz
tar -zxvf helm-v3.1.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm

helm version
helm get -h
# https://helm.sh/docs/intro/using_helm/
# You can see which repositories are configured using helm repo list
helm repo list

# Init default repo: https://hub.helm.sh/charts | https://kubernetes-charts.storage.googleapis.com/
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm search repo
helm search hub
helm search repo mongodb

helm repo update
# https://www.nginx.com/products/nginx/kubernetes-ingress-controller
helm install ingress stable/nginx-ingress --namespace ingress
helm upgrade --install ingress stable/nginx-ingress --namespace ingress
ing_ctl_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
```

### Task 4: Deploy a service using a Helm chart



### Optionnal: Monitor
```sh
az aks enable-addons --resource-group $rg_name --name $cluster_name --addons monitoring
```

### Download files your local machine to be able to drag&drop them to CloudShell window

[]()

Drag&drop the above files to CloudShell and check the files location in CloudShell
```sh
pwd
ls -al *.yaml
ls -al *.json
```

### Optionnal Play: Create Analytics Workspace


```sh
# https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli
# /!\ ATTENTION : check & modify location in the JSON template from https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli#create-and-deploy-template

# You can use `VIM <file you want to edit>` in Azure Cloud Shell to open the built-in text editor.
# You can upload files to the Azure Cloud Shell by dragging and dropping them
# You can also do a `curl -o filename.ext https://file-url/filename.ext` to download a file from the internet.

az group deployment create --resource-group $rg_name --template-file $analytics_workspace_template --name $analytics_workspace_name --parameters=workspaceName=$analytics_workspace_name

```

### Create Azure Container Registry
Note: Premium sku is a requirement to enable replication

```sh
az acr create --resource-group $rg_name --name $acr_registry_name --sku standard --location $location
# az acr create --resource-group $rg_name --name $acr_registry_name --sku Premium --location $location

# Get the ACR registry resource id
acr_registry_id=$(az acr show --name $acr_registry_name --resource-group $rg_name --query "id" --output tsv)
echo "ACR registry ID :" $acr_registry_id
az acr repository list  --name $acr_registry_name # --resource-group $rg_name
az acr check-health --yes -n $acr_registry_name 

# Create role assignment
az role assignment create --assignee $sp_id --role acrpull --scope $acr_registry_id

docker_server="$(az acr show --name $acr_registry_name --resource-group $rg_name --query "name" --output tsv)"".azurecr.io"
echo "Docker server :" $docker_server

kubectl -n $target_namespace create secret docker-registry acr-auth \
        --docker-server="$docker_server" \
        --docker-username="$sp_id" \
        --docker-email="youremail@groland.grd" \
        --docker-password="$sp_password"

kubectl get secrets -n $target_namespace
```

```sh
# check eventual errors:
k get events -n $target_namespace | grep -i "Error"

```
If there were no errors, you can skip the snippet below. In case of error, use the snippet below to found the error
```sh
for pod in $(k get po -n $target_namespace -o=name)
do
	k describe $pod | grep -i "Error"
	k logs $pod | grep -i "Error"
    k exec $pod -n $target_namespace -- wget http://localhost:8080/manage/health
    k exec $pod -n $target_namespace -- wget http://localhost:8080/manage/info
    # k exec $pod -n $target_namespace -it -- /bin/sh
done

# kubectl describe pod petclinic-YOUR-POD-ID -n $target_namespace
# kubectl logs petclinic-YOUR-POD-ID -n $target_namespace
# kubectl  exec -it "POD-UID" -n $target_namespace -- /bin/sh
```

### Configure DNS

Free DNS: http://xip.io , https://nip.io, https://freedns.afraid.org, https://dyn.com/dns
```sh
$custom_dns
#service_ip=$(kubectl get service petclinic-lb-service -n $target_namespace -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
service_ip=$(kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
echo $service_ip

# https://docs.microsoft.com/en-us/cli/azure/network/public-ip?view=azure-cli-latest
# Get the resource-id of the public ip
managed_rg="mc_$rg_name"_"$cluster_name""_""$location"
echo $managed_rg

public_ip_id=$(az network public-ip list --subscription $subId --resource-group $managed_rg --query "[?ipAddress!=null]|[?contains(ipAddress, '$service_ip')].[id]" --output tsv)
echo $public_ip_id

In the Azure portal, go to All services / Public IP addresses / kubernetes-xxxx - Configuration ( the Ingress Controller IP) , then there is a field "DNS name label (optional)" ==> An "A record" that starts with the specified label and resolves to this public IP address will be registered with the Azure-provided DNS servers. Example: mylabel.westus.cloudapp.azure.com.

az network public-ip show --ids $public_ip_id --subscription $subId --resource-group $managed_rg

az network public-ip update --ids $public_ip_id --dns-name $dnz_zone --subscription $subId --resource-group $managed_rg

#http://petclinic.francecentral.cloudapp.azure.com
#http://petclinic.internal.cloudapp.net

#http://petclinic.kissmyapp.francecentral.cloudapp.azure.com/ 
dnz_zone="kissmyapp.francecentral.cloudapp.azure.com" 
az network dns zone create -g $rg_name -n $dnz_zone
az network dns zone list -g $rg_name
az network dns record-set a add-record -g $rg_name -z $dnz_zone -n www -a ${service_ip}
az network dns record-set list -g $rg_name -z $dnz_zone

az network dns record-set cname create -g $rg_name -z $dnz_zone -n petclinic-ingress
az network dns record-set cname set-record -g $rg_name -z $dnz_zone -n petclinic-ingress -c www.$dnz_zone
az network dns record-set cname show -g $rg_name -z $dnz_zone -n petclinic-ingress
http://petclinic-ingress.kissmyapp.francecentral.cloudapp.azure.com/ 

```

###  Clean-Up
```sh
az aks delete --name $cluster_name --resource-group $rg_name
az acr delete --resource-group $rg_name --name $acr_registry_name
az keyvault delete --location $location --name $vault_name --resource-group $rg_name
az network vnet delete --name $vnet_name --resource-group $rg_name --location $location
az network vnet subnet delete --name $subnet_name --vnet-name $vnet_name --resource-group $rg_name 
az network route-table route delete -g  $rg_name --route-table-name $route_table -n $route
az network route-table delete -g  $rg_name -n $route_table
az network dns record-set a delete -g $rg_name -z $dnz_zone -n www 
az network dns zone delete -g $rg_name -n $dnz_zone
az network dns zone list -g $rg_name
az group delete --name $rg_name
```

<span style="color:red">**/!\ IMPORTANT** </span> : Decide to keep or delete your service principal
```sh
az ad sp delete --id $sp_id
rm spp.txt
rm spid.txt
```