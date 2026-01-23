# AWS Services And Best Practices

## VPC

### Connect VPC to Internet

The foundational resources required to expose an AWS VPC to the internet remain consistent, with an emphasis on **Internet Gateways** for public access and **NAT Gateways** for secure outbound connectivity

#### Essential Connectivity Resources

To enable internet access, you must deploy and configure these core components: 
- **Internet Gateway (IGW)**: This is the "main gate" for your VPC. It must be created and explicitly attached to your VPC to allow any communication with the internet.
- **Public Subnet**: A subnet becomes "public" only when its Route Table contains a route (typically `0.0.0.0/0`) targeting the attached Internet Gateway.
- **Elastic IP (EIP)**: A static, public IPv4 address. You will need one for a **NAT Gateway** or to manually assign to a specific EC2 instance that must be reachable directly from the internet. 

#### Resources for Secure Outbound Access

If you have sensitive resources (like databases) in a Private Subnet that need to download updates without being exposed to incoming internet traffic, use: 
- **NAT Gateway**: Resides in a public subnet and allows private instances to initiate outbound connections while blocking unsolicited inbound ones.
- **Egress-Only Internet Gateway**: Used specifically for IPv6 traffic to allow outbound connectivity while preventing inbound access. 

#### Security & Control Resources
Opening a VPC to the internet requires multiple layers of defense-in-depth: 
- **Security Groups**: Act as a virtual firewall for your instances, controlling traffic at the resource level.
- **Network ACLs (NACLs)**: Act as a firewall for the subnet, providing a stateless second layer of security.
- **AWS WAF & Shield**: Highly recommended in 2026 to protect **internet-facing Application Load Balancers (ALBs)** from web exploits and DDoS attacks. 

### Transit Gateway

**AWS Transit Gateway (TGW)** is the standard hub-and-spoke router for interconnecting multiple VPCs and on-premises networks.  
As of mid-2025, AWS introduced **Native Network Firewall** integration, which fundamentally changed how "firewall rules" are applied to **Transit Gateway** traffic. 

#### Centralized Traffic Inspection

Previously, you had to build a complex **"Inspection VPC"** to filter traffic. In 2026, you can attach an **AWS Network Firewall** directly to the **Transit Gateway** as a native attachment. 
- **Native Attachment**: You no longer need to manage dedicated subnets or route tables in an inspection VPC. The firewall is attached directly to the TGW.
- **Appliance Mode**: This is automatically enabled for native firewall attachments, ensuring traffic flows are symmetric (the same firewall sees both the request and response) across multiple Availability Zones.
- **Cost Allocation**: A new 2025 feature allows for flexible cost allocation, meaning the data processing costs of the centralized firewall can be automatically billed to the individual "spoke" accounts that generated the traffic, rather than just the central security account

#### Firewall Rule Types for Transit Gateway

Security is implemented using **AWS Network Firewall Policies**, which consist of two rule categories: 
- **Stateless Rules**:
  - **Function**: Standard 5-tuple filtering (Source/Dest IP, Port, Protocol).
  - **Behavior**: Fast, but "dumb"—they do not track session state. Best for high-volume, simple "Allow" or "Drop" actions based on CIDR blocks.  
- **Stateful Rules**:
  - **Function**: **Deep Packet Inspection (DPI)** using the **Suricata engine**.
  - **Capabilities**: Domain filtering (e.g., allow `*.aws.com` but block others), TLS inspection, and signature-based **Intrusion Prevention (IPS)**. 

#### Implementing "Rules" via Routing

In a Transit Gateway architecture, "firewall rules" are only effective if traffic is forced through the firewall. This is handled by **TGW Route Tables**: 
- **Spoke Route Table**: All "spoke" VPCs (your apps) are associated with a route table that has a default route (`0.0.0.0/0`) pointing to the Firewall Attachment.
- **Inspection/Return Route Table**: The Firewall Attachment itself is associated with a separate route table that contains routes back to the specific destination VPCs. 

#### Traditional "Firewall" Layers  

While the Network Firewall handles the heavy lifting, standard AWS security resources still apply to the Transit Gateway environment:  
- **Security Groups**: Still used at the instance level (Layer 4) within individual VPCs.
- **Network ACLs**: Stateless subnet-level protection (Layer 4) used for defense-in-depth.
- **TGW Flow Logs**: Essential for auditing and troubleshooting rule hits/misses in 2026. 

