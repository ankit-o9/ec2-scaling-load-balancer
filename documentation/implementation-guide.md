# EC2 Auto Scaling and Load Balancing - Implementation Guide

This console-oriented guide describes the architecture documented in the repository README. It is a learning example, not a deployable infrastructure template.

## Architecture overview

The Application Load Balancer (ALB) receives client traffic and routes requests to healthy EC2 instances registered in a target group. An Auto Scaling group (ASG) maintains a configured fleet across multiple Availability Zones using a launch template. Target-group health checks can be used as an ASG health check source so unhealthy instances are replaced. A target-tracking scaling policy uses CloudWatch metrics such as average CPU utilization to adjust desired capacity.

## Prerequisites

- An AWS account and a VPC with public subnets in at least two Availability Zones for an internet-facing ALB.
- An EC2 launch template using a supported AMI and instance type.
- An application that listens on the target group's configured port and responds successfully to its health-check path.
- An IAM identity with permission to configure EC2, Elastic Load Balancing, Auto Scaling, and CloudWatch resources.

## 1. Create a launch template

In the EC2 console, create a launch template with the selected AMI, instance type, and application bootstrap/user data. Configure an instance security group to allow the application port only from the ALB security group. Do not expose SSH to all internet addresses; restrict administrative access to a trusted source range or use AWS Systems Manager Session Manager.

The original example uses Amazon Linux 2 and a small instance type for testing. Choose an AMI and instance family that are currently supported in your region and suitable for your application.

## 2. Create a target group

Create a target group with target type **Instances**, the same VPC as the ALB and ASG, and the application's protocol and port (the example uses HTTP on port 80). Configure a health-check path that returns a successful response only when the application is ready to receive traffic.

## 3. Create an Application Load Balancer

Create an internet-facing ALB in at least two public subnets/AZs. Add a listener and route its default action to the target group. For real public applications, configure HTTPS with a suitable certificate; do not send sensitive traffic over plain HTTP. The ALB security group should permit the listener traffic from intended clients.

## 4. Create an Auto Scaling group

Create an ASG from the launch template. Select multiple Availability Zones in the ALB's VPC, attach the ASG to the target group, and enable Elastic Load Balancing health checks. Set minimum, desired, and maximum capacity based on the workload and available subnet capacity.

The source example suggests desired capacity 2, minimum 1, and maximum 4. These are sample values only; set limits and instance sizes deliberately because running instances and load balancers incur charges.

## 5. Add a target-tracking scaling policy

In the ASG automatic scaling settings, create a target-tracking policy using average EC2 CPU utilization. The source example uses a target of 50%. Choose a target that reflects your application's performance and scaling characteristics. Allow CloudWatch time to collect metrics and the group to launch and warm up instances before assessing the response.

## 6. Verify behavior

Use the ALB DNS name to make requests and confirm that the target group reports instances as healthy. Observe the ASG's desired and actual capacity and review CloudWatch metrics and alarms. In a non-production environment, conduct controlled load tests and verify that capacity increases and later scales in as designed. Also terminate a test instance to confirm unhealthy-instance replacement and check application availability throughout.

Useful AWS CLI inspection commands include:

```bash
aws elbv2 describe-load-balancers --names web-app-alb --query 'LoadBalancers[0].DNSName'
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names web-app-asg
aws elbv2 describe-target-health --target-group-arn YOUR_TARGET_GROUP_ARN
```

Replace example names and ARN values with resources in your own account. Do not run cleanup commands against resources you do not own.

## Troubleshooting checklist

| Symptom | Checks |
| --- | --- |
| Instances are not registering | Confirm VPC/subnet selection, target-group attachment, application port, and security-group traffic from the ALB. |
| Target health checks fail | Confirm the application is running, the health-check path returns success, and the target port is correct. |
| Scaling does not occur | Review ASG policy settings, CloudWatch metrics/alarms, scaling limits, cooldown/warm-up settings, and service quotas. |
| Requests fail intermittently | Check target health, application logs, ALB listener rules, and capacity across Availability Zones. |

## Cleanup and cost

When finished, remove only the test resources you created. Delete or reduce the ASG first, then remove the ALB, target group, and launch template after confirming that no other environment depends on them. Review the AWS billing dashboard for any remaining resources.

## Attribution

This guide is adapted from the implementation notes in [VanshPatyal/ec2-auto-scaling](https://github.com/VanshPatyal/ec2-auto-scaling) and is included with permission. The original source commit history was not imported.
