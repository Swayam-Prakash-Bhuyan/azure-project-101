# DevOps+SRE Challenge 1 - Complete Manual CLI Implementation

## Prerequisites
- Azure CLI installed and configured (v2.56.0 or later)
- Appropriate Azure subscription permissions (Owner or Contributor + User Access Administrator)
- PowerShell 7.0+ or Bash terminal
- Owner permissions on the subscription for policy assignments

## Phase 1: Initial Setup and Variables

### 1.1 Set Variables
```powershell
# Core Variables
$resourceGroupName = "rg-network-dev"
$location = "eastus"
$subscription = "dummy-sub"  # Replace with actual subscription ID

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

# DNS Variables
$aksPrivateDnsZone = "privatelink.$location.azmk8s.io"
$acrPrivateDnsZone = "privatelink.azurecr.io"
$kvPrivateDnsZone = "privatelink.vaultcore.azure.net"

# Standard Tags for all resources
$standardTags = @{
    environment = "dev"
    project = "aks-sre"
    owner = "swayam"
    challenge = "challenge1"
    costcenter = "engineering"
    department = "devops"
    region = $location
    createddate = (Get-Date -Format "yyyy-MM-dd")
}
```

### 1.2 Login and Set Subscription
```powershell
# Login and set subscription
az login
az account set --subscription $subscription
az account show

# Verify permissions
$currentUser = az ad signed-in-user show --query id -o tsv
Write-Host "Current User Object ID: $currentUser"
```

## Phase 2: Core Infrastructure

### 2.1 Create Resource Group with Proper Tags
```powershell
az group create `
  --name $resourceGroupName `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops region=$location createddate=(Get-Date -Format "yyyy-MM-dd")
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
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops region=$location
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

## Phase 3: Private DNS Zones (Create Before Resources)

### 3.1 Create All Private DNS Zones
```powershell
# AKS Private DNS Zone
az network private-dns zone create `
  --resource-group $resourceGroupName `
  --name $aksPrivateDnsZone `
  --tags environment=dev project=aks-sre owner=swayam

# ACR Private DNS Zone
az network private-dns zone create `
  --resource-group $resourceGroupName `
  --name $acrPrivateDnsZone `
  --tags environment=dev project=aks-sre owner=swayam

# Key Vault Private DNS Zone
az network private-dns zone create `
  --resource-group $resourceGroupName `
  --name $kvPrivateDnsZone `
  --tags environment=dev project=aks-sre owner=swayam
```

### 3.2 Link Private DNS Zones to VNet
```powershell
# AKS DNS Zone Link
az network private-dns link vnet create `
  --resource-group $resourceGroupName `
  --zone-name $aksPrivateDnsZone `
  --name "aks-dnslink" `
  --virtual-network $vnetName `
  --registration-enabled false `
  --tags environment=dev project=aks-sre owner=swayam

# ACR DNS Zone Link
az network private-dns link vnet create `
  --resource-group $resourceGroupName `
  --zone-name $acrPrivateDnsZone `
  --name "acr-dnslink" `
  --virtual-network $vnetName `
  --registration-enabled false `
  --tags environment=dev project=aks-sre owner=swayam

# Key Vault DNS Zone Link
az network private-dns link vnet create `
  --resource-group $resourceGroupName `
  --zone-name $kvPrivateDnsZone `
  --name "keyvault-dnslink" `
  --virtual-network $vnetName `
  --registration-enabled false `
  --tags environment=dev project=aks-sre owner=swayam
```

## Phase 4: Private Container Registry

### 4.1 Create Azure Container Registry (Premium SKU for Private Endpoints)
```powershell
az acr create `
  --resource-group $resourceGroupName `
  --name $acrName `
  --sku Premium `
  --location $location `
  --public-network-enabled false `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops region=$location
```

### 4.2 Create Private Endpoint for ACR
```powershell
az network private-endpoint create `
  --name pe-acr `
  --resource-group $resourceGroupName `
  --vnet-name $vnetName `
  --subnet $subnetAppName `
  --private-connection-resource-id $(az acr show --name $acrName --resource-group $resourceGroupName --query id -o tsv) `
  --group-ids registry `
  --connection-name acr-connection `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam
```

### 4.3 Configure Private DNS Zone Group for ACR
```powershell
az network private-endpoint dns-zone-group create `
  --resource-group $resourceGroupName `
  --endpoint-name pe-acr `
  --name acr-dns-zone-group `
  --private-dns-zone $acrPrivateDnsZone `
  --zone-name acr
