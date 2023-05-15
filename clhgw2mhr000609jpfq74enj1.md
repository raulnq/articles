---
title: "How to Deploy a .NET App on AWS Elastic Beanstalk using Terraform (Windows Server)"
datePublished: Tue May 09 2023 23:14:34 GMT+0000 (Coordinated Universal Time)
cuid: clhgw2mhr000609jpfq74enj1
slug: how-to-deploy-a-net-app-on-aws-elastic-beanstalk-using-terraform-windows-server
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683673461405/ed462407-1107-4674-970b-62263f2c8c7f.png
tags: aws, net, elastic-beanstalk, windows-server

---

A couple of weeks ago, we discussed [how to deploy a .NET application on AWS Elastic Beanstalk using Terraform](https://blog.raulnq.com/how-to-deploy-a-net-app-on-aws-elastic-beanstalk-using-terraform), but on a Linux server. After reviewing the official [documentation](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/dotnet-core-tutorial.html), it appears somewhat outdated when it comes to using Windows Server as a platform. As a result, we aim to modify our initial project to accommodate this support.

So, download or clone [this](https://github.com/raulnq/aws-beanstalk) repository. Open the `variables.tf` file and append the following content at the end:

```json
variable "platform" {
  type = string
}
```

Open the `main.tf` file and replace its content with:

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
  region  = "us-east-2"
}

resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket
}

resource "aws_s3_object" "bucket_object" {
  bucket = aws_s3_bucket.bucket.id
  key    = "app-${uuid()}.zip"
  source = "app.zip"
}


resource "aws_elastic_beanstalk_application" "application" {
  name        = var.application
}

resource "aws_elastic_beanstalk_environment" "environment" {
  name                = "${var.application}-env"
  application         = aws_elastic_beanstalk_application.application.name
  solution_stack_name = var.platform
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
    value     =  "aws-elasticbeanstalk-ec2-role"
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
    value     = "aws-elasticbeanstalk-service-role"
  }

  setting {
      namespace = "aws:elasticbeanstalk:environment:process:default"
      name      = "HealthCheckPath"
      value     = var.health_check_path
  }
}

resource "aws_elastic_beanstalk_application_version" "version" {
  bucket      = aws_s3_bucket.bucket.id
  key         = aws_s3_object.bucket_object.id
  application = aws_elastic_beanstalk_application.application.name
  name        = "${var.application}-app-${uuid()}"
}
```

With this new variable, we are parameterizing the platform that will be used. It's time to add the [deployment manifest file](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/deployment-beanstalk-custom-netcore.html). Create a file named `aws-windows-deployment-manifest.json` at the solution level with the following content:

```json
{
    "manifestVersion": 1,
    "deployments": {
        "aspNetCoreWeb": [
        {
            "name": "dotnet-api",
            "parameters": {
                "appBundle": "site.zip",
                "iisPath": "/",
                "iisWebSite": "Default Web Site"
            }
        }
        ]
    }
}
```

Run the following commands to create the bundle to deploy:

```bash
mkdir terraform/publish
mkdir terraform/bundle
dotnet publish ./src/WeatherApi.csproj --output "terraform/publish" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained
Compress-Archive -Path terraform/publish/* -DestinationPath terraform/bundle/site.zip
copy .\aws-windows-deployment-manifest.json .\terraform\bundle
Compress-Archive -Path terraform/bundle/* -DestinationPath terraform/app.zip
```

**Tip**: To deploy .NET versions such as .NET 5 or .NET 7, you can create a self-contained artifact using a command like:

```bash
dotnet publish ./src/WeatherApi5/WeatherApi5.csproj --output "terraform/publish" --configuration "Release" --framework "net5.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --self-contained /p:PublishTrimmed=false
```

Run the following commands to create the environment, application, and application version:

```bash
cd terraform
terraform init
terraform plan -out app.tfplan -var="health_check_path=/swagger/index.html" -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>" -var="platform=64bit Windows Server 2019 v2.11.3 running IIS 10.0"
terraform apply 'app.tfplan'
```

And finally, the deployment itself of our application version:

```bash
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENV_NAME> --version-label <OUTPUT_APP_VERSION>
```

And that's it, the same application we deployed to a Linux server now running in a Windows Server under AWS Elastic Beanstalk. The code and scripts are available [here](https://github.com/raulnq/aws-beanstalk/tree/windows). Thanks, and happy coding.