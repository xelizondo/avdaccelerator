# Azure Virtual Desktop Accelerator for Terraform Guide

This guide is designed to help you get started with deploying Azure Virtual Desktop using the provided Terraform template(s) within this repository. Before you deploy, it is recommended to review the template(s) to understand the resources that will be deployed and the associated costs.

This accelerator is to be used as starter kit and you can expand its functionality by developing your own deployments. This scenario deploys a new Azure Virtual Desktop workload, so it cannot be used to maintain, modify or add resources to an existing or already deployed Azure Virtual Desktop workload from this accelerator.

***Note*** This terraform accelerator requires the Custom Image Build before deploying the Baseline. If you prefer to use the marketplace image with no customization [see](https://docs.microsoft.com/en-us/azure/developer/terraform/create-avd-session-host)

## Table of Contents

- [Prerequisites](#prerequisites)  
- [Planning](#planning) 
- [Custom Image Build](#Custom-Image-Build)    
- [AVD Baseline](#AVD-Baseline)  
- [Backend Setup](#Backends)  
- [Terraform file Structure](#Files)  

This guide describes how to deploy Azure Virtual Desktop Accelerator using the [Terraform](https://www.terraform.io/).
To get started with Terraform on Azure check out their [tutorial](https://learn.hashicorp.com/collections/terraform/azure-get-started/).

## Prerequisites

- Meet the prerequisites listed [here](https://github.com/Azure/avdaccelerator/wiki/Getting-Started#Getting-Started)
- Current version of the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- Current version of the Terraform CLI
- An Azure Subscription(s) where you or an identity you manage has `Owner` [RBAC permissions](https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#owner)

## Planning

### Decide on a Prefix

The deployments will require a "Prefix" which will be included in all the deployed resources name.
Resource Groups and resource names are derived from the `Prefix` parameter. Pick a unique resource prefix that is 3-5 alphanumeric characters in length without whitespaces.

## Custom-Image-Build

Deploy a customer image based on the latest version of the Azure Marketplace image for Windows 11 21H2 with M365 using Azure Image Builder to an Azure Compute Gallery. The custom image is optimized using [Virtual Desktop Optimization Tool (VDOT)](https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool) and patched with the latest Windows updates.

![Custom Image diagram](../../workload/docs/diagrams/avd-accelerator-terraform-aib-custom-image.png)
## AVD-Baseline

Azure Virtual Desktop (Azure Virtual Desktop) resources and dependent services for establishing the baseline.

- Azure Virtual Desktop resources:
  - 2 Host Pools – 1 personal and 1 pooled
  - 2 Workspaces – 1 personal and 1 pooled
  - Associated Desktop Application Group for personal
  - Associated Desktop Application Group and Remote Application Group for pooled
- Azure Files Storage with FSLogix share, RBAC role assignment and private endpoint
- Application Security group and Network Security group
- New VNet, subnet with baseline NSG, DNS zones for private endpoints, route table and peering to the hub virtual network
- Key Vault and private endpoint

![Custom Image diagram](../../workload/docs/diagrams/avd-accelerator-terraform-baseline-image.png)

## Backends

The default templates write a state file directly to disk locally to where you are executing terraform from. If you wish to AzureRM backend please see [AzureRM Backend](https://www.terraform.io/docs/language/settings/backends/azurerm.html). This deployment highlights using Azure Blog Storage to store state file and Key Vault

### Backends using Azure Blob Storage

<details>
<summary>Click to expand</summary>
#### Using Azure CLI

[Store state in Azure Storage](https://docs.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage)

```cli
RESOURCE_GROUP_NAME=tstate
STORAGE_ACCOUNT_NAME=tstate$RANDOM
CONTAINER_NAME=tstate
```

### Create resource group

```cli
az group create --name $RESOURCE_GROUP_NAME --location <eastus>
```

### Create storage account

```cli
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob
```

#### Get storage account key

```cli
ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query '[0].value' -o tsv)
```

#### Create blob container

```cli
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME --account-key $ACCOUNT_KEY

echo "storage_account_name: $STORAGE_ACCOUNT_NAME"
echo "container_name: $CONTAINER_NAME"
echo "access_key: $ACCOUNT_KEY"
```

### Create a key vault

[Create Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-cli)

```cli
az keyvault create --name "<Azure Virtual Desktopkeyvaultdemo>" --resource-group $RESOURCE_GROUP_NAME --location "<East US>"
```

#### Add storage account access key to key vault

```cli
az keyvault secret set --vault-name "<Azure Virtual Desktopkeyvaultdemo>" --name terraform-backend-key --value "<W.........................................>"
```

</details>

## Files

The Custom Image Terraform files structure:
| file Name           | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| aib.tf              | This file deploys Azure Image Builder and Compute Gallery |
| outputs.tf          | This will contains the outputs post deployment |
| variables.tf        | Variables have been created in all files for various properties and names, these are placeholders and are not required to be changed unless there is a need to. See below |
| terraform.tfvars    | This file contains all variables to be changed from the defaults, you are only required to change these as per your requirements |

The Azure Virtual Desktop Baseline Terraform files are all written as individual files each having a specific function. Variables have been created in all files for consistency, all changes to defaults are to be changed from the terraform.tfvars.sample file. The structure is as follows:

| file Name           | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| main.tf             | This file deploys Azure Virtual Desktop |
| host.tf             | This file deploys session host using the custom image in the Azure Compute Gallery |
| provider.tf         | This file contains the Terraform provider settings and version |
| afstorage.tf        | This file creates the Storage account and Azure files shares with RBAC |
| networking.tf       | This file creates the Virtual Network and subnets to be used |
| nsg.tf              | This file creates a nsg |
| keyvault.tf         | This file creates the Key Vault to be used     |
| appsecgrp.tf        | This file creates the Application security group to be used     |
| routetable.tf       | This file creates the a Route Table to be used     |
| rbac.tf             | This will creates the rbac permissions |
| outputs.tf          | This will contains the outputs post deployment |
| variables.tf        | Variables have been created in all files for various properties and names, these are placeholders and are not required to be changed unless there is a need to. See below |
| terraform.tfvars    | This file contains all variables to be changed from the defaults, you are only required to change these as per your requirements |

Validated on provider versions:
hashicorp/random v3.3.2
hashicorp/azuread v2.26.1
hashicorp/azurerm v3.61.0
## Deployment Steps

1. Modify the `terraform.tfvars` file to define the desired names, location, networking, and other variables
2. Before deploying, confirm the correct subscription
3. Change directory to the Terraform folder
4. Run `terraform init` to initialize this directory
5. Run `terraform plan` to view the planned deployment
5. Run `terraform apply` to confirm the deployment

## Confirming Deployment

## Additional References

<details>
<summary>Click to expand</summary>

- [Terraform Download](https://www.terraform.io/downloads.html)
- [Visual Code Download](https://code.visualstudio.com/Download)
- [Powershell VS Code Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell)
- [HashiCorp Terraform VS Code Extension](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)
- [Azure Terraform VS Code Extension Name](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureterraform)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli)
- [Configure the Azure Terraform Visual Studio Code extension](https://docs.microsoft.com/en-us/azure/developer/terraform/configure-vs-code-extension-for-terraform)
- [Setup video](https://youtu.be/YmbmpGdhI6w)

</details>

## Reporting issues

Microsoft Support is not yet handling issues for any published tools in this repository. However, we would like to welcome you to open issues using GitHub [issues](https://github.com/Azure/avdaccelerator/issues) to collaborate and improve these tools.
