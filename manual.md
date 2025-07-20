# DevOps+SRE Challenge 1 - Manual CLI Implementation

## Prerequisites
- Azure CLI installed and configured
- Appropriate Azure subscription permissions
- PowerShell or Bash terminal

## Phase 1: Initial Setup and Variables

### 1.1 Set Variables
```powershell
# Core Variables
$resourceGroupName = "rg-network-dev"
$location = "eastus"
$subscription = ""  # Replace with actual subscription ID

# Network Variables
$vnetName = "vnet-app-dev"
$vnetAddressPrefix = "10.0.0.0/16"
$subnetAppName = "snet-app"
$subnetAppPrefix = "10.0.1.0/24"
$subnetBastionName = "AzureBastionSubnet"
$subnetBastionPrefix = "10.0.254.0/27"

# AKS Variables
$aksName = "aks-app-dev"
$nodeVmSize = "Standard_B2s"
$nodeCount = 3
$serviceCidr = "10.240.0.0/16"
$dnsServiceIp = "10.240.0.10"
$dockerBridgeCidr = "172.17.0.1/16"

# Jumpbox Variables
$vmName = "vm-jumpbox"
$vmUser = "azureuser"
$vmPassword = "YourSecureP@ssword123!"  # Use a strong password

# Bastion Variables
$bastionIpName = "pip-bastion"
$bastionName = "bastion-host"

# Key Vault Variables
$keyVaultName = "kv-aks-dev-$((Get-Random -Maximum 9999))"

# Container Registry Variables
$acrName = "acraksdev$((Get-Random -Maximum 9999))"

# Tags
$tags = @{
    environment = "dev"
    project = "aks-sre"
    owner = "swayam"
    challenge = "challenge1"
}
```

### 1.2 Login and Set Subscription
```powershell
az login
az account set --subscription $subscription
az account show
```

## Phase 2: Core Infrastructure

### 2.1 Create Resource Group
```powershell
az group create `
  --name $resourceGroupName `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1
```

### 2.2 Create Virtual Network and Application Subnet
```powershell
az network vnet create `
  --name $vnetName `
  --resource-group $resourceGroupName `
  --location $location `
  --address-prefix $vnetAddressPrefix `
  --subnet-name $subnetAppName `
  --subnet-prefix $subnetAppPrefix `
  --tags environment=dev project=aks-sre owner=swayam
```

### 2.3 Create Azure Bastion Subnet
```powershell
az network vnet subnet create `
  --resource-group $resourceGroupName `
  --vnet-name $vnetName `
  --name $subnetBastionName `
  --address-prefix $subnetBastionPrefix
```

### 2.4 Get Subnet ID for AKS
```powershell
$subnetId = az network vnet subnet show `
  --resource-group $resourceGroupName `
  --vnet-name $vnetName `
  --name $subnetAppName `
  --query id -o tsv

Write-Host "Subnet ID: $subnetId"
```

## Phase 3: Container Registry

### 3.1 Create Azure Container Registry
```powershell
az acr create `
  --resource-group $resourceGroupName `
  --name $acrName `
  --sku Basic `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam
```

## Phase 4: Private AKS Cluster Deployment

### 4.1 Create Private AKS Cluster with Managed Identity
```powershell
az aks create `
  --resource-group $resourceGroupName `
  --name $aksName `
  --enable-private-cluster `
  --network-plugin azure `
  --network-plugin-mode overlay `
  --vnet-subnet-id $subnetId `
  --node-vm-size $nodeVmSize `
  --node-count $nodeCount `
  --service-cidr $serviceCidr `
  --dns-service-ip $dnsServiceIp `
  --enable-managed-identity `
  --enable-workload-identity `
  --enable-oidc-issuer `
  --attach-acr $acrName `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 `
  --generate-ssh-keys `
  --yes
```

### 4.2 Create Private DNS Zone for AKS
```powershell
az network private-dns zone create `
  --resource-group $resourceGroupName `
  --name "privatelink.eastus.azmk8s.io"

az network private-dns link vnet create `
  --resource-group $resourceGroupName `
  --zone-name "privatelink.eastus.azmk8s.io" `
  --name "aks-dnslink" `
  --virtual-network $vnetName `
  --registration-enabled false
