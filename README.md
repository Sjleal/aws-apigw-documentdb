# Using AWS Cloudformation to automate the setup of API Gateway and DocuemntDB Resources
## Overview

The REST API in this project can perform operations of insert update, delete and read against a collection of an Amazon DocumentDB resource. This design includes the creation of resources in AWS, like an Amazon DocumentDB, AWS Lambda functions and an AWS API Gateway, all of this  will be deployed through a templete in AWS CloudFormation. 

To meet the goal of this project, it has been divided into 3 sections. The first one prepares the code, upload the functions to Amazon S3. The second is in charge of designing a template to create our stack in CloudFormation. Finally, the third section will be responsible for testing the REST API.

#### Initial setup
+ Step 1. **Preparing Lambda layer.**
+ Step 2. **Preparing Lambda functions.**
#### CloudFormation template 
+ Step 3. **Creating a VPC.**
+ Step 4. **Creating a DocumentDB cluster.**
+ Step 5. **Creating a Secret.**
+ Step 6. **Setting up IAM permissions.**
+ Step 7. **Creating the Lambda resources.**
+ Step 8. **Creating an API Gateway.**
+ Step 9. **Creating the stack with Cloudformation.**
#### Final steps
+ Step 10. **Setting up DocumentDB.**
+ Step 11. **Testing CRUD operations**

Most of the steps will be automated in an AWS CloudFormation template, which will be explained throughout this document. The goal is automate the process of deploying the resources used to host the solution.

<br>

## Architecture

The following image provides a view of the proposed architecture for deploying website resources.

![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/diagram/aws-apigw-documentdb-architecture.png)

#### Architect
  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/b1.svg) A CloudFormation template is designed to provision the resources for this solution

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/b2.svg) The stack is created in CloudFormation with the appropriate role, adhering to the least privilege principle

<br>

#### Client Application
  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g1.svg) Client makes the request to the REST API 

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g2.svg) API Gateway trigger the Authentication lambda function to allow or deny access to the CRUD function

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g3.svg) Once authenticated, API Gateway trigger the CRUD function, delivering the query parameters requiered

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g4.svg) Lamda gets the database connection stored in AWS Secrets    

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g5.svg) Lambda performs the requiered query against DocumentDB

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g6.svg) Lambda gets the query response from the DocumentdDB

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g7.svg) Lambda sent the query response to API Gateway

  ![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/g8.svg) API Gateway sent the response to the client

<br>

## Services