```

## Phase 5: Private AKS Cluster Deployment

### 5.1 Create Private AKS Cluster with Custom DNS Zone
```powershell
# First, get the private DNS zone resource ID
$aksPrivateDnsZoneId = az network private-dns zone show `
  --resource-group $resourceGroupName `
  --name $aksPrivateDnsZone `
  --query id -o tsv

# 1. Create user-assigned identity
$identityName = "aks-private-dns-identity"
az identity create --resource-group $resourceGroupName --name $identityName
$identityId = az identity show -g $resourceGroupName -n $identityName --query id -o tsv

# 2. Assign DNS permissions to identity
az role assignment create `
  --assignee $(az identity show -g $resourceGroupName -n $identityName --query principalId -o tsv) `
  --role "Private DNS Zone Contributor" `
  --scope $aksPrivateDnsZoneId

# Create AKS cluster with private DNS zone configuration
az aks create --resource-group $resourceGroupName --name $aksName --enable-private-cluster --private-dns-zone $aksPrivateDnsZoneId --assign-identity $identityId --enable-aad --enable-azure-rbac --network-plugin azure --network-plugin-mode overlay --vnet-subnet-id $subnetId --node-vm-size $nodeVmSize --node-count $nodeCount --service-cidr $serviceCidr --dns-service-ip $dnsServiceIp --enable-workload-identity --enable-oidc-issuer --attach-acr $acrName --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops region=$location --generate-ssh-keys  --yes
```

### 5.2 Enable Azure Policy Add-on for AKS
```powershell
az aks enable-addons `
  --resource-group $resourceGroupName `
  --name $aksName `
  --addons azure-policy
```

## Phase 6: Key Vault with Private Endpoints

### 6.1 Create Key Vault
```powershell
az keyvault create `
  --name $keyVaultName `
  --resource-group $resourceGroupName `
  --location $location `
  --enable-rbac-authorization `
  --enable-purge-protection `
  --public-network-access Disabled `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops region=$location
```

### 6.2 Create Private Endpoint for Key Vault
```powershell
az network private-endpoint create `
  --name pe-keyvault `
  --resource-group $resourceGroupName `
  --vnet-name $vnetName `
  --subnet $subnetAppName `
  --private-connection-resource-id $(az keyvault show --name $keyVaultName --resource-group $resourceGroupName --query id -o tsv) `
  --group-ids vault `
  --connection-name keyvault-connection `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam
```

### 6.3 Configure Private DNS Zone Group for Key Vault
```powershell
az network private-endpoint dns-zone-group create `
  --resource-group $resourceGroupName `
  --endpoint-name pe-keyvault `
  --name keyvault-dns-zone-group `
  --private-dns-zone $kvPrivateDnsZone `
  --zone-name keyvault
```

## Phase 7: Jumpbox VM Deployment

### 7.1 Create Network Security Group for Jumpbox
```powershell
az network nsg create `
  --resource-group $resourceGroupName `
  --name nsg-jumpbox `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam

# Add rule to allow SSH from Bastion subnet
az network nsg rule create `
  --resource-group $resourceGroupName `
  --nsg-name nsg-jumpbox `
  --name AllowBastionSSH `
  --protocol Tcp `
  --priority 100 `
  --destination-port-range 22 `
  --source-address-prefixes $subnetBastionPrefix `
  --access Allow
```

### 7.2 Deploy Jumpbox VM (No Public IP)
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
  --nsg nsg-jumpbox `
  --size Standard_B2s `
  --public-ip-address '""' `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops region=$location
```

## Phase 8: Azure Bastion Deployment

### 8.1 Create Public IP for Bastion
```powershell
az network public-ip create `
  --resource-group $resourceGroupName `
  --name $bastionIpName `
  --sku Standard `
  --location $location `
  --allocation-method Static `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops
```

### 8.2 Deploy Azure Bastion
```powershell
az network bastion create `
  --name $bastionName `
  --resource-group $resourceGroupName `
  --location $location `
  --vnet-name $vnetName `
  --public-ip-address $bastionIpName `
  --sku Standard `
  --enable-tunneling true `
  --tags environment=dev project=aks-sre owner=swayam challenge=challenge1 costcenter=engineering department=devops
