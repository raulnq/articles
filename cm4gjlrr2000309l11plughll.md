---
title: "Hosting Blazor WebAssembly Applications on AWS CloudFront"
datePublished: Mon Dec 09 2024 04:38:46 GMT+0000 (Coordinated Universal Time)
cuid: cm4gjlrr2000309l11plughll
slug: hosting-blazor-webassembly-applications-on-aws-cloudfront
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1733586251018/a02d9c2d-bde9-4d9a-9811-a0f9c9b6a2fd.png
tags: aws, cloudfront, dotnet, blazor, s3-static-website-hosting

---

[Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-9.0) is becoming a popular choice for developing modern web applications using C# and .NET, offering two distinct hosting models:

* [Blazor Server](https://learn.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-9.0#blazor-server): In this model, the application runs on the server, and UI updates are sent to the browser through a SignalR connection.
    
* [Blazor WebAssembly](https://learn.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-9.0#blazor-webassembly): In this model, the application runs directly in the browser using WebAssembly.
    

Within the Blazor WebAssembly model, there are two options for deploying our application:

* **Hosted deployment**: In this option, the Blazor WebAssembly application is delivered from an ASP.NET Core app.
    
* **Standalone deployment**: In this deployment, the Blazor WebAssembly application is delivered from a static web server. This option allows us to use various hosting environments like Amazon S3, GitHub Pages, Azure Static Web Apps, Azure Storage, and more.
    

In this article, we will deploy a Blazor WebAssembly application in an [AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) bucket and deliver it to our users using a CDN such as [AWS CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html).

## **Pre-requisites**

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## Blazor WebAssembly App

We will use the default Blazor WebAssembly application provided by the templates. Run the following commands:

```powershell
dotnet new blazorwasm -o MyApp
dotnet new sln -n Blazor
dotnet sln add --in-root MyApp
```

## AWS SAM template

At the solution level, create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: "<MY_BUCKET_NAME>"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  MyCloudFrontOriginAccessControl:
    Type: AWS::CloudFront::OriginAccessControl
    Properties:
      OriginAccessControlConfig:
        Name: !Sub "OAC for ${MyBucket}"
        OriginAccessControlOriginType: s3
        SigningBehavior: always
        SigningProtocol: sigv4

  MyCloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    DependsOn:
      - MyBucket
    Properties:
      DistributionConfig:
        Origins:
          - DomainName: !GetAtt MyBucket.DomainName
            Id: !Sub "origin-${MyBucket}"
            S3OriginConfig:
              OriginAccessIdentity: ""
            OriginAccessControlId: !GetAtt MyCloudFrontOriginAccessControl.Id
        Enabled: "true"
        PriceClass: "PriceClass_200"
        IPV6Enabled : "false"
        DefaultRootObject: index.html
        ViewerCertificate:
          CloudFrontDefaultCertificate: true
        DefaultCacheBehavior:
          AllowedMethods:
            - DELETE
            - GET
            - HEAD
            - OPTIONS
            - PATCH
            - POST
            - PUT
          CachedMethods :
            - GET
            - HEAD
          Compress: true
          TargetOriginId: !Sub "origin-${MyBucket}"
          ForwardedValues:
            QueryString: "false"
            Cookies:
              Forward: none
          ViewerProtocolPolicy: allow-all

  MyBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref MyBucket
      PolicyDocument:
        Version: 2008-10-17
        Statement:
          - Action:
            - 's3:GetObject'
            Effect: Allow
            Principal:
              Service: cloudfront.amazonaws.com
            Resource: !Sub "${MyBucket.Arn}/*"
            Condition:
              StringEquals:
                AWS:SourceArn: !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${MyCloudFrontDistribution.Id}"

Outputs:
  CloudFrontURL:
    Description: URL of CloudFront distribution.
    Value: !GetAtt MyCloudFrontDistribution.DomainName
```

The script above defines the following resources:

* [AWS::S3::Bucket](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3-bucket.html): This resource creates the AWS S3 bucket, blocking all public access.
    
* [AWS::CloudFront::OriginAccessControl](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-originaccesscontrol.html): This resource creates a new origin access control (OAC). Once created, it can be added to an origin in a CloudFront distribution. This allows CloudFront to send authenticated requests to the origin, allowing us to block public access.
    
* [AWS::CloudFront::Distribution](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-distribution.html): This resource creates the distribution that instructs CloudFront where to locate the content to deliver and provides details on how to track and manage it.
    
* [AWS::S3::BucketPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3-bucketpolicy.html): This resource configures our AWS S3 bucket to allow access for requests from the CloudFront distribution with OAC.
    

## Deploying our application

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

Once the resources are deployed, it's time to deploy our Blazor application to AWS S3:

```powershell
dotnet publish -c Release
aws s3 sync '.\MyApp\bin\Release\net8.0\publish\wwwroot' s3://<MY_BUCKET_NAME>
```

## Testing our application

Browse to the output URL to view the Blazor application up and running:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733713607840/f9ff7d1e-855b-469d-bed7-e6fc37854203.png align="center")

You can find all the code [here](https://github.com/raulnq/aws-cloudfront-blazor). Thanks, and happy coding.