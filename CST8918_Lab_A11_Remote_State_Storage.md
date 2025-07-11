
# CST8918 Lab A11 — Remote Terraform State Storage with Azure Blob Storage

## Step 1: Setup project folder and initialize Git

```bash
mkdir -p cst8918-w25-a11/terraform
cd cst8918-w25-a11/terraform
touch terraform.tf main.tf
cd ..
git init
```

(Optional) Create `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.backup
.terraform.lock.hcl
*.plan
*.log
.env
```

## Step 2: Set variables for your college username and Azure region

```bash
export TF_USER=degu0055
export TF_LOCATION=canadacentral
```

## Step 3: Create an Azure resource group for backend storage

```bash
az group create \
  --name ${TF_USER}-cst8918-tf-backend \
  --location $TF_LOCATION
```

## Step 4: Create an Azure Storage Account

```bash
export TF_STORAGE_NAME=${TF_USER}tfstorage

az storage account create \
  --name $TF_STORAGE_NAME \
  --resource-group ${TF_USER}-cst8918-tf-backend \
  --location $TF_LOCATION \
  --sku Standard_LRS
```

## Step 5: Create a container inside the storage account for Terraform state

Export the storage account key:

```bash
export ARM_ACCESS_KEY=$(az storage account keys list \
  --account-name $TF_STORAGE_NAME \
  --resource-group ${TF_USER}-cst8918-tf-backend \
  --query '[0].value' --output tsv)
```

Create the container:

```bash
az storage container create \
  --name tfstate \
  --account-name $TF_STORAGE_NAME \
  --auth-mode key
```

## Step 6: Configure Terraform backend in `terraform.tf`

```hcl
terraform {
  required_version = "~> 1.5"

  backend "azurerm" {
    resource_group_name  = "degu0055-cst8918-tf-backend"
    storage_account_name = "degu0055tfstorage"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "3.96.0"
    }
  }
}
```

## Step 7: Initialize Terraform backend

```bash
terraform init
```

## Step 8: Add provider and resources in `main.tf`

```hcl
provider "azurerm" {
  features {}
}

variable "resource_prefix" {
  description = "A prefix to add to all resources"
  type        = string
  default     = "degu0055-a11"
}

resource "azurerm_resource_group" "rg" {
  name     = "${var.resource_prefix}-rg"
  location = "canadacentral"
}

resource "azurerm_virtual_network" "vnet" {
  name                = "${var.resource_prefix}-vnet"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "subnet" {
  name                 = "${var.resource_prefix}-test-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

## Step 9: Format, validate, and plan Terraform

```bash
terraform fmt && terraform validate && terraform plan -out=a11.tfplan
```

## Step 10: Apply Terraform changes

```bash
terraform apply "a11.tfplan"
```

## Step 11: Verify remote state storage

- No local `terraform.tfstate` file should exist.
- State is stored remotely in Azure Blob Storage in `tfstate` container with key `dev.terraform.tfstate`.

## Step 12: Submit your work

- Take a screenshot of `dev.terraform.tfstate` blob overview in Azure portal.
- Download `dev.terraform.tfstate` file and submit.

## Optional Step 13: Clean up resources

```bash
terraform destroy
```
