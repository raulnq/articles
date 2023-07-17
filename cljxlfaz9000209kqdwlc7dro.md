---
title: "AWS Elastic Beanstalk: How to Include an Additional Application Load Balancer"
datePublished: Tue Jul 11 2023 01:08:00 GMT+0000 (Coordinated Universal Time)
cuid: cljxlfaz9000209kqdwlc7dro
slug: aws-elastic-beanstalk-how-to-include-an-additional-application-load-balancer
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688837152029/0f745fc7-0d03-454c-bda1-bc4ae16de8c3.png
tags: aws, net, aws-elastic-beanstalk

---

AWS Elastic Beanstalk is a fully managed service that allows developers to easily deploy, manage, and scale applications in the AWS cloud. One of the key components of Elastic Beanstalk is the Application Load Balancer, which ensures the efficient distribution of incoming traffic across multiple instances. In this article, we will examine how simple it is to incorporate an additional Application Load Balancer using .ebextensions. If you are new to this topic, you can refer to our previous article, [Customizing Your AWS Elastic Beanstalk Environment with .ebextensions.](https://blog.raulnq.com/customizing-our-aws-elastic-beanstalk-environment-with-ebextensions)

Why do we need two load balancers? Using both internal and internet-facing load balancers can be beneficial for applications that require secure internal communication while also serving external traffic. Internal load balancers distribute traffic among instances within a private network, while internet-facing load balancers handle traffic from the public internet. This setup can enhance security and provide better control over traffic routing.

We will begin with the code featured in the previous .ebextensions article, which can be downloaded [here](https://github.com/raulnq/aws-beanstalk/tree/ebextensions). Navigate to the `Terraform` folder and open the `main.tf` file. Locate the `aws_elastic_beanstalk_environment` resource and append the following lines at the end:

```bash
  setting {
      namespace = "aws:elasticbeanstalk:customoption"
      name      = "PrivateELBSubnets"
      value     = var.private_elb_subnets
  }

  setting {
      namespace = "aws:elasticbeanstalk:customoption"
      name      = "PrivateSSLCertificateArns"
      value     = var.private_ssl_certificate
  }
```

Open the `variables.tf` file and add the following:

```bash
variable "private_elb_subnets" {
  type = string
}

variable "private_ssl_certificate" {
  type = string
}
```

Navigate to the `.ebextensions` folder and create a file called `add-load-balancer.config`, then insert the following content:

```bash
Resources:
  AdditionalAWSEBLoadBalancerSecurityGroup:
    Properties:
      GroupDescription: Load Balancer Security Group
      VpcId:
        Fn::GetOptionSetting: 
          Namespace: 'aws:ec2:vpc'
          OptionName: 'VPCId'
          DefaultValue: 'abc'
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0
          FromPort: '80'
          ToPort: '80'
          IpProtocol: tcp
        - CidrIp: 0.0.0.0/0
          FromPort: '443'
          ToPort: '443'
          IpProtocol: tcp
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0
          FromPort: '80'
          ToPort: '80'
          IpProtocol: tcp
    Type: 'AWS::EC2::SecurityGroup'
  AdditionalAWSEBV2LoadBalancer:
    Properties:
      SecurityGroups:
        - { "Ref" : "AdditionalAWSEBLoadBalancerSecurityGroup" }
      Subnets:
        Fn::Split:
          - ','
          - Fn::GetOptionSetting:
              Namespace: 'aws:elasticbeanstalk:customoption'
              OptionName: 'PublicELBSubnets'
              DefaultValue: 'abc'
      Scheme: internet-facing
    Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
  AddtionalAWSEBV2LoadBalancerListener:
    Properties:
      LoadBalancerArn: { "Ref" : "AdditionalAWSEBV2LoadBalancer" }
      DefaultActions:
        - Type: redirect
          RedirectConfig:
            Path: '/#{path}'
            Query: '#{query}'
            Port: '443'
            Host: '#{host}'
            Protocol: HTTPS
            StatusCode: HTTP_301
      Port: 80
      Protocol: HTTP
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
  AdditionalAWSEBV2LoadBalancerListener443:
    Properties:
      SslPolicy: ELBSecurityPolicy-2016-08
      LoadBalancerArn: { "Ref" : "AdditionalAWSEBV2LoadBalancer" }
      Port: '443'
      DefaultActions:
        - TargetGroupArn: { "Ref" : "AdditionalAWSEBV2LoadBalancerTargetGroup" }
          Type: forward
      Certificates:
        - CertificateArn:
            Fn::GetOptionSetting: 
              Namespace: 'aws:elasticbeanstalk:customoption'
              OptionName: 'PublicSSLCertificateArns'
              DefaultValue: 'abc'
      Protocol: HTTPS
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
  AdditionalAWSEBV2LoadBalancerTargetGroup:
    Properties:
      HealthCheckIntervalSeconds: 15
      VpcId:
        Fn::GetOptionSetting: 
          Namespace: 'aws:ec2:vpc'
          OptionName: 'VPCId'
          DefaultValue: 'abc'
      HealthyThresholdCount: 3
      HealthCheckPath: 
        Fn::GetOptionSetting:
          Namespace: 'aws:elasticbeanstalk:environment:process:default'
          OptionName: 'HealthCheckPath'
          DefaultValue: 'abc'
      Port: 80
      TargetGroupAttributes:
        - Value: '20'
          Key: deregistration_delay.timeout_seconds
      Protocol: HTTP
      UnhealthyThresholdCount: 5
      HealthCheckTimeoutSeconds: 5
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
  AWSEBAutoScalingGroup :
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      TargetGroupARNs:
        - { "Ref" : "AWSEBV2LoadBalancerTargetGroup" }
        - { "Ref" : "AdditionalAWSEBV2LoadBalancerTargetGroup" }
  AWSEBSecurityGroup:
    Properties:
      SecurityGroupIngress:
        - FromPort: '80'
          ToPort: '80'
          IpProtocol: tcp
          SourceSecurityGroupId: { "Ref" : "AWSEBLoadBalancerSecurityGroup" }
        - FromPort: '80'
          ToPort: '80'
          IpProtocol: tcp
          SourceSecurityGroupId: { "Ref" : "AdditionalAWSEBLoadBalancerSecurityGroup" }
    Type: 'AWS::EC2::SecurityGroup'
```

Let's discuss the resources created by the script (the default resources created by AWS Elastic Beanstalk can be located [here](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-format-resources-eb.html)):

* `AdditionalAWSEBLoadBalancerSecurityGroup`: Security Group associated with the Load Balancer.
    
* `AdditionalAWSEBV2LoadBalancer`: The new internet-facing Load Balancer.
    
* `AddtionalAWSEBV2LoadBalancerListener`: The listener on port 80 with the default action to redirect to port 443.
    
* `AdditionalAWSEBV2LoadBalancerListener443`: The listener for the new Load Balancer on port 443.
    
* `AdditionalAWSEBV2LoadBalancerTargetGroup`: The new Load Balancer Target Group routes requests to one or more registered targets.
    
* `AWSEBAutoScalingGroup`: The Auto Scaling group attached to our environment.
    
    Now there are two Load Balancers attached, the new Target Group and the default one.
    
* `AWSEBSecurityGroup`: The security group attached to our Auto Scaling group. We are allowing traffic from the new Security Group and the default one.
    

It's time to create the bundle for deployment. Execute the following commands:

```bash
dotnet publish ./src/WeatherApi/WeatherApi.csproj --output "terraform/publish/webapi" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained
mkdir terraform/bundle
Compress-Archive -Path terraform/publish/webapi/* -DestinationPath terraform/bundle/site.zip
dotnet publish ./src/WeatherWs/WeatherWs.csproj --output "terraform/bundle/ws" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime win-x64 --no-self-contained
copy .\install.ps1 .\terraform\bundle
copy .\aws-windows-deployment-manifest.json .\terraform\bundle
mkdir .\terraform\bundle\.ebextensions
copy .\.ebextensions\* .\terraform\bundle\.ebextensions
Compress-Archive -Path terraform/bundle/* -DestinationPath terraform/app.zip
```

Run the terraform scripts with the following commands:

```bash
cd terraform
terraform init
terraform plan -out app.tfplan -var="health_check_path=/swagger/index.html" -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>" -var="platform=64bit Windows Server 2019 v2.11.3 running IIS 10.0" -var ssl_certificate="<MY_SSL_CERTIFICATE>" -var="public_elb_subnets=<MY_SUBNETS>" -var public_ssl_certificate="<MY_SSL_CERTIFICATE>"
terraform apply 'app.tfplan'
```

And deploy the application version with the following command:

```json
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENV_NAME> --version-label <OUTPUT_APP_VERSION>
```

If we navigate to the Auto Scaling Group associated with the AWS Elastic Beanstalk application, we will now see two Load Balancers linked to it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689036649769/4b8e7724-1590-4b88-8074-3890873e71d4.png align="center")

All the code is available [here](https://github.com/raulnq/aws-beanstalk/tree/additional-load-balancer). Thanks, and happy coding.