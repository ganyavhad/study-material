# AWS & Serverless Architecture - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> *"Migrated Java Spring Boot services to Node.js... deployed as AWS Lambda functions to adopt a serverless architecture"*

---

## 1. AWS Overview

Amazon Web Services (AWS) is the world's largest cloud platform with 200+ services.

### Core Services Map

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Cloud                             │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Compute │  │ Storage │  │ Database │  │  Networking  │   │
│  │─────────│  │─────────│  │──────────│  │──────────────│   │
│  │ EC2     │  │ S3      │  │ RDS      │  │ VPC          │   │
│  │ Lambda  │  │ EBS     │  │ DynamoDB │  │ Route 53     │   │
│  │ ECS     │  │ EFS     │  │ Aurora   │  │ CloudFront   │   │
│  │ Fargate │  │ Glacier │  │ ElastiC. │  │ API Gateway  │   │
│  └─────────┘  └─────────┘  └──────────┘  └──────────────┘  │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ Security │  │  DevOps  │  │ Messaging│  │ Analytics  │   │
│  │──────────│  │──────────│  │──────────│  │────────────│   │
│  │ IAM     │  │ CodePipe │  │ SQS      │  │ Athena     │   │
│  │ KMS     │  │ CodeBuild│  │ SNS      │  │ Kinesis    │   │
│  │ WAF     │  │ CodeDeploy│ │ EventBr. │  │ QuickSight │   │
│  │ Cognito │  │ CloudForm│  │ Step Fn  │  │ Glue       │   │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. AWS Lambda (Serverless Compute)

### What is Serverless?
- **No server management** — AWS handles infrastructure
- **Pay-per-execution** — billed per invocation + duration
- **Auto-scaling** — scales from 0 to thousands concurrently
- **Event-driven** — triggered by API Gateway, S3, SQS, etc.

### Lambda Handler (Node.js)

```typescript
// handler.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';

export const getUsers = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  try {
    const { limit = '10', offset = '0' } = event.queryStringParameters || {};

    const users = await userService.getUsers(
      parseInt(limit, 10),
      parseInt(offset, 10)
    );

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ data: users })
    };
  } catch (error) {
    console.error('Error fetching users:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal Server Error' })
    };
  }
};

export const createUser = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  try {
    const body = JSON.parse(event.body || '{}');

    // Input validation
    if (!body.name || !body.email) {
      return {
        statusCode: 400,
        body: JSON.stringify({ error: 'Name and email are required' })
      };
    }

    const user = await userService.createUser(body);

    return {
      statusCode: 201,
      body: JSON.stringify({ data: user })
    };
  } catch (error) {
    console.error('Error creating user:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal Server Error' })
    };
  }
};
```

### Lambda with Serverless Framework

```yaml
# serverless.yml
service: user-service

provider:
  name: aws
  runtime: nodejs20.x
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  memorySize: 256
  timeout: 30
  environment:
    DATABASE_URL: ${ssm:/user-service/${self:provider.stage}/database-url}
    NODE_ENV: ${self:provider.stage}

  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:Query
            - dynamodb:Scan
          Resource: !GetAtt UsersTable.Arn

functions:
  getUsers:
    handler: src/handlers/user.getUsers
    events:
      - httpApi:
          path: /users
          method: GET

  createUser:
    handler: src/handlers/user.createUser
    events:
      - httpApi:
          path: /users
          method: POST

  processUserEvent:
    handler: src/handlers/user.processEvent
    events:
      - sqs:
          arn: !GetAtt UserEventQueue.Arn
          batchSize: 10

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: users-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH

    UserEventQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: user-events-${self:provider.stage}
        VisibilityTimeout: 60
```

---

## 3. API Gateway

```yaml
# API Gateway with Lambda integration
functions:
  api:
    handler: src/handlers/api.handler
    events:
      - httpApi:
          path: /api/{proxy+}
          method: ANY
      - httpApi:
          path: /api
          method: ANY

# With authorizer
  securedEndpoint:
    handler: src/handlers/secured.handler
    events:
      - httpApi:
          path: /secure/data
          method: GET
          authorizer:
            name: jwtAuthorizer
            type: jwt
            id: !Ref HttpApiAuthorizer
```

---

## 4. DynamoDB

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  GetCommand,
  PutCommand,
  QueryCommand,
  ScanCommand
} from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

const TABLE_NAME = process.env.USERS_TABLE || 'users';

// Get item by key
async function getUserById(id: string) {
  const result = await docClient.send(new GetCommand({
    TableName: TABLE_NAME,
    Key: { id }
  }));
  return result.Item;
}

// Query by GSI
async function getUsersByEmail(email: string) {
  const result = await docClient.send(new QueryCommand({
    TableName: TABLE_NAME,
    IndexName: 'email-index',
    KeyConditionExpression: 'email = :email',
    ExpressionAttributeValues: { ':email': email }
  }));
  return result.Items;
}

