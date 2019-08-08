brew install terraform azure-cli kubectl jq

az login

terraform init

# https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade
export TF_VAR_client_id=2c652d75-e01c-4644-bf77-4887341b189c
export TF_VAR_client_secret=ii:-uCspOjYCM6=?gDpjcTSU1ynCp6C1

mkdir -p output
az ad sp create-for-rbac --name ServicePrincipalName > output/create-for-rbac.json
appId=`cat output/create-for-rbac.json | jq -r '.["appId"]'`

az role assignment create --assignee $appId --role Reader > output/assign-create-role-reader.json
az role assignment delete --assignee $appId --role Contributor




echo "$(terraform output kube_config)" > ./azurek8s
export KUBECONFIG=./azurek8s
kubectl get nodes
