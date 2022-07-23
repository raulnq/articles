## Deploying AWS Lambda Functions with Terraform

In a previous post, we reviewed how to [build AWS Lambda Functions with .NET 6](https://blog.raulnq.com/building-aws-lambda-functions-with-net-6). In today's post, we will see how to deploy them using [Terraform](https://blog.raulnq.com/terraform-resources-variables-outputs-and-locals).

Before starting, we need to fulfill a few pre-requisites:

- Have a [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
- Setup your [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) locally.
- Install [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli).

As the first step, clone [this](https://github.com/raulnq/aws-lambda-sandbox) GitHub repository. Create a `terraform` folder with a `main.tf` file and add the Terraform Provider for AWS :

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
  region= "us-east-2"
  profile = "default"
}
``` 

Next, we will add the resource to create the IAM Role to be used by the Lamba and the permissions attached to it:

```json
resource "aws_iam_role" "role" {
name   = "ASPNETCoreWebAPI_role"
assume_role_policy = <<EOF
{
 "Version": "2012-10-17",
 "Statement": [
   {
     "Action": "sts:AssumeRole",
     "Principal": {
       "Service": "lambda.amazonaws.com"
     },
     "Effect": "Allow",
     "Sid": ""
   }
 ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "policy_attachment" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
``` 

To end with the Terraform script, we will add the resource to create the Lambda and the URL to expose it (an output was added to see the URL of the lambda at the end of the deployment):

```json
resource "aws_lambda_function" "lambda" {
  filename      = "ASPNETCoreWebAPI.zip"
  function_name = "ASPNETCoreWebAPI"
  role          = aws_iam_role.role.arn
  handler       = "ASPNETCoreWebAPI"
  memory_size   = 256
  source_code_hash = filebase64sha256("ASPNETCoreWebAPI.zip")
  timeout = 30
  runtime = "dotnet6"
}

resource "aws_lambda_function_url" "function_url" {
   function_name      = aws_lambda_function.lambda.function_name
   authorization_type = "NONE"
 }

output "lambda_url" {
  value = aws_lambda_function_url.function_url.function_url
}
``` 

Run the following commands at the solution level to generate the zip file:

```powershell
mkdir terraform/publish
dotnet publish "ASPNETCoreWebAPI" --output "terraform/publish" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime linux-x64
Compress-Archive -Path terraform/publish/* -DestinationPath terraform/ASPNETCoreWebAPI.zip
``` 

And finally, we will run the Terraform commands to start the deployment:

```powershell
cd terraform
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
``` 

At the end of the outputs of the last command, we will find the URL of the application:

```powershell
aws_iam_role.role: Creating...
aws_iam_role.role: Creation complete after 2s [id=ASPNETCoreWebAPI_role]
aws_iam_role_policy_attachment.policy_attachment: Creating...
aws_lambda_function.lambda: Creating...
aws_iam_role_policy_attachment.policy_attachment: Creation complete after 0s [id=ASPNETCoreWebAPI_role-20220716154419988700000001]
aws_lambda_function.lambda: Still creating... [10s elapsed]
aws_lambda_function.lambda: Still creating... [20s elapsed]
aws_lambda_function.lambda: Creation complete after 25s [id=ASPNETCoreWebAPI]
aws_lambda_function_url.function_url: Creating...
aws_lambda_function_url.function_url: Creation complete after 1s [id=ASPNETCoreWebAPI]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

lambda_url = "https://siaauogvzqot475ipvmvnlolx40ftrnd.lambda-url.us-east-2.on.aws/"
``` 

Open in your browser that URL adding `swagger` at the end:


![terraform_lambda.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1657986657154/jG5LDwOUf.PNG align="left")

To clean up all the created resources, run:

```powershell
terraform plan -out app.tfplan -destroy
terraform apply -destroy 'app.tfplan'  
``` 

You can see the final ` main.tf`  file [here](https://github.com/raulnq/aws-lambda-sandbox/blob/terraform/terraform/main.tf). Thanks, and happy coding.






