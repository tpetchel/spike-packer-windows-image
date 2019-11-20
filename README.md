
```bash
SP_NAME=http://mslearn-packer-sp

ARM_SUBSCRIPTION_ID=$(az account list \
  --query "[?isDefault][id]" \
  --all \
  --output tsv)

ARM_CLIENT_SECRET=$(az ad sp create-for-rbac \
  --name $SP_NAME \
  --role Contributor \
  --scopes "/subscriptions/$ARM_SUBSCRIPTION_ID" \
  --query password \
  --output tsv)

ARM_CLIENT_ID=$(az ad sp show \
  --id $SP_NAME \
  --query appId \
  --output tsv)

ARM_TENANT_ID=$(az ad sp show \
  --id $SP_NAME \
  --query appOwnerTenantId \
  --output tsv)

echo $ARM_SUBSCRIPTION_ID
echo $ARM_CLIENT_SECRET
echo $ARM_CLIENT_ID
echo $ARM_TENANT_ID
```

```bash
RG_NAME=spike-packer-rg
az group create --location northeurope --name $RG_NAME
```

```bash
packer build \
  -var 'client_id=$ARM_CLIENT_ID' \
  -var 'client_secret=$ARM_CLIENT_SECRET' \
  -var 'tenant_id=$ARM_TENANT_ID' \
  -var 'subscription_id=$ARM_SUBSCRIPTION_ID' \
  windows.json
```

```bash
az vm create \
  --resource-group $RG_NAME \
  --name spike-windows-vm \
  --admin-username azureuser \
  --image spike-packer-image
```