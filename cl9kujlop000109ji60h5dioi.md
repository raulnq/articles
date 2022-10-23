# AWS Lambda Functions: Finding the right memory size

Once we have our first Lambda Function up and running, a new question arises, How much memory should we use? As we know, the [AWS Lambda pricing model](https://aws.amazon.com/lambda/pricing/?nc1=h_ls) is based on request charges(number of requests) and compute charges(GB-seconds). In addition, we need to notice that the amount of memory is related to the CPU power (check [this](https://www.sentiatechblog.com/aws-re-invent-2020-day-3-optimizing-lambda-cost-with-multi-threading?utm_source=reddit&utm_medium=social&utm_campaign=day3_lambda) post for more details). These characteristics make finding the right memory size not as simple as it seems. Here is where [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) comes to the rescue:

> AWS Lambda Power Tuning is a state machine powered by AWS Step Functions that helps you optimize your Lambda functions for cost and/or performance in a data-driven way.

> The state machine is designed to be easy to deploy and fast to execute. Also, it's language agnostic so you can optimize any Lambda functions in your account.

Today we will test this tool with an AWS Lambda Function that:

- Receives an HTTP request from API Gateway.
- Store the request in a DynamoDb table.
- Send a message to an SQS queue.
- Return an HTTP response

The code can be found [here](https://github.com/raulnq/aws-lambda-power-tuning). See the post [Deploying AWS Lambda Functions with Serverless Framework](https://blog.raulnq.com/deploying-aws-lambda-functions-with-serverless-framework) to get the application up.

Let's start going [here](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:451282441545:applications~aws-lambda-power-tuning) to install the AWS Lambda Power Tuning in our AWS Account, with just a few clicks in the AWS Management Console.

![tool.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1666490996207/QN03WcKHc.PNG align="left")

![installing.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1666491113455/0uXIPz0uc.PNG align="left")

Once the tool is installed, go to AWS Step Functions to locate our state machine:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1666491304734/WzAPj0OXW.png align="left")

Open the state machine and start a new execution:

![image (5).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1666491544710/r_F3lUBV0.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1666492004858/_mibhuN5b.png align="left")

Use the following as input to the state machine:

```json
{
    "lambdaARN": "<YOUR-LAMBDA-ARN>",
    "powerValues": [
      128,
      256,
      512,
      1024,
      1280,
      1536,
      1792,
      2048
    ],
    "num": 50,
    "payload": {
      "Resource": null,
      "Path": null,
      "HttpMethod": null,
      "Headers": null,
      "MultiValueHeaders": null,
      "QueryStringParameters": null,
      "MultiValueQueryStringParameters": null,
      "PathParameters": null,
      "StageVariables": null,
      "RequestContext": null,
      "Body": "{\"Name\":\"Hi\"}",
      "IsBase64Encoded": false
    }
  }
``` 

- `lambdaARN`: Unique identifier of the Lambda function you want to optimize.
- `powerValues`: The list of memory values to be tested.
- `num`: The number of invocations for each memory configuration.
- `payload`: The static payload that will be used for every invocation.

Wait until the execution ends,  go to the output tab, copy the visualization URL and open it in a browser:

![image (6).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1666493492972/9Ha6inCXW.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1666493938918/oZmvVZDnh.png align="left")

Looking at the graph, 512Mb provides a good balance between invocation time and cost for our Lambda function. But if time is a concern 1024Mb is a good choice as well. Thanks, and happy coding.

