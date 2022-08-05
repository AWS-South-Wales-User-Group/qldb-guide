---
title: "QLDB Streams"
date: 2020-07-10T18:41:23Z
lastmod: 2020-07-11
weight: 70
draft: false
---

QLDB Streams is a feature that allows changes made to the journal to be continuously written in near real time to a destination Kinesis Data Stream. Consumers can subscribe to the stream, and take appropriate action. There are a number of advantages of this approach:

* QLDB Streams provides a continuous flow of data from a specified ledger in near real time
* QLDB Streams provides an at-least-once delivery guarantee
* Multiple streams can be created with different start/end dates and times. This provides the ability to go back and replay all document revisions from a specific point in time.
* Up to 20 consumers (soft limit) can be configured to consume data from a Kinesis Data Stream

To try out QLDB Streams for yourself, take a look at our demo application - [QLDB Demo App](https://github.com/AWS-South-Wales-User-Group/qldb-simple-demo).

{{< spacer >}}

## Configuring QLDB Stream Resource

There is CloudFormation support for configuring a QLDB Stream resource. There are a number of required properties that need to be set:

* **InclusiveStartTime** - the start date and time from which to start streaming journal data, and which cannot be in the future. If it is before the ledger's creation, QLDB defaults this value to the ledger's creation date time.
* **KinesisConfiguration** - the configuration settings for the destination Kinesis data stream, which specifies whether aggregation should be enabled and the ARN of the stream
* **LedgerName** - the name of the ledger
* **RoleArn** - the ARN of the IAM role to grant QLDB permission to write to data to Kinesis
* **StreamName** - the name of the QLDB journal stream

QLDB Streams are supported by CloudFormation. The following CloudFormation template will create:

* A QLDB Ledger
* A Kinesis Data Stream
* A QLDB Stream to publish changes in the ledger to the data stream

{{< codeblock "language-yaml" >}}
AWSTemplateFormatVersion: 2010-09-09
Resources:
  QLDBLedger:
    Type: AWS::QLDB::Ledger
    Properties:
      Name: qldb-guide-ledger
      DeletionProtection: false
      PermissionsMode: ALLOW_ALL

  QLDBKinesisStream:
    Type: AWS::Kinesis::Stream
    Properties: 
      Name: qldb-kinesis-stream
      ShardCount: 1

  QLDBStreamRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
              - qldb.amazonaws.com
          Action:
            - 'sts:AssumeRole'
      Path: /
      ManagedPolicyArns:
        - !Ref QLDBStreamManagedPolicy

  QLDBStreamManagedPolicy:
    Type: 'AWS::IAM::ManagedPolicy'
    Properties:
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: QLDBGuideStreamPermissionsSID
            Effect: Allow
            Action:
              - 'kinesis:PutRecord*'
              - 'kinesis:DescribeStream'
              - 'kinesis:ListShards'
            Resource: 
              Fn::GetAtt: [QLDBKinesisStream, Arn]

  QLDBStream:
    Type: AWS::QLDB::Stream
    Properties: 
      InclusiveStartTime: "2020-05-29T00:00:00Z"
      KinesisConfiguration: 
        AggregationEnabled: true
        StreamArn:
          Fn::GetAtt: [QLDBKinesisStream, Arn]
      LedgerName: !Ref QLDBLedger 
      RoleArn: 
        Fn::GetAtt: [QLDBStreamRole, Arn]
      StreamName: qldb-stream
{{< /codeblock  >}}

This template can be run using the following command:

```
aws cloudformation deploy --template-file ./qldb-stream.yaml \
--stack-name qldb-guide-demo --capabilities CAPABILITY_IAM
```


{{< spacer >}}

## QLDB Stream Record Types

There are three different types of records written by QLDB. All of them use a common top-level format consisting of the QLDB Stream ARN, the record type, and the payload:

{{< codeblock "language-json" >}}
{
  qldbStreamArn: string,
  recordType: CONTROL | BLOCK | REVISION_DETAILS,
  payload: {
    // data
  }
}
{{< /codeblock  >}}

**CONTROL Record**

A CONTROL record is the first record written to Kinesis, and the last record written when an end date/time is specified. The payload simply states whether this is the first event 'CREATED' or the last event 'COMPLETED'.

{{< codeblock "language-json" >}}
{
  controlRecordType:"CREATED/COMPLETED"
}
{{< /codeblock  >}}

**BLOCK Record**

A block summary record represents the details of a block that has been committed to QLDB as part of a transaction. All interaction with QLDB takes place within a transaction. In the demo application, when a new Bicycle Licence is created, there are 3 steps carried out:

1. A lookup is made on the table to check the provided email address is unique
2. A new licence record is created
3. The licence record is updated to include the document ID generated and returned by QLDB in step 2

The resulting BLOCK record for this is shown below:

{{< codeblock "language-json" >}}
{
  blockAddress: {...},
  ...
  transactionInfo: {
    statements: [
      {
        statement: "SELECT Email FROM BicycleLicence AS b WHERE b.Email = ?\",
        startTime: 2020-07-05T09:37:11.253Z,
        statementDigest: {{rXJNhQbB4tyQLAqYYCj6Ahcar2D45W3ySfxy1yTVTBY=}}
      },
      {
          statement: "INSERT INTO BicycleLicence ?\",
          startTime: 2020-07-05T09:37:11.290Z,
          statementDigest: {{DnDQJXtKop/ap9RNk9iIyrJ0zKSFYVciscrxiOZypqk=}}
      },
      {
          statement: "UPDATE BicycleLicence as b SET b.GUID = ?, b.LicenceId = ? WHERE b.Email = ?\",
          startTime: 2020-07-05T09:37:11.314Z,
          statementDigest: {{xxEkXzdXLX0/jmz+YFoBXZFFpUy1H803ph1OF2Lof0A=}}
      }
    ],
    documents: {...}
  },
  revisionSummaries: [{...}]
}
{{< /codeblock  >}}

All of the PartiQL statements executed are included in the BLOCK record, including SELECT statements, as they form part of the same transaction. If multiple tables are used, then statements against all tables carried out in the same transaction will appear in the BLOCK record.


**REVISION_DETAILS record**

The REVISION_DETAILS record represents a document revision that is committed to the ledger. The payload contains the latest committed view, along with the associated table name and Id. If three tables are updated within one transaction, this will result in one BLOCK record and three REVISION_DETAILS records. An example of one of the records is shown below: 

{{< codeblock "language-json" >}}
{
  tableInfo: {
    tableName: "Orders",
    tableId: "LY4HO2JU3bX99caTIXJonG"
  },
  revision: {
    blockAddress: {...},
    hash: {{hrhsCwsNPzLjCsOBHRtSkMCh2JGrB6q0eOGFswyQBPU=}},
    data: {
      OrderId: "12345",
      Item: "ABC12345",
      Quantity: 1
    },
    metadata: {
      id: "3Ax1in3Mt7L0YvVb6XhYyn",
      version: 0,
      txTime: 2020-07-05T18:22:14.019Z,
      txId: "84MQSpihZfxFzpQ4fGyXtX"
    }
  }
}
{{< /codeblock  >}}

{{< spacer >}}

## Processing Events in AWS Lambda

By default, the QLDB Stream is configured to support record aggregation in Kinesis Data Streams. This allows QLDB to publish multiple stream records in a single Kinesis Data Stream record. This can greatly improve throughput, and improve cost optimisation as pricing for PUTs are by 25KB payload “chunks”.

The demo application makes use of the [Nodejs Kinesis Aggregation and Disaggregation Modules](https://www.npmjs.com/package/aws-kinesis-agg). A Kinesis record event consists of an array of Kinesis records in the structure below: 


{{< codeblock "language-json" >}}
{
  Records: [
    {
      kinesis: {
          ...
          data: '...',
          approximateArrivalTimestamp: 1593728523.059
      },
      ...
    }
  ]
};
{{< /codeblock  >}}

Inside the handler of the AWS Lambda function, the records passed in are processed one at a time for each element in the array using the `map()` function. Each record calls out to `promiseDeaggregate` and then to `processRecords`.

{{< codeblock "language-javascript" >}}
await Promise.all(
  event.Records.map(async (kinesisRecord) => {
    const records = await promiseDeaggregate(kinesisRecord.kinesis);
    await processRecords(records);
  })
);
``{{< /codeblock  >}}

The `promiseDeaggregate` function uses the `deaggregateSync` interface which handles the record aggregation, with each deaggregated record being returned as a resolved `Promise`.

{{< codeblock "language-javascript" >}}
const promiseDeaggregate = (record) =>
  new Promise((resolve, reject) => {
    deagg.deaggregateSync(record, computeChecksums, (err, responseObject) => {
      if (err) {
        //handle/report error
        return reject(err);
      }
      return resolve(responseObject);
    });
});
{{< /codeblock  >}}

Once returned, the record is then processed. This involves decoding the base64 encoded data. The payload is the actual Ion binary record published by QLDB to the stream. This is loaded into memory using `ion-js`, and then any relevant processing can take place. 


{{< codeblock "language-javascript" >}}
async function processRecords(records) {
  await Promise.all(
    records.map(async (record) => {
      // Kinesis data is base64 encoded so decode here
      const payload = Buffer.from(record.data, "base64");

      // payload is the actual ion binary record published by QLDB to the stream
      const ionRecord = ion.load(payload);
      ...
    })
  );
}
{{< /codeblock  >}}

Ion values implement the strongly-typed `dom.Value` interface, so you can use DOM methods to make it easy to work with in `JavaScript`.

{{< codeblock "language-javascript" >}}
  // retrieve the version and id from the metadata section of the message
  const version = ionRecord.payload.revision.metadata.version.numberValue();
  const id = ionRecord.payload.revision.metadata.id.stringValue();
{{< /codeblock  >}}

In the case of the demo, the only record types processed were REVISION_DETAILS with all others being skipped.

{{< codeblock "language-javascript" >}}
  // Only process records where the record type is REVISION_DETAILS
  if (ionRecord.recordType.stringValue() !== REVISION_DETAILS) {
    console.log(`Skipping record of type ${ion.dumpPrettyText(ionRecord.recordType)}`);
  } else {
    // process record
  }
{{< /codeblock  >}}

If a document has been deleted, there will be no `data` section present in the `REVISION_DETAILS` record. The `metadata` section will still be present, which allows the `id` to be retrieved.

{{< codeblock "language-javascript" >}}
  // Check to see if the data section exists.
  if (! ionRecord.payload.revision.data) {
    Log.debug('No data section so handle as a delete');
    ...
  } else {
    const points = ionRecord.payload.revision.data.penaltyPoints.numberValue();
    const postcode = ionRecord.payload.revision.data.postcode.stringValue();
    ...
  }
{{< /codeblock  >}}

{{< spacer >}}


## Handling duplicate and out-of-order records

The following statements about QLDB are set out in the AWS developer guide:

> QLDB streams provide an *at-least-once* delivery guarantee. Each data record that is produced by a QLDB stream is delivered to Kinesis Data Streams at least once. The same records can appear in a Kinesis data stream multiple times. So you must have deduplication logic in the consumer application layer if your use case requires it.

> There are also no ordering guarantees. In some circumstances, QLDB blocks and revisions can be produced in a Kinesis data stream out of order.

Each BLOCK record includes the `blockAddress`:

{{< codeblock "language-json" >}}
blockAddress: {
  strandId: "GJMmYanMuDRHevK9X6MX3h",
  sequenceNo: 3
}
{{< /codeblock  >}}

This details the sequence number of the block within the ledger. As QLDB is immutable, each block gets appended to the end of the journal.

Each REVISION_DETAILS record includes the `version` number of the document in the `metadata` section. Each document uses an incrementing version number with the creation of the record being version `0`. 

If necessary, the use of one or both of these values can help to handle duplicate or out-of-order records.

{{< spacer >}}

## Streaming data from QLDB to DynamoDB

The source code for streaming data from QLDB to DynamoDB can be found [here](https://github.com/AWS-South-Wales-User-Group/qldb-simple-demo/tree/master/streams-dynamodb). For more details about streaming data from QLDB to DynamoDB, see this [blog post](https://dev.to/aws-heroes/real-time-streaming-for-amazon-qldb-3c3c)

As the full document revision is sent each time, creates and updates to DynamoDB can be handled by an upsert. To make this work, the `id` and `version` of the document that is contained in the `metadata` section are used. The `id` is used as the primary key, and the `version` as an attribute of the item. The `upsert` code sample is shown below:  

{{< codeblock "language-json" >}}
const updateLicence = async (id, points, postcode, version) => {
  const params = {
    TableName: TABLE_NAME,
    Key: { pk: id },
    UpdateExpression: 'set penaltyPoints=:points, postcode=:code, version=:version',
    ExpressionAttributeValues: {
      ':points': points,
      ':code': postcode,
      ':version': version,
    },
    ConditionExpression: 'attribute_not_exists(pk) OR version < :version',
  };

  try {
    await dynamodb.update(params).promise();
  } catch(err) {
    Log.error(`Unable to update licence: ${id}. Error: ${err}`);
  }
};
{{< /codeblock  >}}

The critical part is the `ConditionExpression`. This specifies that the item will be created **ONLY** if one of the following conditions is true:

* There is no existing item with this `id` as the primary key (to allow for creates), OR
* The version is greater than the value of the current version attribute

If neither of these conditions are true, then no update will take place, and the following error is thrown:

```
ConditionalCheckFailedException: The conditional request failed
```

There is a challenge with deletes. This is because an older update document revision may arrive after the delete revision has been received. This maybe the case when a QLDB stream is first created and streams all current revisions of documents to date. In the demo, we handled this by using a concept of a 'soft delete' or 'tombstone', by marking the item with an `isDeleted` attribute but without actually deleting it. This means we use an `update` and not a `delete`.

{{< codeblock "language-json" >}}
  const params = {
    TableName: TABLE_NAME,
    Key: { pk: id },
    UpdateExpression: 'set version=:version, isDeleted=:isDeleted',
    ExpressionAttributeValues: {
      ':version': version,
      ':isDeleted': true,
    },
    ConditionExpression: 'attribute_not_exists(pk) OR version < :version',
  };
{{< /codeblock  >}}

{{< spacer >}}

## Streaming data from QLDB to Elasticsearch

For more details about streaming data from QLDB to Elasticsearch, see this [blog post](https://dev.to/aws-heroes/streaming-data-from-amazon-qldb-to-elasticsearch-78c)