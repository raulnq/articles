---
title: "How to Deploy a .NET App on AWS Elastic Beanstalk using Terraform"
datePublished: Sun Apr 30 2023 15:38:37 GMT+0000 (Coordinated Universal Time)
cuid: clh3ktlhs000109mjgvib1g07
slug: how-to-deploy-a-net-app-on-aws-elastic-beanstalk-using-terraform
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682801875582/9ed0152f-940c-4d2f-aa03-3b11dee5c6e6.png
tags: aws, net, terraform, elastic-beanstalk, aws-elastic-beanstalk

---

In certain instances, we might prefer deploying our applications on an [EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html) instance without the burden of managing the associated infrastructure. [AWS Elastic Beanstalk](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/Welcome.html) provides streamlined deployment, automatic scaling, and application management, enabling developers to concentrate on coding without being concerned about infrastructure configuration and management. So, let's explore how to deploy it using [Terraform](https://developer.hashicorp.com/terraform/intro).

### Pre-requisites

* Have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Set up your [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) locally.
    
* Install [**Terraform CLI**](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    
* Have an [AWS EC2 Key Pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html) ready to use.
    

### The application

We'll use a standard .NET 6 application, which can be downloaded [here](https://github.com/raulnq/aws-beanstalk/tree/main/src). It is important to note that the `Procfile` is used to instruct Elastic Beanstalk which applications to run, specifically for the .NET on a Linux environment:

```bash
web: dotnet exec ./WeatherApi.dll --urls http://0.0.0.0:5000/
```

Run the following command to create the artifact to deploy:

```bash
mkdir terraform/publish
dotnet publish ./src/WeatherApi.csproj --output "terraform/publish" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime linux-x64 --no-self-contained
Compress-Archive -Path terraform/publish/* -DestinationPath terraform/app.zip
```

### Defining the inputs

Create a `variables.tf` file containing the following content:

```json
variable "application" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "ec2_subnets" {
  type = string
}

variable "elb_subnets" {
  type = string
}

variable "instance_type" {
  type = string
}

variable "keypair" {
  type = string
}

variable "bucket" {
  type = string
}
```

### Publishing the artifact to S3

Create a `main.tf` with the following initial content:

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

resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket
}

resource "aws_s3_object" "bucket_object" {
  bucket = aws_s3_bucket.bucket.id
  key    = "app-${uuid()}.zip"
  source = "app.zip"
}
```

### Adding the Elastic Beanstalk Application and Environment

Add to the `main.tf` file the following content:

```json
data "aws_iam_policy_document" "assume_service_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["elasticbeanstalk.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]

    condition {
        test     = "StringEquals"
        variable = "sts:ExternalId"

        values = [
        "elasticbeanstalk"
        ]
    }
  }
}

resource "aws_iam_role" "service_role" {
  name               = "aws-elasticbeanstalk-service-role"
  assume_role_policy = data.aws_iam_policy_document.assume_service_role.json
}

resource "aws_iam_role_policy_attachment" "AWSElasticBeanstalkEnhancedHealth-attach" {
  role       = aws_iam_role.service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSElasticBeanstalkEnhancedHealth"
}

resource "aws_iam_role_policy_attachment" "AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy-attach" {
  role       = aws_iam_role.service_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy"
}

data "aws_iam_policy_document" "assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "role" {
  name               = "aws-elasticbeanstalk-ec2-role"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

resource "aws_iam_role_policy_attachment" "CloudWatchFullAccess-attach" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchFullAccess"
}

resource "aws_iam_role_policy_attachment" "AWSElasticBeanstalkWebTier-attach" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSElasticBeanstalkWebTier"
}

resource "aws_iam_role_policy_attachment" "AWSElasticBeanstalkWorkerTier-attach" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSElasticBeanstalkWorkerTier"
}

resource "aws_iam_role_policy_attachment" "AWSElasticBeanstalkMulticontainerDocker-attach" {
  role       = aws_iam_role.role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSElasticBeanstalkMulticontainerDocker"
}

resource "aws_iam_instance_profile" "instance_profile" {
  name = "aws-elasticbeanstalk-ec2-role"
  role = aws_iam_role.role.name
}

resource "aws_elastic_beanstalk_application" "application" {
  name        = var.application
}