## Lambda

In 2026, AWS Lambda remains the core of serverless architecture, recently enhanced with features like **Durable Functions** for long-running workflows and **Lambda Managed Instances** for specialized compute  

### Lambda's Roles and Policies

Lambda security is defined by two primary types of permissions: 
- **Execution Role (IAM Role)**: This "identity card" allows the Lambda function to access other AWS services once it is running.
  - **Trust Policy**: Must explicitly allow `lambda.amazonaws.com` to assume the role.
  - **Permissions Policy**: An inline or managed policy attached to the role that grants specific actions (e.g., `s3:GetObject`, `logs:CreateLogGroup`).
- **Resource-Based Policy**: Attached directly to the function, this policy defines who or what can invoke the Lambda. For example, S3 needs a resource-based policy to "knock on the door" and start your function

```hcl
# 1. The Execution Role (Trust Policy)
resource "aws_iam_role" "lambda_exec_role" {
  name = "my_lambda_execution_role"

  # Trust policy: Allows Lambda service to use this role
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

# 2. Permissions Policy (S3 & Logs)
resource "aws_iam_policy" "lambda_s3_logging_policy" {
  name        = "lambda_s3_logging_policy"
  description = "Allows Lambda to log to CloudWatch and access a specific S3 bucket"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # Permission for CloudWatch Logs
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Effect   = "Allow"
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        # Permission for S3 Bucket Access
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Effect   = "Allow"
        Resource = [
          "arn:aws:s3:::my-secure-data-bucket",
          "arn:aws:s3:::my-secure-data-bucket/*"
        ]
      }
    ]
  })
}

# Attach the Policy to the Role
resource "aws_iam_role_policy_attachment" "lambda_attach" {
  role       = aws_iam_role.lambda_exec_role.name
  policy_arn = aws_iam_policy.lambda_s3_logging_policy.arn
}

# 3. Resource-Based Policy (S3 Trigger)
resource "aws_lambda_permission" "allow_s3_bucket" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.my_lambda.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = "arn:aws:s3:::my-secure-data-bucket"
}
```

### Lambda's Triggering Options

A trigger is an event source that initiates your function. Modern options include:  
- **S3 Event Notification**s: Triggers whenever an object is created, deleted, or modified (e.g., `s3:ObjectCreated:*`).
- **Message Queues/Streams**: Integration with **SQS** (now with 3x faster scaling in 2026), **DynamoDB Streams**, **Kinesis**, and **Amazon MSK**.
- **Synchronous Triggers**: **API Gateway** (now supporting response streaming for large payloads) and **Application Load Balancer**.
- **Schedules**: Using **Amazon EventBridge** to run functions at specific intervals (cron/rate).
- **Async Invocations**: In 2026, the maximum payload size for asynchronous triggers (like S3 or EventBridge) has increased to **1 MB**. 


### Asynchronous Invocations

**Asynchronous Invocations** are the preferred way to scale Lambda for non-blocking workloads, such as processing S3 uploads, handling SNS messages, or executing EventBridge rules.  

When you invoke a function asynchronously, the caller receives a 202 Accepted response immediately, and AWS places the event in an **internal managed queue**

#### Key Features in 2026

- **Increased Payload Size**: As of early 2026, the maximum payload size for async invocations is **1 MB** (up from 256 KB in previous years), allowing for larger event data without needing to store it in S3 first.
- **Managed Retry Logic**: AWS automatically retries failed async executions. By default, it retries two more times (total of 3 attempts) with exponential backoff.
- **Event Age Filtering**: You can configure the Maximum Event Age (up to 6 hours). If the event sits in the internal queue longer than this, it is discarded rather than executed, preventing "stale" processing.

#### Destinations (The modern DLQ)

While **Dead Letter Queues (DLQ)** still exist, **Lambda Destinations** are the standard in 2026 for handling execution results. You can route results to different services based on success or failure:
- **On Success**: Send a record of completion to **EventBridge** or **SQS**.
- **On Failure**: Send the full execution context and error stack trace to an **SQS queue** or **SNS topic** for debugging or automated recovery.

#### Error Handling & Idempotency

Because async invocations involve retries, your Lambda function must be **idempotent**.
If a function fails halfway through and retries, it should not create duplicate database entries or charge a customer twice.  

**Tip**: Use a unique ID from the event (like the S3 request ID or a custom UUID) to check if the work has already been completed before processing