- [Amazon DocumentDB](https://docs.aws.amazon.com/documentdb/) is a fully managed, scalable, NoSQL database service, It’s designed to be compatible with MongoDB
- [Amazon VPC](https://docs.aws.amazon.com/vpc/) enables you to provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you've defined.
- [Amazon S3](https://docs.aws.amazon.com/s3/) is a cloud-based object storage service that helps you store, protect, and retrieve any amount of data.
- [AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/iam/) helps you securely manage access to your AWS resources by controlling who is authenticated and authorized to use them.
- [AWS Lambda)](https://docs.aws.amazon.com/lambda/) is a serverless compute service that runs your code in response to events and automatically manages the underlying compute resources.
- [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/) is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale.
- [Amazon CloudFront](https://docs.aws.amazon.com/cloudfront/) speeds up distribution of your web content by delivering it through a worldwide network of data centers, which lowers latency and improves performance.
- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/) helps you set up AWS resources, provision them quickly and consistently, and manage them throughout their lifecycle. You can use a template to describe your resources and their dependencies, and launch and configure them together as a stack, instead of managing resources individually. You can manage and provision stacks across multiple AWS accounts and AWS Regions.

<br>

## Step by Step

>**Initial setup:**<br>
In these initial steps, we will focus on preparing the source code for the lambda functions, and upload them to the s3 buckets.

**1. Preparing Lambda layer**

- Create or use a pre-built basic html website template
- Create a folder named _proyect_name_ and copy any HTML documents, images, CCS stylesheets, and JavaScript files
- Setup your authentication method on GitHub [Authentication to GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github#authenticating-with-the-command-line)
- Create a new repository on GitHub: [Creating a new repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository)
- Setup your remote repository: [Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
- Modify the necessary files to update website and Commit changes to the GitHub repository. 

**2. Preparing Lambda functions**

You can register a domain name on [Amazon Route 53](https://docs.aws.amazon.com/route53/) or [Namecheap](https://www.namecheap.com//) or [GoDaddy](https://www.godaddy.com/domains/) or [Hostinger](https://www.hostinger.com/) or any domains registrars of your choice.  

<br>

---

>**CloudFormation Template:**<br>
The resources will be created using AWS Cloudformation in order to maintain infrastructure integrity, reduce errors, and track changes over time. Taking advantage of Cloudformation's ability to automate resource deployment, a template will be designed to handle the creation and configuration of the resources involved in the solution.<br>
In the following steps, part of the code used in each section will be shown, the complete template is available in a public repository called [apigw-docuemntdb](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/dev/docdb.yaml).<br>
Some inputs will be requested at the stack creation and others will be captured during template execution. The YAML format was chosen for this template.


**3. Creating a VPC**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  MainVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      InstanceTenancy: default
      Tags:
        - Key: Name
          Value: !Ref punitag
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Ref punitag
  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref MainVPC
  MainSubnetPublic:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      CidrBlock: 10.0.10.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  MainSubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      CidrBlock: 10.0.100.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  SecondSubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1b
      CidrBlock: 10.0.200.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  RouteIGW:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  SubnetMainPubRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref MainSubnetPublic
      RouteTableId: !Ref PublicRouteTable
  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NATGatewayEIP.AllocationId
      SubnetId: !Ref MainSubnetPublic
      Tags:
        - Key: Name
          Value: !Ref punitag
  NATGatewayEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Ref punitag
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  RouteNATGateway:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NATGateway
  SubnetMainPrivRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref MainSubnetPrivate
      RouteTableId: !Ref PrivateRouteTable
  SubnetSecondPrivRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref SecondSubnetPrivate
      RouteTableId: !Ref PrivateRouteTable

~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**4. Creating a DocumentDB cluster**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  DocumentDBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: DocumentDB Security Group
      VpcId: !Ref MainVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 27017
          ToPort: 27017
          CidrIp: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Ref punitag
  DocumentDBSubnetGroup:
    Type: AWS::DocDB::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: DocumentDB Subnet Group
      SubnetIds:
        - !Ref MainSubnetPrivate
        - !Ref SecondSubnetPrivate
      Tags:
        - Key: Name
          Value: !Ref punitag
  DocumentDBCluster:
    Type: AWS::DocDB::DBCluster
    Properties:
      DBClusterIdentifier: docdbcv2024
      MasterUsername: !Ref DDBMasterUser
      MasterUserPassword: !Ref DDBMasterPassword
      Port: 27017
      StorageEncrypted: true
      EngineVersion: 5.0.0
      AvailabilityZones:
        - us-east-1a
        - us-east-1b
      VpcSecurityGroupIds:
        - !Ref DocumentDBSecurityGroup
      DBSubnetGroupName: !Ref DocumentDBSubnetGroup
      Tags:
        - Key: Name
          Value: !Ref punitag
  DocumentDBInstance1:
    Type: AWS::DocDB::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DocumentDBCluster
      DBInstanceClass: db.t3.medium
      Tags:
        - Key: Name
          Value: !Ref punitag
  DocumentDBInstance2:
    Type: AWS::DocDB::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DocumentDBCluster
      DBInstanceClass: db.t3.medium
      Tags:
        - Key: Name
          Value: !Ref punitag

~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**5. Creating a Secret**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  DocDBSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: !Sub ${punitag}-DocDBSecret
      Description: This secret has the credentials for the DocumentDB cluster
      SecretString: !Sub '{"username": "${DDBMasterUser}","password":
        "${DDBMasterPassword}"}, "ssl": true'
      Tags:
        - Key: Name
          Value: !Ref punitag
  SecretDocDBClusterAttachment:
    Type: AWS::SecretsManager::SecretTargetAttachment
    Properties:
      SecretId: !Ref DocDBSecret
      TargetId: !Ref DocumentDBCluster
      TargetType: AWS::DocDB::DBCluster
    DependsOn: DocumentDBCluster

~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**6. Setting up IAM permissions**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
        - arn:aws:iam::aws:policy/AWSLambdaExecute
      Policies:
        - PolicyName: DocumentDBSecret
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - secretsmanager:GetSecretValue
                Resource: !Ref DocDBSecret
      Tags:
        - Key: Name
          Value: !Ref punitag
      RoleName: !Sub ${punitag}-Role-Lambda

~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**7. Creating the Lambda resources**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  LambdaLayer:
    Type: AWS::Lambda::LayerVersion
    Properties:
      Content:
        S3Bucket: layers-aws-api-documentdb
        S3Key: lambda_layer.zip
      CompatibleRuntimes:
        - python3.11
      Description: Lambda layer for DocumentDB dependencies and PEM file
      LayerName: !Sub ${punitag}-LambdaLayer
      LicenseInfo: MIT
  LambdaAPPFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub ${punitag}-LambdaAPP
      Description: Lambda function to retrieve DocumentDB credentials
      Handler: app.lambda_handler
      Runtime: python3.11
      Layers:
        - !Ref LambdaLayer
      Code:
        S3Bucket: functions-aws-api-documentdb
        S3Key: app.zip
      Role: !GetAtt LambdaExecutionRole.Arn
      Environment:
        Variables:
          DB_SECRET_NAME: !Sub ${punitag}-DocDBSecret
      VpcConfig:
        SecurityGroupIds:
          - !Ref DocumentDBSecurityGroup
        SubnetIds:
          - !Ref MainSubnetPrivate
          - !Ref SecondSubnetPrivate
      Timeout: 30
      Tags:
        - Key: Name
          Value: !Ref punitag
  LambdaAuthFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub ${punitag}-LambdaAuth
      Description: Lambda function to handle authentication
      Handler: auth.lambda_handler
      Runtime: python3.11
      Code:
        S3Bucket: functions-aws-api-documentdb
        S3Key: auth.zip
      Role: !GetAtt LambdaExecutionRole.Arn
      Environment:
        Variables:
          USERNAME: !Ref APIUsername
          PASSWORD: !Ref APIPassword
      Timeout: 30
      Tags:
        - Key: Name
          Value: !Ref punitag
  LambdaPermissionAPP:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt LambdaAPPFunction.Arn
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${APIGatewayRestAPI}/*/*
  LambdaPermissionAuth:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt LambdaAuthFunction.Arn
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${APIGatewayRestAPI}/*/*

~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**8. Creating an API Gateway**

To host the website, it is necessary to create two S3 buckets, one of them as subdomain to store the static website and the other to redirect traffic to the subdomain. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  APIGatewayRestAPI:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub ${punitag}-API
      Description: An API to perform CRUD operations on a DocumentDB cluster
      ApiKeySourceType: HEADER
      EndpointConfiguration:
        Types:
          - REGIONAL
      Tags:
        - Key: Name
          Value: !Ref punitag
  APIGatewayResourcedocdb:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      ParentId: !GetAtt APIGatewayRestAPI.RootResourceId
      PathPart: docdb
  APIGatewayResourcedb:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      ParentId: !GetAtt APIGatewayResourcedocdb.ResourceId
      PathPart: '{general_db}'
  APIGatewayResourcecollection:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      ParentId: !GetAtt APIGatewayResourcedb.ResourceId
      PathPart: '{general_collection}'
  APIGatewayDeployment:
    Type: AWS::ApiGateway::Deployment
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      StageName: v1
    DependsOn:
      - APIGatewayMethodANYAuth
  APIGatewayAuthorizer:
    Type: AWS::ApiGateway::Authorizer
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      Name: !Sub ${punitag}-Authorizer
      Type: REQUEST
      IdentitySource: method.request.header.Authorization
      AuthorizerUri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaAuthFunction.Arn}/invocations
      AuthorizerResultTtlInSeconds: 300
  APIGatewayModel:
    Type: AWS::ApiGateway::Model
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      Name: Emptyresponse
      ContentType: application/json
      Schema:
        title: Empty Schema
        type: object
  APIGatewayMethodANYAuth:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref APIGatewayRestAPI
      ResourceId: !Ref APIGatewayResourcecollection
      HttpMethod: ANY
      AuthorizationType: CUSTOM
      AuthorizerId: !Ref APIGatewayAuthorizer
      RequestParameters:
        method.request.path.general_db: true
        method.request.path.general_collection: true
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaAPPFunction.Arn}/invocations
        PassthroughBehavior: WHEN_NO_MATCH
        ContentHandling: CONVERT_TO_TEXT
        IntegrationResponses:
        - StatusCode: 200
      MethodResponses:
        - StatusCode: 200
          ResponseModels:
            application/json: !Ref APIGatewayModel
  APIGatewayResponses1:
    Type: AWS::ApiGateway::GatewayResponse
    Properties:
      StatusCode: 401
      RestApiId: !Ref APIGatewayRestAPI
      ResponseType: UNAUTHORIZED
      ResponseParameters:
        gatewayresponse.header.WWW-Authenticate: '''Basic'''
      ResponseTemplates:
        application/json: '{"message":$context.error.messageString}'

~~~
As you can see, no access policy has been applied to these buckets because this policy will be created in the next steps and depends on an Origin Access Identity, which will be created later on and it will be the only one with access to the bucket content.


<br>

**9. Creating the stack with Cloudformation**

Once we have finished designing the template for our stack, it is time to build it. As I mentioned before, the complete template is available in a public repository on GitHub called [apigw-docuemntdb](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/dev/docdb.yaml).

You can use a tool named [Application Composer](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/app-composer-for-cloudformation.html) in CloudFormation console mode to validate your template and also you can drag, drop, configure, and connect a variety of resources onto a visual canvas. The following image shows the resources involved in the template and a canvas representation of them.

![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/diagram/aws-apigw-documentdb-composer-canvas.png)

To [create the stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) you need to follow these steps:

1. Logging in to the AWS Management console and open the AWS CloudFormation console. 
2. Choose Create Stack to start the Create Stack wizard.
3. __Selecting a stack template.__ On the Specify template page, choose Upload a template file to select the CloudFormation template designed.
4. __Specifying stack parameters.__ On the Specify stack details page, type a stack name in the Stack name box. In the Parameters section, specify parameters domain, subdomain, and the unique tag.
5. __Setting AWS CloudFormation stack options.__ On _Permissions - optional_, select a role that allow to this stack create, update or delete the resources involved, __IMPORTANT:__ you can create this role on IAM console but remember using the least privilege principle. In _Stack failure options_ you can set the stack behavior in case of provisioning failure.
6. __Reviewing your stack.__ Here You can review the selected options and press Submmit button to start the creation process.

<br>

---

>**Final steps:**<br>
In this section, we need to upload the files needed to update the website. To do this, we need to create a pipeline that will allow us to update any changes in our GitHub repository and automatically upload them to the S3 bucket.

**10. Setting up DocumentDB.****

A pipeline has been designed with two stages: a souce stage that will be triggered when a commit is made to the GitHub repository, and then a deployment stage that will update the files to the S3 bucket.

To create a [Pipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/pipelines-create.html) you need to follow these steps:

1. Logging in to the AWS Management console and open the CodePipeline console.
2. Choose Pipeline in the left column and click in Create pipeline to start the pipeline wizard.
   1. __Choose pipeline settings.__
        - Pipeline name: _Project-name_
        - Execution mode: Queued
        - Service role: New service role
   2. __Add source stage.__
        - Source provider: GitHub (Version 2)
        - Connection: Connect to GitHub 
          - Connection name: _Project-connection_
          - GitHub Apps: Install a new app
            - Login to [GitHub.com](https://github.com/)
            - Install the AWS Connector for GitHub
            - Repository access: Only selected repositories
        - Repository name: _Project-name_
        - Default branch: main
        - Trigger
          - Trigger type: No filter
   3. __Add build stage.__
        - Skip build stage
   4. __Add deploy stage.__
        - Deploy provider: Amazon S3
        - Bucket: Project-Bucket
        - Extract file before deploy: Check
   5. __Review.__
        - Create pipeline

<br>

**11. Testing CRUD operations**

Once the steps above are complete, you can make changes to your website code (any HTML, CSS, or Java files) and push the files to your remote repository ([GitHub.com](https://github.com/)). You can then verify in the pipeline that the source and deploy stages were triggered and the changes are uploaded to the repository within seconds. You can then visit your website to verify those changes were applied.

<br>

## Summary

When the resources created for this solution are no longer required, it is important to perform an appropriate cleanup of the resources created by the stack. To do this, you can use the CloudFormation console and follow this [guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html). Remember to review the retention policies in the template (especially for S3 buckets) and the events in the stack deletion process to avoid additional charges to the AWS account.
