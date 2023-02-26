# AWS SAM: SNS and SQS Lambda Triggers

In the previous post [Deploying AWS Lambda Functions with AWS SAM](https://blog.raulnq.com/deploying-aws-lambda-functions-with-aws-sam), we saw how easy it is to deploy an AWS Lambda function with Amazon API Gateway. Today we want to complement it by using [SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) and [SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) as triggers for the Lambda function when dealing with event-driven architectures.

## **Pre-requisites**

* Have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g` [`Amazon.Lambda.Tools`](http://Amazon.Lambda.Tools))
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## SQS

Main features:

* Lambda polls the SQS queue and invokes the function [synchronously](https://docs.aws.amazon.com/lambda/latest/dg/invocation-sync.html).
    
* Lambda can read messages in batches and invoke the function once per batch.
    
* On errors, the message is automatically returned to the queue.
    

Create a Lambda function to handle SQS events running the following command:

```bash
dotnet new lambda.SQS -n SQSTrigger -o .
```

Create a `template.yaml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SQS

Globals:
  Function:
    Timeout: 15
    MemorySize: 512
    Runtime: dotnet6
    Architectures:
      - x86_64

Resources:
  SQSQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: "mySQSQueue"

  SQSFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: SQSTrigger::SQSTrigger.Function::FunctionHandler
      CodeUri: ./src/SQSTrigger/
      Policies:  
        - SQSPollerPolicy:
            QueueName: !GetAtt SQSQueue.QueueName
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt SQSQueue.Arn
            BatchSize: 10

Outputs:
  SQSConsumerFunction:
    Description: SQS consumer function name
    Value: !Ref SQSFunction
```

The event type `SQS` has the following properties:

* `BatchSize`: Number of records that will be read at once and delivered to the function.
    
* `Queue`: The ARN of the queue.
    
* `FilterCriteria`: [Defines the criteria](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventfiltering.html) to determine whether Lambda should process an event.
    
* `FunctionResponseTypes`: A list of the response types currently applied to the event source mapping. There is only one value allowed [`ReportBatchItemFailures`](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html#services-sqs-batchfailurereporting).
    
* `MaximumBatchingWindowInSeconds`: The maximum amount of time to gather records before invoking the function.
    
* `ScalingConfig`: With only one property `MaximumConcurrency` limits the number of concurrent instances that the Amazon SQS event source can invoke.
    

Run `sam build` to build the application, and then `sam deploy --guided` to deploy it.

## SNS

Main features:

* SNS invokes the function [asynchronously](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html).
    
* Batching is not available, messages are delivered one by one.
    
* On errors, the event is lost unless DLQ is configured in the function.
    

Create a Lambda function to handle SNS events running the following command:

```bash
dotnet new lambda.SNS -n SNSTrigger -o .
```

Create a `template.yaml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SNS

Globals:
  Function:
    Timeout: 15
    MemorySize: 512
    Runtime: dotnet6
    Architectures:
      - x86_64

Resources:
  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: "mySNSTopic"

  SNSFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: SNSTrigger::SNSTrigger.Function::FunctionHandler
      CodeUri: ./src/SNSTrigger/
      Events:
        SNSEvent:
          Type: SNS
          Properties:
            Topic: !Ref SNSTopic

Outputs:
  SNSConsumerFunction:
    Description: SNS consumer function name
    Value: !Ref SNSFunction
```

The event type `SNS` has the following properties:

* `Topic`: The ARN of the topic to subscribe to.
    
* `RedrivePolicy`: Sends undeliverable messages to the specified Amazon SQS dead-letter queue.
    
    ```json
    {
      "deadLetterTargetArn": "arn:aws:sqs:us-east 2:123456789012:MyDeadLetterQueue"
    }
    ```
    
* `FilterPolicy`: Enables the subscriber to [filter out unwanted messages](https://docs.amazonaws.cn/en_us/sns/latest/dg/example-filter-policies.html).
    
* `SqsSubscription`: Enable batching SNS topic notifications in an SQS queue.
    

Run `sam build` to build the application and then `sam deploy --guided` to deploy it.

## SNS to SQS

There are several advantages to having an SQS between SNS and Lambda, especially when you do not want to lose any messages or reprocess them in case of failures. Therefore, you can change the `template.yaml` above with:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SNS to SQS

Globals:
  Function:
    Timeout: 15
    MemorySize: 512
    Runtime: dotnet6
    Architectures:
      - x86_64

Resources:
  SQSSubscriptionQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: "mySubscriptionSQSQueue"

  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: "mySNSTopic"

  SNSFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: SNSTrigger::SNSTrigger.Function::FunctionHandler
      CodeUri: ./src/SNSTrigger/
      Events:
        SNSEvent:
          Type: SNS
          Properties:
            Topic: !Ref SNSTopic
            SqsSubscription:
              BatchSize: 10
              QueueArn: !GetAtt SQSSubscriptionQueue.Arn
              QueueUrl: !Ref SQSSubscriptionQueue
              Enabled: true

Outputs:
  SNSConsumerFunction:
    Description: SNS consumer function name
    Value: !Ref SNSFunction
```

Thanks, and happy coding.