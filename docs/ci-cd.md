CI/CD Pipeline (Conceptual Design)

Event Triggers

Push to main

Pull Request created


Jobs

build:

Install dependencies

Run unit tests for backend and frontend


package:

Build frontend bundle

(Conceptually) build Docker image and push to ECR


deploy-dev:

Sync frontend to S3

CloudFront invalidation

Update ECS task definition for dev


e2e-tests:

Run integration/smoke tests


manual-approval:

Reviewer approves promotion


deploy-prod:

Same as dev but targeting prod resources


rollback strategy:

Revert ECS task definition

Restore previous S3 artifact