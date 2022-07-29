When QLDB was first launched, it provided a set of actions for interacting with the control plane API to manage ledgers [(see here)](https://docs.aws.amazon.com/qldb/latest/developerguide/API_Operations.html), but only a single action for interacting with a ledger over the data plane API. This meant any user or role required the `qldb:sendCommand` permission for issuing a `PartiQL` command against a ledger. With this IAM permission, you were able to execute all `PartiQL` commands from simple lookups, to mutating current state with updates and deletes, and querying all revision history.

The latest release from the Amazon QLDB team provides support for fine-grained IAM permissions when interacting with a ledger, which helps enforce least-privilege. This blog post will show you how to get started, using the `QLDB Shell`.

All code and setup instructions can be found in the [QLDB access control demo repo](https://github.com/AWS-South-Wales-User-Group/qldb-access-control-demo)

## Pre-Requisites

In order to run the demo, the following is required:

* The AWS Commmand Line Interface `AWS CLI` is installed. For more details see [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

* The `jq` library is installed. For more details see [here](https://stedolan.github.io/jq/download/)

* An AWS Profile is configured with a user with administrative permissions for initial setup.

* QLDB Shell is installed - For more details see [here](https://docs.aws.amazon.com/qldb/latest/developerguide/data-shell.html)


The current QLDB shell is written in Python, but there is also a branch available written in Rust that has additional features. A huge thank you to [Mark Bowes](https://twitter.com/marcbowes) and [Ian Davies](https://twitter.com/this_alpian) who rapidly turned around my feature request for multi-line support and added a tonne of new functionality. AWS provide prebuilt binaries for Linux, Windows and macOS. On `macOS` the shell is integrated with the `aws/tap` Homebrew tap:

```shell
xcode-select install # required to use Homebrew
brew tap aws/tap # Add AWS as a Homebrew tap
brew install qldbshell
qldb --ledger <your-ledger>
```

## Setup

To setup the demo, clone the github repository and navigate to the shell-demo folder.

```shell
git clone https://github.com/AWS-South-Wales-User-Group/qldb-access-control-demo.git
cd qldb-access-control-demo/shell-demo
```

Follow the instructions to edit the `qldb-access-control.yaml` CloudFormation template with your user, and create a new stack running the following command:

```shell
aws cloudformation deploy --template-file ./qldb-access-control.yaml --stack-name qldb-access-control --capabilities CAPABILITY_NAMED_IAM
```

This creates a new QLDB ledger with the name `qldb-access-control` using the new `STANDARD` permissions mode. The snippet that does this is shown below:

```yaml
QLDBAccessControl:
  Type: "AWS::QLDB::Ledger"
  Properties:
    DeletionProtection: false
    Name: "qldb-access-control"
    PermissionsMode: "STANDARD"
    Tags:
      - Key: "name"
        Value: "qldb-access-control"
```

Before this release, the only permissions mode supported was `ALLOW_ALL`, which allowed any user with this permission to execute any PartiQL command. This is now marked as legacy and should not be used. Deletion protection is disabled to allow simpler clean up at the end.

## Role Permissions

As well as creating a QLDB Ledger with the name `qldb-access-control` the cloudformation template sets up the following roles with associated permissions:

![QLDB IAM Roles](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/996rrlv4n4ih4ll33akf.png)

Each role has its own policy document setting out the permissions allowed. In order to execute any PartiQL command, permission must be given to the `sendCommand` API action for the ledger resource. Explicit permission to PartiQL commands can then be given, taking into account that requests to run all PartiQL commands are denied unless explicitly allowed here. An example of a policy document is shown below:

![PartiQL Statement](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/94yzmzcxnppfdvhdhjwi.jpg)

## Assuming a role

A number of helper scripts are provided to help assume the various roles:

```shell
source setupSuperUser.sh
source setupAdmin.sh
source setupAudit.sh
source setupReadOnly.sh
```

When running one of these scripts, it prints out details of the current user with the following command, which can also be run separately.

```shell
# print out the current identity
aws sts get-caller-identity
```

Finally, in order to assume another role, you will need to unset the current assumed role. This is because none of the roles have permission to perform the `sts:AssumeRole` command. You can unset the current role using the following command:

```shell
source unset.sh
```

## Testing Permissions

The demo gives a set of tasks with accompanying PartiQL statements to create tables, indexes, insert and update data, and query the revision history using various roles. Note how if the correct permission is not explicitly assigned to the role, then the command will fail with an error message like the following:

```shell
"Message":"Access denied. Unable to run the statement due to insufficient permissions or an improper variable reference"
```

## Creating a policy for a specific table

Permissions can applied at a table level as well as a ledger level. The `table-demo` folder in the repository shows an example of how this can be applied automatically using a custom resource.

This folder uses the Serverless Framework to create a custom resource and a new role with a policy attached to it that allows read access to a `Keeper` table.

The original cloudformation stack in the `shell-demo` folder outputs the value of the new QLDB ledger name it creates through the `Outputs` section of the template as shown below:

```yaml
Outputs:
  qldbAccessControlLedger:
    Value:
      Ref: QLDBAccessControl
    Export:
      Name: qldb-access-control-demo
```

This value can then be referenced in the `serverless.yml` file using the `Fn::ImportValue` intrinsic function as follows:

```yaml
!ImportValue qldb-access-control-demo
```

The custom resource lambda function is responsible for creating a `Keeper` table and a `Vehicle` table. When a table is created, the unique ID for the table is returned. This value is retrieved, and stored as a name/value pair. This gets returned in the optional data section as shown below:

```javascript
const keeperResult = await createTable(txn, keeperTable);
const keeperIdArray = keeperResult.getResultList();
keeperId = keeperIdArray[0].get('tableId').stringValue();

const responseData = { requestType: event.RequestType, 
                       'keeperId': keeperId  };

await response.send(event, context, response.SUCCESS, responseData);
```

Finally, this value can be referenced using the `Fn::GetAtt` instrinic function, and the full resource name created using the `Fn::Join` instrinic function as follows:

```yaml
- Effect: Allow
    Action:
    - 'qldb:PartiQLSelect'
    Resource:
    - !Join
        - ''
        - - 'arn:aws:qldb:#{AWS::Region}:#{AWS::AccountId}:ledger/'
        - !ImportValue qldb-access-control-demo
        - '/table/#{qldbTable.keeperId}'
```

Now when the new role is assumed, data can be queried successfully from the `Keeper` table but not the `Vehicle` table.

## Conclusion

This blog post and associated code repository shows how you can take advantage of the new fine grained permissions now available with QLDB. This is a great addition, that enables the principle of least-privilege to be easily assigned to all resources in a QLDB ledger.

To find out more, read the [QLDB Guide](https://qldbguide.com/), follow the curated list of articles, tools and resources at [Awesome QLDB](https://github.com/mlewis7127/awesome-qldb) or try it out our online demo to see QLDB in action at [QLDB Demo](https://demo.qldbguide.com/)