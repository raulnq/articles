---
title: "Creating an Amazon  Aurora Serverless v2 Database Using AWS SAM"
datePublished: Tue Jan 30 2024 18:52:57 GMT+0000 (Coordinated Universal Time)
cuid: cls0puryk000908l8fq07cpbh
slug: creating-an-amazon-aurora-serverless-v2-database-using-aws-sam
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706566947687/5655faa5-166d-4de4-8a3d-8e0719a2e256.png
tags: postgresql, aws, serverless, aws-rds, aws-sam

---

[Amazon Aurora Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html) has been available for a while and has proven to be a cost-effective option for certain types of workloads. [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html), as an extension of [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html), can be used to create resources beyond Lambda functions, such as databases. In this article, we will delve into the process of using AWS SAM to create our Amazon Aurora Serverless v2 database.

## Pre-requisites

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

We will require at least four resources, assuming we already have a VPC.

## DB Subnet Group

A [DB subnet group](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbsubnetgroup.html#cfn-rds-dbsubnetgroup-subnetids) is a collection of subnets that you create in a VPC and that you then designate for your DB instances:

```yaml
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Aurora DB subnet group
      SubnetIds:
        - <MY_SUBNET_1>
        - <MY_SUBNET_2>
```

The only requirement is that we must specify two or more subnets located in at least two availability zones.

## VPC Security Group

A [VPC security group](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-securitygroup.html#cfn-ec2-securitygroup-securitygroupingress) is required to control inbound and outbound traffic to the DB instance:

```yaml
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
```

In our case, we define an inbound rule that allows access from any source IP to port 5432.

> Always aim to make your ingress rules as restrictive as possible. The current configuration is for testing purposes only.

## DB Cluster

An [Amazon Aurora DB cluster](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbcluster.html) consists of a DB instance and a cluster volume that holds the data for the DB cluster:

```yaml
  DBCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Delete
    Properties:
      DatabaseName: mydatabase
      DBClusterIdentifier: my-dbcluster
      DBSubnetGroupName: !Ref DBSubnetGroup
      VpcSecurityGroupIds:
        - !Ref DBSecurityGroup
      Engine: aurora-postgresql
      EngineVersion: 15.4
      MasterUsername: <MY_USER>
      MasterUserPassword: <MY_PASSWORD>
      Port: 5432
      ServerlessV2ScalingConfiguration:
        MaxCapacity: 1.0
        MinCapacity: 0.5
```

> The `DeletionPolicy` set to `Delete` will remove the resource and all its content, if applicable, during stack deletion. The current configuration is for testing purposes only. For more information, please refer to the following [documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html).

The properties defined in the cluster are:

* `DatabaseName`: The name of our initial database, which only accepts alphanumeric characters in the case of the PostgreSQL engine.
    
* `DBClusterIdentifier`: The cluster identifier must contain from 1 to 63 letters, numbers, or hyphens.
    
* `DBSubnetGroupName`: The reference to the DB subnet group previously created.
    
* `VpcSecurityGroupIds`: The reference to the VPC security group previously created.
    
* `Engine`: The name of the database engine to be used for the DB cluster, with `aurora-mysql` and `aurora-postgresql` as serverless options.
    
* `EngineVersion`: The version number of the database engine to use. We can list all available options by running the command `aws rds describe-db-engine-versions --engine <MY_ENGINE> --query '*[].[EngineVersion]' --output text --region <MY_REGION>`.
    
* `MasterUsername`: The name of the master user for the DB cluster.
    
* `MasterUserPassword`: The master password for the DB instance. It can include any printable ASCII character except `/`, `'`, `"`, `@`, or a space.
    
* `Port`: The port number on which the DB instances in the DB cluster accept connections. Since we are not setting up an `EngineMode` (because the `serverless` option is only for v1), the port number must be specified for the PostgreSQL engine; otherwise, it will use the default of 3306.
    
* `ServerlessV2ScalingConfiguration`: To define our scaling configuration there are two properties:
    
    * `MaxCapacity`: must be higher than 0.5.
        
    * `MinCapacity`: the smallest value is 0.5.
        

## DB Instance

The final building block for our cluster is the [DB instance](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbinstance.html#cfn-rds-dbinstance-port):

```yaml
  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DBCluster
      DBInstanceIdentifier: my-dbinstance
      DBInstanceClass: db.serverless
      Engine: aurora-postgresql
```

The properties defined in the instance are:

* `DBClusterIdentifier`: The identifier of the DB cluster to which the instance will belong.
    
* `DBInstanceIdentifier`: A name for the DB instance.
    
* `DBInstanceClass`: The compute and memory capacity of the DB instance. In our case, we will use the special class type `db.serverless`.
    
* `Engine`: The name of the database engine we want to use for this DB instance. In our case, we must use the same value specified in the cluster.
    

## Deploy

To create all the resources, run:

```powershell
sam build
sam deploy --guided
```

Once completed, run the following command to get our cluster's endpoints:

```powershell
aws docdb describe-db-clusters --region us-east-2 --db-cluster-identifier my-dbcluster --query 'DBClusters[*].[DBClusterIdentifier,Port,Endpoint,ReaderEndpoint]'
```

Download a PostgreSQL client like [PgAdmin](https://www.pgadmin.org/download/) and test the connection against the database.

## Clean Up

Run the following command to delete all the resources:

```powershell
sam delete
```

The final script can be found [here](https://github.com/raulnq/aws-sam-aurora). Thanks, and happy coding.