---
title: "Scaling out SignalR with Redis Backplane"
datePublished: Sat Jun 10 2023 05:21:38 GMT+0000 (Coordinated Universal Time)
cuid: clipju2tm000009jucge1aoot
slug: scaling-out-signalr-with-redis-backplane
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686345713362/585ba7e3-99ac-40b4-825a-170dbf997067.png
tags: aws, reactjs, net, terraform, signals

---

Scale-out generally involves increasing the number of application instances and placing a load balancer in front of them. This approach presents challenges when using SignalR, as each instance will only maintain records of its connected clients.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686354735094/36399e86-a512-43bf-b31b-377c96832930.png align="center")

In the image above, clients #1 and #2 are connected to server #1, while client #3 is connected to server #2. Therefore, when calling `Clients.All.SendAsync()`, messages are only sent to clients connected to the respective instance. Fortunately, the Redis backplane provides a solution to this problem.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686354804185/c9aa3763-81f1-4258-ac40-81874085c03f.png align="center")

The Redis backplane employs the publish/subscribe pattern to synchronize the servers, working as follows:

* All servers subscribe to the Redis backplane.
    
* Whenever a new message arrives, the server holding the connection sends the message to the Redis backplane.
    
* The Redis backplane publishes the message.
    
* The servers subscribed to the Redis backplane receive the message and forward it to their connected clients.
    

In our previous article, [Getting Started with SignalR in .NET 6](https://blog.raulnq.com/getting-started-with-signalr-in-net-6), we developed a basic SignalR application. And now we intend to scale it out. To accomplish this, we will use [Terraform](https://developer.hashicorp.com/terraform/intro), [AWS Elastic Beanstalk](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/Welcome.html), and [Amazon ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html).

### Pre-requisites

* Have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Set up your [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) locally.
    
* Install [**Terraform CLI**](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    
* Have an [**AWS EC2 Key Pair**](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html).
    
* Have an [Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) and [Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html).
    

### The Application

Download the code from [here](https://github.com/raulnq/signalr). Open the solution and within the `WebAPI` project, add a Procfile file with the following content:

```bash
web: dotnet exec ./WebAPI.dll --urls http://0.0.0.0:5000/
```

Run the following command to create the artifact to deploy:

```bash
mkdir terraform/publish
dotnet publish ./WebAPI/WebAPI.csproj --output "terraform/publish" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime linux-x64 --no-self-contained
Compress-Archive -Path terraform/publish/* -DestinationPath terraform/app.zip
```

### The Terraform script

Under the `terraform` folder, create a `variables.tf` file with the following content:

```json
variable "aws_region" {
    type    = string
    default = "us-east-2"
}
variable "aws_profile" {
    type    = string
    default = "default"
}

variable "application" {
    type    = string
    default = "<MY_APP_NAME>"
}

variable "vpc" {
    type    = string
    default = "<MY_VPC>"
}

variable "subnets" {
    type    = list
    default = ["<MY_SUBNET>"]
}

variable "keypair"  {
    type    = string
    default = "<MY_KEY_PAIR>"
}
```

Create a `main.tf` file as follow:

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
  region      = var.aws_region
  profile     = var.aws_profile
}

resource "aws_s3_bucket" "bucket" {
  bucket = "${var.application}-bucket"
}

resource "aws_s3_object" "bucket_object" {
  bucket = aws_s3_bucket.bucket.id
  key    = "app-${uuid()}.zip"
  source = "app.zip"
}

resource "aws_security_group" "security_group" {
  vpc_id       = var.vpc
  name         = var.application
  description  = "Security group for elastic beanstalk app ${var.application}"

  ingress {
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }  

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
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
  name               = "aws-elasticbeanstalk-service-role-${var.application}"
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
  name               = "aws-elasticbeanstalk-ec2-role-${var.application}"
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
  name = aws_iam_role.role.name
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
    value     = var.vpc
  }

  setting {

    namespace = "aws:ec2:vpc"
    name      = "Subnets"
    value     =  join(",",var.subnets) 
  }

  setting {
    namespace = "aws:ec2:vpc"
    name      = "ELBScheme"
    value     = "internal"
  }

  setting {
    namespace = "aws:ec2:vpc"
    name      = "ELBSubnets"
    value     = join(",",var.subnets) 
  }

  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "IamInstanceProfile"
    value     = aws_iam_role.role.name
  }

  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "SecurityGroups"
    value     =  aws_security_group.security_group.id
  }

  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "InstanceType"
    value     = "t2.small"
  }

  setting {
    namespace = "aws:autoscaling:asg"
    name      = "MinSize"
    value     = 2
  }

  setting {
    namespace = "aws:autoscaling:asg"
    name      = "MaxSize"
    value     = 2
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
      value     = "/swagger/index.html"
  }

  setting {
      namespace = "aws:elasticbeanstalk:environment:process:default"
      name      = "StickinessEnabled"
      value     = "true"
  }

    setting {
    namespace = "aws:elasticbeanstalk:environment:proxy"
    name      = "ProxyServer"
    value     = "none"
  }
}

