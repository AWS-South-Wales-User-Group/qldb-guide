Many applications today benefit from the ability to capture all changes to an item or document, at the point in time at which they occur, and emit these changes to downstream consumers. Common use cases include:

* [**Event driven architecture**](https://aws.amazon.com/event-driven-architecture/) - event driven architectures use events (such as state changes) to trigger further processing, with the events naturally decoupling the systems involved.
* **Analytics and Search** - Stream processing allows the core system of record to emit events downstream to allow purpose-built database engines to be used for analytics or to support requirements like full text search

The advantage of stream processing over traditional batch processing approaches is that changes can be reflected in near real time. A natural choice for these applications has been DynamoDB. Recently, another AWS purpose-built database, Amazon QLDB, announced in-built support for stream processing. This blog looks to give some tips to anyone looking to get started with either DynamoDB Streams or QLDB Streams, and to provide a brief comparison between them.

## Architecture
I am a big advocate of serverless technologies. As such, building out working prototypes for this blog, I used the [Serverless Application Model](https://aws.amazon.com/serverless/sam/), and `AWS Lambda` functions to consume messages from the stream. The architecture for this looks like:

![Stream Processing Architecture](https://dev-to-uploads.s3.amazonaws.com/i/hvpv876s9cukbnrg3a14.png)

In both cases, DynamoDB and QLDB write a stream record that represents a single modification of an item or document to the stream. The Lambda services polls shards in the stream for records, and when available, invokes the Lambda function and waits for the result.

## Overview of Streams
There is a lot of similarity between DynamoDB Streams and Kinesis Data Streams. At a high level, the basic components of each are as follows:

![DynamoDB Streams Terminology](https://dev-to-uploads.s3.amazonaws.com/i/t635xz43yrgamb7za7qq.png)

In both services, data streams are composed of shards, which are containers for stream records.

## DynamoDB Streams

![DynamoDB Streams Overview](https://dev-to-uploads.s3.amazonaws.com/i/j9sapw580mhnzhsozjga.png)

A DynamoDB Stream consists of stream records which represent a single data modification to an item in the DynamoDB table for which the stream is enabled. Each stream record is assigned a sequence number, reflecting the order in which the record was published to the stream. Stream records are organised into groups, or shards. Each shard acts as a container for multiple stream records, and contains information required for accessing and iterating through these records. The stream records within a shard are removed automatically after 24 hours. If you disable a stream, any shards that are open will be closed. The data in the stream will continue to be readable for 24 hours.

Lambda polls shards in the stream for records at a base rate of 4 times per second. When records are available, Lambda invokes your function and waits for the result. If processing succeeds, Lambda resumes polling until it receives more records. By default, Lambda invokes your function as soon as records are available in the stream. If the batch that Lambda reads from the stream only has one record in it, Lambda sends only one record to the function. To avoid invoking the function with a small number of records, you can tell the event source to buffer records for up to five minutes by configuring a batch window. Up to 2 lambda functions can be subscribed to a single stream.

If your function returns an error, Lambda retries the batch until processing succeeds or the data expires. To avoid stalled shards, you can configure the event source mapping to retry with a smaller batch size, limit the number of retries, or discard records that are too old. To retain discarded events, you can configure the event source mapping to send details about failed batches to an SQS queue or SNS topic.

You can also increase concurrency by processing multiple batches from each shard in parallel. Lambda can process up to 10 batches in each shard simultaneously. If you increase the number of concurrent batches per shard, Lambda still ensures in-order processing at the partition-key level.

There is no direct control of the number of shards in a DynamoDB Stream. The number of shards depends on the volume of data in the table as well as the amount of read and write throughput provisioned for the table. In general, when a tables ability to serve a set amount of traffic doubles, so does the number of shards in the stream.


## Kinesis Data Streams

![Kinesis Data Streams Overview](https://dev-to-uploads.s3.amazonaws.com/i/gexw98ghjzwug1hxzxnc.png)

A Kinesis Data Stream consists of stream records which represent all state changes to documents in a QLDB ledger. Each stream record is assigned a sequence number by Kinesis Data Streams. Stream records are organised into groups, or shards. Shards act as units of scale and capacity, and provide a pre-defined write and read capacity. A partition key is used to group data by shard within a stream. Kinesis Data Streams segregates the data records belonging to a stream into multiple shards. It uses the partition key that is associated with each data record to determine which shard a given data record belongs to. Lambda processes records in each shard in order. It stops processing additional records in a shard if your function returns an error. With more shards, there are more batches being processed at once, which lowers the impact of errors on concurrency. Up to 5 lambda functions can be subscribed to a single stream.

The Lambda service polls the shard once per second for a set of records. The Lambda service synchronously invokes the Lambda function with the batch of records. By default, Lambda invokes your function as soon as records are available in the stream. If the batch that Lambda reads from the stream only has one record in it, Lambda sends only one record to the function. To avoid invoking the function with a small number of records, you can tell the event source to buffer records for up to 5 minutes by configuring a batch window. Before invoking the function, Lambda continues to read records from the stream until it has gathered a full batch, or until the batch window expires.

If the Lambda returns successfully, the Lambda service advances to the next set of records. If your function returns an error, Lambda retries with the same set of records until processing succeeds or the data expires. To avoid stalled shards, you can configure the event source mapping to retry with a smaller batch size, limit the number of retries, or discard records that are too old. To retain discarded events, you can configure the event source mapping to send details about failed batches to an SQS queue or SNS topic.

You can map a Lambda function to a shared-throughput consumer (standard iterator), or to a dedicated-throughput consumer with enhanced fan-out. For standard iterators, Lambda polls each shard in your Kinesis stream for records at a base rate of once per second. When more records are available, Lambda keeps processing batches until the function catches up with the stream. The event source mapping shares read throughput with other consumers of the shard. Enhanced fan-out consumers get a dedicated connection to each shard that doesn't impact other applications reading from the stream, and uses HTTP/2 to reduce latency by pushing records to Lambda over a long-lived connection and by compressing request headers.

You can also increase concurrency by processing multiple batches from each shard in parallel. Lambda can process up to 10 batches in each shard simultaneously. If you increase the number of concurrent batches per shard, Lambda still ensures in-order processing at the partition-key level.

Kinesis Data Streams supports the concept of aggregation. This refers to the combining of multiple records in a Kinesis Data Streams record. Aggregation allows customers to increase the number of records sent per API call, which effectively increases producer throughput.


## Using Lambda and Event Source Mapping

The examples all use Lambda functions to subscribe to the stream. Both DynamoDB Streams and Kinesis Data Streams act as event sources for Lambda. Lambda gives many benefits when used for Stream Processing

* there are no servers to manage
* there are no stream consumption costs when no new records to process
* automatically scales to handle number of shards
* handles the checkpoint for reading the stream


This blog doesn't cover setting up QLDB Streams. Detailed information can be found in the [QLDB Guide](https://qldbguide.com/docs/introduction/qldb-streams/).

DynamoDB Streams can be enabled by specifying the `StreamSpecification` in the CloudFormation template:

```yaml
  DynamoDBStreamsSource:
    Type: AWS::DynamoDB::Table
    Properties: 
      TableName: DynamoDBStreamsSource
      AttributeDefinitions: 
        - AttributeName: id
          AttributeType: S
      KeySchema: 
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
      StreamSpecification:
        StreamViewType: NEW_IMAGE
```

There are 4 different options for `StreamViewType`:

The possible values are:

* `KEYS_ONLY` - only the key attributes of the modified item are written to the stream
* `NEW_IMAGE` - the entire item, as it appears after it was modified, is written to the stream
* `OLD_IMAGE` - the entire item, as it appears before it was modified, is written to the stream
* `NEW_AND_OLD_IMAGES` - both the new and the old images of the item are written to the stream

The Lambda function that consumes messages from the stream must be configured to allow permission to write to an `on failure destination`, which in this case is an SQS queue.

``` yaml
  QldbStreamsKinesisLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      ...
      Policies:
        - AWSLambdaBasicExecutionRole
        - Version: 2012-10-17 
          Statement: 
            - Effect: Allow 
              Action:
                - "sqs:SendMessage"
              Resource:
                Fn::GetAtt: [StreamsFailureQueue, Arn]

  StreamsFailureQueue:
    Type: AWS::SQS::Queue
```

Finally, you can configure the `EventSourceMapping` which creates a mapping between an event source (DynamoDB Streams or Kinesis Data Stream) and the Lambda function:

```yaml
  MyEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      BatchSize: 50
      BisectBatchOnFunctionError: true
      DestinationConfig: 
        OnFailure: 
          Destination: !GetAtt StreamsFailureQueue.Arn
      Enabled: true
      EventSourceArn: !GetAtt LicenceStreamKinesis.Arn
      FunctionName: !GetAtt QldbStreamsKinesisLambdaFunction.Arn
      MaximumBatchingWindowInSeconds: 30
      MaximumRecordAgeInSeconds: -1
      MaximumRetryAttempts: -1
      ParallelizationFactor: 1
      StartingPosition: "TRIM_HORIZON"
```

The `EventSourceMapping` provides a number of important configuration options

**MaximumBatchingWindowInSeconds**

The `MaximumBatchingWindowInSeconds` parameter allows you to tune how long to wait before processing a batchm ideal for low traffic or tasks that aren't time sensitive. Lambda reads records from a stream at a fixed cadence and invokes a function with a batch of records. This parameter allows you to wait as long as 300s to build a batch before invoking a function. Now a function is invoked when one of the following conditions is met:

* the payload size reaches 6MB
* the Batch Window reaches its maximum value
* the Batch Size reaches its maximum value

With Batch Window you can increase the average number of records passed to the function with each invocation. This is helpful when you want to reduce the number of invocations and optimise cost.

**ParallelizationFactor**

The `ParallelizationFactor` parameter specifies the number of batches to process from each shard concurrently. This enables faster stream processing of ordered records at high volume, using concurrent Lambda invocations per shard. Each shard has uniquely identified sequences of data records. Each record contains a partition key to guarantee order and are organized separately into shards based on that key. The records from each shard must be polled to guarantee that records with the same partition key are processed in order. When there is a high volume of data traffic, you want to process records as fast as possible. Before this release, customers were solving this by updating the number of shards on a Kinesis data stream. Increasing the number of shards increases the number of functions processing data from those shards. One Lambda function invocation processes one shard at a time. The Parallelization Factor allows you to specify the number of concurrent batches that Lambda polls from a single shard. This feature introduces more flexibility in scaling options for Lambda and Kinesis. The default factor of one exhibits normal behavior. A factor of two allows up to 200 concurrent invocations on 100 Kinesis data shards. The Parallelization Factor can be scaled up to 10. Each parallelized shard contains messages with the same partition key. This means record processing order will still be maintained and each parallelized shard must complete before processing the next.

Lambda processes data records from Kinesis and DynamoDB streams in batches. Previously when your function returns an error, Lambda stops processing any data in the impacted shard and retries the entire batch of records. These records are continuously retried until they are successfully processed by Lambda or expired by the event source. The following four failure-handling features allow for more customised responses to data processing failures:

**BisectBatchOnFunctionError**

With Bisect on Function Error enabled, Lambda breaks the impacted batch of records into two when a function returns an error, and retries them separately. This allows you to easily separate the malformed data record from the rest of the batch, and process the rest of data records successfully.


**MaximumRecordAgeInSeconds**

Your Lambda function can skip processing a data record when it has reached its Maximum Record Age, which can be configured from 60 seconds to 7 days.  


**MaximumRetryAttempts**

Your Lambda function can skip retrying a batch of records when it has reached the Maximum Retry Attempts, which can be configured from 0 to 10,000.  

**DestinationConfig**

Now your Lambda function can continue processing a shard even when it returns an error. When a data record reaches its Maximum Retry Attempts or Maximum Record Age, you can send its metadata like shard ID and stream ARN to one of these two destinations for further analysis: an SQS queue or SNS topic.  

I configured a separate Lambda function to use SQS as an event source and consume these error messages. The Lambda function requires permission to either the `dynamodb` or `kinesis` `GetShardIterator` and `GetRecords` actions. For Kinesis, the stream name is required to retrieve the shard iterator, so I configure this as an environment variable within the SAM template.


``` yaml
  DLQHandlerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/get-dlq-message.lambda_handler
      Runtime: nodejs12.x
      Timeout: 10
      Environment:
        Variables:
          StreamName: !Ref LicenceStreamKinesis
      Policies:
        - AWSLambdaBasicExecutionRole
        - Version: 2012-10-17 
          Statement: 
            - Effect: "Allow"
              Action:
                - "kinesis:GetShardIterator"
                - "kinesis:GetRecords"
              Resource:
                Fn::GetAtt: LicenceStreamKinesis.Arn
      Events:
        MySQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt StreamsFailureQueue.Arn
            BatchSize: 1
```

## Processing Failed Messages

When a data record cannot be processed, it is now possible to send metadata about the record to SQS or SNS for futher analysis. The actual records are not included in the metadata. Instead, you must process the record and retrieve them from the stream before they are lost. The approach for DynamoDB Streams and Kinesis Data Streams is roughly the same in this respect. The following code example shows how the original stream records from DynamoDB Streams can be retrieved



```javascript
const AWS = require('aws-sdk');
const dynamodbStreams = new AWS.DynamoDBStreams();

exports.lambda_handler = async (event, context) => {

    for(const record of event.Records) {
      const { body } = record;
      const msg = JSON.parse(body);
      const ShardId =  msg.DDBStreamBatchInfo.shardId;
      const StreamArn =  msg.DDBStreamBatchInfo.streamArn;
      const SequenceNumber =  msg.DDBStreamBatchInfo.startSequenceNumber;

      try {
        const iterator = await getShardIterator(ShardId, StreamArn, SequenceNumber);
        const records = await dynamodbStreams.getRecords(iterator).promise();

        records.Records.forEach(message => { 
          const messageData = JSON.stringify(message.dynamodb);
          console.log(`messageData: ${messageData}`);
          // Process the message
        });

      } catch(err) {
        console.log(`Error: " ${err}`);
      }

    }
};

async function getShardIterator(ShardId, StreamArn, SequenceNumber) {
  const params = {
    ShardId,
    ShardIteratorType: "AT_SEQUENCE_NUMBER",
    StreamArn,
    SequenceNumber
  };
  return await dynamodbStreams.getShardIterator(params).promise();
}
```

The one difference is if the Kinesis Data Stream has aggregation enabled. In which case, you will need to use a deaggregation library. In addition, with aggregation enabled, a Kinesis record may contain multiple QLDB document revisions. If just a single one failed, the record itself which contains multiple revisions will be placed on the DLQ. Best practice is to re-process all of them and treat the stream processing steps as idempotent.


## Conclusion

There are many similarities when utilising Stream Processing with either DynamoDB Streams or QLDB Streams, but there are also some important distinctions. The main ones observed are called out below:

**Replay**

Replay is the abilility to replay all the modifications in a stream that have taken place to date or between a specific time window. Many applications need to maintain a history of item-level revisions for audit or compliance purposes and to be able to retrieve the most recent version easily. There is a design patten to support this with DynamoDB when [using sort keys for version control](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-sort-keys.html). In some circumstances, you may want to replay all events:

* a new requirement has come along which requires a new purpose-built database to be used that needs to be loaded
* an error in business logic has happened, and now the downstream data is inaccurate and needs to be replayed.

There is no out of the box approach to replay all item modifications with DynamoDB. A stream is either enabled or not, and data in an existing stream is retained for 24 hours.

QLDB provides the ability to create up to 5 streams for a given ledger. When a QLDB Stream is created, you can optionally specify both a start data and an end date for all document revisions. This means it is possible to setup a new stream and replay all document revisions that have ever taken place since the ledger was created.


**Modification Information**

DynamoDB is much more explicit with information about modifications or changes to an item. An `eventName` attribute specifies whether the item is new, an update, or a deletion.

```yaml
{
    "Records": [
        {
            "eventID": "c5d7dc7d2712678cf0505387f94a98eb",
            "eventName": "INSERT" | "MODIFY" | "REMOVE",
            "eventVersion": "1.1",
            "eventSource": "aws:dynamodb",
            ...
```

In addition, it provides options to transfer both old and new images, or the associated keys.

QLDB does not provide the same level of detail. It adopts a pattern commonly referred to as [Event-Carried State Transfer](https://martinfowler.com/articles/201701-event-driven.html), which is the equivalent of the `NEW_IMAGE` setting for `StreamViewType`. You can determine the state of the revision based on the following:

* A revision with a `version` of 0 is a new document
* A revision with no `data` section is a deletion


**Delivery Guarantee**

DynamoDB Streams offers strong guarantees, including the following:
* Each stream record appears exactly once in the stream.
* For each item that is modified in a DynamoDB table, the stream records appear in the same sequence as the actual modifications to the item.

QLDB Streams does not offer these strong guarantees. It provides:
* QLDB streams provides an at-least-once delivery guarantee. Each data record produced by a QLDB stream is delivered to Kinesis Data Streams at least once. The same records can appear in a Kinesis data stream multiple times
* There are no ordering guarantees. In some circumstances, QLDB blocks and revisions can be produced in a Kinesis data stream out of order

Luckily, each `REVISION_DETAILS` record from QLDB includes the `version` number of the document in the `metadata` section. Each document includes an incrementing version number with the creation of the record being version `0`.


**Next Steps**

Stream processing is growing in popularity, and both DynamoDB Streams and QLDB Streams provide fully managed serverless options to support highly available, resilient and decoupled solutions. Why not try it out for yourself. The following repositorities will allow you to quickly get up and running, and reach out if you have any questions:

* [DynamoDB Streams PoC](https://github.com/mlewis7127/dynamodb-streams)
* [QLDB Streams PoC](https://github.com/AWS-South-Wales-User-Group/qldb-bicycle-licence-sam)

My next step is to try out the recent launch of [Kinesis Data Stream support for DynamoDB](https://aws.amazon.com/about-aws/whats-new/2020/11/now-you-can-use-amazon-kinesis-data-streams-to-capture-item-level-changes-in-your-amazon-dynamodb-table/)