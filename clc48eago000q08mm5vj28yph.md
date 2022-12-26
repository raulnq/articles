# Migrating a database from Microsoft Azure SQL to Amazon Aurora (PostgreSQL)

Database migration, especially between heterogeneous engines, is always a complex task. Luckily for us, AWS offers a set of services/tools from the schema conversion until the data migration to make our life easier.

> AWS Database Migration Service (AWS DMS) is a cloud service that makes it possible to migrate relational databases, data warehouses, NoSQL databases, and other types of data stores. You can use AWS DMS to migrate your data into the AWS Cloud or between combinations of cloud and on-premises setups.

## Concepts

*   **Source Endpoint**: Source database you wish to migrate.
    
*   **Target Endpoint**: Database to which your data will be migrated.
    
*   **Replication instance**: It's an EC2 instance on which the migration task runs.
    
*   **Database migration task**: Used to move data from a source to a target in a Replication instance.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671984873529/789eb034-d249-4ce6-8a82-4774da654f3e.png align="center")

In this post, we will migrate an Azure SQL Server database to an Amazon Aurora database. Let's start.

## Prerequisites

*   An Azure SQL Server, up and running.
    
*   An Amazon Aurora Server (PostgreSQL compatible edition), up and running.
    
*   Create a Northwind database (Azure SQL) and run [this script](https://github.com/microsoft/sql-server-samples/blob/master/samples/databases/northwind-pubs/instnwnd%20(Azure%20SQL%20Database).sql).
    
*   Create a Northwind database (Amazon Aurora).
    

## Migration Process

### Convert the Azure SQL database schema to a PostgreSQL schema

Currently, there are two options for schema conversion:

*   [AWS DMS Schema Conversion (DMS SC)](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_SchemaConversion.html)
    
*   [AWS Schema Conversion Tool (AWS SCT)](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)
    

> You can use the AWS Schema Conversion Tool (AWS SCT) to convert your existing database schema from one database engine to another. You can convert relational OLTP schema or data warehouse schema. Your converted schema is suitable for an Amazon Relational Database Service (Amazon RDS) MySQL, MariaDB, Oracle, SQL Server, PostgreSQL DB, an Amazon Aurora DB cluster, or an Amazon Redshift cluster

Follow the instructions [here](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Installing.html) to install the AWS SCT. We will need to download the following database drivers:

*   [Azure SQL Database](https://learn.microsoft.com/en-us/sql/connect/jdbc/release-notes-for-the-jdbc-driver?view=sql-server-ver15#72)
    
*   [Amazon Aurora PostgreSQL-Compatible Edition](https://jdbc.postgresql.org/download/postgresql-42.2.19.jar)
    

Open the AWS SCT, go to the **File** menu, and select the **New Project Wizard**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671987336349/1da2645c-c076-4d4c-a231-f55e793314ed.png align="center")

Click **Next** and fulfill all the connection data against the source database (the first time the wizard will ask for the driver location path):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671734474007/6e97b92d-cd84-445e-ac2d-a835ab53a271.png align="center")

Test the connection (make sure your source database allows connections from your IP address) and, if everything is okay, click **Next** and choose the schema to migrate:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671734612852/b6a81680-9e0f-43c9-9f89-45b82833fc25.png align="center")

Click **Next** to see a migration assessment of your schema to different database engines:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671737962184/d2a58e9d-356d-4d61-b1d4-72ad8c3426a0.png align="center")

Click **Next** and fulfill all the connection data against the target database (the first time the wizard will ask for the driver location path):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671738034801/62183bd0-2660-47bd-843c-236b1ea48115.png align="center")

Click Finish to see a summary of the schema migration. Select the **Action Items** to see the suggested SQL changes. Right-click on the source database and click **Convert Schema**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671738593819/b4e640a3-2ded-4a29-801e-769f0cf86add.png align="center")

At this point, we can see the generated SQL script for every entity on our source database. We can edit it, especially if you have a blue issue (those require manual intervention to fix it):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671741445654/0e897924-35c8-423c-9577-1cb93f6069f2.png align="center")

Go to the target database, right-click on the target schema and click **Apply to database**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671741524735/f3799a1c-f3b3-440d-bda3-5acddc89843a.png align="center")

And that's it. You have your schema fully migrated into the target database:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671743232708/f3568a34-7637-4785-a387-1278f9326d60.png align="center")

### Create an AWS DMS replication instance

In the AWS Console, go to the [Database Migration Service](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html)**, click on Replication instances** and create a new one:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672021508007/f862bb33-b5a2-4c59-ac33-c9af6630ce99.png align="center")

### Define an AWS DMS endpoints

In the AWS Console, go to the **Database Migration Service, click on Endpoints** and create the source endpoint:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672022352331/63e319e8-89be-491b-940e-b7e78e20e7cf.png align="center")

Optionally you can test the connection against your database (use the **Replication instance** created in the previous step). Make sure your database allows connections from the IP address of the **Replication instance**. Let's create the target endpoint:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672022973434/3b6fbaf0-ce77-4d58-ad9b-0301a0a9d92f.png align="center")

### Create an AWS DMS migration task

In the AWS Console, go to the **Database Migration Service, click on Database migration tasks** and create a new one:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672023621481/8e8eb8c2-7f2c-4e30-bc93-7d560d182437.png align="center")

**Nothing** was chosen as the **Target table preparation mode** because we will manually truncate all the tables before every run of this task. As AWS DMS does not follow any order during the migration, disabling foreign key constraints is highly recommended. Open our migration task (wait until it completes) and see the result of our migration:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672024269288/801834e6-d404-49ce-b006-02f2e610744b.png align="center")

Thanks, and happy coding.