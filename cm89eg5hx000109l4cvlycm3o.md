---
title: "Securing AWS CloudFront Content with Lambda@Edge and Azure EntraID"
datePublished: Fri Mar 14 2025 23:18:52 GMT+0000 (Coordinated Universal Time)
cuid: cm89eg5hx000109l4cvlycm3o
slug: securing-aws-cloudfront-content-with-lambdaedge-and-azure-entraid
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1741887767861/00be1d5b-019f-4939-83ee-b81cd33f54cb.png
tags: authentication, aws-cloudfront, azure-ad, lambdaedge

---

A everyday use case for [AWS CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html), with Amazon Simple Storage Service (S3) as the origin, is hosting a Single Page Application (SPA). This method offers the benefits of serverless hosting, mainly lower costs. However, there's a downside: even though the SPA only allows access to valid users, any unauthenticated user can download the source code. A malicious user could look at the source code to understand the SPA's functionality and identify backend API endpoints, making it easier for them to launch attacks.

In this article, we focus on preventing unauthenticated users from downloading the SPA's source code by using Lambda@Edge functions, Azure Entra ID, and the Authorization Code Flow with PKCE. The image below summarizes the entire solution:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741895342499/e4aacee8-2fbe-46b9-b155-489ab96ade4d.png align="center")

1. The user's browser sends a request to the `/` path.
    
2. AWS CloudFront uses the `check_auth` Lambda@edge function to look for the `access_token` and `refresh_token` cookies in the request.
    
3. Since the cookies are not present, AWS CloudFront responds by redirecting the browser to Azure EntraID, creating the `code_verifier` cookie in the process.
    
4. Azure EntraID prompts the sign-in form.
    
5. The user enters their credentials, and once authenticated, Azure EntraID redirects the browser to AWS CloudFront.
    
6. The user's browser sends a request to the `/callback` path.
    
7. AWS CloudFront invokes the `handle_callback` Lambda@edge function.
    
8. The Lambda@Edge function exchanges the authorization code for Azure EntraID's access and refresh tokens.
    
9. AWS CloudFront responds by redirecting the browser to the `/` path, creating the `access_token` and `refresh_token` cookies in the process.
    
10. The user’s browser sends a request to the `/` path.
    
11. AWS CloudFront repeats the process described in step 2 and also validates the access token.
    
12. Now that the cookies are present and the access token is valid, AWS CloudFront grants access to the resource in the S3 bucket.
    
13. The browser downloads the resource from AWS CloudFront.
    

## What does Lambda@Edge?

[Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html) enables us to run AWS Lambda functions at AWS CloudFront's edge locations, allowing real-time modification of requests and responses. This powerful feature enables the manipulation at four points in the AWS CloudFront request/response [cycle](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-cloudfront-trigger-events.html):

* **Viewer Request**: This occurs when AWS CloudFront receives a request from a client before cache checking.
    
* **Origin Request**: This takes place before AWS CloudFront forwards the request to the origin. The function is bypassed if the resource is already cached.
    
* **Origin Response**: This happens when AWS CloudFront receives the origin's response before caching the resource.
    
* **Viewer Response**: This occurs just before AWS CloudFront sends the response back to the client.
    

While Lambda@Edge provides great flexibility, it operates under several important [restrictions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge-function-restrictions.html):

* Only a specific version of a function can be used.
    
* Functions must be located in the US East region.
    
* Functions must be completed within 5 seconds for viewer triggers and 30 seconds for origin triggers.
    
* Environment variables are not supported.
    
* Layers are not supported.
    
* Supported runtimes are limited to Node.js and Python.
    
* A maximum of 128 MB of memory is allowed per function for viewer triggers and 10 GB for origin triggers.
    

Now, let's set up a solution to secure our AWS CloudFront resources using Lambda@Edge and Azure Entra ID.

