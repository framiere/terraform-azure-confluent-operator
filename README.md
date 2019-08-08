# Setup local tooling

```sh
brew install terraform azure-cli kubectl helm jq
```

# Azure 

```sh
az login
open https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade
export TF_VAR_client_id=2c652d75-e01c-4644-bf77-4887341b189c
export TF_VAR_client_secret=ii:-uCspOjYCM6=?gDpjcTSU1ynCp6C1

mkdir -p ./output
az ad sp create-for-rbac --name ServicePrincipalName > .output/create-for-rbac.json
appId=`cat output/create-for-rbac.json | jq -r '.["appId"]'`

az role assignment create --assignee $appId --role Reader > .output/assign-create-role-reader.json
az role assignment delete --assignee $appId --role Contributor
```sh

# terraform

```sh
terraform init
terraform plan -out .output/terraform.plan
terraform apply .output/terraform.plan
```

# Configure kubectl

```sh
echo "$(terraform output kube_config)" > ~/.azurek8s-$appId.json
export KUBECONFIG=~/.azurek8s-$appId.json
kubectl get nodes
```

# Download confluent operator 

```sh
mkdir confluent-operator
cd confluent-operator
wget https://platform-ops-bin.s3-us-west-1.amazonaws.com/operator/confluent-operator-20190726-v0.65.0.tar.gz
tar -xvf *
```

# Setup Kubernetes

```sh
echo Create an RBAC service account.
kubectl create serviceaccount tiller -n kube-system

echo Bind the cluster-admin role to the service account.
kubectl create clusterrolebinding tiller \
    --clusterrole=cluster-admin \
    --serviceaccount kube-system:tiller

helm init --service-account tiller
```

# Setup confluent operator

```sh
rm -f ./helm/providers/private.yaml
location=`az aks list | jq -r '.[]["location"]'`
echo your region is $location
echo """
global:
  provider:
    region: $location
""" > ./helm/providers/private.yaml
```

# Deploy the operator
cd helm

```sh
helm install \
    -f ./providers/azure.yaml \
    --name operator \
    --namespace operator \
    --set operator.enabled=true \
    ./confluent-operator    

kubectl -n operator patch serviceaccount default -p '{"imagePullSecrets": [{"name": "confluent-docker-registry" }]}'

echo let's verify operator and manager are up
kubectl get pods -n operator
```

# Install Zookepeer

```sh
echo "installing zookeeper"
helm install \
    -f ./providers/azure.yaml \
    --name zookeeper \
    --namespace operator \
    --set zookeeper.enabled=true \
    ./confluent-operator

echo """
        Zookeeper Cluster Deployment

Zookeeper cluster is deployed through CR.

  1. Validate if Zookeeper Custom Resource (CR) is created

     kubectl get zookeeper -n operator | grep zookeeper

  2. Check the status/events of CR: zookeeper

     kubectl describe zookeeper zookeeper -n operator

  3. Check if Zookeeper cluster is Ready

     kubectl get zookeeper zookeeper -ojson -n operator

     kubectl get zookeeper zookeeper -ojsonpath='{.status.phase}' -n operator

  4. Update/Upgrade Zookeeper Cluster

     The upgrade can be done either through the helm upgrade or by editing the CR directly as below;

     kubectl edit zookeeper zookeeper  -n operator
"""
```

# Install Kafka

```sh
helm install \
    -f ./providers/azure.yaml \
    --name kafka \
    --namespace operator \
    --set kafka.enabled=true \
    ./confluent-operator
```