# Serverless Framework: Variables

In our last [post](https://blog.raulnq.com/deploying-aws-lambda-functions-with-serverless-framework), we introduced Serverless Framework as a great tool to deploy our AWS Lambda functions. Today we will review Serverless Variables as a way to dynamically replace values in the `serverless.yml` file.  

## Syntax
Through the `${}` syntax we can reference several values from different sources.

```yaml
SomeProperty: ${someVariable}
``` 

## Custom Properties
We can define new properties under the `custom` section:

```yaml
service: new-service
provider:
  name: aws
custom:
  properyA: 1
  properyB: 'value'
functions:
  hello:
    handler: handler.hello
``` 

## Overwriting Variables
We can reference multiple variables as a fallback strategy in case one of the variables is missing:
```yaml
 propertyA: ${someVariable, otherVariable}
 propertyB: ${someVariable, 'value'}
 propertyC: ${someVariable, false}
 propertyD: ${someVariable, 1024}
``` 

## Sources

### Properties
We can reference any property in the `serverless.yml` file, using the `${self:someProperty}` syntax:

```yaml
service: new-service
provider:
  name: aws
custom:
  prefix: 'abc'
functions:
  hello:
    name: ${self:custom.prefix}-hello
    handler: handler.hello
``` 

### Environment Variables
To reference environment variables, use the `${env:someVariable}` syntax:

```yaml
service: new-service
provider:
  name: aws
functions:
  hello:
    name: ${env:PREFIX}-hello
    handler: handler.hello
``` 

### Parameters
To reference parameters, use the `${param:someParameter}` syntax:

```yaml
service: new-service
provider:
  name: aws
functions:
  hello:
    name: ${param:prefix}-hello
    handler: handler.hello
``` 

Parameters can be passed directly via CLI `--param` flag:

```powershell
serverless deploy --param="prefix=abc" --param="otherParameter=otherValue"
``` 

### Files
To reference properties in other files, use the `${file(./someFile.xyz):someProperty}` syntax.

#### Yaml

```yaml
# file.yml file
prefix: 'abc'
```
 
```yaml
service: new-service
provider:
  name: aws
functions:
  hello:
    name: ${file(./file.yml):prefix}-hello
    handler: handler.hello
``` 

#### Json

```json
# file.json file
{
  prefix: 'abc'
}
```
 
```yaml
service: new-service
provider:
  name: aws
functions:
  hello:
    name: ${file(./file.yml):prefix}-hello
    handler: handler.hello
``` 

#### Javascript

```js
// file.js file
module.exports.prefix= 'abc';
```
 
```yaml
service: new-service
provider:
  name: aws
functions:
  hello:
    name: ${file(./file.js):prefix}-hello
    handler: handler.hello
``` 

### CLI Options
To reference CLI options, use the `${opt:someOption}` syntax. Common options are:

- `${opt:stage}`: The stage in your service you want to deploy.
- `${opt:stage}`: The region in your service you want to deploy.

Options can be passed directly via CLI `--stage` and `--region` flags:

```powershell
serverless deploy --stage production --region eu-central-1
``` 

### Serverless Variables
Serverless variables are used internally by the Framework itself. The following variables are available:

- `${sls:instanceId}`: A random id generated when the CLI runs.
- `${sls:stage}`: The stage used by the CLI.

### AWS Resources

- **CloudFormation Outputs**: Use the `${cf:someStackName.someOutputKey}` syntax.
- **S3 Objects**: Use the `${s3:someBucketName/someKey}` syntax.
- **SSM Parameter Store**: Use the `${ssm:/path/to/param}`  syntax.
- **AWS Secrets Manager**: Use the `${ssm:/aws/reference/secretsmanager/secret_ID_in_Secrets_Manager}` syntax.
- **AWS Variables**: `${aws:accountId}` and `${aws:region}`.

## Multiple Configuration Files
We can add entire `Resources` sections from multiple files:

```yaml
resources:
  - ${file(resources/first-cf-resources.yml)}
  - ${file(resources/second-cf-resources.yml)}
``` 

Each of your CloudFormation files has to start with a ` Resources`  entity:

```yaml
Resources:
  Type: 'AWS::S3::Bucket'
  Properties:
    BucketName: some-bucket-name
``` 

## Nesting Variable References
We can nest variable references within each other. So you can reference certain variables based on another variable:

```yaml
service: new-service
provider:
  name: aws
functions:
  hello:
    name: ${param:${env:PARAM_NAME}}-hello
    handler: handler.hello
``` 

Thanks, and happy coding.