## **Pre-requisites**

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    
* Ensure you have an [Azure Account](https://azure.microsoft.com/en-us/free/)
    
* [Node.js](https://nodejs.org/en/download) installed
    

## **App Registration**

Let's create the app registration for our SPA:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739573270762/fc9fcdf5-6da0-4103-89c4-b5bd12afa7be.png?auto=compress,format&format=webp align="left")

Go to the **Expose an API** option and click **Add a scope:**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741905518880/37d74471-fc0d-4274-8788-7666edd7a7e9.png align="center")

Go to the **API permissions** option, click **Add a permission**, select **APIs my organization uses**, and search for **My SPA**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741905588400/c2106421-7821-42de-a846-fa25a131291b.png align="center")

There is a missing step, which is setting up the Platform for this App Registration. We will add that once we have the domain for our application.

## Lambda@Edge Functions

Create the `functions/check-auth/index.mjs` file with the following content:

```javascript
import crypto from 'crypto';
import jwt  from 'jsonwebtoken';
import jwksClient  from 'jwks-rsa';

const TENANT_ID = '<MY_TENANT_ID>';
const CLIENT_ID =  '<MY_CLIENT_ID>';

export const Handler = async (event, context) => {
    const request = event.Records[0].cf.request;
    console.info(request);
    const cookies = parseCookies(request.headers.cookie || []);
    if(cookies.access_token && cookies.refresh_token){
      try {
        const verifiedToken = verifyToken(cookies.access_token);
        if(verifiedToken)
        {
          return request;
        }
      } catch (error) {
        console.error('Verify token error:', error.name, error.message);
        if (error.name=='TokenExpiredError') {
          const newtokens = await exchangeRefreshToken(cookies.refresh_token, request.headers.host[0].value);
          if(newtokens){
            redirectToRoot(newtokens);
          }
        }
      }
    } 
    return redirectToAzure(request.headers.host[0].value);
  };

function redirectToRoot(tokens)
{
  return { 
    status: '302', 
    headers: { 
      location: [
        { key: 'Location', value: '/' }
      ],
      'set-cookie': [
        { key: 'Set-Cookie', value: `access_token=${tokens.access_token}; Path=/; Secure; HttpOnly` },
        { key: 'Set-Cookie', value: `refresh_token=${tokens.refresh_token}; Path=/; Secure; HttpOnly` }
      ] 
    } 
  };
}

function redirectToAzure(domainName){
  console.info("Redirecting to Azure...");
  const codeVerifier = crypto.randomBytes(32).toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
  const codeChallenge = crypto.createHash('sha256').update(codeVerifier).digest('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
  const state = crypto.randomBytes(16).toString('hex');
  const authUrl = `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/authorize?` +
      `client_id=${CLIENT_ID}` +
      `&response_type=code` +
      `&redirect_uri=${encodeURIComponent(`https://${domainName}/callback`)}` +
      `&scope=${encodeURIComponent(`openid profile email api://${CLIENT_ID}/download`)}` +
      `&code_challenge=${encodeURIComponent(codeChallenge)}` +
      `&code_challenge_method=S256` +
      `&state=${encodeURIComponent(state)}` +
      `&response_mode=query`;
  const response = {
        status: '302',
        statusDescription: 'Found',
        headers: {
            location: [{
                key: 'Location',
                value: authUrl
            }],
            'set-cookie': [{
                key: 'Set-Cookie',
                value: `code_verifier=${codeVerifier}; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=600`
            }]
        }
    };
  return response;
}

async function verifyToken(access_token){
  console.info("Verifying token...");
  const client = jwksClient({jwksUri: `https://login.microsoftonline.com/${TENANT_ID}/discovery/v2.0/keys`});
  const decoded = jwt.decode(access_token, { complete: true });
  if(!decoded){
    console.error('Decode token error');
    return null;
  }
  const key = await client.getSigningKey(decoded.header.kid);
  return jwt.verify(access_token, key.getPublicKey(), {
    audience: `api://${CLIENT_ID}`,
    issuer: `https://sts.windows.net/${TENANT_ID}/`,
    algorithms: ["RS256"]
  });
}