resource "aws_elastic_beanstalk_environment" "environment" {
  name                = "${var.application}-env"
  application         = aws_elastic_beanstalk_application.application.name
  solution_stack_name = "64bit Amazon Linux 2 v2.5.2 running .NET Core"
  tier                = "WebServer"

  setting {
    namespace = "aws:ec2:vpc"
    name      = "VPCId"
    value     = var.vpc_id
  }

  setting {
    namespace = "aws:ec2:vpc"
    name      = "Subnets"
    value     = var.ec2_subnets
  }

  setting {
    namespace = "aws:ec2:vpc"
    name      = "ELBScheme"
    value     = "internet facing"
  }

  setting {
    namespace = "aws:ec2:vpc"
    name      = "ELBSubnets"
    value     = var.elb_subnets
  }

  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "IamInstanceProfile"
    value     =  aws_iam_role.role.name
  }

  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "InstanceType"
    value     = var.instance_type
  }

  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "EC2KeyName"
    value     = var.keypair
  }

  setting {
    namespace = "aws:elasticbeanstalk:environment"
    name      = "LoadBalancerType"
    value     = "application"
  }

  setting {
    namespace = "aws:elasticbeanstalk:environment"
    name      = "ServiceRole"
    value     = aws_iam_role.service_role.name
  }

  setting {
    namespace = "aws:elasticbeanstalk:environment:proxy"
    name      = "ProxyServer"
    value     = "none"
  }

  setting {
      namespace = "aws:elasticbeanstalk:environment:process:default"
      name      = "HealthCheckPath"
      value     = "/swagger/index.html"
  }
}
```

Depending on our requirements, we can refer to the [documentation](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-general.html) for all the available options for the environment. Let's discuss the ones we are using here:

* `VPCID`: The ID for your Amazon VPC.
    
* `Subnets`: The IDs of the Auto Scaling group subnet or subnets.
    
* `ELBScheme`: Specify `internal` if we wish to create an internal load balancer in your Amazon VPC, which will prevent your Elastic Beanstalk application from being accessed outside your Amazon VPC.
    
* `ELBSubnets`: The IDs of the subnet or subnets for the elastic load balancer.
    
* `IamInstanceProfile`: An instance profile enables IAM users and AWS services to access temporary security credentials to make AWS API calls.
    
* `InstanceType`: The instance type that's used to run our application in an Elastic Beanstalk environment.
    
* `EC2KeyName`: Key pair used to securely log into your EC2 instance.
    
* `LoadBalancerType`: The type of [load balancer](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.managing.elb.html) for our environment.
    
* `ServiceRole`: The [service role](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/concepts-roles-service.html) is the IAM role that Elastic Beanstalk assumes when calling other services on your behalf.
    
* `HealthCheckPath`: The path that HTTP requests for health checks are sent to.
    
* `ProxyServer`: [Elastic Beanstalk](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-specific.html#command-options-dotnet-core-linux) comes with Nginx as a reverse proxy server that we can use if we want.
    

### Adding the Application Version

An [Application Version](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/applications-versions.html) is a specific, labeled iteration of deployable code for an application. Add to the `main.tf` file the following content:

```json
resource "aws_elastic_beanstalk_application_version" "version" {
  bucket      = aws_s3_bucket.bucket.id
  key         = aws_s3_object.bucket_object.id
  application = aws_elastic_beanstalk_application.application.name
  name        = "${var.application}-app-${uuid()}"
}
```

### Defining the outputs

Create a `outputs.tf` file containing the following content:

```json
output "app_version" {
  value = aws_elastic_beanstalk_application_version.version.name
}
output "env_name" {
  value = aws_elastic_beanstalk_environment.environment.name
}
output "cname" {
  value = aws_elastic_beanstalk_environment.environment.cname
}
```

### The Deployment

Run the following commands:

```bash
cd terraform
terraform init
terraform plan -out app.tfplan -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>"
terraform apply 'app.tfplan'
```

Terraform will create the environment, application, and application version, but it will not deploy the application. To do this, we need to use the AWS CLI to update the environment's application version (use the outputs of the previous command):

```bash
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENV_NAME> --version-label <OUTPUT_APP_VERSION>
```

Open the URL `http://<OUTPUT_CNAME>/swagger/index.html` to see our application up and running. To clean up all the created resources, execute the following commands:

```bash
terraform plan -destroy -out app.tfplan -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>"
terraform apply -destroy 'app.tfplan'
```

You can see the code and scripts [here](https://github.com/raulnq/aws-beanstalk). Thanks, and happy coding.