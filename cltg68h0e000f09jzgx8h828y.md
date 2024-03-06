---
title: "AWS SAM: Provisioning AWS AppSync with Amazon RDS Data Source"
datePublished: Wed Mar 06 2024 19:07:45 GMT+0000 (Coordinated Universal Time)
cuid: cltg68h0e000f09jzgx8h828y
slug: aws-sam-provisioning-aws-appsync-with-amazon-rds-data-source
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1709672772886/5c3cf2dd-d80c-401c-9cad-305be9e62161.png
tags: rds, appsync, aws-sam, aws-aurora-serverless-v2

---

[AWS AppSync](https://docs.aws.amazon.com/appsync/latest/devguide/what-is-appsync.html) now supports [Aurora Serverless v2 and Aurora provisioned clusters](https://aws.amazon.com/es/blogs/database/introducing-the-data-api-for-amazon-aurora-serverless-v2-and-amazon-aurora-provisioned-clusters/) as data sources. However, the `DataSource` property within the `AWS::Serverless::GraphQLApi` resource of [AWS SAM](https://aws.amazon.com/es/blogs/mobile/aws-sam-now-supports-graphql-applications-with-aws-appsync/) doesn't yet support RDS. Until an update from the AWS SAM team arises, leveraging [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) resources emerges as the sole avenue for provisioning AWS AppSync with these data sources. So, let's get started.

## **Pre-requisites**

* Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## **The Database**

In a previous article, we saw [how to create an Amazon Aurora serverless v2 using AWS SAM](https://blog.raulnq.com/creating-an-amazon-aurora-serverless-v2-database-using-aws-sam). We will use the script presented there as a starting point. Create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM Template

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Aurora DB subnet group
      SubnetIds:
        - <MY_SUBNET_1>
        - <MY_SUBNET_2>

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: Aurora DB SG
      GroupDescription: Ingress rules for Aurora DB
      VpcId: <MY_VPC>
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          CidrIp: 0.0.0.0/0

  DBCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Delete
    Properties:
      DatabaseName: mydatabase
      DBClusterIdentifier: my-dbcluster
      DBSubnetGroupName: !Ref DBSubnetGroup
      Engine: aurora-postgresql
      EngineVersion: 15.4
      MasterUsername: <MY_USER>
      ManageMasterUserPassword: True
      Port: 5432
      EnableHttpEndpoint: true
      ServerlessV2ScalingConfiguration:
        MaxCapacity: 1.0
        MinCapacity: 0.5
      VpcSecurityGroupIds:
        - !Ref DBSecurityGroup

  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DBCluster
      DBInstanceIdentifier: my-dbinstance
      DBInstanceClass: db.serverless
      Engine: aurora-postgresql

Outputs:
  DBSecret:
    Description: Secret arn
    Value: !GetAtt DBCluster.MasterUserSecret.SecretArn
  DBCluster:
    Description: Cluster arn
    Value: !GetAtt DBCluster.DBClusterArn
```

We set `ManageMasterUserPassword` to `true` to manage the password using [AWS Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html). We also enable the RDS Data API by setting the property `EnableHttpEndpoint` to `true`. Run the following commands to create the database:

```powershell
sam build
sam deploy --guided
```

Next, create a table using the AWS CLI by running the following commands:

```powershell
aws rds-data execute-statement --resource-arn "<MY_CLUSTER_ARN>" --database "mydatabase" --secret-arn "<MY_SECRET_ARN>" --sql "CREATE TABLE Tasks (Id VARCHAR(255) PRIMARY KEY, Name VARCHAR(50), Description VARCHAR(255));"
```

## AWS AppSync

Add the following resources to the `template.yml` file:

```yaml
  AppSyncAPI:
    Type: AWS::AppSync::GraphQLApi
    Properties:
      Name: my-appsyncapi
      AuthenticationType: API_KEY

  AppSyncApiKey:
    Type: AWS::AppSync::ApiKey
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
```

The [`AWS::AppSync::GraphQLApi`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-appsync-graphqlapi.html) resource sets up a new AWS AppSync GraphQL API, and the [`AWS::AppSync::ApiKey`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-appsync-apikey.html) resource generates a unique key for client distribution. Create a `schema.graphql` file with the following content:

```graphql
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

type Mutation {
  createTasks(input: CreateTasksInput!): Tasks
  deleteTasks(condition: TableTasksConditionInput, input: DeleteTasksInput!): Tasks
  updateTasks(condition: TableTasksConditionInput, input: UpdateTasksInput!): Tasks
}

type Query {
  getTasks(id: String!): Tasks
  listTasks(filter: TableTasksFilterInput, limit: Int, nextToken: String, orderBy: [OrderByTasksInput]): TasksConnection
}

type Tasks {
  description: String
  id: String!
  name: String
}

type TasksConnection {
  items: [Tasks]
  nextToken: String
}

enum ModelSortDirection {
  ASC
  DESC
}

input CreateTasksInput {
  description: String
  id: String
  name: String
}

input DeleteTasksInput {
  id: String!
}

input ModelSizeInput {
  between: [Int]
  eq: Int
  ge: Int
  gt: Int
  le: Int
  lt: Int
  ne: Int
}

input OrderByTasksInput {
  description: ModelSortDirection
  id: ModelSortDirection
  name: ModelSortDirection
}

input TableStringFilterInput {
  attributeExists: Boolean
  beginsWith: String
  between: [String]
  contains: String
  eq: String
  ge: String
  gt: String
  le: String
  lt: String
  ne: String
  notContains: String
  size: ModelSizeInput
}

input TableTasksConditionInput {
  and: [TableTasksConditionInput]
  description: TableStringFilterInput
  name: TableStringFilterInput
  not: [TableTasksConditionInput]
  or: [TableTasksConditionInput]
}

input TableTasksFilterInput {
  and: [TableTasksFilterInput]
  description: TableStringFilterInput
  id: TableStringFilterInput
  name: TableStringFilterInput
  not: [TableTasksFilterInput]
  or: [TableTasksFilterInput]
}

input UpdateTasksInput {
  description: String
  id: String!
  name: String
}
```

The schema above covers all the CRUD (Create, Read, Update, Delete) operations for the `Tasks` table we created earlier. Now, let's add the following resource to the `template.yaml` file:

```yaml
  AppSyncSchema:
    Type: AWS::AppSync::GraphQLSchema
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      DefinitionS3Location: ./src/schema.graphql
```

The [`AWS::AppSync::GraphQLSchema`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-appsync-graphqlschema.html) resource specifies the schema that defines the data model for our API. The next step is to add the data source resources:

```yaml
AppSyncDataSource:
    Type: AWS::AppSync::DataSource
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      Name: RDSDataSource
      Type: RELATIONAL_DATABASE
      ServiceRoleArn: !GetAtt AppSyncDataSourceRole.Arn
      RelationalDatabaseConfig:
        RelationalDatabaseSourceType: RDS_HTTP_ENDPOINT
        RdsHttpEndpointConfig:
          DatabaseName: mydatabase
          AwsRegion: !Ref AWS::Region
          DbClusterIdentifier: !GetAtt DBCluster.DBClusterArn
          AwsSecretStoreArn: !GetAtt DBCluster.MasterUserSecret.SecretArn

  AppSyncDataSourceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Action: sts:AssumeRole
            Principal:
              Service: appsync.amazonaws.com
      Policies:
        - PolicyName: DataSourceRDSPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - rds-data:BatchExecuteStatement
                  - rds-data:BeginTransaction
                  - rds-data:CommitTransaction   
                  - rds-data:RollbackTransaction
                  - rds-data:ExecuteStatement
                Resource: 
                  - !GetAtt DBCluster.DBClusterArn
              - Effect: Allow
                Action:
                  - secretsmanager:GetSecretValue
                Resource: 
                  - !GetAtt DBCluster.MasterUserSecret.SecretArn
```

The [`AWS::AppSync::DataSource`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-appsync-datasource.html) resource creates data sources for resolvers in this scenario against Amazon RDS. AWS AppSync will use the created `AWS::IAM::Role` resource to access the data source. In this case, we are giving it permission to access the RDS Data API and read the secret containing the database credentials. For the resolvers, we will use [JavaScript](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-reference-js-version.html) instead of [VTL](https://docs.aws.amazon.com/appsync/latest/devguide/tutorials.html). Create a `createTasks.js` file with the following content:

```javascript
import { util } from '@aws-appsync/utils';
import { insert, createPgStatement, toJsonObject } from '@aws-appsync/utils/rds';

export function request(ctx) {
    const { input } = ctx.args;
    const insertStatement = insert({
        table: 'tasks',
        values: input,
        returning: '*',
    });
    return createPgStatement(insertStatement)
}

export function response(ctx) {
    const { error, result } = ctx;
    if (error) {
        return util.appendError(
            error.message,
            error.type,
            result
        )
    }
    return toJsonObject(result)[0][0]
}
```

Create an `updateTasks.js` file with the following content:

```javascript
import { util } from '@aws-appsync/utils';
import { update, createPgStatement, toJsonObject } from '@aws-appsync/utils/rds';

export function request(ctx) {
    const { input: { id, ...values }, condition = {} } = ctx.args;
    const where = {
        ...condition,
        id: {
            eq: id,
        },
    };
    const updateStatement = update({
        table: 'tasks',
        values,
        where,
        returning: '*',
    });
    return createPgStatement(updateStatement)
}

export function response(ctx) {
    const { error, result } = ctx;
    if (error) {
        return util.appendError(
            error.message,
            error.type,
            result
        )
    }
    return toJsonObject(result)[0][0]
}
```

Create a `deleteTasks.js` file with the following content:

```javascript
import { util } from '@aws-appsync/utils';
import { remove, createPgStatement, toJsonObject } from '@aws-appsync/utils/rds';

export function request(ctx) {
    const { input: { id }, condition = {} } = ctx.args;
    const where = {
        ...condition,
        id: {
            eq: id,
        },
    };
    const deleteStatement = remove({
        table: 'tasks',
        where,
        returning: '*',
    });
    return createPgStatement(deleteStatement)
}

export function response(ctx) {
    const { error, result } = ctx;
    if (error) {
        return util.appendError(
            error.message,
            error.type,
            result
        )
    }
    return toJsonObject(result)[0][0]
}
```

Create a `getTasks.js` file with the following content:

```javascript
import { util } from '@aws-appsync/utils';
import { select, createPgStatement, toJsonObject } from '@aws-appsync/utils/rds';

export function request(ctx) {
    const { id } = ctx.args;
    const where = {
        id: {
            eq: id,
        },
    };
    const statement = select({
        table: 'tasks',
        columns: '*',
        where,
        limit: 1,
    });
    return createPgStatement(statement)
}

export function response(ctx) {
    const { error, result } = ctx;
    if (error) {
        return util.appendError(
            error.message,
            error.type,
            result
        )
    }
    return toJsonObject(result)[0][0]
}
```

Create a `listTasks.js` file with the following content:

```javascript
import { util } from '@aws-appsync/utils';
import { select, createPgStatement, toJsonObject } from '@aws-appsync/utils/rds';

export function request(ctx) {
    const { filter = {}, limit = 100, orderBy: _o = [], nextToken } = ctx.args;
    const offset = nextToken ? +util.base64Decode(nextToken) : 0;
    const orderBy = _o.map((x) => Object.entries(x)).flat().map(([ column, dir ]) => ({ column, dir }));
    const statement = select({
        table: 'tasks',
        columns: '*',
        limit,
        offset,
        where: filter,
        orderBy,
    });
    return createPgStatement(statement)
}

export function response(ctx) {
    const { args: { limit = 100, nextToken }, error, result } = ctx;
    if (error) {
        return util.appendError(
            error.message,
            error.type,
            result
        )
    }
    const offset = nextToken ? +util.base64Decode(nextToken) : 0;
    const items = toJsonObject(result)[0];
    const endOfResults = items?.length < limit;
    const token = endOfResults ? null : util.base64Encode(`${offset + limit}`);
    return {
        items,
        nextToken: token,
    }
}
```

With all the resolvers in place, let's add the corresponding resources to the `template.yaml` file:

```yaml
  ListTasksResolver:
    Type: AWS::AppSync::Resolver
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      CodeS3Location: ./src/listTasks.js
      FieldName: listTasks
      TypeName: Query
      DataSourceName: !GetAtt AppSyncDataSource.Name
      Runtime:
        Name: APPSYNC_JS
        RuntimeVersion: 1.0.0

  GetTasksResolver:
    Type: AWS::AppSync::Resolver
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      CodeS3Location: ./src/getTasks.js
      FieldName: getTasks
      TypeName: Query
      DataSourceName: !GetAtt AppSyncDataSource.Name
      Runtime:
        Name: APPSYNC_JS
        RuntimeVersion: 1.0.0

  CreateTasksResolver:
    Type: AWS::AppSync::Resolver
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      CodeS3Location: ./src/createTasks.js
      FieldName: createTasks
      TypeName: Mutation
      DataSourceName: !GetAtt AppSyncDataSource.Name
      Runtime:
        Name: APPSYNC_JS
        RuntimeVersion: 1.0.0

  UpdateTasksResolver:
    Type: AWS::AppSync::Resolver
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      CodeS3Location: ./src/updateTasks.js
      FieldName: updateTasks
      TypeName: Mutation
      DataSourceName: !GetAtt AppSyncDataSource.Name
      Runtime:
        Name: APPSYNC_JS
        RuntimeVersion: 1.0.0

  DeleteTasksResolver:
    Type: AWS::AppSync::Resolver
    Properties:
      ApiId: !GetAtt AppSyncAPI.ApiId
      CodeS3Location: ./src/deleteTasks.js
      FieldName: deleteTasks
      TypeName: Mutation
      DataSourceName: !GetAtt AppSyncDataSource.Name
      Runtime:
        Name: APPSYNC_JS
        RuntimeVersion: 1.0.0
```

The [`AWS::AppSync::Resolver`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-appsync-resolver.html#cfn-appsync-resolver-fieldname) resource specifies the resolver that we link to fields within a schema, using `APPSYNC_JS`(JavaScript) as the runtime:

* `FieldName`: The field on a type that invokes the resolver.
    
* `TypeName`: The type that invokes this resolver.
    
* `Kind`: The resolver type, `UNIT` (default), and `PIPELINE`.
    

So, it's time to redeploy our script. Run the following commands:

```powershell
sam build
sam deploy
```

After deploying, you can test the API using the AWS Console:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1709748136875/0c92ce9d-5bfe-4074-9a5e-f4ba3317e1db.png align="center")

The final scripts can be found [here](https://github.com/raulnq/appsync-aurora-serverless). Thanks, and happy coding.