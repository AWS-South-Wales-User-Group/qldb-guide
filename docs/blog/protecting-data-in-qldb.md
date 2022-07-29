## Background

When Amazon QLDB was first launched, it only supported AWS owned keys to encrypt data at rest. Amazon QLDB launched support for customer managed AWS KMS keys on July 22, 2021. For many organisations, especially those in regulated environments, this is a big deal. So let's dive deeper into what this means by taking a look at the AWS KMS service

## Customer Master Keys (CMK)

The primary resource in AWS KMS is a customer master key (CMK), which is sometimes referred to as the root or master key. The CMK can be used to encrypt, decrypt, re-encrypt or generate a data key. There are three types of CMK supported by AWS KMS:

* AWS Owned CMK
* AWS Managed CMK
* Customer Managed CMK

AWS Owned and Customer Managed CMKs are the ones now supported by QLDB, so lets compare these two types:

### Key Material

Key material is the secret string of bits used in a cryptographic algorithm. It must be kept secret to protect the cryptographic operations that use it. With AWS Owned CMKs, all of this is taken care of by AWS. With Customer Managed CMKs, there are 3 options for the source of the key material:

* **AWS_KMS** - AWS KMS creates and manages the key material for the CMK in its own key store. This is the default and the recommended value for most CMKs. By default, these keys are protected by hardware security modules (HSMs) that are FIPS 140-2 validated crypytographic modules. The key material exists in plaintext only within the HSMs, and only when in use. Otherwise, the key material is encrypted and stored in durable persistent storage. The key material that AWS KMS generates for CMKs never leaves the boundary of AWS KSM HSMs unencrypted, and is never exported or transmitted in any AWS KMS API operations.

* **AWS_CLOUDHSM** - AWS KMS creates the key material for the CMK in a custom key store that is backed by FIPS 140-2 Level 3 HSMs in an AWS CloudHSM cluster that you provision and manage. Cryptographic operations are then performed entirely within the AWS CloudHSM cluster. This adds an extra layer of maintenance responsibility and an additional dependency. This comes at a cost, including more operational overhead. However, if you have legal requirements around not storing key material in a shared environment, mandating that it be subject to a secondary, independent audit path, or that the HSMs must be certified to FIPS 140-2 Level 3, then this is the option for you.

* **EXTERNAL** - The CMK is created with no key material. Later, this is imported into the CMK. Although this puts you in complete control of the key material, there are many considerations to take into account. For example, you are responsible for generating the key material, and transferring it to AWS. You are responsible for the overall availability and durability of the key material, and will still need to retain a copy of the key material in a system you control. Once you import key material into a CMK, you cannot import different key material into that CMK, or enable automatic key rotation.

### Key Storage

With AWS Owned CMKs, the keys are not stored in your AWS account. They are part of a collection of KMS keys that an AWS service owns and manages for use in multiple AWS accounts. In contrast, with Customer Managed CMKs the keys are stored in your own AWS account.

### Key Rotation

You cannot manage key rotation for AWS Owned CMKs. The key rotation strategy for an AWS Owned CMK is determined by the AWS service that creates and manages the CMK.

With Customer Managed CMKs, you control the key rotation policy. For CMKs with AWS KMS generated key material, AWS provide the option to automatically rotate the key material every year. AWS KMS also saves the CMK's older key material in perpetuity so it can be used to decrypt data that is encrypted. AWS KMS does not delete any rotated key material until you delete the CMK.

Customer Managed CMKs with custom key stores or imported key material are not eligible for automatic key rotation, and you must manually rotate these keys yourself.

### View, Track and Audit Keys

With AWS Owned CMKs, you do not have the capability to view or track keys, or audit their use.

With Customer Managed CMKs, you can use AWS CloudTrail and Amazon CloudWatch logs to track and audit the requests that QLDB sends to AWS KMS on your behalf. For example, when AWS KMS automatically rotates the key material, it writes a CMK Rotation event to CloudWatch Events, and a RotateKey event to your AWS CloudTrail log. You can use these records to verify that the CMK was rotated.

With Customer Managed CMKs, you are responsible for defining and maintaining the key policy, IAM policy and grants to control access to the key. You also have the ability to enable or disable the key, create key tags and aliases, and schedule the key for deletion. None of these are possible with AWS Owned CMKs.

### Cost

You are not charged a monthly fee or a usage fee for AWS Owned CMKs, and they don't count against the AWS KMS quotas for your account, unlike with Customer Managed CMKs.