async function exchangeRefreshToken(refresh_token, domainName) {
  console.info("Refreshing token...");
  const url = `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token`;
  const body = new URLSearchParams({
    client_id: CLIENT_ID,
    grant_type: 'refresh_token',
    refresh_token: refresh_token
  });
  const response = await fetch(url, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Origin': `https://${domainName}`
    },
    body: body
  });
  if (!response.ok) {
    console.error('Exchange refresh token error:', response.status);
    return null;
  }
  return await response.json();
}

function parseCookies(cookieHeaders) {
  const cookies = {};
  if (!cookieHeaders || !cookieHeaders.length) {
      return cookies;
  }
  cookieHeaders.forEach(header => {
      if (header.value) {
          header.value.split(';').forEach(cookie => {
              const parts = cookie.trim().split('=');
              if (parts.length >= 2) {
                  cookies[parts[0].trim()] = parts.slice(1).join('=');
              }
          });
      }
  });
  return cookies;
}
```

This function intercepts the viewer's request and tries to extract the `access_token` and `refresh_token` from the cookies, then executes the following logic:

1. If both tokens are present:
    
    * Attempt to verify the access token.
        
    * If verification succeeds, return the original request.
        
    * If the token is expired, attempt to get new tokens using the refresh token.
        
    * If the refresh is successful, redirect to the root with the new tokens stored in the cookies.
        
2. If the tokens are absent or the refresh fails:
    
    * Redirect to the Azure EntraID login page to initiate the [authorization code flow with PKCE](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow).
        

To start the Node project, run `npm init`. Next, install the required dependencies by executing `npm install --save jwks-rsa` and `npm install jsonwebtoken`. Then, create the `functions/handle-callback/index.mjs` file with the following content:

```javascript
const TENANT_ID = '<MY_TENANT_ID>';
const CLIENT_ID =  '<MY_CLIENT_ID>';

export const Handler = async (event, context) => {
    const request = event.Records[0].cf.request;
    console.info(request);
    const queryParams = request.querystring ? Object.fromEntries(new URLSearchParams(request.querystring)) : {};
    const cookies = parseCookies(request.headers.cookie || []);
    if(queryParams.code && cookies.code_verifier)
    {
      const tokens = await exchangeCode(queryParams.code, cookies.code_verifier, request.headers.host[0].value);
      if(tokens){
        return redirectToRoot(tokens);
      }
    }
    return returnError();
  };

function returnError()
{
  return {
    status: '400',
    statusDescription: 'Bad Request',
    headers: {
      'content-type': [{
        key: 'Content-Type',
        value: 'text/html'
      }]
    },
    body: '<html><body><h1>Error</h1><p>Authentication error</p></body></html>'
  };
}

function redirectToRoot(tokens)
{
  return { 
    status: '302', 
    headers: { 
      location: [
        { key: 'Location', value: '/' }
      ],
      'set-cookie': [
        { key: 'Set-Cookie', value: `access_token=${tokens.access_token}; Path=/; Secure; HttpOnly` },
        { key: 'Set-Cookie', value: `refresh_token=${tokens.refresh_token}; Path=/; Secure; HttpOnly` }
      ] 
    } 
  };
}

async function exchangeCode(code, codeVerifier, domainName) {
  console.info("Exchanging code...");
  const url = `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token`;
  const body = new URLSearchParams({
    client_id: CLIENT_ID,
    grant_type: 'authorization_code',
    code: code,
    redirect_uri: `https://${domainName}/callback`,
    code_verifier: codeVerifier
  });
  const response = await fetch(url, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Origin': `https://${domainName}`
    },
    body: body
  });
  if (!response.ok) {
    console.error('Exchange code error:', response.status);
    return null;
  }
  return await response.json();
}

