---
title: "Avoid Data Loss When Updating the AzureRM Terraform Provider"
date: 2024-11-25T21:19:21Z
draft: false
keywords: "terraform azurerm azure storage account data-loss update"
summary: "![](/images/terraform-4.9.0-azurerm-storage-container-update/title.webp)
With the release of [AzureRM provider 4.9.0](https://registry.terraform.io/providers/hashicorp/azurerm/4.9.0/docs/resources/storage_container), the azurerm_storage_container resource deprecates the storage_account_name argument in favor of the storage_account_id argument. Updating your Terraform template to use storage_account_id directly will force Terraform to recreate the storage container, leading to potential data loss.

Here’s a step-by-step guide to safely update your configuration without losing data in the storage container."
description: "How not to lose the data in storage account container by updating the terraform template for azurerm 4.9.0"
---

![](/images/terraform-4.9.0-azurerm-storage-container-update/title.webp)

With the release of [AzureRM provider 4.9.0](https://registry.terraform.io/providers/hashicorp/azurerm/4.9.0/docs/resources/storage_container), the azurerm_storage_container resource deprecates the storage_account_name argument in favor of the storage_account_id argument. Updating your Terraform template to use storage_account_id directly will force Terraform to recreate the storage container, leading to **potential data loss**.

Here’s a step-by-step guide to safely update your configuration without losing data in the storage container.

## Step 1: Update AzureRM to Version 4.9.0

The storage_account_id argument is introduced in version 4.9.0, while storage_account_name is still supported but marked as deprecated. Ensure you're using this version to perform the migration smoothly without data loss.

``` hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.9.0"
    }
  }
}
```
## Step 2: Ensure Terraform Plan Shows No Changes

Before making any updates, confirm your current Terraform configuration matches the infrastructure.

Run:

``` bash
terraform plan
```

The output should indicate: `No changes. Your infrastructure matches the configuration.`

This ensures your Terraform template reflects the current state of your infrastructure, avoiding unnecessary changes or errors during migration.

## Step 3: Backup Your Terraform State File

Before proceeding, back up your Terraform state file. This is critical, as modifications to the state file are part of the process.

For remote backends (e.g., Azure Blob Storage), ensure you download a copy of the state file.

## Step 4: Remove the Existing Container from Terraform State

Remove the storage container resource from the Terraform state to dissociate it temporarily from management.

``` bash
terraform state rm azurerm_storage_container.sc
```

Replace `sc` with the name of your storage container resource in the Terraform configuration.

> **Note**: This action removes the container only from the state file, not from Azure.

## Step 5: Update the Terraform Configuration

Modify the azurerm_storage_container resource in your Terraform template by replacing storage_account_name with storage_account_id.

Before:

``` hcl

resource "azurerm_storage_container" "sc" {
  name                  = "mycontainer"
  storage_account_name  = azurerm_storage_account.sa.name
  container_access_type = "private"
}

```

After:

``` hcl

resource "azurerm_storage_container" "sc" {
  name                  = "mycontainer"
  storage_account_id    = azurerm_storage_account.sa.id
  container_access_type = "private"
}

```

### Step 6: Import the Existing Storage Container

Re-associate the existing storage container with Terraform using the import command. You’ll need the storage container’s resource ID.

Find the resource ID:

``` bash
az storage account show \
  --name <storage-account-name> \
  --resource-group <resource-group-name> \
  --query id \
  --output tsv

```

Run the import:

``` bash
terraform import azurerm_storage_container.sc \
  "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.Storage/storageAccounts/<storage-account-name>/blobServices/default/containers/<container-name>"
```

Replace placeholders (**<<subscription-id>>**, **<<resource-group-name>>**, etc.) with actual values.

## Step 7: Verify Terraform Plan

Finally, ensure Terraform recognizes the updated configuration and state without attempting changes.

Run:

``` bash
terraform plan
```

The output should again confirm: `No changes. Your infrastructure matches the configuration.`

## Conclusion

This step-by-step process ensures a smooth transition to the new storage_account_id argument without risking data loss in your storage container. Always back up your Terraform state file before making any significant changes and validate each step to prevent issues.

If you’ve encountered similar challenges or have tips to share, feel free to leave a comment below!

Feel free to adapt this further for tone or audience!