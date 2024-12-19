# https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver

# Create RG
az group create --name myResourceGroup --location eastus2

# Create cluster
# --enable-addons azure-keyvault-secrets-provider will create a SP in the infra RG + deploy the required PODS to the cluster.
# --enable-oidc-issuer =  Enables OIDC issuer which allows AAD to discover the API server's public signing keys.
# --enable-workload-identity = Allows AAD to trust ID tokens created by AKS which allows PODs to access Azure resources without the need for Application credentials.

az aks create --name myAKSCluster --resource-group myResourceGroup --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys --node-count 1

#######--enable-managed-identity \
    
# Connect to the cluster
az aks get-credentials --name myAKSCluster --resource-group myResourceGroup

# Check the required PODs are installed into the cluster
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)'

## Create a new Azure key vault
$kv = az keyvault create --name dx25vault --resource-group myResourceGroup --location eastus2 --enable-rbac-authorization

# Grant yourself permissions to add a secret to the KV
$userID = az ad signed-in-user show --query id -o tsv
az role assignment create --assignee $userID --role "Key Vault Administrator" --scope $($kv | convertfrom-json).id

# Create a sample secret
$secretObject = az keyvault secret set --vault-name dx25vault --name ExampleSecret --value MyAKSExampleSecret

##Take note of the following properties for future use:
# The name of the secret object in the key vault
$secretName = ($secretObject | convertfrom-json).name

# The object type (secret, key, or certificate)
$secretType = "secret"

# The name of your key vault resource
$kvID = ($kv | convertfrom-json).id

# The Azure tenant ID of the subscription
$tenantID = az account show --query tenantId --output tsv

# https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-identity-access

# There are 3 ways you can connect an AKS cluster to KV, 
# Service Connector with managed Identity (still in preview)
# The Service Connector in Azure is a managed service that simplifies the process of connecting Azure compute services to other backing services. It handles the configuration of network settings and connection information, such as generating environment variables, between compute services and target backing services

# *Workload ID

# *Managed identity

# From now one, we will use the new Service Connector method.
az provider register -n Microsoft.ServiceLinker
az provider register -n Microsoft.KubernetesConfiguration

# Create the service connection - --target-resource-group = <resource-group-of-key-vault>
az aks connection create keyvault --connection kv25ServiceConnection --resource-group myResourceGroup --name myAKSCluster --target-resource-group myResourceGroup --vault dx25vault --enable-csi --client-type none

# Original samples taken from https://github.com/Azure-Samples/serviceconnector-aks-samples.git in the "azure-keyvault-csi-provider" folder.


# Deploy the sample CSI configuration and POD app:
kubectl apply -f ./secret_provider_class.yaml
kubectl apply -f ./pod.yaml

# Test POD is running
kubectl get pod/sc-demo-keyvault-csi

# Check you can read the secrets
kubectl exec sc-demo-keyvault-csi -- cat /mnt/secrets-store/ExampleSecret
kubectl exec sc-demo-keyvault-csi -- env



#####################################
##  az keyvault purge --name dx25vault