```

### 4.3 Get AKS Private Endpoint IP and Add DNS Record
```powershell
# Get the AKS private FQDN
$aksFqdn = az aks show --resource-group $resourceGroupName --name $aksName --query privateFqdn -o tsv
Write-Host "AKS Private FQDN: $aksFqdn"

# Get the private endpoint IP (you may need to check the private endpoint in the MC_ resource group)
$mcResourceGroup = az aks show --resource-group $resourceGroupName --name $aksName --query nodeResourceGroup -o tsv
Write-Host "MC Resource Group: $mcResourceGroup"

# List private endpoints to find the IP
az network private-endpoint list --resource-group $mcResourceGroup --output table
```

## Phase 5: Key Vault with Private Endpoints

### 5.1 Create Key Vault
```powershell
az keyvault create `
  --name $keyVaultName `
  --resource-group $resourceGroupName `
  --location $location `
  --enable-rbac-authorization `
  --tags environment=dev project=aks-sre owner=swayam
```

### 5.2 Disable Public Network Access
```powershell
az keyvault update `
  --name $keyVaultName `
  --resource-group $resourceGroupName `
  --public-network-access Disabled
```

### 5.3 Create Private Endpoint for Key Vault
```powershell
az network private-endpoint create `
  --name pe-keyvault `
  --resource-group $resourceGroupName `
  --vnet-name $vnetName `
  --subnet $subnetAppName `
  --private-connection-resource-id $(az keyvault show --name $keyVaultName --resource-group $resourceGroupName --query id -o tsv) `
  --group-ids vault `
  --connection-name keyvault-connection `
  --location $location
```

### 5.4 Create Private DNS Zone for Key Vault
```powershell
az network private-dns zone create `
  --resource-group $resourceGroupName `
  --name "privatelink.vaultcore.azure.net"

az network private-dns link vnet create `
  --resource-group $resourceGroupName `
  --zone-name "privatelink.vaultcore.azure.net" `
  --name keyvault-dnslink `
  --virtual-network $vnetName `
  --registration-enabled false
```

## Phase 6: Jumpbox VM Deployment

### 6.1 Deploy Jumpbox VM (No Public IP)
```powershell
az vm create `
  --resource-group $resourceGroupName `
  --name $vmName `
  --image Ubuntu2204 `
  --admin-username $vmUser `
  --admin-password $vmPassword `
  --authentication-type password `
  --vnet-name $vnetName `
  --subnet $subnetAppName `
  --size Standard_B2s `
  --public-ip-address '""' `
  --nsg '""' `
  --tags environment=dev project=aks-sre owner=swayam
```

## Phase 7: Azure Bastion Deployment

### 7.1 Create Public IP for Bastion
```powershell
az network public-ip create `
  --resource-group $resourceGroupName `
  --name $bastionIpName `
  --sku Standard `
  --location $location `
  --allocation-method Static `
  --tags environment=dev project=aks-sre owner=swayam
```

### 7.2 Deploy Azure Bastion
```powershell
az network bastion create `
  --name $bastionName `
  --resource-group $resourceGroupName `
  --location $location `
  --vnet-name $vnetName `
  --public-ip-address $bastionIpName `
  --tags environment=dev project=aks-sre owner=swayam
```

## Phase 8: RBAC and Access Configuration

### 8.1 Get Current User Object ID
```powershell
$currentUserObjectId = az ad signed-in-user show --query id -o tsv
Write-Host "Current User Object ID: $currentUserObjectId"
```

### 8.2 Assign Key Vault Roles
```powershell
# Key Vault Administrator role
az role assignment create `
  --assignee $currentUserObjectId `
  --role "Key Vault Administrator" `
  --scope $(az keyvault show --name $keyVaultName --resource-group $resourceGroupName --query id -o tsv)

# AKS Cluster Admin role
az role assignment create `
  --assignee $currentUserObjectId `
  --role "Azure Kubernetes Service Cluster Admin Role" `
  --scope $(az aks show --name $aksName --resource-group $resourceGroupName --query id -o tsv)
```

### 8.3 Get AKS Managed Identity and Assign ACR Role
```powershell
$aksManagedIdentity = az aks show --name $aksName --resource-group $resourceGroupName --query identityProfile.kubeletidentity.clientId -o tsv

az role assignment create `
  --assignee $aksManagedIdentity `
  --role "AcrPull" `
  --scope $(az acr show --name $acrName --resource-group $resourceGroupName --query id -o tsv)