#### Concurrency & Throttling

- If your async invocation triggers more functions than your **Reserved Concurrency** allows, the events stay in the internal queue for up to **6** hours while they wait for capacity.

- In 2026, AWS improved **Recursive Loop Detection**, which automatically stops an async Lambda if it detects it is triggering itself in a loop (e.g., S3 → Lambda → S3 → Lambda).

#### Async Triggers

- **S3**: "Object Created" events.
- **Amazon SNS**: Fan-out messaging.
- **EventBridge**: Scheduled events or cross-account bus events.
- **SES**: Triggered upon receiving an email.  

**Terraform Snippet for Destinations & Retries**:

```hcl
resource "aws_lambda_function_event_invoke_config" "example" {
  function_name                = aws_lambda_function.my_lambda.function_name
  maximum_event_age_in_seconds = 3600 # 1 hour
  maximum_retry_attempts       = 1

  destination_config {
    on_failure {
      destination = aws_sqs_queue.failures.arn
    }
    on_success {
      destination = aws_sns_topic.success_notifications.arn
    }
  }
}
```


## Organization Account

AWS Organizations has shifted the paradigm of identity management from local "IAM Users" to centralized **Workforce Identities** managed via AWS **IAM Identity Center** (the successor to Single Sign-On)

### The Hierarchy: Organization & Accounts

An AWS Organization consists of a **Management Account** and multiple **Member Accounts**. 

- **Management Account**: Handles billing, creates new accounts, and applies global security policies.
- **Member Accounts**: Where your actual workloads (Dev, Staging, Prod) live. In 2026, best practice dictates that these accounts should contain zero local IAM users. 

### IAM Identities

The way you manage users has changed to support multi-account security:
- **Workforce Identities (The New Standard)**: Instead of creating a "User" inside every account, you create one identity in the **IAM Identity Center** (attached to the **Management Account** or a delegated admin).
- **Permission Sets**: These act like "Group Templates." You define a policy (e.g., `AdministratorAccess`) and assign it to a user or group across multiple accounts.
- **IAM Roles (The "How")**: When a user logs in, they "assume" a role in the target account. This role is temporary and uses short-lived credentials, which is significantly more secure than the old-fashioned permanent **Access Keys**. 

### Roles vs. Users vs. Groups

| Resource | Role in 2026 | Usage in an Organization |
|----------|--------------|--------------------------|
| **IAM Users** | Legacy / Discouraged | Only used for legacy automated systems that cannot support IAM Roles. Never used for human login. |
| **IAM Groups** | Legacy | Replaced by Identity Center Groups. You group users (e.g., "DevOps Team") and assign them to specific accounts with specific Permission Sets. |
| **IAM Roles** | Mandatory | The primary way humans and services (EC2, Lambda) access resources. Cross-account access is handled entirely via roles. |

### Security Governance: Service Control Policies (SCPs)

In an Organization, the "Firewall for IAM" is the Service Control Policy (SCP).  

- **What they do**: SCPs set the maximum permissions for all identities (including the Root user) in a member account.
- **Example**: You can apply an SCP to a "Production" account that prevents anyone—even an Administrator—from deleting S3 buckets or disabling CloudTrail.
- **Logic**: Even if a local IAM Role has `FullAccess`, if the Organization SCP denies an action, the action is blocked.  

### Management Best Practices

- **Delegate Administration**: In 2026, you should delegate "Identity Management" and "Security" to a dedicated Security account so you rarely have to log into the sensitive Management Account.
- **Abuse IAM Roles for Services**: Use IAM Roles for Service Accounts (IRSA) or EKS Pod Identities if running Kubernetes, rather than passing secret keys to pods.
- **Root Account Security**: Use Phishing-resistant MFA (FIDO2/Security Keys) for the Management Account's root user and keep it locked in a physical safe. 

### Multi-Account Setup

| Account Type | Services Delegated to It | Purpose |
|-------------|--------------------------|---------|
| Management | Billing, Organizations, SCPs | Minimal use only. High-security vault. |
| Security | Security Hub, GuardDuty, Amazon Detective, Inspector | Centralized threat detection and response. |
| Identity | IAM Identity Center (Successor to SSO) | Managing user access and workforce identities. |
| Log Archive | CloudTrail, S3 (Centralized logs) | Immutable storage for all organization-wide logs. |

