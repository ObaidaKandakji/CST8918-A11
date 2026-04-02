CST8918-W25-A11.  
Prof. Robert McKenney

# Lab A11: Remote state storage with Azure Blob Storage

When using Terraform with teams or in a CI/CD pipeline, it is important to store the state file securely and to share it with the team. In this lab, you will configure Terraform to store the state file in Azure Blob Storage. You will create a storage account and a container in Azure, and configure Terraform to use the storage account as the backend. You will also set up an environment variable to store the access key for the storage account.

## Setup your project folders

```bash
mkdir -p cst8918-w25-a11/terraform
cd cst8918-w25-a11/terraform
touch terraform.tf
touch main.tf
cd ..
# copy the .gitignore file from the previous project
git init
```

## Remote storage of Terraform state

The terraform state file is a file that contains the state of the infrastructure that is being managed by terraform. This file is used to map the real world resources to the configuration that is being used to manage them. The state file is used to plan and apply changes to the infrastructure. **The state file is stored locally by default**, but it can be stored remotely. This is useful when working in a team, as it allows the team to share the state file and work on the same infrastructure. It also allows the state file to be stored securely, as it can be encrypted and stored in a secure location.

### Terraform backend

The terraform backend is a configuration that tells terraform where to store the state file. The backend can be configured to store the state file in a number of different locations, including S3, Azure Blob Storage, Google Cloud Storage, and more. The backend can also be configured to encrypt the state file, which is important for storing the state file securely.

### Remote state storage with Azure Blob Storage

In this example, we will configure terraform to store the state file in Azure Blob Storage. First, we need to create a storage account in Azure. We can do this using the Azure portal, or using the Azure CLI. Once the storage account is created, we need to create a container to store the state file. We can do this using the Azure portal, or using the Azure CLI.

#### Create a separate resource group
You want to place the blob storage for terraform state in a dedicated resource group. This prevents accidental deletion with your managed infrastructure. Using the az cli create a resource group using the naming patter `<your-college-username>-cst8918-tf-backend` => **replace <your-college-username> with your college username.**

```sh
az group create --name <your-college-username>-cst8918-tf-backend --location westus3
```

#### Create a storage account

You can then manually create a new storage account using the that new resource group. Update the Azure CLI command below with your college id and run it in the terminal. 

```bash
az storage account create \
--name <your-college-id>tfstorage \
--resource-group <your-college-username>-cst8918-tf-backend \
--location westus3 --sku Standard_LRS
```

> [!TIP]
> Remember that the storage account name must be unique across all of Azure, and must be between 3 and 24 characters in length. Only use lowercase letters and numbers (no dashes, underscores, or other symbols).

#### Create a container

You will need a container to store the state file. You can create a container using the Azure portal, or using the Azure CLI. Here is an example of how to create a container using the Azure CLI:

```bash
az storage container create --name tfstate --account-name <your-college-id>tfstorage
```
> [!TIP]
> You may see a warning message about access credentials. Read it carefully to understand the default behaviour. At the end you should see an indication that the storage container was created successfully.

#### Configure Terraform to use the storage account as the backend

Once the storage account and container are created, we can configure terraform to use the storage account as the backend. We can do this by adding a `backend` block to the terraform configuration file. The `backend` block tells terraform where to store the state file, and how to access it. Add this `backend` block to your usual `terraform` initialization block in the `terraform.tf` file. That configures terraform to use Azure Blob Storage as the backend:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "<your-college-id>-cst8918-tf-backend"
    storage_account_name = "<your-college-id>tfstorage"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
  }
}
```

> [!TIP]
> The example uses `dev.terraform.tfstate` as the key for the state file. This is the name of the state file that will be stored in the container. You can use any name you like, but it must end with `.tfstate`. In this case we are using the `dev` prefix to indicate that this is the state file for the development environment. You can use any prefix you like, or no prefix at all.

#### Add an environment variable for the storage account access key

You will need to copy the Azure storage account `access_key`. You can find the `access_key` in the Azure portal, under the `Access keys` section of the storage account. You can also use the Azure CLI to get the `access_key` and `connection_string`:

```bash
az storage account keys list --account-name <your-college-id>tfstorage --resource-group <your-college-id>-cst8918-tf-backend
```

Copy the `value` property of the first key in the list. You will use this value to set the `ARM_ACCESS_KEY` shell environment variable. Terraform will use it to access the storage account. You can set this environment variable manually in your shell, or from secure storage in your CI/CD pipeline.

Set this environment variable locally for now using the following command:

```bash
export ARM_ACCESS_KEY=<your-access-key>
```

> [!IMPORTANT]
> This is a special variable name. You cannot use a regular `TF_VAR_` prefix for this variable. Terraform will automatically use this environment variable to access the storage account.

##### Check your work

Your complete `terraform.tf` file should look like this:

```hcl