```

## Phase 9: Post-Deployment Configuration

### 9.1 Install Azure CLI Extension for Bastion
```powershell
az extension add --name bastion
```

### 9.2 Connect to Jumpbox via Bastion (Test Connection)
```powershell
# Test connection to jumpbox
az network bastion ssh `
  --name $bastionName `
  --resource-group $resourceGroupName `
  --target-resource-id $(az vm show --resource-group $resourceGroupName --name $vmName --query id -o tsv) `
  --auth-type password `
  --username $vmUser
```

### 9.3 Install Tools on Jumpbox (Run these commands after connecting via Bastion)
```bash
# Update system
sudo apt-get update

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Docker (optional)
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER

# Verify installations
az --version
kubectl version --client
```

### 9.4 Configure kubectl on Jumpbox
```bash
# Login to Azure
az login

# Get AKS credentials
az aks get-credentials --resource-group rg-network-dev --name aks-app-dev

# Test kubectl access
kubectl get nodes
kubectl get pods --all-namespaces
```

### 9.5 Store AKS Configuration in Key Vault
```powershell
# Get AKS credentials locally first
az aks get-credentials --name $aksName --resource-group $resourceGroupName --file kubeconfig-temp

# Store in Key Vault (run this from a machine that can access Key Vault privately or temporarily enable public access)
az keyvault secret set `
  --vault-name $keyVaultName `
  --name "aks-kubeconfig" `
  --file kubeconfig-temp

# Clean up temporary file
Remove-Item kubeconfig-temp
```

## Phase 10: Cloud Policies and Compliance

### 10.1 Apply Tag Policy
```powershell
# Create policy assignment for required tags
az policy assignment create `
  --name "require-project-tag" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "/providers/Microsoft.Authorization/policyDefinitions/1e30110a-5ceb-460c-a204-c1c3969c6d62" `
  --params '{"tagName": {"value": "project"}}'
```

### 10.2 Apply Location Restriction Policy
```powershell
az policy assignment create `
  --name "allowed-locations" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c" `
  --params '{"listOfAllowedLocations": {"value": ["eastus", "eastus2"]}}'
```

## Phase 11: Validation and Testing

### 11.1 Validate Resources
```powershell
# List all resources in the resource group
az resource list --resource-group $resourceGroupName --output table

# Check AKS cluster status
az aks show --name $aksName --resource-group $resourceGroupName --query provisioningState -o tsv

# Check Key Vault status
az keyvault show --name $keyVaultName --resource-group $resourceGroupName --query properties.provisioningState -o tsv

# Check Bastion status
az network bastion show --name $bastionName --resource-group $resourceGroupName --query provisioningState -o tsv
```

### 11.2 Test Private DNS Resolution (from Jumpbox)
```bash
# Test AKS private endpoint resolution
nslookup your-aks-cluster-fqdn.privatelink.eastus.azmk8s.io

# Test Key Vault private endpoint resolution
nslookup your-keyvault.privatelink.vaultcore.azure.net
```

## Summary

You have successfully deployed:

✅ **Virtual Network & Subnets**: VNet with app subnet and Bastion subnet  
✅ **Private AKS Cluster**: With managed identity, workload identity, and private endpoints  
✅ **Azure Container Registry**: Integrated with AKS for container image storage  
✅ **Key Vault**: With private endpoints for secure secret storage  
✅ **Azure Bastion**: For secure access to private resources  
✅ **Jumpbox VM**: Private VM for managing AKS cluster  
✅ **RBAC Configuration**: Proper role assignments for security  
✅ **Private DNS Zones**: For private endpoint resolution  
✅ **Cloud Policies**: Tag enforcement and location restrictions  

## Next Steps for Production

1. **High Availability**: Deploy AKS across multiple availability zones
2. **Backup Strategy**: Implement backup for AKS and Key Vault
3. **Monitoring**: Set up Azure Monitor and Log Analytics
4. **Security**: Implement network security groups and Azure Firewall
5. **CI/CD**: Set up GitHub Actions or Azure DevOps pipelines
6. **Workload Identity**: Configure specific workload identities for applications