function parseCookies(cookieHeaders) {
  const cookies = {};
  if (!cookieHeaders || !cookieHeaders.length) {
      return cookies;
  }
  cookieHeaders.forEach(header => {
      if (header.value) {
          header.value.split(';').forEach(cookie => {
              const parts = cookie.trim().split('=');
              if (parts.length >= 2) {
                  cookies[parts[0].trim()] = parts.slice(1).join('=');
              }
          });
      }
  });
  return cookies;
}
```

This function handles the callback from Azure EntraID. It attempts to extract the authorization `code` parameter and the `code_verifier` cookie, then proceeds with the following steps:

1. If both are present:
    
    * Exchange the `code` for access and refresh tokens.
        
    * If successful, redirect to the root path with tokens set as secure cookies.
        
2. If either the `code` or `code_verifier` is missing or if the exchange fails:
    
    * Display an error page.
        

To start the Node project, run `npm init`. Create the `functions/template.yaml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  AWS SAM

Resources:
  MyAuthFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 5
      MemorySize: 128
      CodeUri: check-auth/
      Handler: index.Handler
      Role: !GetAtt MyLambdaFunctionRole.Arn 
      Runtime: nodejs20.x
      AutoPublishAlias: live
      Architectures:
        - x86_64

  MyCallbackFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 5
      MemorySize: 128
      CodeUri: handle-callback/
      Handler: index.Handler
      Role: !GetAtt MyLambdaFunctionRole.Arn 
      Runtime: nodejs20.x
      AutoPublishAlias: live
      Architectures:
        - x86_64

  MyLambdaFunctionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - 'lambda.amazonaws.com'
                - 'edgelambda.amazonaws.com'
            Action:
              - 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole'

Outputs:
  AuthArn:
    Description: Auth Lambda edge ARN
    Value: !Ref MyAuthFunction.Version
  CallbackArn:
    Description: Callback Lambda edge ARN
    Value: !Ref MyCallbackFunction.Version
```

The file above will create both AWS Lambda functions with the necessary permissions and automatically publish a new version with each deployment. Run the following commands in the `functions` folder to deploy the resources to AWS, and remember to select `us-east-1` as the region:

```powershell
sam build
sam deploy --guided
```

## AWS CloudFront

Create the `cloudfront/template.yaml` file with the following content:

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
          ViewerProtocolPolicy: redirect-to-https
          LambdaFunctionAssociations: 
            - EventType: viewer-request
              LambdaFunctionARN: <MY_CHECK_AUTH_FUNCTION_ARN>
        CacheBehaviors:
          - PathPattern: /callback
            TargetOriginId: !Sub "origin-${MyBucket}"
            ViewerProtocolPolicy: redirect-to-https
            ForwardedValues:
              QueryString: false
              Cookies:
                Forward: none
            LambdaFunctionAssociations:
              - EventType: viewer-request
                LambdaFunctionARN: <MY_HANDLE_CALLBACK_FUNCTION_ARN>

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

The file above sets up an AWS CloudFront distribution linked with two AWS Lambda@Edge functions:

* The `check-auth` function is triggered for all viewer requests in the default behavior.
    
* The `handle-callback` function is triggered for viewer requests matching the `/callback` path pattern.
    

Run the following commands in the `cloudfront` folder to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

Create the `cloudfront/site/index.html` file with the following content:

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

Deploy our application to AWS S3 by running the following command:

```powershell
aws s3 sync '.\cloudfront\site' s3://<MY_BUCKET_NAME>
```

## Callback URL

At this point, we can complete the app registration. Go to our App Registration, select the **Authentication** option, click **Add a platform**, and choose **Single-page application**.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741976672462/55e9793a-0f28-4ba9-bf61-f91fda0eb0c9.png align="center")

Navigate to our AWS CloudFront URL and start using the application. All the code can be found [here](https://github.com/raulnq/aws-lambda-edge). Thanks, and happy coding.