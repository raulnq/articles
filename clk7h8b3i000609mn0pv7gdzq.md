---
title: "AWS Elastic Beanstalk: Integrating a Network Load Balancer with an Application Load Balancer"
datePublished: Mon Jul 17 2023 23:08:17 GMT+0000 (Coordinated Universal Time)
cuid: clk7h8b3i000609mn0pv7gdzq
slug: aws-elastic-beanstalk-integrating-a-network-load-balancer-with-an-application-load-balancer
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1689607605936/11830ef3-7702-48bf-81a8-38e88cdf54ef.png
tags: aws, net, elastic-beanstalk

---

Integrating a [Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html) with AWS Elastic Beanstalk is essential for applications that need static IP addresses for consistent accessibility. This article shows how to overcome the limitations of an [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) by placing a Network Load Balancer in front of it, providing our application the best of both worlds: consistent accessibility through static IP addresses and advanced routing capabilities.

Our starting point will be the article [AWS Elastic Beanstalk: How to Include an Additional Application Load Balancer](https://blog.raulnq.com/aws-elastic-beanstalk-how-to-include-an-additional-application-load-balancer) (the code can be downloaded [here](https://github.com/raulnq/aws-beanstalk/tree/additional-load-balancer)). Navigate to the `Terraform` folder and open the [`main.tf`](http://main.tf) file. Locate the `aws_elastic_beanstalk_environment` resource and append the following lines at the end:

```json
  setting {
      namespace = "aws:elasticbeanstalk:customoption"
      name      = "NELBbSubnetA"
      value     = var.nelb_subnetA
  }

  setting {
      namespace = "aws:elasticbeanstalk:customoption"
      name      = "NELBIPSubnetA"
      value     = var.nelb_ip_subnetA
  }
```

Open the [`variables.tf`](http://variables.tf) file and add the following:

```json
variable "nelb_subnetA" {
  type = string
}

variable "nelb_ip_subnetA" {
  type = string
}
```

These two variables will serve as [inputs](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions-functions.html#ebextensions-functions-getoptionsetting) for our Network Load Balancer. Navigate to the `.ebextensions` folder and create a file called `add-network-load-balancer.config`, then insert the following content:

```json
Resources:
  NetworkLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internal
      Type: network
      SubnetMappings: 
        - PrivateIPv4Address:
            Fn::GetOptionSetting:
              Namespace: 'aws:elasticbeanstalk:customoption'
              OptionName: 'NELBIPSubnetA'
              DefaultValue: '0.0.0.0'
          SubnetId:
            Fn::GetOptionSetting:
              Namespace: 'aws:elasticbeanstalk:customoption'
              OptionName: 'NELBbSubnetA'
              DefaultValue: 'abc'
  NetworkAWSEBV2LoadBalancerTargetGroup:
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
      Protocol: TCP
      UnhealthyThresholdCount: 5
      HealthCheckTimeoutSeconds: 5
      Targets:
        - Id:  { "Ref" : "AWSEBV2LoadBalancer" }
          Port: 80
      TargetType: 'alb'
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
  NetworkAWSEBV2LoadBalancerTargetGroup443:
    Properties:
      HealthCheckIntervalSeconds: 15
      VpcId:
        Fn::GetOptionSetting: 
          Namespace: 'aws:ec2:vpc'
          OptionName: 'VPCId'
          DefaultValue: 'abc'
      HealthyThresholdCount: 3
      HealthCheckProtocol: HTTPS
      HealthCheckPath:
        Fn::GetOptionSetting:
          Namespace: 'aws:elasticbeanstalk:environment:process:default'
          OptionName: 'HealthCheckPath'
          DefaultValue: 'abc'
      Port: 443
      Protocol: TCP
      UnhealthyThresholdCount: 5
      HealthCheckTimeoutSeconds: 5
      Targets:
        - Id:  { "Ref" : "AWSEBV2LoadBalancer" }
          Port: 443
      TargetType: 'alb'
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
  NetworkAWSEBV2LoadBalancerListener:
    Properties:
      LoadBalancerArn: { "Ref" : "NetworkLoadBalancer" }
      DefaultActions:
        - TargetGroupArn: { "Ref" : "NetworkAWSEBV2LoadBalancerTargetGroup" }
          Type: forward
      Port: 80
      Protocol: TCP
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
  NetworkAWSEBV2LoadBalancerListener443:
    Properties:
      LoadBalancerArn: { "Ref" : "NetworkLoadBalancer" }
      DefaultActions:
        - TargetGroupArn: { "Ref" : "NetworkAWSEBV2LoadBalancerTargetGroup443" }
          Type: forward
      Port: 443
      Protocol: TCP
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
```

Let's review the resources created by the script (the default resources created by AWS Elastic Beanstalk can be located [here](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-format-resources-eb.html)):

* `NetworkLoadBalancer`: The new internal Network Load Balancer.
    
* `NetworkAWSEBV2LoadBalancerTargetGroup` and `NetworkAWSEBV2LoadBalancerTargetGroup443`:
    
    The Network Load Balancer's Target Groups route requests to the default Application Load Balancer for ports 80 and 443(`AWSEBV2LoadBalancer`).
    
* `NetworkAWSEBV2LoadBalancerListener` and `NetworkAWSEBV2LoadBalancerListener443`: Listeners for new Load Balancer for each port.
    

Execute the following commands to create the deployment bundle:

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

```json
cd terraform
terraform init
terraform plan -out app.tfplan -var="health_check_path=/swagger/index.html" -var="bucket=app-tf-001" -var="keypair=<MY_KEY_PAIR>" -var="instance_type=t2.medium" -var="application=app-tf-001" -var="vpc_id=<MY_VPC>" -var="ec2_subnets=<MY_SUBNETS>" -var="elb_subnets=<MY_SUBNETS>" -var="platform=64bit Windows Server 2019 v2.11.3 running IIS 10.0" -var ssl_certificate="<MY_SSL_CERTIFICATE>" -var="public_elb_subnets=<MY_SUBNETS>" -var public_ssl_certificate="<MY_SSL_CERTIFICATE>" -var nelb_subnetA="<MY_SUBNET>" -var nelb_ip_subnetA="<MY_IP>"
terraform apply 'app.tfplan'
```

And deploy the application version with the following command:

```json
aws --region us-east-2 elasticbeanstalk update-environment --environment-name <OUTPUT_ENV_NAME> --version-label <OUTPUT_APP_VERSION>
```

Wait for the deployment to finish, then go to the Load Balancer section within the EC2 service to view the newly created Network Balancer. All the code is available [here](https://github.com/raulnq/aws-beanstalk/tree/network-load-balancer). Thanks, and happy coding.