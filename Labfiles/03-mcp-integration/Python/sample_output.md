Agent created (id: MyAgent:1, name: MyAgent, version: 1)
Created conversation (id: conv_0e4848e6f70933c000AlBcZ7hIW71emWesXPxQlIIudReVffun)

Agent response: Below are Azure CLI commands for both managed identity types. These examples use Bash.

## Option 1: System-assigned managed identity

A system-assigned identity is created and managed by Azure and is tied to the Container App.

```azurecli
# Variables
RESOURCE_GROUP="my-containerapps-rg"
LOCATION="eastus"
ENVIRONMENT="my-containerapp-env"
APP_NAME="my-containerapp"

# Sign in and select a subscription
az login
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"

# Optional: install/update the Container Apps CLI extension
az extension add --name containerapp --upgrade

# Create a resource group
az group create \
  --name "$RESOURCE_GROUP" \
  --location "$LOCATION"

# Create a Container Apps environment
az containerapp env create \
  --name "$ENVIRONMENT" \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION"

# Create the Container App
az containerapp create \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --environment "$ENVIRONMENT" \
  --image "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest" \
  --target-port 80 \
  --ingress external

# Assign a system-assigned managed identity
PRINCIPAL_ID=$(az containerapp identity assign \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --system-assigned \
  --query principalId \
  --output tsv)

echo "Managed identity principal ID: $PRINCIPAL_ID"
```

## Option 2: User-assigned managed identity

A user-assigned identity is an independent Azure resource that can be reused by multiple apps.

```azurecli
# Variables
IDENTITY_NAME="my-containerapp-identity"

# Create the user-assigned identity
az identity create \
  --name "$IDENTITY_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION"

# Get the identity resource ID and principal ID
IDENTITY_ID=$(az identity show \
  --name "$IDENTITY_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query id \
  --output tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --name "$IDENTITY_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query principalId \
  --output tsv)

# Assign the identity to the Container App
az containerapp identity assign \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --user-assigned "$IDENTITY_ID"

echo "Identity resource ID: $IDENTITY_ID"
echo "Identity principal ID: $IDENTITY_PRINCIPAL_ID"
```

## Grant the identity access to an Azure resource

A managed identity must have an Azure RBAC role before it can access resources such as Key Vault, Storage, or Service Bus.

For example:

```azurecli
TARGET_RESOURCE_ID="<RESOURCE_ID_TO_ACCESS>"

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "<ROLE_NAME>" \
  --scope "$TARGET_RESOURCE_ID"
```

For the user-assigned identity, use:

```azurecli
az role assignment create \
  --assignee-object-id "$IDENTITY_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "<ROLE_NAME>" \
  --scope "$TARGET_RESOURCE_ID"
```

For example, to grant a system-assigned identity access to read secrets from a Key Vault:

```azurecli
az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.KeyVault/vaults/<KEY_VAULT_NAME>"
```

Microsoft documentation: [Managed identities in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/managed-identity)
Agent deleted