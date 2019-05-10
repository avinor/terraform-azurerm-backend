# Remote backend

Terraform module to deploy a remote backend storage with key-vault to store access keys. To access the remote state the SAS Token that is stored in key vault should be used. It will create an automation account that rotates the key every day. This ensures that nobody needs the root access key to access storage account and each backend has it's own key-vault with SAS Token scoped to only that backends container.

## Usage

Since this will create the remote backend where state should be stored it requires special setup. Remote state does not exist first time running so first use module without declaring a remote state:

```terraform
provider "azurerm" {}

module "backend" {
    source  = "avinor/remote-backend/azurerm"
    version = "0.1.0"

    name           = "remotebackendsa"
    resource_group = "terraform-rg"
    location       = "westeurope"

    backends = [
        "shared",
        "dev",
        "test",
        "prod",
    ]

    tags = {
        whatever = "remote-state"
    }
}
```

Run terraform script without any backend

```bash
terraform init -backend=false
terraform apply
```

Once this is done the remote backend should be created and state is stored locally. To upload it to the remote state that has just been created reconfigure terraform.

Create a `terraform.tf` file in same folder

```terraform
terraform {
    backend "azurerm" {
        storage_account_name = "remotebackendsa"
        container_name       = "shared"
        key                  = "bootstrap.terraform.tfstate"
    }
}
```

If required configure an access key for storing state (`ARM_ACCESS_KEY` environment variable for instance). Reconfigure terraform:

```bash
terraform init -reconfigure
```

State should now be stored remotely. Any changes after this will use the remote state that have been created with same template.

## CI/CD

To use the remote state in a CI/CD process (for instance Azure DevOps Pipelines) it is recommended to create a service principal that is granted access to the key-vault only. It can then read the SAS Token from key-vault to access the storage account. CI/CD process should not have access to storage account directly, or use the root access key.

## Key-vault token

As of now it does not set the token in key-vault since it does not have access during deployment to add secrets.