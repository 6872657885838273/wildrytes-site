Operations Runbook

1. ALB 5xx Errors

Check ECS task logs in CloudWatch

Verify health checks

Restart tasks

Scale out if CPU > 70%


2. High RDS Connections

Check for long-lived connections

Enable connection pooling (pgBouncer)

Scale read replicas

Increase database instance size


3. CloudFront Serving Old Assets

Perform CloudFront invalidation

Verify S3 update completed


4. Deployment Failure

Rollback ECS task definition

Redeploy last known good version

Notify on-call


5. ECS Tasks Unhealthy

Check CPU/memory

Check recent deployment

Restart service if required

Investigate logs