```

## Phase 9: RBAC and Access Configuration

### 9.1 Get Current User Object ID
```powershell
$currentUserObjectId = az ad signed-in-user show --query id -o tsv
Write-Host "Current User Object ID: $currentUserObjectId"
```

### 9.2 Assign Comprehensive RBAC Roles
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

# AKS RBAC Cluster Admin role
az role assignment create `
  --assignee $currentUserObjectId `
  --role "Azure Kubernetes Service RBAC Cluster Admin" `
  --scope $(az aks show --name $aksName --resource-group $resourceGroupName --query id -o tsv)

# ACR Admin role
az role assignment create `
  --assignee $currentUserObjectId `
  --role "Owner" `
  --scope $(az acr show --name $acrName --resource-group $resourceGroupName --query id -o tsv)
```

### 9.3 Configure AKS Managed Identity Permissions
```powershell
# Get AKS managed identity
$aksManagedIdentityClientId = az aks show --name $aksName --resource-group $resourceGroupName --query identityProfile.kubeletidentity.clientId -o tsv
$aksManagedIdentityObjectId = az aks show --name $aksName --resource-group $resourceGroupName --query identityProfile.kubeletidentity.objectId -o tsv

# Assign ACR Pull role to AKS managed identity
az role assignment create `
  --assignee $aksManagedIdentityClientId `
  --role "AcrPull" `
  --scope $(az acr show --name $acrName --resource-group $resourceGroupName --query id -o tsv)

# Assign Key Vault Secrets User role to AKS managed identity
az role assignment create `
  --assignee $aksManagedIdentityObjectId `
  --role "Key Vault Secrets User" `
  --scope $(az keyvault show --name $keyVaultName --resource-group $resourceGroupName --query id -o tsv)
```

## Phase 10: Comprehensive Cloud Policies and Compliance

### 10.1 Install Azure Policy Extension
```powershell
az extension add --name policy
```

### 10.2 Apply Resource Tagging Policies
```powershell
# Require specific tags policy
# Save to file
$requiredTagsPolicy = @"
{
  "if": {
    "allOf": [
      {
        "field": "type",
        "notEquals": "Microsoft.Resources/subscriptions/resourceGroups"
      },
      {
        "anyOf": [
          {
            "field": "tags['environment']",
            "exists": "false"
          },
          {
            "field": "tags['project']",
            "exists": "false"
          },
          {
            "field": "tags['owner']",
            "exists": "false"
          },
          {
            "field": "tags['costcenter']",
            "exists": "false"
          }
        ]
      }
    ]
  },
  "then": {
    "effect": "deny"
  }
}
"@

$policyFilePath = "./require-standard-tags.json"
$requiredTagsPolicy | Set-Content -Path $policyFilePath -Encoding utf8

# Create custom policy definition
az policy definition create `
  --name "require-standard-tags" `
  --display-name "Require Standard Tags" `
  --description "Requires environment, project, owner, and costcenter tags on all resources" `
  --rules $policyFilePath `
  --mode Indexed

# Assign the policy
az policy assignment create `
  --name "require-standard-tags-assignment" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "require-standard-tags" `
  --display-name "Require Standard Tags Assignment"
```

### 10.3 Apply Location Restriction Policy
```powershell
az policy assignment create `
  --name "allowed-locations" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "e56962a6-4747-49cd-b67b-bf8b01975c4c" `
  --params '{\"listOfAllowedLocations\": {\"value\": [\"eastus\", \"eastus2\", \"westus2\"]}}' `
  --display-name "Allowed Locations Policy"
```

### 10.4 Apply AKS Security Policies
```powershell
# AKS cluster should not allow privileged containers
az policy assignment create `
  --name "aks-no-privileged-containers" `
  --scope $(az aks show --name $aksName --resource-group $resourceGroupName --query id -o tsv) `
  --policy "95edb821-ddaf-4404-9732-666045e056b4" `
  --display-name "AKS Should Not Allow Privileged Containers"

# AKS cluster should use managed identities
az policy assignment create `
  --name "aks-use-managed-identity" `
  --scope $(az aks show --name $aksName --resource-group $resourceGroupName --query id -o tsv) `
  --policy "da6e2401-19da-4532-9141-fb8fbde08431" `
  --display-name "AKS Should Use Managed Identity"

# AKS cluster should enable private cluster
az policy assignment create `
  --name "aks-enable-private-cluster" `
  --scope $(az aks show --name $aksName --resource-group $resourceGroupName --query id -o tsv) `
  --policy "040732e8-d947-40b8-95d6-854c95024bf8" `
  --display-name "AKS Should Enable Private Cluster"
```

