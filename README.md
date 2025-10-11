# Using AWS Cloudformation to automate the setup of API Gateway and DocuemntDB Resources
## Overview

This project represents a first exploration into serverless design, demonstrating how AWS services can be orchestrated to create a scalable, secure, and maintenance-free architecture.
By replacing traditional compute instances with Lambda functions, the solution eliminates the need to manage servers, patch operating systems, or handle manual scaling — allowing full focus on logic and functionality.

The experience was both instructive and enriching, revealing how serverless components can integrate seamlessly with managed services like DocumentDB and Secrets Manager within a private network, achieving a robust architecture with minimal operational overhead.

The REST API in this project can perform operations of insert update, delete, and read against a collection of an Amazon DocumentDB resource. This design includes the creation of resources in AWS, like an Amazon DocumentDB, AWS Lambda functions and an AWS API Gateway, all of this  will be deployed through a templete in AWS CloudFormation. 

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

The following image provides a view of the proposed architecture for deploying the solution resources.

![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/diagram/aws-apigw-documentdb-architecture.png)


<br>

#### From the Client’s Perspective
For the client, the interaction with the system is simple and seamless. They only need to know the public endpoint of the API Gateway, which serves as the single entry point to the backend logic.
1. __API Request:__
The client sends an HTTP request (GET, POST, PUT, DELETE) to the API Gateway endpoint. This could be from a web application, mobile client, or even directly via Postman.
2. __Secure Routing:__
The API Gateway authenticates and routes the request to the appropriate Lambda function, based on the method and resource being accessed.
3. __Lambda Execution:__
   - One Lambda function retrieves the database credentials from AWS Secrets Manager, ensuring that no credentials are exposed in code and performs CRUD operations on the database.
   - The second Lambda function manages the API username and password 
4. __Response to the Client:__
The Lambda function returns the result (for example, a query result or confirmation of a successful insert/update).
From the client’s point of view, it feels like a traditional API — fast, reliable, and secure — but it’s powered by fully managed, auto-scaling services behind the scenes.

<br>

#### Technical Overview

The solution is built on a secure and cost-efficient VPC design:
- __VPC with 2 public and 2 private subnets__ across __two Availability Zones__, ensuring high availability.
- __Internet Gateway (IGW)__ for public subnet connectivity.
- __NAT Gateway__ in the public subnet, enabling private resources to make outbound connections without being directly exposed.
- __Amazon DocumentDB cluster__ deployed in private subnets, with a __primary__ and __replica instance__ to ensure redundancy.
- __AWS Secrets Manager__ for secure storage of database credentials.
- __Identity Access Manager__ to create rol 
- __AWS Lambda Functions:__
  - A Lambda Layer providing shared dependencies and libraries.
  - A CRUD Lambda function with permissions to retrieve the secret from Secrets Manager and to perform CRUD operations on the database.
  - An Authentication Lambda to handle basic API authorization.
- __AWS IAM__ plays a central role in defining access boundaries:
  - A dedicated IAM role is created for the Lambda functions, allowing:
    - Execution of the Lambda service.
    - Secure access to the VPC.
    - Permission to read database credentials from AWS Secrets Manager.
  - IAM policies are attached to this role to enforce the principle of least privilege, granting only the necessary actions for each Lambda’s purpose.
- __API Gateway__ to expose RESTful endpoints to external users.

This design ensures that every service communicates securely, that data remains private within the VPC, and that all actions are governed by explicit permissions defined in IAM — achieving a robust, auditable, and secure serverless architecture.

<br>

## Services

