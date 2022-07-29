In previous posts I described how to:

* [Stream data from QLDB to DynamoDB](https://dev.to/aws-heroes/real-time-streaming-for-amazon-qldb-3c3c) to support single-digit latency and infinitely scalable key-value enquiries, and
* [Stream data from QLDB to ElasticSearch](https://dev.to/aws-heroes/streaming-data-from-amazon-qldb-to-elasticsearch-78c) to support rich text search and downstream analytics.

This was all created in a source code repository that anyone could try out for themselves. Recently, [Sheen Brisals](https://twitter.com/sheenbrisals) wrote a great post on ["Why Serverless Teams should embrace Continuous Refactoring"](https://betterprogramming.pub/why-serverless-teams-should-embrace-continuous-refactoring-217d4e67db5b).

Given that, I thought I would go back and update the repository, in line with new features and changes over the past 12 months:

* Updating the QLDB permissions mode to `STANDARD`
* Implementing fine grained access control for all functions accessing QLDB
* Updating to the latest version of Nodejs
* Switching from `webpack` to `esbuild` for tree shaking
* Migrating from ElasticSearch to OpenSearch
* Configuring Amazon Cognito authentication for OpenSearch Dashboards
* Implementing custom metrics

This blog post focuses on the last three updates.

## Amazon OpenSearch Service

AWS [announced](https://aws.amazon.com/blogs/aws/amazon-elasticsearch-service-is-now-amazon-opensearch-service-and-supports-opensearch-10/) on 8th Sept 2021 that they had renamed Amazon ElasticSearch Service to Amazon OpenSearch Service. This is an Apache 2.0-licensed fork of ElasticSearch that is community-driven and open source.

In the previous deployment, ElasticSearch was configured to run within your VPC. This is still the recommended approach from a security standpoint. However, to make things simpler for people to get up and running, I wanted to deploy OpenSearch with a public endpoint instead. In addition, I wanted to protect access to OpenSearch Dashboards using Amazon Cognito.

The full source code can be found at [QLDB Simple Demo](https://github.com/AWS-South-Wales-User-Group/qldb-simple-demo), but lets walk through the main steps involved.

## Setting up Amazon Cognito

The first thing to setup in our `serverless.yml` file is the Cognito User Pool. This provides the user directory to control access to OpenSearch Dashboards. The setup below asks for name and email attributes at sign up, as well as a username and password. The email will be verified by entering a code that is sent to the specified email address.

```yaml
OSUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
    UsernameConfiguration:
        CaseSensitive: false
    AutoVerifiedAttributes:
        - email
    UserPoolName: !Sub qldb-demo-user-pool
    Schema:
        - Name: email
        AttributeDataType: String
        Mutable: false
        Required: true
        - Name: name
        AttributeDataType: String
        Mutable: true
        Required: true
```

The next step is the UserPoolDomain. This provides a domain name to be used as part of the hosted UI.

```yaml
OSUserPoolDomain: 
    Type: AWS::Cognito::UserPoolDomain 
    Properties:
    UserPoolId: !Ref OSUserPool 
    Domain: "qldb-opensearch-demo"
```

After this, we define the Cognito Identity Pool. We use this to provide a way to grant temporary AWS credentials for users. This is necessary, as to support a public endpoint on the OpenSearch domain, we need to enable fine-grained access control or apply a restrictive access policy. We don't configure the CognitoIdentityProviders, as these will be created for us by the OpenSearch service.

```yaml
OSIdentityPool:
    Type: AWS::Cognito::IdentityPool
    Properties:
    IdentityPoolName: qldb-demo-identity-pool
    AllowUnauthenticatedIdentities: true
```

Next we create two roles, one for an authenticated identity, and one for an unauthenticated identity. The one for the authenticated identity is shown below:

```yaml
CognitoAuthorizedRole:
    Type: "AWS::IAM::Role"
    Properties:
    AssumeRolePolicyDocument: 
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
            Principal: 
            Federated: "cognito-identity.amazonaws.com"
            Action: 
            - "sts:AssumeRoleWithWebIdentity"
            Condition:
            StringEquals: 
                "cognito-identity.amazonaws.com:aud": !Ref OSIdentityPool
            ForAnyValue:StringLike:
                "cognito-identity.amazonaws.com:amr": authenticated
```

The above is a trust policy for an authenticated role. It allows a federated user from `cognito-identity.amazonaws.com` (the issuer of the OpenID Connect token) to assume this role. It also places a condition, that restricts the `aud` of the token (the client ID of the relying party) to be the Cognito Identity Pool, and the `amr` of the token contains the value `authenticated`. When Amazon Cognito creates a token, it will set the `amr` of the token to be either `unauthenticated` or `authenticated`. There is no policy attached to this identity, as we are going to control access to OpenSearch through a resource policy attached to the OpenSearch domain.

After defining the two roles with the associated policies, we map them to the Identity Pool using an IdentityPoolRoleAttachment

```yaml
IdentityPoolRoleMapping:
    Type: "AWS::Cognito::IdentityPoolRoleAttachment"
    Properties:
    IdentityPoolId: !Ref OSIdentityPool
    Roles:
        authenticated: !GetAtt CognitoAuthorizedRole.Arn
        unauthenticated: !GetAtt CognitoUnAuthorizedRole.Arn
```

Then its time to define a role that the OpenSearch service can assume, that includes permissions to configure the Cognito user and identity pools and use them for authentication. This can be done using the `AmazonOpenSearchServiceCognitoAccess` AWS-managed policy as shown below:

```yaml
OSCognitoRole:
    Type: 'AWS::IAM::Role'
    Properties:
    RoleName: 'CognitoAccessForAmazonOpenSearch'
    AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
            Principal:
            Service:
                - es.amazonaws.com
            Action:
            - 'sts:AssumeRole'
    Path: "/"
    ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonOpenSearchServiceCognitoAccess
```

Finally, its time to create the OpenSearch domain, with the `CognitoOptions` referencing the role defined above, as well as the Cognito user and identity pool.

```yaml
OpenSearch:
    Type: AWS::OpenSearchService::Domain
    Properties:
    DomainName: "#{AWS::StackName}"
    EngineVersion: 'OpenSearch_1.0'
    ...
    CognitoOptions:
        Enabled: true
        IdentityPoolId: !Ref OSIdentityPool
        RoleArn: !GetAtt OSCognitoRole.Arn
        UserPoolId: !Ref OSUserPool
    ...
    AccessPolicies:
        Version: "2012-10-17"
        Statement:
        - Effect: Allow
            Principal:
            AWS: 
                - !GetAtt CognitoAuthorizedRole.Arn
            Action: 
            - es:ESHttpGet
            - es:ESHttpPost
            Resource: arn:aws:es:${self:provider.region}:#{AWS::AccountId}:domain/#{AWS::StackName}/*
        - Effect: Allow
            Principal:
                AWS: "*"
            Action: es:ESHttp*
            Resource: arn:aws:es:${self:provider.region}:#{AWS::AccountId}:domain/#{AWS::StackName}/*
            Condition:
                StringLike:
                    "aws:PrincipalArn": arn:aws:iam::#{AWS::AccountId}:role/qldb-streams-es-lambda-role-${self:provider.stage}
```

All access to the domain is controlled through the resource policy that is applied. The first statement allows the authenticated Cognito user to access the domain. The second statement allows access to the domain from the execution role attached to the AWS Lambda function. You might notice that this is defined in a different way. A circular dependency exists, as the Lambda function needs the OpenSearch domain endpoint which is set up as an environment variable. Using a condition and the `aws:PrincipalArn` key allows you to compare the ARN of the principal that made the request with the ARN specified in the policy at runtime, rather than at deployment time which otherwise failed. 

## Accessing OpenSearch Dashboard

Now the stack is deployed, we can access the OpenSearch Dashboard. The easiest place to start is by launching the Hosted UI. You can find the link in the Cognito User Pool under App Client Settings:

![Cognito Launch HostedUI](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n5ojzhsvonhh4ovq4yp1.png)

This will allow you to sign up and verify your new account with a code sent to your email address. Once verified and signed in, you can click the heading to visualise and analyse your data.

![OpenSearch Dashboards](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xu1gxownymzf0e20n7zj.png)

From here, click the button to add your data:

![OpenSearch-Dashboards-AddData](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qovhkdf8b6kf29xy3x80.png)

Next, create an index pattern. If you are using the sample applications, then the index pattern is simply `licence`:

![Define-Index-Pattern](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/319f8hdke2bj2m5gsso0.png)

Now, you can go into `Dev Tools` and run queries, use metrics aggregation and combine filter and query contexts amongst other features. You can also create visualisations from the raw data in OpenSearch.

## Custom CloudWatch Metrics

In a previous blog post on [tips to prevent a serverless wreck](https://dev.to/aws-heroes/tips-to-prevent-a-serverless-wreck-15af), I strongly advocated the use of metrics to monitor an application. The CloudWatch Embedded Metric Format allows you to create custom metrics, that can be used for visualisations and alarming for real-time incident detection.

In this sample application, an AWS Lambda function is used to consume an aggregated set of records from a Kinesis Data Stream, and write any new records (inserts, updates or deletes) to OpenSearch. Each request to OpenSearch hits a REST API endpoint, and returns an HTTP Status Code. We can use the returned HTTP Status Code as a custom metric as follows:

```javascript
    const { createMetricsLogger, Unit } = require('aws-embedded-metrics');

    ...
    const metrics = createMetricsLogger();
    metrics.setNamespace('OpenSearch Status Codes');
    metrics.putDimensions({ StatusCode: `HTTP_${statusCode}` });
    metrics.putMetric('ProcessedRecords', 1, Unit.Count);
    await metrics.flush();
```

This code sets the namespace to be 'OpenSearch Status Codes'. This is the name that will appear in custom namespaces in CloudWatch metrics. We use the returned status code as the dimension. A dimension is a name/value pair that is part of the identity of a metric. Each time we process a record, we use a count of 1 as the unit.

This results in a log entry in CloudWatch that looks similar to below:

```json
{
    "LogGroup": "qldb-streams-es-dev",
    "ServiceName": "qldb-streams-es-dev",
    "ServiceType": "AWS::Lambda::Function",
    "StatusCode": "HTTP_200",
    "executionEnvironment": "AWS_Lambda_nodejs14.x",
    "memorySize": "512",
    "functionVersion": "$LATEST",
    "logStreamId": "2021/12/11/[$LATEST]6924daa324e8482099ebcad5c1168c9a",
    "_aws": {
        "Timestamp": 1639244821514,
        "CloudWatchMetrics": [
            {
                "Dimensions": [
                    [
                        "LogGroup",
                        "ServiceName",
                        "ServiceType",
                        "StatusCode"
                    ]
                ],
                "Metrics": [
                    {
                        "Name": "ProcessedRecords",
                        "Unit": "Count"
                    }
                ],
                "Namespace": "OpenSearch Status Codes"
            }
        ]
    },
    "ProcessedRecords": 1
}
```

When streaming records from Kinesis to OpenSearch, there were a handful of status codes commonly returned:

* HTTP 200 - a successful update to an existing document with an HTTP POST
* HTTP 201 - a successful insert of a new document, or completely overwriting an existing document with an HTTP PUT
* HTTP 409 - an error where the version of the document attempted to be inserted is less than or equal to the one that already exists. This can happen as each data record produced by a QLDB stream is delivered to Kinesis at least once, the same records can appear multiple times, and there are no ordering guarantees. The sample application implements external versioning to resolve this.

After streaming a number of records to OpenSearch, we can create a simple CloudWatch Dashboard from the custom metric that looks as follows:

![OpenSearch-Custom-Metrics](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wkbstxlq56ib5ryx1bdg.png)

## Conclusion

So there we have it. This post has shown how to quickly get up and running with OpenSearch, configuring Cognito to protect OpenSearch Dashboards, and set up a custom CloudWatch Metrics dashboard for monitoring the HTTP Status Codes that are returned. Please reach out if you have any feedback or suggestions.

To find out more, read the [QLDB Guide](https://qldbguide.com/), follow the curated list of articles, tools and resources at [Awesome QLDB](https://github.com/mlewis7127/awesome-qldb) or try it out our online demo to see QLDB in action at [QLDB Demo](https://demo.qldbguide.com/)