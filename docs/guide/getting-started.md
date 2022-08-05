---
title: "Getting Started"
date: 2020-02-21T18:32:44Z
lastmod: 2020-02-21
weight: 30
draft: false
---

## Using AWS Console

The first step is to create a ledger. In the AWS console, you specify a name for the ledger, the permissions mode to use, and any optional tags.

![Create Ledger through Console](/images/qldb-create-ledger-console.png)

After creating a ledger, the next step is to create a table and optionally indexes. QLDB is a schemaless database, with no schema enforced on the data in documents within any table.

Indexes are used to improve query performance, and there are a number of limitations:

  * They can only be created on a single field
  * They cannot be dropped once created
  * There is a maximum of 5 indexes per table
  * Query performance is only improved when you use an equality predicate e.g. fieldName = XYZ

At launch, indexes could only be created at the time of table creation. An [update](https://aws.amazon.com/about-aws/whats-new/2020/09/amazon-qldb-launches-index-improvements/) allowed indexes to be created on non-empty tables.

In the console, you click on `Query editor` and then select a ledger. You can then execute the relevant PartiQL statement.

The PartiQL to create a table is:

{{< codeblock "language-shell" >}}
CREATE TABLE table
{{< /codeblock  >}}

The PartiQL to create an index is:

{{< codeblock "language-shell" >}}
CREATE INDEX ON table (field)
{{< /codeblock  >}}

![Create Table through Console](/images/qldb-create-table-console.png)

{{< spacer >}}

## Using AWS CLI

You can create a ledger directly via the AWS Command Line Interface (CLI), using the `createLedger` call. With this, you must specify a ledger name and a permissions mode. The only permissions mode currently supported is `ALLOW_ALL`.

{{< codeblock "language-shell" >}}
aws qldb create-ledger --name qldb-guide --permissions-mode STANDARD
{{< /codeblock  >}}

When you create a ledger, deletion protection is enabled by default. This is a feature in QLDB that prevents ledgers from being deleted by any user. You can disable deletion protection on ledger creation by using the `--no-deletion-protection` parameter.

Optionally, you can also specify tags to attach to your ledger.

> **NOTE**: There is no current way to create a table or index through the AWS CLI

{{< spacer >}}

## Using AWS CloudFormation

There is CloudFormation support for creating a QLDB ledger, but not for creating a table or index.

To ensure that any required tables or indexes are created at deployment time, along with the ledger, you can use a `custom resource` in CloudFormation. Custom resources allows you to write custom provisioning logic in templates that AWS CloudFormation runs anytime you create, update or delete a stack.

QLDB is a fully serverless database. The code examples throughout this guide use AWS Lambda to integrate with QLDB. Many frameworks exist for building serverless applications such as AWS SAM and Serverless Framework. It is strongly recommended to use an abstraction framework on top of CloudFormation for deploying QLDB as part of a serverless application. 

{{< spacer >}}

### Using AWS SAM

The [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) is an open-source framework for building serverless applications. It provides shorthand syntax to express functions, APIs, databases, and event source mappings.

AWS SAM templates are an extension of AWS CloudFormation templates. The following snippet from an AWS SAM template shows the creation of a QLDB ledger, a custom resource to create a table which is dependent upon the ledger being created first, and a custom resource to create an index which is dependent upon the table being created.

{{< codeblock  "language-yaml" >}}
Resources:

  QldbLedger:
    Type: AWS::QLDB::Ledger
    Properties:
      Name: qldb-bicycle-licence-sam
      DeletionProtection: false
      PermissionsMode: STANDARD
      Tags:
        - 
          Key: name
          Value: qldb-bicycle-licence-sam

  QldbTable:
    Type: Custom::CreateQldbTable
    DependsOn: QldbLedger
    Properties:
      ServiceToken: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${CreateQldbTable}
      Version: 1.0 

  QldbIndex:
    Type: Custom::qldbIndexes
    DependsOn: QldbTable
    Properties:
      ServiceToken: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${CreateQldbIndex}
      Version: 1.0 

{{< /codeblock >}}

The `ServiceToken` is the ARN of the AWS Lambda function that CloudFormation invokes when you create, update or delete the stack. This Lambda function can be defined in the same template. The `QldbTable` logical name invokes the `CreateQldbTable` logical name which is defined below:

{{< codeblock  "language-yaml" >}}
  CreateQldbTable:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/functions/createQldbTable.handler
      Runtime: nodejs12.x
      Environment:
        Variables:
          LEDGER_NAME: !Ref QldbLedger
          LICENCE_TABLE_NAME: BicycleLicence
      Policies:
        - AWSLambdaBasicExecutionRole
        - Version: 2012-10-17
          Statement:
            - Effect: Allow 
              Action:
                - qldb:SendCommand
              Resource: !Sub arn:aws:qldb:${AWS::Region}:${AWS::AccountId}:ledger/${QldbLedger}

{{< /codeblock >}}

{{< spacer >}}

### Using Serverless Framework

The [Serverless Framework](https://www.serverless.com/) is a popular open source framework for building serverless applications, that has a wide range of plugins available to extend its functionality.

The serverless framework supports CloudFormation in a `resources` section. The following shows the similar approach for creating a QLDB ledger, table and index as used by AWS SAM.

{{< codeblock  "language-yaml" >}}
resources:
  Resources:
    qldbGuideLedger:
      Type: AWS::QLDB::Ledger
      Properties:
        Name: qldb-simple-demo-${self:provider.stage}
        DeletionProtection: false
        PermissionsMode: STANDARD
        Tags:
          - 
            Key: name
            Value: qldb-simple-demo

    qldbTable:
      Type: Custom::qldbTable
      DependsOn: qldbGuideLedger
      Properties:
        ServiceToken: !GetAtt CreateTableLambdaFunction.Arn
        Version: 1.0  #change this to force redeploy

    qldbIndex:
      Type: Custom::qldbIndexes
      DependsOn: qldbTable
      Properties:
        ServiceToken: !GetAtt CreateIndexLambdaFunction.Arn
        Version: 1.0  #change this to force redeploy
{{< /codeblock >}}

{{< spacer >}}
