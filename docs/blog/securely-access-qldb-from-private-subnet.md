## Background

The serverless demo applications I built out previously use `AWS Lambda` running in service VPCs to interact with `QLDB`. Historically, there were significant cold start penalties when Lambda was configured to connect to your own VPC. This was a result of setting up a new Elastic Network Interface (ENI) and creating a cross-account attachment. However, this changed dramatically with a new release at the end of 2019 described in this [blog post](https://aws.amazon.com/blogs/compute/announcing-improved-vpc-networking-for-aws-lambda-functions/). As a result, it is increasingly common to see Lambda functions "running" in a customer VPC. This post shows how to run a Lambda function in a private subnet, and intefact with QLDB, with no traffic leaving the AWS network.

## QLDB Interface VPC Endpoint

All code for this demo can be found in this [GitHub repo](https://github.com/AWS-South-Wales-User-Group/qldb-vpc). The overall high level architecture is shown below:

![QLDB VPC Access](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l3hbvhgpghd8heaz4sg0.jpg)

API Gateway provides a REST API endpoint, and can control access to back end services. It uses lambda proxy integration to route requests to a Lambda function that is configured to execute within two private subnets. The Lambda function interacts with the ledger in QLDB through a VPC interface endpoint. This enables traffic to flow between QLDB and the VPC through this endpoint. In addition, the endpoint is configured to only allow traffic from the Lambda function to ensure least privilege is adopted.

Now let's see how this is setup using the `Serverless Framework`.

## Configuration

The first step is to configure a new VPC. The following configuration sets the `EnableDNSSupport` and `EnableDNSHostnames` to true. This ensures that instances launched in the VPC receive corresponding public DNS hostnames to their public IP addresses, and the Amazon Route 53 Resolver server can resolve Amazon-provided private DNS hostnames. 

This is necessary as we are using `PrivateLink` by configuring a VPC interface endpoint. This endpoint will be accessed using the private DNS name associated with the service.

```yaml
QLDBVPC:
    Type: AWS::EC2::VPC
    Properties:
    CidrBlock: "10.0.0.0/16"
    EnableDnsSupport: true
    EnableDnsHostnames: true
    Tags:
        - Key: Name
        Value: qldb-private
```

The next step is to configure private subnets in the VPC. A private subnet has no capability to send outbound traffic directly to the internet, meaning there is no route out to an Internet Gateway.

```yaml
QLDBSubnetA:
    DependsOn: QLDBVPC
    Type: AWS::EC2::Subnet
    Properties:
    VpcId:
        Ref: QLDBVPC
    AvailabilityZone: ${self:provider.region}a
    CidrBlock: "10.0.2.0/24"
    Tags:
        - Key: Name
        Value: qldb-private-a
```

A security group is then created. This is a virtual interface that controls inbound and outbound traffic. Note there are no outbound or inbound rules associated with this security group. However, by default, a security group includes an outbound rule that allows all outbound traffic.

```yaml
QLDBSecurityGroup:
    DependsOn: QLDBVPC
    Type: AWS::EC2::SecurityGroup
    Properties:
    GroupDescription: SecurityGroup for QLDB Private
    VpcId:
        Ref: QLDBVPC
    Tags:
        - Key: Name
        Value: qldb-private-sg
```

The lambda function configured to interact with QLDB is assigned to the security group in the last step, with the private subnets also specified.

```yaml
  createLicence:
    name: qldb-private-${self:provider.stage}
    ...
    vpc:
      securityGroupIds:
        - !GetAtt QLDBSecurityGroup.GroupId
      subnetIds:
        - Ref: QLDBSubnetA
        - Ref: QLDBSubnetB
```

Another security group is required to follow best practice, that will then be assigned to the VPC endpoint. This security group is set up to only allow traffic coming from the other security group assigned to the lambda function.

```yaml
VpcEndpointSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
    VpcId: 
        Ref: QLDBVPC
    GroupDescription: 'Security group for VPC Endpoint'
    Tags:
        - Key: Name
        Value: qldb-vpce-sg
    SecurityGroupIngress:
        - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        SourceSecurityGroupId: !GetAtt QLDBSecurityGroup.GroupId
```

Finally, the VPC interface endpoint is configured. It specifies that is used for the QLDB service by using the specific service name. A policy document is attached to the endpoint that controls access to QLDB. This policy allows access only to the role that is assumed by the lambda function, and only to make a `qldb:SendCommand` call to the specific ledger.

```yaml
QLDBEndpoint:
    Type: 'AWS::EC2::VPCEndpoint'
    Properties:
    PolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
            Principal:
            AWS:
                - arn:aws:iam::#{AWS::AccountId}:role/qldb-vpc-${self:provider.stage}-#{AWS::Region}-lambdaRole
            Action: 
            - 'qldb:SendCommand'
            Resource: arn:aws:qldb:#{AWS::Region}:#{AWS::AccountId}:ledger/qldb-private-${self:provider.stage}
    PrivateDnsEnabled: True
    SecurityGroupIds:
        - !GetAtt VpcEndpointSecurityGroup.GroupId
    ServiceName: !Sub 'com.amazonaws.${AWS::Region}.qldb.session'
    SubnetIds:
        - Ref: QLDBSubnetA
        - Ref: QLDBSubnetB
    VpcEndpointType: Interface
    VpcId: !Ref QLDBVPC
```

With this set up, you can now invoke the Lambda function via API Gateway, and all traffic from the private subnet to the QLDB and back to your VPC will remain on the AWS network.