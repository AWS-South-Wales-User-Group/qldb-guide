---
title: "Interacting with QLDB"
date: 2020-02-21T18:32:44Z
lastmod: 2020-07-11
weight: 40
draft: false
---

The recommended way to interact with QLDB is by using the official QLDB driver. The driver is open sourced on GitHub and available for the following programming languages:

  * [Java](https://docs.aws.amazon.com/qldb/latest/developerguide/getting-started.java.html)
  * [.NET](https://docs.aws.amazon.com/qldb/latest/developerguide/getting-started.dotnet.html)
  * [Go](https://docs.aws.amazon.com/qldb/latest/developerguide/getting-started.golang.html)
  * [Node.js](https://docs.aws.amazon.com/qldb/latest/developerguide/getting-started.nodejs.html)
  * [Python](https://docs.aws.amazon.com/qldb/latest/developerguide/getting-started.python.html)

This guide currently provides examples and code snippets in Node.js, but will be extended to support other programming languages.

## Environment Setup

A typical setup of dependencies required to interact with QLDB is shown below:

{{< codeblock "language-json" >}}
{
 "dependencies": {
    "amazon-qldb-driver-nodejs": "^2.0.0",
    "aws-sdk": "^2.778.0",
    "aws-xray-sdk-core": "^3.2.0",
    "ion-js": "^4.0.1",
    "jsbi": "^3.1.4"
  },
}
{{< /codeblock  >}}

* The `amazon-qldb-driver-nodejs` module is the official driver provided by AWS
* The `aws-sdk` and `aws-xray-sdk-core` modules are needed to support tracing with X-Ray
* The `ion-js` and `jsbi` modules are needed to easily interact with Amazon ION documents

{{< spacer >}}

## Connect to Ledger

The first step in writing to code to interact with QLDB is to create a `QldbDriver`. This driver provides a high-level abstraction layer above the transaction data API (QLDB Session). It takes care of connection pooling and managing sessions, transactions and retry policies.

{{< codeblock "language-javascript" >}}

const { QldbDriver, RetryConfig } = require('amazon-qldb-driver-nodejs');
const qldbDriver = createQldbDriver();
function createQldbDriver(
      ledgerName = process.env.LEDGER_NAME,
      serviceConfigurationOptions = {},
) {
      const retryConfig = new RetryConfig(4);
      const qldbDriver = new QldbDriver(ledgerName, serviceConfigurationOptions, 10, retryConfig);
      return qldbDriver;
}
function getQldbDriver() {
  return qldbDriver;
}
{{< /codeblock  >}}


The constructor for a new `QldbDriver` can take four parameters:

* *ledgerName* - the name of the ledger to connect to
* *qldbClientOptions* - an object that contains options for configuring the low level client. More details can be found [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/QLDBSession.html#constructor-details).
* *maxConcurrentTransactions* - the number of sessions the driver can hold in the pool, with the default set to the maximum number of sockets specified in the global agent
* *RetryConfig* - config to specify max number of retries, base and custom backoff strategy for retries

The `RetryConfig` was introduced in version 2 of the Nodejs driver. It consists of two attributes:

* *retryLimit* - tells the driver how many times to retry when there are failures. The value must be greater than 0. The default value is 4.
* *backoffFunction* - A custom function that accepts a retry count, error, transaction id and returns the amount of time to delay in milliseconds. A default backoff function is provided

{{< spacer >}}

## Execute Lambda

The `executeLambda` method on the `QldbDriver` is the primary method to execute a transaction against a QLDB ledger. When this method is invoked, the driver acquires a `Transaction` and hands it to the `TransactionExecutor` that is passed in. Once all execution is complete, the driver attempts to commit the transaction. If there is a failure, then the driver will attempt to retry the entire transaction block, so your code should be idempotent.

{{< codeblock "language-javascript" >}}
const createLicence = async (param1, param2, ...) => {
  const qldbDriver = await getQldbDriver();
  await qldbDriver.executeLambda(async (txn) => {
    // all functionality committed as one atomic transaction
    ...
  });
};
{{< /codeblock  >}}

In the example above, if multiple statements are executed, they will all either succeed or be rolled back in one atomic transaction. The driver will only attempt to commit the transaction once all the execution is carried out. If there is a failure, then the driver will attempt to retry the entire transaction block.

{{< spacer >}}


## CRUD Operations

#### Creating a record

To insert a new document into a table, the following syntax is used:

{{< codeblock "language-javascript" >}}
await qldbDriver.executeLambda(async txn => {
    const document = [{'Name': 'name', 'Email': 'name@email.com', 'Telephone': '01234'}];
    const statement = 'INSERT INTO Table ?';
    const result = await txn.execute(statement, document);
    const docIdArray = result.getResultList()
    console.log(JSON.stringify(values, null, 2))
})

{{< /codeblock  >}}

The `execute` method returns a Promise that resolves to a `Result` object. This class represents the fully buffered set of results from QLDB in an array. When a new record is inserted, the result object contains the document ID of the record.

{{< codeblock "language-json" >}}
[
  {
    "documentId": "7ISClqWTgkcLNnBlgdtKYa"
  }
]

{{< /codeblock  >}}

To extract the `documentId` from the returned array, you can use the following syntax:

{{< codeblock "language-javascript" >}}
  const result = await insertDocument(txn, docToInsert);
  const docIdArray = result.getResultList();
  const docId = docIdArray[0].get('documentId').stringValue();

{{< /codeblock  >}}

{{< spacer >}}

#### Reading a record

To read a record, execute a select statement against a table:

{{< codeblock "language-javascript" >}}
const getRecord = async (id) => {
    await qldbDriver.executeLambda(async (txn) => {
        const query = `SELECT * FROM Table WHERE Attribute = ?`;
        const result = await txn.execute(query, id);
        const resultList = result.getResultList();
        ...
    };
};

{{< /codeblock  >}}


The `Result` object returned represents the buffered set of results in an array. The `getResultList()` function returns the list of Ion values returned from the enquiry. You can check the `length` property to determine how many results were returned:

{{< codeblock "language-javascript" >}}
  const resultList = result.getResultList();
  ...
  if (resultList.length === 0) {
      // no record found processing
  } else if (resultList.length === 1) {
      // single record found processing
  } else {
      // multiple records found processing
  }
};

{{< /codeblock  >}}

If you just want to log the document revision returned by QLDB, you can use `JSON.stringify()` to cast the ION to JSON:

{{< codeblock "language-javascript" >}}
  JSON.stringify(resultList[0], null, 2)};

{{< /codeblock  >}}

Otherwise, you can use simple calls to retrieve specific values out of the record that is returned. The following code will retrieve the name as a string value, and a points attribute as a number value:

{{< codeblock "language-javascript" >}}
  const statement = ‘SELECT name, points FROM Licence where ID =  ?’;
  const result = txn.execute(statement, ID);
  const resultList = result.getResultList();
  const name = resultList[0].get('name').stringValue();
  const points = resultList[0].get('points').numberValue();

{{< /codeblock  >}}

You can also use multiple attributes with a WHERE predicate clause.

{{< codeblock "language-javascript" >}}
  const statement = `SELECT * FROM Table WHERE a = ? AND b = ?`;
  const result = await txn.execute(statement, "valueA", "valueB");
  ...
};

{{< /codeblock  >}}

{{< spacer >}}

#### Updating a record

To update a record, execute an update statement against a table:

{{< codeblock "language-javascript" >}}
await qldbDriver.executeLambda(async (txn) => {
  const statement = `UPDATE Table SET A = ?, B = ?, C = ? WHERE D = ?`;
  const result = await txn.execute(statement, valueA, valueB, valueC, valueD);
  const resultList = result.getResultList();
  ...
};

{{< /codeblock  >}}

In the same way as for creating a record, the result object returned contains the document ID of the record that has been updated.

{{< spacer >}}

#### Deleting a record

To delete a record, execute a delete statement against a table:

{{< codeblock "language-javascript" >}}
await qldbDriver.executeLambda(async (txn) => {
  const statement = `DELETE FROM Table WHERE id = ?`;
  const result = await txn.execute(statement, id);
  const resultList = result.getResultList();
  ...
};

{{< /codeblock  >}}

The result object returned contains the document ID of the record that has been deleted.

{{< spacer >}}

## QLDB Shell

Amazon QLDB provides a command line shell for interaction with the transactional data plane. The QLDB shell enables you to run PartiQL statements on ledger data. This shell is written in Python and is open-sourced in a [GitHub repository](https://github.com/awslabs/amazon-qldb-shell). The shell is a great way to try and PartiQL statements and interact with a ledger from a developer perspective.

More details can be found in the [developer guide](https://docs.aws.amazon.com/qldb/latest/developerguide/data-shell.html)

![Using the QLDB Shell](/images/qldbshell.png)

{{< spacer >}}