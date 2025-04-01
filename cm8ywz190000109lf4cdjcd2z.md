---
title: "AWS CloudFront Functions: What Are They For?"
datePublished: Tue Apr 01 2025 19:51:41 GMT+0000 (Coordinated Universal Time)
cuid: cm8ywz190000109lf4cdjcd2z
slug: aws-cloudfront-functions-what-are-they-for
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1743441679526/50b33548-6529-4437-9b80-8d495ff10f0f.png
tags: aws, aws-cloudfront, aws-sam

---

Let's imagine we have our resources under [AWS CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html), and we want to perform any of the following tasks:

* Remove unnecessary query strings or headers to improve our cache hit ratio.
    
* Redirect users from an old URL path to a new one without hitting our origin server.
    
* Add headers like `Strict-Transport-Security` or `Content-Security-Policy` into responses.
    
* Validate a simple token or header before allowing access to content.
    
* Change the request path before it is forwarded to our origin server.
    

All these tasks are simple, and for such straightforward tasks, we can use [AWS CloudFront functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions.html). These short-lived, lightweight JavaScript functions run directly at AWS's global edge locations. They allow us to modify the requests and responses that go through AWS CloudFront to complete the tasks mentioned and more. Some of its key features include:

* **Runtime:** Only supports JavaScript that complies with [ECMAScript (ES) version 5.1](https://262.ecma-international.org/5.1/). The [JavaScript runtime 2.0](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-javascript-runtime-20.html) also includes some features from ES versions 6 through 12.
    
* **Triggers:** These can only be triggered by viewer request events (when we receive a request from a viewer) and viewer response events (before we return the response to the viewer).
    
* **Scale:** Scales automatically to handle millions of requests per second.
    
* **Performance:** Designed for high performance and extremely low latency (typically sub-millisecond execution).
    

While both AWS CloudFront functions and [Lambda@Edge](https://blog.raulnq.com/securing-aws-cloudfront-content-with-lambdaedge-and-azure-entraid) allow us to run them at edge locations, they are designed for different purposes and have significant differences:

* CloudFront functions only support JavaScript (ES 5.1+), while Lambda@Edge functions support Node.js and Python.
    
* Lambda@Edge supports all CloudFront events, while CloudFront functions only support viewer request and response events.
    
* For viewer events, the timeout for Lambda@Edge is 5 seconds, while for CloudFront functions, it is 1 millisecond.
    
* CloudFront functions do not have access to the network or file system.
    
* The maximum memory used by CloudFront functions is about 2MB, while Lambda@Edge functions can use up to 128MB for viewer events.
    
* The cost model for CloudFront functions is based on the number of requests, while for Lambda@Edge functions, it is based on both the number of requests and the duration (GB per second).
    

**Choose CloudFront Functions when:** We need low latency, simple request/response manipulation, and don't require network access or significant computing resources.

**Choose Lambda@Edge when:** We need more time to execute, access the network (such as calling APIs), or need to interact with the origin request/response lifecycle.

In this post, we will deploy a static website in an AWS S3 bucket and deliver it to our users using AWS CloudFront. The peculiarity is that any path entered will always display the same page.

## Pre-requisites

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## AWS SAM template

Create a `template.yml` file with the following content:

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
            QueryString: false
            Cookies:
              Forward: none
          ViewerProtocolPolicy: allow-all
          FunctionAssociations: 
            - EventType: viewer-request
              FunctionARN: !GetAtt MyUrlRewriteFunction.FunctionARN

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

  MyUrlRewriteFunction:
    Type: AWS::CloudFront::Function
    Properties:
      Name: "MyUrlRewriteFunction"
      AutoPublish: true
      FunctionConfig:
        Comment: "Rewrites function"
        Runtime: cloudfront-js-2.0
      FunctionCode: !Sub |
        function handler(event) {
            var request = event.request;
            request.uri = "/index.html";
            return request;
        }

Outputs:
  CloudFrontURL:
    Description: URL of CloudFront distribution.
    Value: !GetAtt MyCloudFrontDistribution.DomainName
```

The script above defines the following resources:

* [AWS::S3::Bucket](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3-bucket.html)**:** Defines an AWS S3 bucket resource.
    
* [AWS::CloudFront::OriginAccessControl](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-originaccesscontrol.html)**:** Defines a CloudFront Origin Access Control (OAC) configuration, which is the recommended method for AWS CloudFront to securely access private S3 buckets.
    
* [AWS::CloudFront::Distribution](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-distribution.html): Defines a AWS CloudFront CDN distribution.
    
* [AWS::S3::BucketPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-s3-bucketpolicy.html): Defines a policy attached directly to the AWS S3 bucket.
    
* [AWS::CloudFront::Function](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudfront-function.html): Defines a AWS CloudFront Function. Includes the code to change any requested path to `index.html`. This practice is common for SPAs, where client-side routing handles different paths, but the server always needs to provide the main `index.html` file.
    

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

## The Static Website

In a `site` folder, create an `index.html` file with the following content:

```xml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Under Construction</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f3f3f3;
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        .container {
            max-width: 600px;
            padding: 20px;
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
            text-align: center;
            animation: pulse 1.5s infinite alternate;
        }
        @keyframes pulse {
            0% {
                transform: scale(1);
            }
            100% {
                transform: scale(1.05);
            }
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Under Construction</h1>
        <p>We're working hard to bring you something awesome!</p>
        <p>In the meantime, please excuse our appearance as we're in the process of building something amazing. Stay tuned for updates.</p>
        <p>Thank you for your patience!</p>
    </div>
</body>
</html>
```

Upload the site to the AWS S3 Bucket by running the following command:

```powershell
aws s3 sync '.\site' s3://<MY_BUCKET_NAME>
```

## **Testing**

Browse to the AWS SAM output URLs to view the static website up and running:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743535561835/6a13655a-c14a-47e2-acef-90905b12255a.png align="center")

Any path in the domain will display the same page. You can find all the code [here](https://github.com/raulnq/aws-cloudfront-functions). Thanks, and happy coding.