### 10.5 Apply Network Security Policies
```powershell
# Subnets should be associated with a Network Security Group
az policy assignment create `
  --name "subnets-require-nsg" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "e71308d3-144b-4262-b144-efdc3cc90517" `
  --display-name "Subnets Should Have NSG"

# Storage accounts should restrict network access
az policy assignment create `
  --name "storage-restrict-network-access" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "34c877ad-507e-4c82-993e-3452a6e0ad3c" `
  --display-name "Storage Accounts Should Restrict Network Access"
```

### 10.6 Apply Monitoring and Diagnostics Policies
```powershell
# Container Registry should have diagnostic settings enabled
az policy assignment create `
  --name "acr-enable-diagnostics" `
  --scope $(az acr show --name $acrName --resource-group $resourceGroupName --query id -o tsv) `
  --policy "c4857be7-912a-4c75-87e6-e30292bcdf78" `
  --display-name "ACR Should Enable Diagnostic Settings"

# Key Vault should have diagnostic settings enabled
az policy assignment create `
  --name "keyvault-enable-diagnostics" `
  --scope $(az keyvault show --name $keyVaultName --resource-group $resourceGroupName --query id -o tsv) `
  --policy "cf820ca0-f99e-4f3e-84fb-66e913812d21" `
  --display-name "Key Vault Should Enable Diagnostic Settings"
```

### 10.7 Apply Cost Management Policies
```powershell
# Virtual machines should have cost-effective SKUs
$costEffectiveVmSkus = @"
{
  "listOfAllowedSKUs": {
    "value": [
      "Standard_B1s",
      "Standard_B1ms",
      "Standard_B2s",
      "Standard_B2ms",
      "Standard_B4ms",
      "Standard_D2s_v3",
      "Standard_D4s_v3"
    ]
  }
}
"@
$costPolicyFilePath = "./cost-effective-vmskus.json"
$costEffectiveVmSkus | Set-Content -Path $costPolicyFilePath -Encoding utf8

az policy assignment create `
  --name "vm-allowed-skus" `
  --scope $(az group show --name $resourceGroupName --query id -o tsv) `
  --policy "cccc23c7-8427-4f53-ad12-b6a63eb452b3" `
  --params @$costPolicyFilePath `
  --display-name "VMs Should Use Cost-Effective SKUs"
```

## Phase 11: DNS Configuration and Validation

### 11.1 Verify DNS Resolution from AKS
```powershell
# Get AKS private FQDN
$aksFqdn = az aks show --resource-group $resourceGroupName --name $aksName --query privateFqdn -o tsv
Write-Host "AKS Private FQDN: $aksFqdn"

# Verify DNS zone has the correct A record
az network private-dns record-set a list `
  --resource-group $resourceGroupName `
  --zone-name $aksPrivateDnsZone `
  --output table
```

### 11.2 Configure Custom DNS Records (if needed)
```powershell
# Get private endpoint IPs for manual DNS configuration if auto-registration fails
Write-Host "Getting private endpoint information..."

# ACR Private Endpoint IP
$acrPrivateIp = az network private-endpoint show `
  --name pe-acr `
  --resource-group $resourceGroupName `
  --query 'customDnsConfigs[0].ipAddresses[0]' -o tsv

Write-Host "ACR Private IP: $acrPrivateIp"

# Key Vault Private Endpoint IP
$kvPrivateIp = az network private-endpoint show `
  --name pe-keyvault `
  --resource-group $resourceGroupName `
  --query 'customDnsConfigs[0].ipAddresses[0]' -o tsv

Write-Host "Key Vault Private IP: $kvPrivateIp"

# Manually create DNS records if not auto-created
if ($acrPrivateIp) {
    az network private-dns record-set a add-record `
      --resource-group $resourceGroupName `
      --zone-name $acrPrivateDnsZone `
      --record-set-name $acrName `
      --ipv4-address $acrPrivateIp
}

if ($kvPrivateIp) {
    az network private-dns record-set a add-record `
      --resource-group $resourceGroupName `
      --zone-name $kvPrivateDnsZone `
      --record-set-name $keyVaultName `
      --ipv4-address $kvPrivateIp
}
```