## QLDB and Customer Managed CMKs

Now that we have a better understanding of AWS KMS and the different key types, let's look at running a simple demo application built using the Serverless Framework. This application exposes two APIs via AWS API Gateway to create or get a record, which invoke separate AWS Lambda functions that integrate with Amazon QLDB, with the ledger data protected by a customer managed CMK with AWS generated key material.

The full source code can be found in the [QLDB KMS Demo](https://github.com/AWS-South-Wales-User-Group/qldb-kms-demo) repository, but we will walk through the main sections.

In the `Resources` section of the `serverless.yml` file, we need to define the KMS Key, an alias for the key, and a new ledger that uses the key, as well as the Lambda functions that integrate with the ledger.

### Define KMS Key

We start off by defining the KMS key:

```yaml
resources:
  Resources:
    QLDBKMSKey:
      Type: AWS::KMS::Key
      Properties: 
        Description: 'QLDB Ledger CMK'
        Enabled: true
        EnableKeyRotation: true
        KeySpec: 'SYMMETRIC_DEFAULT'
        PendingWindowInDays: 7
        KeyPolicy:
          ...
```

The `AWS::KMS::Key` resource specifies a new customer managed CMK in AWS KMS. `EnableKeyRotation` is set to true to support automatic key rotation. The default value of `SYMMETRIC_DEFAULT` for `KeySpec` creates a CMK with a 256-bit symmetric key for encryption and decryption. The `PendingWindowInDays` property specifies the number of days in the waiting period before AWS KMS deletes a CMK that has been removed from a CloudFormation stack. The next step is to define the key policy for this resource.

### Define KMS Key Policy

The key policy is the primary way to control access to a CMK. Every CMK must have *exactly* one key policy. The statements in the key policy document determine who has permission to use the CMK and how they can use it.

```yaml
resources:
  Resources:
    QLDBKMSKey:
      Type: AWS::KMS::Key
      Properties: 
        ...
        KeyPolicy:
          Version: '2012-10-17'
          Id: qldb-cmk-${self:provider.stage}
          Statement:
          - Sid: 'Allow administration of the key'
            Effect: Allow
            Principal:
              AWS: "arn:aws:iam::${aws:accountId}:root"
            Action:
              - kms:*
            Resource: "*"
          - Sid: 'Allow access to principals authorized to use Amazon QLDB'
            Effect: Allow
            Principal:
              AWS: '*'
            Action:
              - kms:DescribeKey
              - kms:CreateGrant
            Resource: '*'
            Condition:
              StringEquals:
                'kms:ViaService': 'qldb.eu-west-2.amazonaws.com'
                'kms:CallerAccount': '${aws:accountId}'
          - Sid: 'Allow access to create and get record roles'
            Effect: Allow
            Principal:
              AWS: 
                - "arn:aws:iam::${aws:accountId}:role/qldb-kms-create-record-role"
                - "arn:aws:iam::${aws:accountId}:role/qldb-kms-get-record-role"
            Action:
              - kms:Decrypt
            Resource: "*"
```

The first statement in the key policy gives the AWS account (root user) that owns the CMK full access to the CMK, and enables IAM policies to control access to the CMK. The rationale is you cannot delete an AWS account's root user, which prevents the possibility of the CMK becoming unmanageable. By default, the root user does not have access to the CMK. AWS KMS performs safety checks when a key policy is created. If no statement was provided, you would get an error message similar to below:

```ascii
The new key policy will not allow you to update the key policy in the future. 
(Service: Kms, Status Code: 400, Request ID: 28e0aca7-46c1-4457-9bdb-c9a216e52d59, 
Extended Request ID: null)
```

If you don't want to provide the root user access, you can provide a separate user or typically a role to carry out administration of the key, using a statement similar to below:

```json
{
    "Sid": "Allow access for Key Administrators",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::{AccountID}:user/MattLewis"
    },
    "Action": [
        "kms:Create*",
        "kms:Describe*",
        "kms:Enable*",
        "kms:List*",
        "kms:Put*",
        "kms:Update*",
        "kms:Revoke*",
        "kms:Disable*",
        "kms:Get*",
        "kms:Delete*",
        "kms:TagResource",
        "kms:UntagResource",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
    ],
    "Resource": "*"
}
```

Note that this grants permissions to administer the key, but not carry out cryptographic operations (Decrypt / Encrypt etc).

The second statement in the key policy provides access to QLDB. The `Resource` element is set to "&#42;", which means "this CMK". The `Principal` element is also set to "&#42;" which gives every identity in every AWS account permission to use the CMK. However, a condition is used to limit the key policy. The `kms:ViaService` condition key limits use the CMK to requests from the QLDB service, with the `kms:CallerAccount` condition key further restricting access to the specific AWS account.

Each role that then accesses QLDB also needs permission to use the key. There are two options:

* Provide permission for the roles in the key policy
* Provide permission for the roles in their own policy

For the purposes of this demo, I have chosen the first approach, and the roles used by the create and get record AWS Lambda functions are given `kms:Decrypt` permission to the CMK.

### Define KMS Alias

The next resource to be created is the KMS Alias:

```yaml
resources:
  Resources:

    QLDBKMSAlias:
      Type: AWS::KMS::Alias
      Properties:
        AliasName: 'alias/qldb-kms-${self:provider.stage}'
        TargetKeyId: !GetAtt QLDBKMSKey.Arn
```

An alias is a friendly name for a CMK. It lets you refer to the CMK using this name, as opposed to the generated key ID, such as `1234abcd-12ab-34cd-56ef-1234567890ab`. The advantage of using an alias is that your code becomes easier to write and maintain. For example, to manually rotate a CMK you would create a new CMK, and then associate the alias with the different CMK. You would not need to change any code.

### Define QLDB Ledger

Finally, we create the new QLDB ledger referencing the alias of the CMK to use for encrypting data at rest in the ledger:

```yaml
resources:
  Resources:

    QLDBKMSLedger:
      Type: AWS::QLDB::Ledger
      Properties:
        Name: qldb-kms-ledger-${self:provider.stage}
        DeletionProtection: false
        KmsKey: !Ref QLDBKMSAlias
        PermissionsMode: STANDARD
        Tags:
          - 
            Key: name
            Value: qldb-kms-ledger
```

### Define AWS Lambda Role Permissions

The demo follows AWS best practice by implementing least privilege, and uses the `serverless-iam-roles-per-function` plugin to automatically create a separate role for each Lambda function.

In the `serverless.yml` file there are 2 Lambda functions defined:

* One to create a new record
* One to retrieve an existing record

The policy statement for the Lambda function that retrieves an existing record is shown below:

```yaml
  iamRoleStatements:
    - Effect: Allow
      Action: 
        - qldb:PartiQLSelect
      Resource: 
        - arn:aws:qldb:#{AWS::Region}:#{AWS::AccountId}:ledger/qldb-kms-ledger-${self:provider.stage}/*
    - Effect: Allow
      Action: 
        - qldb:SendCommand
      Resource: arn:aws:qldb:#{AWS::Region}:#{AWS::AccountId}:ledger/qldb-kms-ledger-${self:provider.stage}
```

There are 2 individual actions that are allowed:

1. The `qldb:SendCommand` action to allow issuing a PartiQL command against the specific ledger
2. The `qldb:PartiQLSelect` action to support executing a `SELECT` statement against the ledger

## QLDB and Data Keys

When you specify a customer managed CMK to protect your ledger, QLDB creates a unique data key. AWS KMS only supports directly encrypting data up to 4 KB (4096 bytes) in size. This also requires remote network calls to transfer the data. Although suitable for small amounts of arbitrary data, this won't work for encrypting large amounts of data. Instead, AWS KMS uses a method called envelope encryption.

In this approach, AWS KMS generates a data key returning both a plaintext copy and one encrypted using the customer master key. This also means that the CMK never leave AWS KMS.

![generate data key](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xndqkheiqnelsq0zuzca.png)

This data key is used by QLDB to encrypt and decrypt data in the ledger. The data keys themselves are not retained or managed by AWS KMS. Instead, the encrypted data key can be decrypted by AWS KMS using your CMK.

A subset of the event record is shown below:

```json
{
    "eventVersion": "1.08",
    "userIdentity": {
        "type": "AWSService",
        "invokedBy": "qldb.amazonaws.com"
    },
    "eventTime": "2021-08-19T22:50:40Z",
    "eventSource": "kms.amazonaws.com",
    "eventName": "GenerateDataKey",
    "awsRegion": "eu-west-2",
    "sourceIPAddress": "qldb.amazonaws.com",
    "userAgent": "qldb.amazonaws.com",
    "requestParameters": {
        "encryptionContext": {
            "key-hierarchy-node-id": "1nmeXO2avkRK43C6qhhkWw",
            "key-hierarchy-node-version": "1"
        },
        "keyId": "arn:aws:kms:eu-west-2:082136225280:key/{keyID}",
        "numberOfBytes": 32
    },
    ...
    "eventCategory": "Management"
}
```

The user is the QLDB service, with the parameters including the ARN of the CMK and the encryption context.

The actual implementation in QLDB was hinted at in a [tweet by Marc Bowes](https://twitter.com/marcbowes/status/1418640809155461121) which states:

> To solve this problem, we create a hierarchy of data keys. The structure is actually pretty complicated, and designing it
> took some care. What it allows us to do is decrypt-encrypt the *keys* and not *the data* ... this design also improves
> several security considerations!

This makes the implementation similar to DynamoDB as shown below:

![DDB generate data key](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ybuc24m9qsfx9070rxgg.png)

In this case, there is a hierarchy of data keys. DynamoDB generates data encryption keys that are encrypted with a data key generated by KMS, which is itself encrypted by the customer master key held in KMS. This means that the data encryption keys are used to protect fewer items limiting the blast radius, and that they can be regularly rotated by the service.

Using a hierarchy of keys makes complete sense for QLDB, as it prevents having an entire ledger protected by a single encryption key. This prevents changing a key resulting in having to encrypt - decrypt huge amounts of data. Indeed, AWS claim they can perform a full key change in under 30 minutes regardless of the amount of data, which is impressive. Key changes themselves in QLDB are asynchronous, with the ledger fully accessible without any performance impact while the key change is being processed. 

## Running the Demo

### Install and Deploy the Demo

The source code can be downloaded, dependencies installed, and the CloudFormation stack deployed using the following:

```shell
git clone https://github.com/AWS-South-Wales-User-Group/qldb-kms-demo
cd qldb-kms-demo
npm ci
sls deploy --stage {stage-name}
```

### Create Table and Index

When you deploy the stack, the QLDB ledger will be created, but you still need to go in and create a table and index. This can be done directly in the console or via the QLDB shell with the following commands:

```shell
CREATE TABLE Test;
CREATE INDEX ON Test (id);
```

### Configure Artillery Scripts

The source code repository includes a load testing tool called [Artillery](https://artillery.io/) that will be used for populating data. When you deploy the stack, you should see the POST and GET endpoints in a format such as below:

```ascii
POST - https://v0gbo1fgz3.execute-api.eu-west-2.amazonaws.com/poc/record
GET - https://v0gbo1fgz3.execute-api.eu-west-2.amazonaws.com/poc/record/{id}
```

Take a note of these values and update the target values in the `create-record.yml` and `get-record.yml` files in the `scripts` directory.

```ascii
config:
    target: "https://v0gbo1fgz3.execute-api.eu-west-2.amazonaws.com/poc"
    phases:
      - duration: 300
        arrivalRate: 5
    processor: "./createRecord.js"
```

The script above will create 5 virtual users every second for 300 seconds.

Now we are ready to test a number of scenarios

### Scenario 1: Create and Query Data

In the root directory create 1500 new records in 5 minutes by running the following command:

```shell
artillery run scripts/create-record.yml
```

Next, we show that we can retrieve these records by running the following command:

```shell
artillery run scripts/get-record.yml
```

This should successfully retrieve all 1500 records in around 2.5 minutes.

### Scenario 2: Rotate Key

This scenario tests what happens when we rotate the key associated with a QLDB ledger.

When we created the ledger, we specified an alias. This means we can simply update the alias to point at a different CMK. To do this, we need to know the key ID of the new CMK. Deploying the stack created a number of keys that we can use for testing.

The first step is to find the key ID of the rotate key. This value was output in the stack, and can be found in the Outputs section of the CloudFormation stack in the AWS console, or by querying CloudFormation using the AWS CLI as below:

```shell
aws cloudformation describe-stacks --region eu-west-2 --stack-name qldb-kms-{stage-name}
```

Next, run the artillery command to retrieve all 1500 records. This should take around 2.5 minutes. When this has started, run the following command to update the alias:

```shell
aws kms update-alias \
    --alias-name alias/qldb-kms-{stage-name} \
    --target-key-id {key-id} \
    --region eu-west-2
```

You should notice that all records were successfully retrieved, proving that the ledger remains fully accessible without any performance impact while the key change is being processed.

Instead of updating an alias, you can also directly update the KMS key used by the ledger. To do this, lookup the key id of the delete key created by the stack, kick off the artillery script to retrieve all records, and then run the following command:

```shell
aws qldb update-ledger \
   --name qldb-kms-ledger-poc \
   --region eu-west-2 \
   --kms-key 48ad14da-5bf9-497a-a4d9-d379bfb0d41e
```

You will see the output of the command being similar to below:

```json
{
    "Name": "qldb-kms-ledger-poc",
    "Arn": "arn:aws:qldb:eu-west-2:{account}:ledger/qldb-kms-ledger-poc",
    "State": "ACTIVE",
    "CreationDateTime": "2021-08-30T16:48:55.600000+01:00",
    "DeletionProtection": false,
    "EncryptionDescription": {
        "KmsKeyArn": "arn:aws:kms:eu-west-2:{account}:key/48ad14da-5bf9-497a-a4d9-d379bfb0d41e",
        "EncryptionStatus": "UPDATING"
    }
}
```

In this case, the `EncryptionStatus` is clearly shown as `UPDATING`, which is carried out asynchronously, without impacting on the performance or availability.

### Scenario 3: Disable (and re-enable) Key

The next scenario involves disabling the KMS key being used, and seeing the impact this has. Again, kick off the artillery script to retrieve all records, and then run the following command:

```shell
aws kms disable-key \
  --region eu-west-2 \
  --key-id 48ad14da-5bf9-497a-a4d9-d379bfb0d41e
```

The records continue to be successfully retrieved for a short period of time, before they begin to fail. An error message is returned that "Amazon QLDB does not have grant access on the AWS KMS customer managed key of the ledger. Restore the grant on the key for the ledger." You can query the status of the ledger using the following command:

```shell
aws qldb describe-ledger \
  --region eu-west-2 \
  --name qldb-kms-ledger-{stage-name}
```

This returns a response similar to the one below:

```json
{
    "Name": "qldb-kms-ledger-poc",
    "Arn": "arn:aws:qldb:eu-west-2:{account}:ledger/qldb-kms-ledger-poc",
    "State": "ACTIVE",
    "CreationDateTime": "2021-08-30T16:48:55.600000+01:00",
    "PermissionsMode": "STANDARD",
    "DeletionProtection": false,
    "EncryptionDescription": {
        "KmsKeyArn": "arn:aws:kms:eu-west-2:{account}:key/48ad14da-5bf9-497a-a4d9-d379bfb0d41e",
        "EncryptionStatus": "KMS_KEY_INACCESSIBLE",
        "InaccessibleKmsKeyDateTime": "2021-08-30T20:01:17.521000+01:00"
    }
}
```

The ledger now has an encryption status of `KMS_KEY_INACCESSIBLE`, meaning it is impaired and won't accept any read or write requests. For example, you won't even be able to connect to the ledger using the QLDB shell utility.

To revert this, you can simply re-enable the key using the following command:

```shell
aws kms enable-key \
  --region eu-west-2 \
  --key-id 48ad14da-5bf9-497a-a4d9-d379bfb0d41e
```

This will take a number of minutes to resolve, but at some point, the encryption status will revert back to `ENABLED` and the ledger is accessible.

### Scenario 4: Invalid Permissions

The final scenario involves using a key for which the roles used by the AWS Lambda functions do not have permission either in their own IAM policy or the key policy.

Update the ledger to use the key id associated with the no access key id in the Outputs of the CloudFormation stack. Next, run a `cURL` command to the HTTP GET endpoint to retrieve the record with the id of 1:

```script
curl 'https://aq1o120co1.execute-api.eu-west-2.amazonaws.com/poc/record/1'
```

This will return the following error message:

```json
{
  "status":500,
  "title":"AccessDeniedException",
  "detail":"The user does not have permissions to use the customer managed KMS key of the ledger (KMS Request ID: 61d6b556-4c6b-4f13-b4c4-fae49a2f11c3) : The ciphertext refers to a customer master key that does not exist, does not exist in this region, or you are not allowed to access."
}
```
### Tidy Up Resources

All the resources generated can be safely removed by running the following command:

```shell
sls remove --stage {stage-name}
```

## Conclusion

For many organisations, the use of customer managed CMK to protect data is essential. Support for these types of KMS keys is a big improvement in QLDB, allowing you to view, audit and track usage of these keys, and giving you complete control over their policies, including revoking all access.

To find out more, read the [QLDB Guide](https://qldbguide.com/), follow the curated list of articles, tools and resources at [Awesome QLDB](https://github.com/mlewis7127/awesome-qldb) or try it out our online demo to see QLDB in action at [QLDB Demo](https://demo.qldbguide.com/)