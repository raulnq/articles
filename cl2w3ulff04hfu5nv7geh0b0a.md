## Terraform: Resources, Variables, Outputs and Locals

> HashiCorp Terraform is an infrastructure as code tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share. You can then use a consistent workflow to provision and manage all of your infrastructure throughout its lifecycle.

In this post, we are going to learn about resources, variables, outputs, and local while we create a web app in Azure. Please follow the links to go to the official documentation.

## Pre-requisites

- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) 
- [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli)

Create an `azure` folder with a `main.tf` file and add the [Terraform Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest) for Azure:

```
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "3.5.0"
    }
  }
}

provider "azurerm" {
    features {}   
}
``` 

## Resources

> Resources are the most important element in the Terraform language. Each resource block describes one or more infrastructure objects, such as virtual networks, compute instances, or higher-level components such as DNS records.

Let's start creating our first [resource](https://www.terraform.io/language/resources/syntax), the [Resource Group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group):

```json
resource "azurerm_resource_group" "resource_group" {
  name     = "myFirstResourceGroupWithTerraform"
  location = "eastus"
}
``` 

The next step is to create the [App Service Plan](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/service_plan):

```json
resource "azurerm_service_plan" "app_service_plan" {
  name                = "myFirstAppServicePlanWithTerraform"
  resource_group_name = azurerm_resource_group.resource_group.name
  location            = azurerm_resource_group.resource_group.location
  os_type             = "Linux"
  sku_name            = "B1"
}
``` 

And finally, the [Web App](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_web_app):

```json
resource "azurerm_linux_web_app" "web_app" {
  name                = "myFirstWebApp-1958666"
  resource_group_name = azurerm_resource_group.resource_group.name
  location            = azurerm_service_plan.app_service_plan.location
  service_plan_id     = azurerm_service_plan.app_service_plan.id
  site_config {}
}
``` 

## Running Terraform Code

The first step is to run the `init` command to [initiate terraform](https://www.terraform.io/cli/commands/init):

```powershell
terraform init
``` 

Before continuing with the following command, we need to [log in](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli) against Azure:

```powershell
 az login
``` 

Time to [provision the infrastructure](https://www.terraform.io/cli/run), run the `plan` command to create the execution plan:

```powershell
terraform plan -out app.tfplan   
``` 

As the last step, we are going to apply the actions proposed in the plan:

```powershell
terraform apply 'app.tfplan'
``` 

## Validate the deployment

We can check the deployment on the Azure Portal:

![azure-portal-webapp.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651876658190/mcmnzzH4W.png align="left")

## Variables

>Input Variables serve as parameters for a Terraform module, so users can customize behavior without editing the source.

Let's introduce some [variables](https://www.terraform.io/language/values/variables) to parametrize the name of the resources:

```json
variable "resource_group_name" {
  description = "Name of the resource group"
  type = string
}

variable "app_service_plan_name" {
  description = "Name of the app service plan"
  type = string
}

variable "web_app_name" {
  description = "Name of the web app"
  type = string
}
``` 

Then change the resources to use the variables:

```json
resource "azurerm_resource_group" "resource_group" {
  name     = var.resource_group_name
  location = "eastus"
}

resource "azurerm_service_plan" "app_service_plan" {
  name                = var.app_service_plan_name
  resource_group_name = azurerm_resource_group.resource_group.name
  location            = azurerm_resource_group.resource_group.location
  os_type             = "Linux"
  sku_name            = "B1"
}

resource "azurerm_linux_web_app" "web_app" {
  name                = var.web_app_name
  resource_group_name = azurerm_resource_group.resource_group.name
  location            = azurerm_service_plan.app_service_plan.location
  service_plan_id     = azurerm_service_plan.app_service_plan.id

  site_config {}
}
``` 

Now during the execution of the plan, we can specify the value for the variables:

```json
terraform plan -out app.tfplan -var="resource_group_name=myRGwithTerraform" -var="app_service_plan_name=mySPWithTerraform" -var="web_app_name=myWAWithTerraform-4589"
``` 

## Outputs
> Output Values are like return values for a Terraform module.

Let's introduce an [output](https://www.terraform.io/language/values/outputs) to see the Web App Id:

```json
output "web_app_id" {
  value = azurerm_linux_web_app.web_app.id
}
``` 

## Local Values

> Local Values are a convenience feature for assigning a short name to an expression.

We are going to create a new variable to ask for the owner of the resource group and then concatenate it with the name of the Resource Group:

```json
variable "owner" {
  description = "Owner of the resource group"
  type = string
}

locals {
  resource_group_name = "${var.owner}-${var.resource_group_name}"
}

resource "azurerm_resource_group" "resource_group" {
  name     = local.resource_group_name
  location = "eastus"
}
``` 

Run the `terraform plan` and `terraform apply` commands to update the infrastructure in Azure. Notice that the Web App Id is visible at the end of the execution.

## Clean up Resources

Run the `plan` command but with the destroy attribute:
```
terraform plan -destroy -out app.tfplan   
``` 
And apply the plan:
```
terraform apply -destroy 'app.tfplan'
``` 

Now you can check the portal, and all the resources should not be there anymore. You can see the final `main.tf` file [here](https://github.com/raulnq/terraform-sandbox/blob/main/azure/main.tf).