- [Amazon DocumentDB](https://docs.aws.amazon.com/documentdb/) is a fully managed, scalable, NoSQL database service, It’s designed to be compatible with MongoDB
- [Amazon VPC](https://docs.aws.amazon.com/vpc/) enables you to provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you've defined.
- [Amazon S3](https://docs.aws.amazon.com/s3/) is a cloud-based object storage service that helps you store, protect, and retrieve any amount of data.
- [AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/iam/) helps you securely manage access to your AWS resources by controlling who is authenticated and authorized to use them.
- [AWS Lambda)](https://docs.aws.amazon.com/lambda/) is a serverless compute service that runs your code in response to events and automatically manages the underlying compute resources.
- [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/) is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale.
- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/) helps you set up AWS resources, provision them quickly and consistently, and manage them throughout their lifecycle. You can use a template to describe your resources and their dependencies, and launch and configure them together as a stack, instead of managing resources individually. You can manage and provision stacks across multiple AWS accounts and AWS Regions.

<br>

## Step by Step

>**Initial setup:**<br>
Before deploying the API and database integration, two preparatory steps are required to build the serverless logic layer.
First, a Lambda Layer must be created to include shared dependencies such as the MongoDB driver and SSL certificate used to connect securely to the Amazon DocumentDB cluster.
Then, the Lambda Functions are packaged — one responsible for authentication and another for executing CRUD operations — both leveraging AWS managed services for scalability and security.
These preparatory steps ensure that all application logic can run in a modular, reusable, and secure way, following AWS best practices for serverless design.

**1. Preparing Lambda layer**

A lambda layer allow us to reuse code and dependencies across functions. It is a .zip file that contains code, library dependencies, configuration files, and addittional files. When you add a layer to a function, Lambda extracts the .zip file contents into the /opt directory in the execution environment of the function, allowing access to the layer content. 
The Lambda Layer includes two key elements:
- __MongoDB Python Driver__ Provides the necessary libraries for the Lambda function to communicate with the Amazon DocumentDB cluster.
- __PEM Certificate File__ Enables SSL/TLS encryption when establishing connections with the DocumentDB cluster, ensuring data in transit remains secure.
The preparation process involved the following high-level steps:
1. Dependency Installation – Install the MongoDB driver (pymongo) into a local directory that matches the Lambda layer’s expected structure (python/lib/...).
2. Certificate Copying – Place the .pem certificate in the same directory so it can be accessed by the Lambda function at runtime.
3. Packaging – Compress the directory into a .zip file containing the required dependencies and SSL certificate.
4. Publishing to Amazon S3 – Upload the package to an S3 bucket, where it can be referenced by the CloudFormation template during stack deployment.
This approach follows the recommended AWS pattern of managing external dependencies through Lambda Layers, ensuring modularity, consistency, and improved security in serverless environments.
After we creating our lambda layer we can add it to the CRUD lambda function and use this dependency and the certificate file. In the future you can add more depencencies to this layer if any function requieres it.


**2. Preparing Lambda functions**

Two AWS Lambda functions were prepared to implement the logic layer of the serverless solution. Each function was developed in Python and designed with a specific, well-defined purpose within the architecture.
- __Authentication Function.__ Handles the authorization process for API requests. When invoked, it processes the event received by the Lambda trigger and validates the provided user credentials against environment variables that store the expected username and password. If the credentials are valid, the function builds and returns an IAM policy document that authorizes the execution of the API request. This function acts as a custom authorizer for the API Gateway.
- __CRUD Function.__ Manages data operations against the Amazon DocumentDB cluster. It securely retrieves the database credentials from AWS Secrets Manager and establishes a connection with the cluster using SSL encryption. The function then executes Create, Read, Update, and Delete (CRUD) operations on the target collections, depending on the API request received.
The packaging and publishing process for both functions mirrors that of the Lambda Layer:
1. Each function’s code and dependencies are packaged into a ZIP file.
2. The ZIP files are uploaded to Amazon S3, where they can be referenced directly by the CloudFormation template during stack deployment.
This modular and automated setup ensures consistency, reusability, and secure deployment of all serverless components within the architecture.

<br>

---

>**CloudFormation Template:**<br>
The resources will be created using AWS Cloudformation in order to maintain infrastructure integrity, reduce errors, and track changes over time. Taking advantage of Cloudformation's ability to automate resource deployment, a template will be designed to handle the creation and configuration of the resources involved in the solution.<br>
In the following steps, part of the code used in each section will be shown, the complete template is available in a public repository called [apigw-docuemntdb](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/dev/docdb.yaml).<br>
Some inputs will be requested at the stack creation and others will be captured during template execution. The YAML format was chosen for this template.


**3. Creating a VPC**

The Virtual Private Cloud (VPC) serves as the foundation of the entire architecture, providing a secure, isolated, and well-organized networking environment for all deployed resources. It defines how services communicate with each other — both within AWS and externally — while enforcing strict access controls and enabling high availability across multiple Availability Zones (AZs).
From a design perspective, the VPC establishes the network segmentation that separates public-facing components from private backend services:
- __Public Subnets__ host resources that require internet connectivity, such as the NAT Gateway and the API Gateway’s public interface. These subnets are connected to the Internet Gateway (IGW), allowing controlled outbound access.
- __Private Subnets__ contain sensitive resources such as the Amazon DocumentDB cluster and the Lambda functions (when configured for VPC access). These resources do not have direct exposure to the internet, ensuring data confidentiality and protection.
- __Route Tables__ define traffic flow between subnets and external networks, with separate configurations for public and private subnets to maintain network isolation.
- __NAT Gateway__ enables instances or functions in private subnets to securely reach external services (for example, to download dependencies or communicate with AWS endpoints) without being directly exposed.
- __VPC Security Groups__ and Network ACLs act as virtual firewalls that control inbound and outbound traffic, enforcing the principle of least privilege at the network layer.

For this project’s Proof of Concept (PoC), a single NAT Gateway was deployed in one Availability Zone to reduce costs, while still maintaining functional connectivity for private resources. In a production-grade, highly available design, a separate NAT Gateway would be deployed in each AZ to eliminate single points of failure and provide true fault tolerance.

This design demonstrates the importance of the VPC as the core of the architecture’s security and resilience model. It defines clear traffic boundaries, supports private connectivity for sensitive services like DocumentDB and Secrets Manager, and enables scalable integration with serverless components such as AWS Lambda and API Gateway. The following code was used to create and configure the resources via cloudformation template:

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
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: 10.0.10.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  MainSubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: 10.0.100.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  SecondSubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone2
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


<br>

**4. Creating a DocumentDB cluster**

The Amazon DocumentDB cluster represents the data tier of this architecture, providing a highly available, managed, and MongoDB-compatible database service. It is deployed within the private subnets of the VPC, ensuring that all database operations remain isolated from public networks and accessible only through authorized internal components, such as the Lambda functions.
In this section of the CloudFormation template, several key resources are defined to build the DocumentDB environment:
- __Cluster Resource.__ The central component that manages database endpoints, replication, and configuration. It defines the cluster’s identifier, engine version, credentials, and network settings.
- __DB Instances (Primary and Replica).__ Two instances are created: one primary instance for write operations and one replica instance for read scalability and fault tolerance. This setup enhances performance and availability across multiple Availability Zones.
- __DB Subnet Group.__ Specifies the private subnets where the cluster and its instances are deployed, ensuring that all traffic between the database and other services remains within the VPC.
- __Security Group.__ Controls inbound access to the cluster, allowing only connections from the Lambda functions or other application components that require database access.

This configuration ensures that the database tier is secure, resilient, and aligned with AWS best practices for private network deployments. By combining private subnet isolation, replication, and managed scaling, the DocumentDB cluster provides a reliable backend capable of supporting production-grade workloads. The following code was used to create and configure the resources via cloudformation template:

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
      DBClusterIdentifier: !Ref DBClusterName
      MasterUsername: !Ref DDBMasterUser
      MasterUserPassword: !Ref DDBMasterPassword
      Port: 27017
      StorageEncrypted: true
      EngineVersion: 5.0.0
      AvailabilityZones:
        - !Ref AvailabilityZone1
        - !Ref AvailabilityZone2
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


<br>

**5. Creating a Secret**

To securely manage authentication to the Amazon DocumentDB cluster, an AWS Secrets Manager secret is created. This secret stores the database username and password in an encrypted format, eliminating the need to hardcode credentials within the Lambda functions or CloudFormation template.

In this section of the CloudFormation template, two key resources are defined:
- __Secret Resource.__ Contains the base credentials (username and password) used by the application to connect to the DocumentDB cluster. The secret is automatically encrypted using the AWS-managed KMS key for Secrets Manager.
- __Secret Target Attachment.__ Associates the secret with the DocumentDB cluster, enabling AWS Secrets Manager to manage rotation and secure integration with the database.

The Lambda function that performs CRUD operations retrieves these credentials programmatically at runtime through its IAM role permissions, ensuring that sensitive information is never exposed in plain text or embedded in the code.

This approach aligns with AWS best practices for secure credential management, providing a centralized, auditable, and scalable solution for secret storage and access control. The following code was used to create and configure the resources via cloudformation template:

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


<br>

**6. Setting up IAM permissions**

The IAM configuration in this architecture establishes the execution role that grants the Lambda functions the necessary permissions to operate securely and interact with other AWS resources. Each permission follows the principle of least privilege, ensuring that functions can perform only the actions they require.

In this section of the CloudFormation template, the following components are created:
- __Lambda Execution Role.__ The core IAM role assumed by both Lambda functions at runtime. This role defines a trust relationship with the Lambda service (lambda.amazonaws.com) and serves as the security identity under which the functions execute.
- __Attached Managed Policies:__
  - _AWSLambdaVPCAccessExecutionRole_ – Grants the permissions required for Lambda to manage Elastic Network Interfaces (ENIs) and securely connect to resources within the VPC, such as the DocumentDB cluster.
  - _AWSLambdaExecute_ – Provides basic Lambda execution capabilities, including permissions to write logs to Amazon CloudWatch Logs and to handle temporary data storage in /tmp.
- __Custom Policy for Secrets Access.__ A custom inline policy that grants the necessary permissions to read the database credentials from AWS Secrets Manager using the _secretsmanager:GetSecretValue_ action. This ensures that the Lambda functions can authenticate with the DocumentDB cluster without embedding credentials in the code.

This configuration ensures that all Lambda functions operate within a secure, auditable, and tightly scoped permission model. IAM manages the execution context and resource access, while the VPC configuration controls network connectivity. Together, they provide a clear separation between security permissions and network access boundaries. The following code was used to create and configure the resources via cloudformation template:

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


<br>

**7. Creating the Lambda resources**

This section of the CloudFormation template defines the Lambda resources that implement the logic and authentication layers of the solution, along with the permissions that allow Amazon API Gateway to invoke these functions securely.
The following resources are created:
- __Lambda Layer.__ A shared dependency layer containing the MongoDB Python driver and the SSL certificate (PEM file) used to establish encrypted connections to the Amazon DocumentDB cluster. This layer is used exclusively by the Application Lambda (CRUD Function), ensuring consistent and secure database connectivity without duplicating dependencies.
- __Application Lambda (CRUD Function).__ – Responsible for executing all Create, Read, Update, and Delete (CRUD) operations on the Amazon DocumentDB cluster. It retrieves credentials securely from AWS Secrets Manager via its IAM role and uses the Lambda Layer for shared dependencies and SSL configuration.
- __Authentication Lambda (Authorizer Function).__ Serves as a custom authorizer for API Gateway. It validates user credentials against environment variables and returns an IAM policy document authorizing or denying access to the requested API method. This function operates independently of the Layer, focusing solely on authentication and policy generation.

To enable communication between the API Gateway and these Lambda functions, the following invocation permissions are explicitly defined:
- __Lambda Permission Resources.__ These grant Amazon API Gateway the right to invoke the respective Lambda functions. Each permission specifies:
  - The target Lambda function ARN.
  - The invoking principal (_apigateway.amazonaws.com_).
  - The corresponding API Gateway resource or method ARN allowed to trigger the function.

This ensures that API Gateway can invoke only the intended functions, maintaining strict security boundaries between the API endpoints and backend logic.

Together, these configurations guarantee that each function operates within a least-privilege, tightly controlled environment:
- __The App Lambda__ securely performs database operations using the Layer and IAM-based secret retrieval.
- __The Auth Lambda__ validates access requests and issues API policies.
- __The API Gateway__ is permitted to invoke these functions only through defined, explicit permissions.

This design enforces strong separation of concerns, clear security boundaries, and full compliance with AWS best practices for serverless and API-integrated architectures. The following code was used to create and configure the resources via cloudformation template:

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


<br>

**8. Creating an API Gateway**

This section of the CloudFormation template defines the Amazon API Gateway that exposes the application’s functionality through a secure and flexible RESTful interface. The API Gateway serves as the single public entry point to the solution, handling authentication, request routing, and response formatting for all client interactions.
The following components are created as part of the API Gateway configuration:
- __REST API Resource.__ The root definition of the API that groups all endpoints under a single logical interface. It defines the base path, stage configuration, and global settings for the API deployment.
- __Resource (Path).__ Represents the application’s main resource (for example, /api or /items) that clients interact with. This resource acts as the logical container for the method and its integration.
- __Method (ANY).__ A single method configured with the HTTP verb ANY, allowing the API Gateway to accept and route any HTTP request type (GET, POST, PUT, DELETE, etc.) to the backend Lambda function.
  - The integration target for this method is the Application Lambda (CRUD Function).
  - The method uses a custom authorizer backed by the Authentication Lambda to validate credentials before allowing access.
  - The method configuration also defines the request integration, response mapping templates, and error handling behavior.
- __Authorizer.__ A custom Lambda authorizer integrated with the Authentication Lambda.
  - When an incoming request is received, API Gateway first invokes the authorizer function.
  - If authentication succeeds, the function returns an IAM policy document granting execution permissions for the request.
- __Model.__ A JSON schema that defines the expected request or response structure.
  - The model is used for validation to ensure that incoming payloads follow the correct format before reaching the backend Lambda function.
- __Method Response.__ Specifies the expected response structure, including status codes (e.g., 200, 400, 500) and headers, to ensure consistent communication with the client.
- __Deployment and Stage.__ The deployment resource captures the API configuration and publishes it to a stage (such as v1). This defines the live, publicly accessible version of the API through a fixed endpoint URL.

By using the ANY method, the API Gateway simplifies integration and reduces configuration complexity, allowing a single Lambda function to process all request types.
This setup is ideal for lightweight or proof-of-concept designs where flexibility and simplicity are prioritized, while still maintaining security, authentication, and structured request handling through the Lambda authorizer. The following code was used to create and configure the resources via cloudformation template:

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


<br>

**9. Creating the stack with Cloudformation**

Once we have finished designing the template for our stack, it is time to build it. As I mentioned before, the complete template is available in a public repository on GitHub called [apigw-docuemntdb](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/dev/docdb.yaml).

You can use a tool named [Application Composer](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/app-composer-for-cloudformation.html) in CloudFormation console mode to validate your template and also you can drag, drop, configure, and connect a variety of resources onto a visual canvas. The following image shows the resources involved in the template and a canvas representation of them.

![Image description](https://github.com/Sjleal/aws-apigw-documentdb/blob/main/images/diagram/aws-apigw-documentdb-composer-canvas.png)

To [create the stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) you need to follow these steps:

1. Logging in to the AWS Management console and open the AWS CloudFormation console. 
2. Choose Create Stack to start the Create Stack wizard.
3. __Selecting a stack template.__ On the Specify template page, choose Upload a template file to select the CloudFormation template designed.
4. __Specifying stack parameters.__ On the Specify stack details page, type a stack name in the Stack name box. In the Parameters section, specify parameters requiered:
  - AvailabilityZone1 / AvailabilityZone2 – Define the two Availability Zones where public and private subnets will be deployed to provide high availability.
  - DBClusterName – The logical name assigned to the Amazon DocumentDB cluster.
  - DDBMasterUser – The master username for the DocumentDB cluster.
  - DDBMasterPassword – The corresponding master password for the DocumentDB cluster (stored securely in AWS Secrets Manager after creation).
  - APIUsername / APIPassword – The credentials used by the Authentication Lambda to validate incoming API requests.
  - TagName – A custom tag applied to all resources in the stack for easy identification and management.
5. __Setting AWS CloudFormation stack options.__ On _Permissions - optional_, select a role that allow to this stack create, update or delete the resources involved, __IMPORTANT:__ you can create this role on IAM console but remember using the least privilege principle. In _Stack failure options_ you can set the stack behavior in case of provisioning failure.
6. __Reviewing your stack.__ Here You can review the selected options and press Submmit button to start the creation process.

<br>

---
  
>**Final steps:**<br>
In the final stage, the Amazon DocumentDB cluster is deployed within private subnets, creating the secure data layer for the application.
Once the API and Lambda functions are fully configured, a series of CRUD tests can be performed through the API Gateway endpoint using simple HTTP requests (GET, POST, PUT, PATCH, DELETE).
These tests confirm that all components — API Gateway, Lambda, Secrets Manager, and DocumentDB — are working together as a unified, secure, and fully automated solution.

**10. Setting up DocumentDB.**

The Amazon DocumentDB cluster is provisioned to serve as the managed database layer for the application. The cluster is created within private subnets of the VPC to ensure network isolation and security. A primary and replica instance are deployed to provide high availability and read scalability, while the DB subnet group, security group, and parameter groups ensure proper configuration and connectivity. The application Lambda function securely retrieves its credentials from AWS Secrets Manager and establishes an SSL-encrypted connection with the cluster to perform database operations.

Populating data in the Amazon DocumentDB cluster is beyond the scope of this lab.However, once the cluster is deployed, you can create and manage any collections required for your application by connecting securely to the DocumentDB endpoint. Using an SSL connection and valid credentials (retrieved from AWS Secrets Manager), you can interact with the database just as you would with any MongoDB-compatible environment — inserting, querying, or updating data as needed.

This flexibility allows you to expand the dataset or simulate real application scenarios without modifying the infrastructure defined in CloudFormation.


<br>

**11. Testing CRUD operations**

Once the API Gateway and backend functions are fully deployed, the final step is to test the API endpoints to verify that the CRUD operations work as expected.
Through the API Gateway’s public endpoint, you can send HTTP requests (GET, POST, PUT, DELETE) that trigger the Application Lambda function, which in turn interacts with the DocumentDB cluster. Each request confirms one aspect of the integration: data creation, retrieval, modification, or deletion. Successful responses validate that the entire stack — API Gateway, Lambda functions, Secrets Manager, and DocumentDB — is fully integrated, functional, and secure.

You can interact with the API endpoint using tools such as cURL, Postman, or any HTTP client. Each request sent to the API Gateway is routed through the Authentication Lambda and then processed by the Application Lambda, which performs the corresponding operation on the Amazon DocumentDB cluster.

The base endpoint typically follows this format:
~~~
https://<api-id>.execute-api.<region>.amazonaws.com/v1
~~~

Each operation targets a specific path corresponding to the database and collection you want to interact with.
For example:

~~~
/<docdb>/<database-name>/<collection-name>
~~~


Below are sample commands for common CRUD operations:

__GET — Retrieve Data__

Fetch all documents or a specific document by ID:
~~~
curl -X GET "https://<api-id>.execute-api.<region>.amazonaws.com/v1/docdb/mydatabase/mycollection"
~~~

or
~~~
curl -X GET "https://<api-id>.execute-api.<region>.amazonaws.com/v1/docdb/mydatabase/mycollection/<document-id>"
~~~
__POST — Create a New Document__

Insert a new record into the specified collection:
~~~
curl -X POST "https://<api-id>.execute-api.<region>.amazonaws.com/v1/docdb/mydatabase/mycollection" \
     -H "Content-Type: application/json" \
     -d '{"name": "John Doe", "email": "john@example.com"}'
~~~
__PUT — Update an Existing Document (Replace)__

Replace an existing document by ID:
~~~
curl -X PUT "https://<api-id>.execute-api.<region>.amazonaws.com/v1/docdb/mydatabase/mycollection/<document-id>" \
     -H "Content-Type: application/json" \
     -d '{"name": "John Doe", "email": "john@newdomain.com"}'
~~~
__PATCH — Partial Update__

Update only specific fields of an existing document:
~~~
curl -X PATCH "https://<api-id>.execute-api.<region>.amazonaws.com/v1/docdb/mydatabase/mycollection/<document-id>" \
     -H "Content-Type: application/json" \
     -d '{"email": "john@update.com"}'
~~~
__DELETE — Remove a Document__

Delete a document from the collection:
~~~
curl -X DELETE "https://<api-id>.execute-api.<region>.amazonaws.com/v1/docdb/mydatabase/mycollection/<document-id>"
~~~

Each response returned by the API Gateway confirms the result of the operation — for example, a 200 OK with the retrieved or updated document, or a 204 No Content for successful deletions. This approach makes it easy to test and validate end-to-end functionality directly from the command line or through any REST client.

<br>

## Summary

This project brought together multiple AWS services to design and deploy a robust, secure, and fully automated solution following a 3-tier architecture pattern.
By combining Infrastructure as Code (IaC), serverless computing, and managed services, the implementation demonstrates how modern cloud architectures can achieve scalability, reliability, and operational simplicity.

The use of AWS CloudFormation automated the deployment of every component — from the VPC and security groups to API Gateway, Lambda functions, IAM roles, Secrets Manager, and the Amazon DocumentDB cluster, ensuring a repeatable and consistent environment.

Adopting a serverless approach eliminated the need to manage infrastructure directly, reducing complexity and allowing the focus to remain on functionality and business logic. The architecture also reflects key principles of:

- __High Availability (HA)__ through multi-AZ distribution and scalable compute.

- __Security__ via IAM roles, private networking, and encrypted credentials in Secrets Manager.

- __Cost Optimization__ using managed and event-driven services that scale with demand.

Beyond its technical robustness, this demonstration served as a hands-on learning experience, reinforcing essential AWS concepts such as Serverless, IaC, HA, Managed Services, and Security by Design. It illustrates how these concepts converge to build cloud-native architectures that are scalable, reliable, and easy to maintain — key pillars for any modern cloud solution.

When the resources created for this solution are no longer required, it is important to perform an appropriate cleanup of the resources created by the stack. To do this, you can use the CloudFormation console and follow this [guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html). Remember to review the retention policies in the template and the events in the stack deletion process to avoid additional charges to the AWS account.

