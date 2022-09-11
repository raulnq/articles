## Reusable infrastructure with Terraform modules

> A module is a container for multiple resources that are used together. Modules can be used to create lightweight abstractions, so that you can describe your infrastructure in terms of its architecture, rather than directly in terms of physical objects.

In today's post, we will understand the benefits of using Terraform modules through a simple example, but first, we need to fulfill some prerequisites:

- Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
- Setup your [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) locally.
- Install [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli).

We will build an API to get quotes from two popular animes using [AWS API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) as a proxy against the [AnimeChan API](https://animechan.vercel.app/):

- `https://<AWS_RANDOM_DOMAIN>/anime/naruto`
- `https://<AWS_RANDOM_DOMAIN>/anime/inuyasha`

Create a `main.tf` file with the following content:

```json
terraform {
  required_providers {
    aws = {
        source = "hashicorp/aws"
        version = "4.22.0"
    }
  }
}

provider "aws"{
  region = "us-east-2"
}

resource "aws_api_gateway_rest_api" "my_api" {
  name        = "my-anime-quotes-api"
  description = "My Anime Quotes API"

  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

resource "aws_api_gateway_resource" "naruto_resource" {
  rest_api_id = aws_api_gateway_rest_api.my_api.id
  parent_id   = aws_api_gateway_rest_api.my_api.root_resource_id
  path_part   = "naruto"
}

resource "aws_api_gateway_method" "naruto_resource_get_method" {
  rest_api_id      = aws_api_gateway_rest_api.my_api.id
  resource_id      = aws_api_gateway_resource.naruto_resource.id
  http_method      = "GET"
  authorization    = "NONE"
  api_key_required = false
}

resource "aws_api_gateway_integration" "naruto_resource_get_method_integration" {
  rest_api_id             = aws_api_gateway_rest_api.my_api.id
  resource_id             = aws_api_gateway_resource.naruto_resource.id
  http_method             = aws_api_gateway_method.naruto_resource_get_method.http_method
  type                    = "HTTP"
  uri                     = "https://animechan.vercel.app/api/quotes/anime?title=naruto"
  integration_http_method = aws_api_gateway_method.naruto_resource_get_method.http_method
}

resource "aws_api_gateway_method_response" "naruto_200_method_response" {
  rest_api_id = aws_api_gateway_rest_api.my_api.id
  resource_id = aws_api_gateway_resource.naruto_resource.id
  http_method = aws_api_gateway_method.naruto_resource_get_method.http_method
  status_code = "200"
}

resource "aws_api_gateway_integration_response" "naruto_200_integration_response" {
  rest_api_id       = aws_api_gateway_rest_api.my_api.id
  resource_id       = aws_api_gateway_resource.naruto_resource.id
  http_method       = aws_api_gateway_method.naruto_resource_get_method.http_method
  status_code       = aws_api_gateway_method_response.naruto_200_method_response.status_code
  selection_pattern = "2\\d{2}"
}

resource "aws_api_gateway_deployment" "deployment_api" {
  rest_api_id = aws_api_gateway_rest_api.my_api.id

  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.naruto_resource.id,
      aws_api_gateway_method.naruto_resource_get_method.id,
      aws_api_gateway_integration.naruto_resource_get_method_integration.id,
      aws_api_gateway_method_response.naruto_200_method_response.id,
      aws_api_gateway_integration_response.naruto_200_integration_response.id
    ]))
  }
    lifecycle {
    create_before_destroy = true
  }
}

resource "aws_api_gateway_stage" "stage_api" {
  rest_api_id   = aws_api_gateway_rest_api.my_api.id
  stage_name    = "anime"
  deployment_id = aws_api_gateway_deployment.deployment_api.id
  description   = "anime"
}

resource "aws_api_gateway_method_settings" "method_settings" {
  rest_api_id = aws_api_gateway_rest_api.my_api.id
  stage_name  = aws_api_gateway_stage.stage_api.stage_name
  method_path = "*/*"

  settings {
    caching_enabled        = false 
    cache_ttl_in_seconds   = 120
    metrics_enabled        = true
    logging_level          = "INFO"
  }
}
``` 

Let's review each Terraform resource to have a general idea of them:

- `aws_api_gateway_rest_api`: Manages an API Gateway REST API.
- `aws_api_gateway_resource`: Provides an API Gateway Resource.
- `aws_api_gateway_method`: Provides a HTTP Method for an API Gateway Resource.
- `aws_api_gateway_integration`: Provides an HTTP Method Integration for an API Gateway Integration.
- `aws_api_gateway_method_response`: Provides an HTTP Method Response for an API Gateway Resource.
- `aws_api_gateway_integration_response`: Provides an HTTP Method Integration Response for an API Gateway Resource.
- `aws_api_gateway_deployment`: Manages an API Gateway REST Deployment. A deployment is a snapshot of the REST API configuration. 
- `aws_api_gateway_stage`: Manages an API Gateway Stage. A stage is a named reference to a deployment.
- `aws_api_gateway_method_settings`: Manages API Gateway Stage Method Settings.

To create the resources, run the following commands:

```powershell
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
``` 

At this point, if we review the AWS Console, we will have our first URL created:

![naruto.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1662853343272/cM6ZiXJu3.PNG align="left")

To add the remaining anime we could copy and paste the resources `aws_api_gateway_resource`, `aws_api_gateway_method`, `aws_api_gateway_integration`, `aws_api_gateway_method_response` and `aws_api_gateway_integration_response` or build a Terrform module. 

### Basics

A Terraform module is any set of Terraform files in a folder. Create the following structure:

```json
|-- modules
|   |-- api_gateway_get_resource
|   |     |-- main.tf
|   |     |-- variables.tf
|   |     `-- outputs.tf
|    `--
`-- main.tf
``` 

Go the ` api_gateway_get_resource`  folder level and open the `main.tf` file and copy the following content:

```json
resource "aws_api_gateway_resource" "anime_resource" {
  rest_api_id = var.rest_api_id
  parent_id   = var.root_resource_id
  path_part   = var.anime_name
}

resource "aws_api_gateway_method" "anime_resource_get_method" {
  rest_api_id      = var.rest_api_id
  resource_id      = aws_api_gateway_resource.anime_resource.id
  http_method      = "GET"
  authorization    = "NONE"
  api_key_required = false
}

resource "aws_api_gateway_integration" "anime_resource_get_method_integration" {
  rest_api_id             = var.rest_api_id
  resource_id             = aws_api_gateway_resource.anime_resource.id
  http_method             = aws_api_gateway_method.anime_resource_get_method.http_method
  type                    = "HTTP"
  uri                     = "https://animechan.vercel.app/api/quotes/anime?title=${var.anime_name}"
  integration_http_method = aws_api_gateway_method.anime_resource_get_method.http_method
}

resource "aws_api_gateway_method_response" "anime_200_method_response" {
  rest_api_id = var.rest_api_id
  resource_id = aws_api_gateway_resource.anime_resource.id
  http_method = aws_api_gateway_method.anime_resource_get_method.http_method
  status_code = "200"
}

resource "aws_api_gateway_integration_response" "anime_200_integration_response" {
  rest_api_id       = var.rest_api_id
  resource_id       = aws_api_gateway_resource.anime_resource.id
  http_method       = aws_api_gateway_method.anime_resource_get_method.http_method
  status_code       = aws_api_gateway_method_response.anime_200_method_response.status_code
  selection_pattern = "2\\d{2}"
}
``` 

### Inputs

The module entries will be in the `variable.tf` file. Copy the following content there:

```json
variable "anime_name" {
  description = "anime"
  type        = string
}

variable "rest_api_id" {
  description = "rest_api_id"
  type        = string
}

variable "root_resource_id" {
  description = "root_resource_id"
  type        = string
}
``` 

### Outputs

The module outputs will be in the `outputs.tf` file. Copy the following content there:

```
output "aws_api_gateway_resource_id" {
  value       = aws_api_gateway_resource.anime_resource.id
}

output "aws_api_gateway_method_id" {
  value       = aws_api_gateway_method.anime_resource_get_method.id
}

output "aws_api_gateway_integration_id" {
  value       = aws_api_gateway_integration.anime_resource_get_method_integration.id
}

output "aws_api_gateway_method_response_id" {
  value       = aws_api_gateway_method_response.anime_200_method_response.id
}

output "aws_api_gateway_integration_response_id" {
  value       = aws_api_gateway_integration_response.anime_200_integration_response.id
}
``` 

### Usage

Go back to the root level and open the `main.tf`, replace the content with:

``` json
terraform {
  required_providers {
    aws = {
        source = "hashicorp/aws"
        version = "4.22.0"
    }
  }
}

provider "aws"{
  region = "us-east-2"
}

resource "aws_api_gateway_rest_api" "my_api" {
  name        = "my-anime-quotes-api"
  description = "My Anime Quotes API"

  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

module "naruto" {
  source           = "./modules/api_gateway_get_resource"
  anime_name       = "naruto"
  rest_api_id      = aws_api_gateway_rest_api.my_api.id
  root_resource_id = aws_api_gateway_rest_api.my_api.root_resource_id
}

module "inuyasha" {
  source           = "./modules/api_gateway_get_resource"
  anime_name       = "inuyasha"
  rest_api_id      = aws_api_gateway_rest_api.my_api.id
  root_resource_id = aws_api_gateway_rest_api.my_api.root_resource_id
}

resource "aws_api_gateway_deployment" "deployment_api" {
  rest_api_id = aws_api_gateway_rest_api.my_api.id

  triggers = {
    redeployment = sha1(jsonencode([
      module.naruto.aws_api_gateway_resource_id,
      module.naruto.aws_api_gateway_method_id,
      module.naruto.aws_api_gateway_integration_id,
      module.naruto.aws_api_gateway_method_response_id,
      module.naruto.aws_api_gateway_integration_response_id,

      module.inuyasha.aws_api_gateway_resource_id,
      module.inuyasha.aws_api_gateway_method_id,
      module.inuyasha.aws_api_gateway_integration_id,
      module.inuyasha.aws_api_gateway_method_response_id,
      module.inuyasha.aws_api_gateway_integration_response_id
    ]))
  }
    lifecycle {
    create_before_destroy = true
  }
}

resource "aws_api_gateway_stage" "stage_api" {
  rest_api_id   = aws_api_gateway_rest_api.my_api.id
  stage_name    = "anime"
  deployment_id = aws_api_gateway_deployment.deployment_api.id
  description   = "anime"
}

resource "aws_api_gateway_method_settings" "method_settings" {
  rest_api_id = aws_api_gateway_rest_api.my_api.id
  stage_name  = aws_api_gateway_stage.stage_api.stage_name
  method_path = "*/*"

  settings {
    caching_enabled        = false 
    cache_ttl_in_seconds   = 120
    metrics_enabled        = true
    logging_level          = "INFO"
  }
}
``` 

Run the following command to destroy the previous resources:

```powershell
terraform plan -destroy -out app.tfplan
terraform apply -destroy 'app.tfplan'
``` 

Then, run:

```powershell
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
``` 

Now, in the AWS Console, we will see:

![inuyasha.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1662857079831/-blz0FAKi.PNG align="left")

I hope this simple example was enough to understand how powerful Terraform modules can be. The final source code is available [here](https://github.com/raulnq/terraform-sandbox/tree/modules). Thanks, and happy coding.