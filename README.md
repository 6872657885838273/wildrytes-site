
DevOps Design — UI + API + Database

Submission by Dhanusri Mahalingam 

Overview

This design describes a scalable, secure, and highly available architecture consisting of:

UI (React SPA hosted on AWS S3 + CloudFront)

API (Containerized backend on ECS Fargate behind ALB)

Database (Amazon RDS PostgreSQL with Multi-AZ)

CI/CD (Conceptual GitHub Actions Pipeline)


Architecture diagrams are included in diagram.pdf.

1. UI (Frontend)

Hosting

Static React SPA stored in S3

Delivered via CloudFront CDN for global latency reduction

Route53 for domain DNS

ACM for HTTPS certificates


Traffic Flow

User → Route53 → CloudFront → S3 (static content) Dynamic API calls → CloudFront → ALB → ECS tasks

Caching & Performance

CloudFront long TTL for hashed assets

Cache invalidation triggered on deployments


Security

HTTPS everywhere

WAF (OWASP rules)

S3 bucket restricted using OAC/OAI

2. Backend API (ECS Fargate)

Deployment

Backend containerized and deployed on ECS Fargate running in private subnets across 2 AZs

ALB in public subnets forwards traffic to target group (ECS)


Scaling

ECS Service Auto Scaling based on:

CPU > 70%

Memory > 70%

ALB RequestCount > threshold



Secrets

All sensitive data stored in AWS Secrets Manager

IAM task role grants permission to read secrets


Networking & Security

ALB SG → ECS SG → RDS SG chain

No public exposure for ECS tasks

Health checks ensure rolling replacement of unhealthy tasks

3. Database (RDS PostgreSQL)

High Availability

Multi-AZ enabled (standby replica auto-failover)

Optional Read Replicas for read scaling


Backups

Automated backups (7–30 days retention)

PITR (point-in-time recovery)

Manual snapshots before deployments


Scaling

Vertical scaling for writes

Read replicas for read-heavy workloads


Migration Strategy

Use schema migration tool (Flyway/Liquibase)

Backward-compatible migrations:

Add column → backfill → enforce constraints late

4. CI/CD (Conceptual GitHub Actions)

Triggers

PR: run unit tests

Push to main: build → test → deploy to dev

Manual approval for stage → prod promotion


Pipeline Stages

A. Build: Lint, unit test frontend & backend


B. Package: Build frontend artifact; conceptually push API image to ECR


C. Deploy (Dev): Upload front-end to S3 + CloudFront invalidation; update ECS task definition


D. E2E Tests: Smoke tests


E. Approval Step: Manual reviewer input


F. Deploy (Prod): Same process for prod


G. Rollback: If health checks fail, revert to previous version

5. Monitoring, Logging & Alerting

Logs

CloudWatch Logs for ECS tasks

CloudFront/S3 access logs stored to S3


Metrics

ALB: 4xx/5xx rates, latency

ECS: CPU/memory

RDS: Connections, IOPS, storage


Alerts

CloudWatch Alarms → SNS → Email/PagerDuty

6. Cost Considerations

S3 + CloudFront: minimal

ECS Fargate: based on vCPU + memory per task

RDS Multi-AZ: higher due to standby replica

NAT Gateways: per-hour + data processing

AWS Budgets recommended

7. Security

IAM least privilege

No public access to ECS/RDS

Secrets in Secrets Manager

TLS termination on CloudFront/ALB

VPC Flow Logs optional

8. Backup & DR

RDS automated backups & snapshots

Cross-region read replica for DR

S3 versioning enabled

9. Migration Strategy

Blue/Green deployments for API

Frontend versioning with S3 + CloudFront




# AWS Project - Build a Full End-to-End Web Application with 7 Services | Step-by-Step Tutorial

This repo contains the code files used in this [YouTube video](https://youtu.be/K6v6t5z6AsU).

## TL;DR
We're creating a web application for a unicorn ride-sharing service called Wild Rydes (from the original [Amazon workshop](https://aws.amazon.com/serverless-workshops)).  The app uses IAM, Amplify, Cognito, Lambda, API Gateway and DynamoDB, with code stored in GitHub and incorporated into a CI/CD pipeline with Amplify.

The app will let you create an account and log in, then request a ride by clicking on a map (powered by ArcGIS).  The code can also be extended to build out more functionality.

## Cost
All services used are eligible for the [AWS Free Tier](https://aws.amazon.com/free/).  Outside of the Free Tier, there may be small charges associated with building the app (less than $1 USD), but charges will continue to incur if you leave the app running.  Please see the end of the YouTube video for instructions on how to delete all resources used in the video.

## The Application Code
The application code is here in this repository.

## The Lambda Function Code
Here is the code for the Lambda function, originally taken from the [AWS workshop](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-3/ ), and updated for Node 20.x:

```node
import { randomBytes } from 'crypto';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const ddb = DynamoDBDocumentClient.from(client);

const fleet = [
    { Name: 'Angel', Color: 'White', Gender: 'Female' },
    { Name: 'Gil', Color: 'White', Gender: 'Male' },
    { Name: 'Rocinante', Color: 'Yellow', Gender: 'Female' },
];

export const handler = async (event, context) => {
    if (!event.requestContext.authorizer) {
        return errorResponse('Authorization not configured', context.awsRequestId);
    }

    const rideId = toUrlString(randomBytes(16));
    console.log('Received event (', rideId, '): ', event);

    const username = event.requestContext.authorizer.claims['cognito:username'];
    const requestBody = JSON.parse(event.body);
    const pickupLocation = requestBody.PickupLocation;

    const unicorn = findUnicorn(pickupLocation);

    try {
        await recordRide(rideId, username, unicorn);
        return {
            statusCode: 201,
            body: JSON.stringify({
                RideId: rideId,
                Unicorn: unicorn,
                Eta: '30 seconds',
                Rider: username,
            }),
            headers: {
                'Access-Control-Allow-Origin': '*',
            },
        };
    } catch (err) {
        console.error(err);
        return errorResponse(err.message, context.awsRequestId);
    }
};

function findUnicorn(pickupLocation) {
    console.log('Finding unicorn for ', pickupLocation.Latitude, ', ', pickupLocation.Longitude);
    return fleet[Math.floor(Math.random() * fleet.length)];
}

async function recordRide(rideId, username, unicorn) {
    const params = {
        TableName: 'Rides',
        Item: {
            RideId: rideId,
            User: username,
            Unicorn: unicorn,
            RequestTime: new Date().toISOString(),
        },
    };
    await ddb.send(new PutCommand(params));
}

function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

function errorResponse(errorMessage, awsRequestId) {
    return {
        statusCode: 500,
        body: JSON.stringify({
            Error: errorMessage,
            Reference: awsRequestId,
        }),
        headers: {
            'Access-Control-Allow-Origin': '*',
        },
    };
}
```

## The Lambda Function Test Function
Here is the code used to test the Lambda function:

```json
{
    "path": "/ride",
    "httpMethod": "POST",
    "headers": {
        "Accept": "*/*",
        "Authorization": "eyJraWQiOiJLTzRVMWZs",
        "content-type": "application/json; charset=UTF-8"
    },
    "queryStringParameters": null,
    "pathParameters": null,
    "requestContext": {
        "authorizer": {
            "claims": {
                "cognito:username": "the_username"
            }
        }
    },
    "body": "{\"PickupLocation\":{\"Latitude\":47.6174755835663,\"Longitude\":-122.28837066650185}}"
}
```

