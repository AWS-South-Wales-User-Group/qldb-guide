## Introduction

Amazon QLDB is a fully managed ledger database. A key feature is that each time a change is made to a record, the existing state of that record is not mutated, but instead an entirely new revision is created. This allows you to automatically track all changes and verify the state of a record at any point in time.

There are many real world examples which benefit from this approach. For example, to accurately calculate the current balance in a bank account, you need to know all the credits and debits that have been applied. Without this complete history, you have lost data lineage and traceability. Amazon QLDB provides this full audit trail without the need to write your own custom triggers or audit logs.

This approach of immutable data is increasingly common, especially with the growing popularity of event sourcing and streaming technologies like Apache Kafka. Martin Kleppmann, author of "Designing Data-Intensive Applications", summarises it well when he argues today, with inexpensive storage, a database should be “[an always-growing collection of immutable facts](https://martin.kleppmann.com/2015/11/05/database-inside-out-at-oredev.html)”.

## Deletion versus Redaction

The announcement of support for redaction now means QLDB supports both delete and redact operations, so let's look at the differences between them.

In QLDB you can execute a DELETE statement against a document in a table using the following syntax:

`DELETE FROM table_name WHERE condition`

The DELETE statement can only be executed against an active document. It results in a new revision being created containing just metadata, with null user data. This final revision indicates the document is deleted. It is, however, only a logical deletion.

In QLDB, all transactions are committed first by appending a block to the journal, before the user data is materialised into the tables previously created using a `CREATE TABLE` statement. Each of these user defined tables has a corresponding system-defined table that contains both the user data, and the QLDB-generated metadata. Once a document has been deleted, it cannot be found when querying the user table with a `SELECT` statement. However, the `history` function in QLDB returns all revisions from the system-defined view of your table, and this can still be used to return data for all revisions of a record, even after it has been deleted.

The new redaction operation is the *only* way to permanently delete user entered data in QLDB. You execute a `REDACT` statement against an inactive document revision in a table using the following syntax:

``EXEC redact_revision `block-address`, 'table-id', 'document-id'``

Note that the block-address is an ION literal, and so is surrounded by backticks `` (`..`) ``. The table-id and document-id are considered string values and surrounded by single quotation marks `('..')`.

This runs a stored procedure with the redaction of data processed asynchronously by QLDB. Once complete, it results in the user data in the specified revision being deleted, but leaves the journal sequence and document metadata unchanged. This maintains the overall data integrity of the ledger, and means that cryptographic verification still works. The `history` function will still return the redacted revisions for a given document, but the user data will have been permanently deleted and is no longer accessible.

Check out the video below to see this in action:

<iframe width="560" height="315" src="https://www.youtube.com/embed/FnaUCg2XhSw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Immutable Data and Data Protection Law

A common challenge is how immutability of data (with all the benefits it brings) works in the face of data protection law in some regions. The General Data Protection Regulation (GDPR) is a regulation in EU law on data protection and privacy, which includes the following two articles:

* Article 16 is the "right to rectification". This gives the consumer the right to have any inaccurate personal data corrected without undue delay.

* Article 17 is the "right to erasure" ('right to be forgotten'). This gives the consumer the right to have their personal data erased when certain criteria are met. This includes when the data is no longer necessary to be collected or processed, or where there is no overriding legitimate grounds or legal obligation.

The exact rules around the correct response to each request can vary. In many cases, an organisation will have legitimate grounds to retain certain information for a period of time, in case of a future dispute. They don't want to end up in a position where data has been destroyed that have could have been provided as evidence in a court case.

With this in mind, let's look at different patterns in QLDB to support these requirements. To do that, we will use a fictional Bicycle Licence use case with the entities below

To do that, we will use a fictional bicycle licence use case with the entities below.

![Bicycle Licence Entity Diagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/esiqk86l6zpobe3v8kya.jpg)

When a customer applies for a licence, a new Licence record and Contact record is created. A licence can have zero-to-many Endorsement records that contain details about the offence. Each endorsement is added to a Mapping table that links the endorsement with the licence.

## Patterns

### Pattern 1: Redact a document revision after a period of time

For this scenario, a specific revision of a licence document must be redacted after a certain period of time has elapsed. This could be because the fact a licence once had a status of 'DISQUALIFIED' needs to be removed.

In order to redact a specific revision, you need to know the block address of that specific revision, the table id, and the unique id of the document. 

To retrieve the block address of a revision which is not the latest revision, you can use the `history` function as shown below:

```sql
SELECT * FROM history(Licence, `2021-01-01`, `2022-11-11`) WHERE metadata.id = ?
```
The history function takes in the table name and an optional start and end time in ISO-8601 format. This will return all revisions that were active during this period. The code example below uses the version number in the condition to retrieve only a single revision. It then retrieves the block address.

```javascript
const qldbDriver = await getQldbDriver();
await qldbDriver.executeLambda(async (txn) => {
    // retrieve revision from history function
    const query = 'SELECT * FROM history(Licence) where metadata.id = ? AND metadata.version = ?';
    const result = await txn.execute(query, id, version); 
    const resultList = result.getResultList();
    const blockAddress = resultList[0].blockAddress;
    ...
});
```

In addition to the block address, we also need to know the tableId. This can be found by querying the system catalog table `information_schema.user_tables`:

```sql
SELECT tableId FROM information_schema.user_tables WHERE name = 'Licence'
```

Finally, we can redact this revision by passing in the required parameters. 

```javascript
const query = `EXEC redact_revision ?, '${tableId}', '${id}'`; 
txn.execute(query, blockAddress);
```

It is important to remember that redaction is an asynchronous operation. You can check the redaction is complete by querying history, calling `GetBlock` or `GetRevision`, checking the block record on a QLDB stream, or checking block details from a journal export.

An example QLDB document pre-redaction is shown below:

```shell
{ 
  blockAddress: { 
    ...
  },
  hash: {{zK0km94yws8HNT7yw20uy2zu8m34TvyTzk/TCT0DdcA=}},
  data: { 
    ...
  },
  metadata: { 
    ...
  }
}
```
After redaction, the `data` section has been replaced by a `dataHash` field. 

```shell
{ 
  blockAddress: { 
    ...
  },
  hash: {{zK0km94yws8HNT7yw20uy2zu8m34TvyTzk/TCT0DdcA=}},
  dataHash: {{rCAQw089uXEYtZ+YeyLaSXWUwc+AVDPZ7gdr35y45Ig=}},
  metadata: { 
    ...
  }
},
```
QLDB also appends a new block to the journal that includes an additional `redactionInfo` entry that contains a reference to the revision that was redacted.

### Pattern 2: Redact a complete record

For this scenario, an endorsement record needs to be redacted after a certain time period has expired. This is a great example where you can use a `DELETE` statement to logically delete the record, so it would only be visible in the history for any forensic investigation. However, in this case all details about the endorsement must be removed.

You cannot redact the current active version of a document. This means you need to execute a `DELETE` statement and then redact all previous revisions.

Because you cannot redact the current active version of the document, the deletion will need to be committed first, before you can execute the redaction. In addition, if you need to redact multiple revisions, you can only redact a single revision per transaction.

To carry this out in a single function, you need to wrap code in separate `executeLambda` methods on the QLDB driver. The code example below shows how to redact multiple revisions by iterating over a result set:

```javascript
const qldbDriver = await getQldbDriver();
await qldbDriver.executeLambda(async (txn) => {
  // Delete endorsement

  // Get the current record
  const query = 'SELECT * FROM history(Endorsement) where metadata.id = ?';
  const result = await txn.execute(query, id); 
  historyResponse = result.getResultList();
  ...
});

for (const element of historyResponse) {
  let blockAddress = element.blockAddress;
  await qldbDriver.executeLambda(async (txn) => {
  try {
    if (element.dataHash === null || element.dataHash === undefined) {
      const redactQuery = `EXEC redact_revision ?, '${tableId}', '${id}'`; 
      txn.execute(redactQuery, blockAddress);
    } else {
      // Revision already been redacted so skipping
    }  
  } catch (error) {
    // Error handling logic
 }  
});
}
```

### Pattern 3: Redact an attribute of a record

For this scenario, there is a requirement to only hold the current mobile number on record and redact older versions. As a result, mobile number is modelled in a separate Contact table.

The way this is implemented in a single update request is as follows:

1. Update the Contact record removing the `mobile` attribute using the `UPDATE-REMOVE` or `FROM-REMOVE` statement. This creates a new revision with the same details, but with the mobile attribute removed
    ```sql
    UPDATE Contact REMOVE mobile WHERE contactId = '123'
    ```

2. Redact the previous revision that will still contain the mobile attribute in history

3. Update the Contact record with the new mobile attribute

This approach enables the full history of the document to be retained but with all references to the mobile attribute removed apart from the current active revision.

## Alternative Approaches

Before the release of the redaction operation, there were other approaches that could be used in isolation or together.

### QLDB Streams

In this option, QLDB remains as the source of truth, with data streamed out to another database engine where data can be physically deleted as needed. An extension is to adopt a Command Query Responsibility Segregation (CQRS) architectural approach, where multiple downstream read models are created, each one satisfying a specific purpose and only containing the minimum data set required. A nice feature of QLDB is that these read models can be re-populated at any point in time by re-streaming the data from the journal.

### QLDB Fine Grained Permissions

In this option, you protect PII data by restricting access to the history function so data is not visible to most users once it has been deleted from the table. Only users with special clearances would be in a group that had the IAM `qldb:PartiQLHistoryFunction` permission to allow them to query history. All other users would be in group that only provided access to the current active record. This option uses the AWS best practice principles around least privilege.

### Separate out PII data

The final option is to store any attributes or entities that would require redaction or hard physical deletion in an alternative database engine, and store the link to the record in QLDB.

## Conclusion

The redaction operation has opened up new approaches for adopting QLDB and taking advantage of all the benefits of a complete cryptographically verifiable audit trail, whilst meeting any strict interpretations of data protection law. 

To find out more and try out redaction for yourself, head over to https://qldbguide.com/ and try out the demo. I've recorded a quick walk through to show how to get started

<iframe width="560" height="315" src="https://www.youtube.com/embed/_8KCvbZrT3Q" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>