terraform {
  required_version = "~> 1.5"

  backend "azurerm" {
    resource_group_name  = "<your-college-id>-Default-WestUS3-rg"
    storage_account_name = "<your-college-id>tfstorage"
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

#### Initialize the backend

In the terminal, make sure that you are in the `terraform` directory, and run `terraform init` to initialize the backend. This will create the state file in the storage account. You should see output similar to the following:

```bash
❯ terraform init

Initializing the backend...

Successfully configured the backend "azurerm"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding hashicorp/azurerm versions matching "~> 3.96.0"...
- Installing hashicorp/azurerm v3.96.0...
- Installed hashicorp/azurerm v3.96.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

## Add some resources

Let's add some resources to the `main.tf` file. We will add a resource group, a virtual network, and a subnet. Here is the `main.tf` file with the resources added:

```hcl
variable "resource_prefix" {
  description = "A prefix to add to all resources"
  type        = string
  default     = "<your-username>-a11"
}

resource "azurerm_resource_group" "rg" {
  name     = "${var.resource_prefix}-rg"
  location = "westus3"
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

### Plan and apply the changes

Check your work.

```bash
terraform fmt && terraform validate && terraform plan -out=a11.tfplan
```

If the plan looks good, apply the changes.

```bash
terraform apply a11.tfplan
```

You should see output similar to the following:

```bash
azurerm_resource_group.rg: Creating...
azurerm_resource_group.rg: Still creating... [10s elapsed]
azurerm_resource_group.rg: Creation complete after 12s [id=/subscriptions/b3f9dff4-91e8-4934-bab4-2c26fae7980d/resourceGroups/mckennr-a11-rg]
azurerm_virtual_network.vnet: Creating...
azurerm_virtual_network.vnet: Creation complete after 9s [id=/subscriptions/b3f9dff4-91e8-4934-bab4-2c26fae7980d/resourceGroups/mckennr-a11-rg/providers/Microsoft.Network/virtualNetworks/mckennr-a11-vnet]
azurerm_subnet.subnet: Creating...
azurerm_subnet.subnet: Creation complete after 6s [id=/subscriptions/b3f9dff4-91e8-4934-bab4-2c26fae7980d/resourceGroups/mckennr-a11-rg/providers/Microsoft.Network/virtualNetworks/mckennr-a11-vnet/subnets/mckennr-a11-test-subnet]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

> [!TIP]
> You can check the state file in the storage account to see the state file that was created. Also, note that you should NOT see a `terraform.tfstate` file in your local directory. The state file is now stored in the storage account.

## Submit your work

Submit two files:

- a screenshot of the Azure portal showing the overview panel of the `dev.terraform.tfstate` file in the storage account. Make sure that your login profile is visible in the upper-right corner of the screen.
- a downloaded copy of the `dev.terraform.tfstate` file from the storage account

## Clean up

When you are done, you can destroy the resources you created using the `terraform destroy` command.

```bash
terraform destroy
```

When it has finished, check the Azure portal to make sure that all the resources have been deleted. You should also check the storage account again. You should see that the `dev.terraform.tfstate` file has NOT been deleted, but contains no resources. It will look similar to this:

```tf
{
  "version": 4,
  "terraform_version": "1.7.4",
  "serial": 4,
  "lineage": "bbeeb1d2-9aee-4d43-939f-7138779410a9",
  "outputs": {},
  "resources": [],
  "check_results": null
}
```

The state file is not deleted when the resources are destroyed. You can delete the state file manually if you want to clean up the storage account. However, we will be using it again next week and it will cost less than 2¢, so feel free to keep the blob storage account.