# spike-packer-windows

This is a prototype solution for building custom VM images in Azure.

## Prerequisites

* An Azure subscription.
* [Packer](https://www.packer.io/downloads.html).
    Cloud Shell comes with Packer installed. Install Packer if you want to build the image from your computer.

## Create a service principal

A service principal authenticates access to Azure resources.

1. Create a name for your service principal. Here's an example:

    ```bash
    SP_NAME=http://mslearn-packer-sp
    ```

1. Fetch details about your service principal:

    ```bash
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
    ```

1. Print your service principal details to the console. Save them somewhere safe for later.

    ```bash
    echo $ARM_SUBSCRIPTION_ID
    echo $ARM_CLIENT_SECRET
    echo $ARM_CLIENT_ID
    echo $ARM_TENANT_ID
    ```

## Create a resource group

Create a resource group that will hold your VM image. Here's an example:

```bash
RG_NAME=spike-packer-rg
az group create --location eastus --name $RG_NAME
```

Save the name for later.

## Choose a name for your VM image

Here's an example:

```bash
IMG_NAME=spike-packer-image
```

Save the name for later.

## Build the image

Perform these steps if you want to build the image locally or see how the process works.

From Bash, run the following `packer build` command to create the image. Give the process around 20 minutes to finish.

```bash
packer build \
  -var "client_id=$ARM_CLIENT_ID" \
  -var "client_secret=$ARM_CLIENT_SECRET" \
  -var "tenant_id=$ARM_TENANT_ID" \
  -var "subscription_id=$ARM_SUBSCRIPTION_ID" \
  -var "managed_image_resource_group_name=$RG_NAME" \
  -var "managed_image_name=$IMG_NAME" \
  windows.json
```

### Create a test VM

To verify the configuration, create a test VM, connect to it, and then tear it down.

1. Create the VM:

    ```bash
    az group create --location eastus --name test-windows-rg

    IMG_URN=$(az image list --resource-group $RG_NAME --query [].id --output tsv)

    az vm create \
      --resource-group test-windows-rg \
      --name test-windows-vm \
      --admin-username azureuser \
      --admin-password "Password1234**4321drowssaP" \
      --image $IMG_URN
    ```

1. Connect to your VM and verify that it's configured as you expect.
1. Tear down your VM:

    ```bash
    az group delete --name test-windows-rg --yes
    ```

## Publish to the image gallery

The image gallery enables you to share your VM with others.

1. Create Bash variables to describe the name of your gallery and its Azure location.

    ```bash
    GAL_NAME=mygallery
    AZ_LOCATION=eastus
    ```

1. Create a resource group to hold your gallery.

    ```bash
    az group create --name $GAL_NAME-rg --location $AZ_LOCATION
    ```

1. Create a shared image gallery.

    ```bash
    az sig create \
      --resource-group $GAL_NAME-rg \
      --gallery-name $GAL_NAME \
      --location $AZ_LOCATION
    ```

1. Create a gallery image definition.

    ```bash
    az sig image-definition create \
      --resource-group $GAL_NAME-rg \
      --gallery-name $GAL_NAME \
      --gallery-image-definition $GAL_NAME-imgdef \
      --publisher MicrosoftLearn \
      --offer WindowsServer \
      --sku 2019-Datacenter-Az-Pwsh \
      --os-type Windows \
      --location $AZ_LOCATION
    ```

1. Create the list of regions that you want to target.

    This example shows one region with a replica count of 1.

    ```bash
    TARGET_REGIONS="eastus=1"
    ```

1. Create a new image version.

    ```bash
    az sig image-version create \
      --resource-group $GAL_NAME-rg \
      --location $AZ_LOCATION \
      --gallery-name $GAL_NAME \
      --gallery-image-definition $GAL_NAME-imgdef \
      --gallery-image-version 1.0.0 \
      --target-regions $TARGET_REGIONS \
      --replica-count 2 \
      --managed-image $IMG_URN
    ```

1. Run this if you need the ID of the image gallery.

    ```bash
    az sig show \
      --resource-group $GAL_NAME-rg \
      --gallery-name $GAL_NAME \
      --query id
    ```

## Grant permissions

From the Azure portal, grant permissions to the user or principal you want to allow access to the gallery.

## Run it in Azure Pipelines

1. Create a project in Azure DevOps.
1. Create a pipeline that's connected to your fork of this repository.
1. From Azure Pipelines, select **Library**. Then add these to a variable group named *Azure*:

    | Variable                          | Your value of: |
    |-----------------------------------|--------------------------------------|
    | client_id                         | `$ARM_CLIENT_ID`       |
    | client_secret                     | `$ARM_CLIENT_SECRET`   |
    | subscription_id                   | `$ARM_SUBSCRIPTION_ID` |
    | tenant_id                         | `$ARM_TENANT_ID`       |
    | managed_image_resource_group_name | `$RG_NAME`             |
    | managed_image_name                | `$IMG_NAME`            |
    | GAL_NAME                          | `$GAL_NAME`            |
    | AZ_LOCATION                       | `$AZ_LOCATION`         |
    | TARGET_REGIONS                    | `$TARGET_REGIONS`      |
    | IMG_URN                           | `$IMG_URN`             |

    Select the lock icon next to `client_secret` to encrypt its value.
1. Create an Azure Resource Manager [service connection](https://docs.microsoft.com/azure/devops/pipelines/library/connect-to-azure?view=azure-devops) named *Azure* that uses your service principal's details. (Select the **Service principal (manual)** option)

    Here's how to get your subscription name and ID.

    ```bash
    az account list --query "[?isDefault][name]" --output tsv
    az account list --query "[?isDefault][id]" --output tsv
    ```

1. Run the pipeline.