resource "aws_elastic_beanstalk_application_version" "version" {
  bucket      = aws_s3_bucket.bucket.id
  key         = aws_s3_object.bucket_object.id
  application = aws_elastic_beanstalk_application.application.name
  name        = "${var.application}-app-${uuid()}"
}

resource "aws_security_group" "security_group_redis" {
  name         = "${var.application}-redis"
  description  = "Security group for ${var.application} redis"
  vpc_id       = var.vpc

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_elasticache_subnet_group" "subnet_group" {
  name       = "${var.application}-redis-subnet"
  subnet_ids = var.subnets
}

resource "aws_elasticache_replication_group" "cluster" {
  replication_group_id       = "${var.application}-redis"
  description                = "Redis for ${var.application}"
  node_type                  = "cache.t2.small"
  engine_version             = "7.0"
  engine                     = "redis"
  port                       = 6379
  parameter_group_name       = "default.redis7.cluster.on"
  automatic_failover_enabled = true
  subnet_group_name          = aws_elasticache_subnet_group.subnet_group.name
  num_node_groups            = 1
  replicas_per_node_group    = 1
  security_group_ids         = [aws_security_group.security_group_redis.id]
  transit_encryption_enabled = true
}
```

The majority of the script's explanation can be found [here](https://blog.raulnq.com/how-to-deploy-a-net-app-on-aws-elastic-beanstalk-using-terraform). If we use Server-Sent Events or Long Polling, the option `StickinessEnabled` must be configured in the load balancer. About Redis, there are three configurations:

* **Redis Cluster**: Single node, no replication.
    
* **Redis Replication Group Cluster mode disabled**: One write node and multiple replica nodes.
    
* **Redis Replication Group Cluster mode enabled**: Multiple write nodes (data partitioning) and multiple replica nodes.
    

For our use case, we are choosing the second configuration, which consists of one write node and one replica node.

### The Deployment

Run the following commands:

```bash
cd terraform
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
```

Wait until the `apply` command ends, copy the outputs into the following script, and run it:

```bash
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENVIRONMENT_NAME> --version-label <OUTPUT_APPLICATION_VERSION_NAME>
```

Open a browser and navigate to `http://<OUTPUT_ENVIRONMENT_CNAME>/swagger` to see the application.

### Adding the Redis backplane

At the solution level, run `dotnet add WebAPI package Microsoft.AspNetCore.SignalR.StackExchangeRedis --version 6.0.16`, then modify the `Program.cs` file as follows:

```csharp
using WebAPI;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddCors();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddSignalR().AddStackExchangeRedis("<OUTPUT_REDIS_CONFIGURATION_ENDPOINT>:6379,ssl=True,abortConnect=False"); ;

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseCors(cp => cp
    .AllowAnyHeader()
    .SetIsOriginAllowed(origin => true)
    .AllowCredentials()
);
app.MapHub<ChatHub>("/chathub");

app.MapControllers();

app.Run();
```

Remove the existing artifact and create a new one by executing:

```bash
dotnet publish ./WebAPI/WebAPI.csproj --output "terraform/publish" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime linux-x64 --no-self-contained
Compress-Archive -Path terraform/publish/* -DestinationPath terraform/app.zip
```

Finally, redeploy the application with the new artifact:

```bash
cd terraform
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENVIRONMENT_NAME> --version-label <OUTPUT_APPLICATION_VERSION_NAME>
```

Open the `client.html` file and change the connection URL with `http://<OUTPUT_ENVIRONMENT_CNAME>/chathub`, then open the file in a browser. In conclusion, scaling out SignalR applications can be achieved by using a Redis backplane to synchronize messages across multiple instances. All the code is available [here](https://github.com/raulnq/signalr/tree/backplane). Thanks, and happy coding.