## Phase 12: Post-Deployment Configuration

### 12.1 Install Required Extensions
```powershell
az extension add --name bastion
az extension add --name aks-preview
```

### 12.2 Connect to Jumpbox via Bastion and Setup Tools
```powershell
# Test connection to jumpbox
Write-Host "Testing Bastion connection to jumpbox..."
az network bastion ssh `
  --name $bastionName `
  --resource-group $resourceGroupName `
  --target-resource-id $(az vm show --resource-group $resourceGroupName --name $vmName --query id -o tsv) `
  --auth-type password `
  --username $vmUser
```

### 12.3 Install Tools on Jumpbox (Run after connecting via Bastion)
```bash
#!/bin/bash
# Run these commands after connecting to jumpbox via Bastion

# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Helm
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y

# Install Docker
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER

# Install additional tools
sudo apt-get install -y git curl wget unzip jq tree htop

# Verify installations
echo "=== Verifying Installations ==="
az --version
kubectl version --client
helm version
docker --version
```

### 12.4 Configure kubectl and Test AKS Access
```bash
# Login to Azure (run on jumpbox)
az login

# Set subscription
az account set --subscription "your-subscription-id"

# Get AKS credentials
az aks get-credentials --resource-group rg-network-dev --name aks-app-dev

# Test kubectl access
kubectl get nodes
kubectl get pods --all-namespaces
kubectl cluster-info

# Test ACR connectivity
az acr login --name your-acr-name
az acr repository list --name your-acr-name

# Test Key Vault connectivity
az keyvault secret list --vault-name your-key-vault-name
```

### 12.5 Store Configuration in Key Vault
```powershell
# Store important configuration values in Key Vault
az keyvault secret set `
  --vault-name $keyVaultName `
  --name "aks-cluster-name" `
  --value $aksName

az keyvault secret set `
  --vault-name $keyVaultName `
  --name "acr-name" `
  --value $acrName

az keyvault secret set `
  --vault-name $keyVaultName `
  --name "jumpbox-username" `
  --value $vmUser

# Store kubeconfig (get from jumpbox or create temporarily)
az aks get-credentials --name $aksName --resource-group $resourceGroupName --file kubeconfig-temp
az keyvault secret set `
  --vault-name $keyVaultName `
  --name "aks-kubeconfig" `
  --file kubeconfig-temp
Remove-Item kubeconfig-temp
```

## Phase 13: Monitoring and Logging Setup

### 13.1 Enable Container Insights
```powershell
# Create Log Analytics workspace
$logAnalyticsWorkspace = "law-aks-$((Get-Random -Maximum 9999))"

az monitor log-analytics workspace create `
  --resource-group $resourceGroupName `
  --workspace-name $logAnalyticsWorkspace `
  --location $location `
  --tags environment=dev project=aks-sre owner=swayam

# Enable Container Insights
az aks enable-addons `
  --resource-group $resourceGroupName `
  --name $aksName `
  --addons monitoring `
  --workspace-resource-id $(az monitor log-analytics workspace show --resource-group $resourceGroupName --workspace-name $logAnalyticsWorkspace --query id -o tsv)
```

### 13.2 Configure Diagnostic Settings
```powershell
# Get Log Analytics workspace ID
$workspaceId = az monitor log-analytics workspace show --resource-group $resourceGroupName --workspace-name $logAnalyticsWorkspace --query id -o tsv

# Enable diagnostics for AKS
az monitor diagnostic-settings create `
  --name "aks-diagnostics" `
  --resource $(az aks show --name $aksName --resource-group $resourceGroupName --query id -o tsv) `
  --workspace $workspaceId `
  --logs '[{"category":"kube-apiserver","enabled":true},{"category":"kube-audit","enabled":true},{"category":"kube-audit-admin","enabled":true},{"category":"kube-controller-manager","enabled":true},{"category":"kube-scheduler","enabled":true},{"category":"cluster-autoscaler","enabled":true}]' `
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Enable diagnostics for ACR
az monitor diagnostic-settings create `
  --name "acr-diagnostics" `
  --resource $(az acr show --name $acrName --resource-group $resourceGroupName --query id -o tsv) `
  --workspace $workspaceId `
  --logs '[{"category":"ContainerRegistryRepositoryEvents","enabled":true},{"category