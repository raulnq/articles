---
title: "Customizing our AWS Elastic Beanstalk environment with .ebextensions"
datePublished: Sun May 28 2023 23:53:40 GMT+0000 (Coordinated Universal Time)
cuid: cli82u30r00000ajyb5n2eiqc
slug: customizing-our-aws-elastic-beanstalk-environment-with-ebextensions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1685149113217/9f7aec55-7939-4b1d-9aa6-83c7afbbc793.png
tags: aws, net, terraform, elastic-beanstalk

---

Deploying a web application on AWS Elastic Beanstalk can sometimes require environment-specific customizations, such as configuring a database, modifying server settings, or installing additional software. [Elastic Beanstalk extensions](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html), or .ebextensions, are a set of configuration files used to customize and configure AWS Elastic Beanstalk environments. These files allow us easily tailor our environment to accommodate these requirements, ensuring a seamless deployment and optimal performance for your application.

To use .ebextensions, create a folder named `.ebextensions` in the root directory of our source bundle. Inside this folder, add one or more configuration files with a `.config` extension. These files should be written in YAML or JSON format and contain instructions to customize your AWS Elastic Beanstalk environment. When you deploy your application, Elastic Beanstalk will read and apply the configurations specified in these files.

The sections of those `.config` files can be grouped into three categories:

* [Options settings](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions-optionsettings.html): Used to change any AWS Elastic Beanstalk [configuration option](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-options-general.html).
    
* [Custom resources](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-resources.html): Let us define additional AWS resources beyond the functionality provided by configuration options. You can add and configure any resources supported by AWS CloudFormation.
    
* [Software customization](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html): Let us configure the EC2 instances. Every time a server is launched in your environment, AWS Elastic Beanstalk runs the operations defined to prepare the operating system.
    

In a previous post, we discussed [**How to Deploy a Windows Service on AWS Elastic Beanstalk using Terraform**](https://blog.raulnq.com/how-to-deploy-a-windows-service-on-aws-elastic-beanstalk-using-terraform). Let's add a couple of requirements to our previous exercise:

* Disable the operative system firewall.
    
* Redirect HTTP traffic to HTTPS at the load balancer level.
    

Create a `.ebextensions` folder at the solution level, and within it, create a `disable-firewall.config` file with the following content:

```yaml
files:
  "C:\\scripts\\firewall.ps1":
    content: |
      Set-NetFirewallProfile -Enabled False

commands:
  disable_firewall:
    command: powershell.exe -ExecutionPolicy Bypass -Command "C:\\scripts\\firewall.ps1"
    ignoreErrors: false
    waitAfterCompletion: 5
```

`Files` and `Commands` are known as Keys, and all of them are:

* [`Packages`](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html#windows-packages): To download and install prepackaged applications and components.
    
* [`Sources`](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html#windows-sources): To download an archive file from a public URL and unpack it in a target directory.
    
* [`Files`](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html#windows-files): To create files on the EC2 instance.
    
* [`Commands`](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html#windows-commands): To execute commands on the EC2 instance.
    
* [`Services`](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html#windows-services): To define which services should be started or stopped when the instance is launched.
    
* [`Container commands`](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html#windows-container-commands): To execute commands that affect your application source code. Container commands run after the application and web server have been set up and the application version archive has been extracted, but before the application version is deployed.
    

To redirect HTTP traffic to HTTPS, we need to modify our Terraform main.tf file to add the listener on port 433:

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
      namespace = "aws:elasticbeanstalk:environment:process:default"
      name      = "HealthCheckPath"
      value     = var.health_check_path
  }

  setting {
      namespace = "aws:elbv2:listener:443"
      name      = "Protocol"
      value     = "HTTPS"
  }

  setting {
      namespace = "aws:elbv2:listener:443"
      name      = "SSLCertificateArns"
      value     = var.ssl_certificate
  }

  setting {
      namespace = "aws:elbv2:listener:443"
      name      = "SSLPolicy"
      value     = "ELBSecurityPolicy-2016-08"
  }
}

resource "aws_elastic_beanstalk_application_version" "version" {
  bucket      = aws_s3_bucket.bucket.id
  key         = aws_s3_object.bucket_object.id
  application = aws_elastic_beanstalk_application.application.name
  name        = "${var.application}-app-${uuid()}"
}
```

Under `.ebextensions` folder, add an `alb-http-to-https-redirection.config` file as follows:

```yaml
Resources:
  AWSEBV2LoadBalancerListener:
   Type: AWS::ElasticLoadBalancingV2::Listener
   Properties:
     LoadBalancerArn:
       Ref: AWSEBV2LoadBalancer
     Port: 80
     Protocol: HTTP
     DefaultActions:
       - Type: redirect
         RedirectConfig:
           Host: "#{host}"
           Path: "/#{path}"
           Port: "443"
           Protocol: "HTTPS"
           Query: "#{query}"
           StatusCode: "HTTP_301"
```

The resources that AWS Elastic Beanstalk creates for our environment can be found [here](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-format-resources-eb.html). To create the bundle for deployment, execute the following commands:

```bash
dotnet publish ./src/WeatherApi/WeatherApi.csproj --output "terraform/publish/webapi" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained

Compress-Archive -Path terraform/publish/webapi/* -DestinationPath terraform/bundle/site.zip

dotnet publish ./src/WeatherWs/WeatherWs.csproj --output "terraform/bundle/ws" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained

copy .\install.ps1 .\terraform\bundle
copy .\aws-windows-deployment-manifest.json .\terraform\bundle

mkdir .\terraform\bundle\.ebextensions
copy .\.ebextensions\* .\terraform\bundle\.ebextensions

Compress-Archive -Path terraform/bundle/* -DestinationPath terraform/app.zip
```

The contents of the bundle will appear as follows:

```bash
|-- .ebextensions
|-- ws
|-- aws-windows-deployment-manifest.json
|-- install.ps1
`-- site.zip
```

Run the terraform scripts with the following commands:

```bash
cd terraform
terraform init
terraform plan -out app.tfplan -var="health_check_path=/swagger/index.html" -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>" -var="platform=64bit Windows Server 2019 v2.11.3 running IIS 10.0" -var ssl_certificate="<MY_SSL_CERTIFICATE>"
terraform apply 'app.tfplan' 
```

And deploy the application version with the following command:

```bash
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENV_NAME> --version-label <OUTPUT_APP_VERSION>
```

In conclusion, AWS Elastic Beanstalk's .ebextensions provide a powerful and flexible way to customize your environment to meet specific requirements. By following the steps outlined in this article, you can easily configure various aspects of your application, such as disabling the operating system firewall and redirecting HTTP traffic to HTTPS. With a better understanding of .ebextensions, you can now further explore and leverage their capabilities to enhance your application's deployment and performance. You can find several `config` files under [this](https://github.com/awsdocs/elastic-beanstalk-samples/tree/main/configuration-files) repository. The code and scripts used in this article are available [here](https://github.com/raulnq/aws-beanstalk/tree/ebextensions). Thanks, and happy coding.