// Put item
async function createUser(user: { id: string; name: string; email: string }) {
  await docClient.send(new PutCommand({
    TableName: TABLE_NAME,
    Item: { ...user, createdAt: new Date().toISOString() },
    ConditionExpression: 'attribute_not_exists(id)'
  }));
  return user;
}
```

---

## 5. S3 (Simple Storage Service)

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });

// Upload file
async function uploadFile(key: string, body: Buffer, contentType: string) {
  await s3.send(new PutObjectCommand({
    Bucket: process.env.BUCKET_NAME,
    Key: key,
    Body: body,
    ContentType: contentType
  }));
}

// Generate pre-signed URL for secure upload
async function getUploadUrl(key: string) {
  const command = new PutObjectCommand({
    Bucket: process.env.BUCKET_NAME,
    Key: key
  });
  return getSignedUrl(s3, command, { expiresIn: 3600 });
}
```

---

## 6. SQS & Event-Driven Architecture

```typescript
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({});

// Producer: Send message
async function publishEvent(event: { type: string; data: unknown }) {
  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.QUEUE_URL,
    MessageBody: JSON.stringify(event),
    MessageGroupId: event.type, // For FIFO queues
    MessageDeduplicationId: `${event.type}-${Date.now()}`
  }));
}

// Consumer: Lambda handler for SQS
export const processEvent = async (event: SQSEvent) => {
  const failedRecords: SQSBatchItemFailure[] = [];

  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body);
      await handleMessage(message);
    } catch (error) {
      console.error(`Failed to process ${record.messageId}:`, error);
      failedRecords.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures: failedRecords };
};
```

---

## 7. Step Functions (Orchestration)

```json
{
  "Comment": "User onboarding workflow",
  "StartAt": "ValidateInput",
  "States": {
    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:validate",
      "Next": "CreateUser",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "HandleError"
      }]
    },
    "CreateUser": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:createUser",
      "Next": "SendWelcomeEmail"
    },
    "SendWelcomeEmail": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:sendEmail",
      "End": true
    },
    "HandleError": {
      "Type": "Fail",
      "Cause": "Input validation failed"
    }
  }
}
```

---

## 8. Lambda Best Practices

### Cold Start Optimization
```typescript
// Initialize outside handler (reused across invocations)
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
const client = new DynamoDBClient({}); // Reused!

export const handler = async (event) => {
  // Handler code uses pre-initialized client
  return await processRequest(event, client);
};
```

### Provisioned Concurrency
```yaml
functions:
  criticalApi:
    handler: src/critical.handler
    provisionedConcurrency: 5  # Keep 5 instances warm
```

### Lambda Layers
```yaml
layers:
  commonDeps:
    path: layers/common
    compatibleRuntimes:
      - nodejs20.x
    description: Common dependencies

functions:
  myFunction:
    handler: src/handler.main
    layers:
      - !Ref CommonDepsLambdaLayer
```

---

## 9. Monitoring & Logging

```typescript
// Structured logging with CloudWatch
const log = (level: string, message: string, meta?: Record<string, unknown>) => {
  console.log(JSON.stringify({
    level,
    message,
    timestamp: new Date().toISOString(),
    requestId: process.env.AWS_REQUEST_ID,
    ...meta
  }));
};

// X-Ray tracing
import AWSXRay from 'aws-xray-sdk-core';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';

const client = AWSXRay.captureAWSv3Client(new DynamoDBClient({}));
```

---

## 10. Security Best Practices

| Practice | Implementation |
|----------|---------------|
| Least Privilege IAM | Only grant permissions functions actually need |
| Secrets Management | Use AWS Secrets Manager / SSM Parameter Store |
| VPC for Databases | Run Lambda in VPC to access RDS/ElastiCache |
| API Authentication | Use Cognito or custom Lambda authorizers |
| Encryption | Enable at-rest (KMS) and in-transit (TLS) |
| Input Validation | Validate all event payloads before processing |

---

## 11. Interview Questions

1. **What is a cold start?** — First invocation initializes the runtime. Mitigate with provisioned concurrency, smaller packages, or keeping functions warm.
2. **Lambda timeout?** — Max 15 minutes. For longer tasks, use Step Functions or ECS.
3. **How does Lambda scale?** — Each concurrent request gets its own instance. Default: 1000 concurrent executions per region.
4. **SQS vs SNS vs EventBridge?** — SQS: queue (pull). SNS: pub/sub (push). EventBridge: event bus with rules and filtering.
5. **What is the Lambda execution environment?** — Container with runtime, handler code, and SDK. Reused across invocations (warm start).
6. **How to handle Lambda failures?** — DLQ (Dead Letter Queue), retry policies, Destinations for async invocations.
7. **What is API Gateway throttling?** — Default 10,000 req/s per region. Configurable per API/stage/method.
8. **Explain SAM vs Serverless Framework** — SAM: AWS-native, CloudFormation extension. Serverless: Multi-cloud, plugin ecosystem, larger community.

---

## 12. Practice Exercises

1. Build a CRUD API with Lambda + API Gateway + DynamoDB
2. Create an event-driven pipeline: S3 → Lambda → SQS → Lambda
3. Implement a Step Function workflow for order processing
4. Set up a Lambda with VPC access to RDS
5. Deploy using Serverless Framework with multiple stages
