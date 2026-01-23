# AWS best practice for exposing a Cell-Based Architecture

AWS best practice for exposing a Cell-Based Architecture (3 independent EKS clusters across 3 AZs) with a round-robin distribution is to use an **Infrastructure-Managed Application Load Balancer (ALB)** combined with the **AWS Load Balancer Controller** using `TargetGroupBinding`.
This approach decouples the lifecycle of the entry point (the ALB) from the individual clusters.

## The Architecture Pattern: "Shared ALB"

Instead of each cluster creating its own Load Balancer, you create one central ALB (using Terraform or CDK) that exists outside the EKS clusters.
- **The ALB**: Exists as a regional resource spanning all 3 AZs.
- **The Target Group**: You create one regional Target Group.
- **The Registration**: Each of the 3 EKS clusters registers its local Pod IPs into that same central Target Group.

## Implementation Steps (2026 Best Practice)

### A. Central Infrastructure
Create one **Application Load Balancer** and one **Target Group** with `target-type: ip`. Direct Pod-IP routing is mandatory for performance and to bypass the "double-hop" latency of NodePorts.

### B. Cluster-Level Configuration
In each of your 3 EKS clusters, install the **AWS Load Balancer Controller** and deploy a `TargetGroupBinding` custom resource:

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: cell-api-binding
spec:
  serviceRef:
    name: api-service # Your local service name
    port: 8080
  targetGroupARN: arn:aws:elasticloadbalancing:us-east-1:1234567890:targetgroup/cell-tg/abcdef12345
```

### C. Service-Level Configuration

You **do not use** a standard `Ingress` manifest for this setup. Instead, you deploy a standard `Service` and a `TargetGroupBinding`

#### 1. Service Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080 # Port your app listens on
```

#### 2. TargetGroupBinding (The Bridge) Manifest

```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: my-app-tgb
spec:
  serviceRef:
    name: my-app-service # Must match Service name above
    port: 80
  targetGroupARN: <YOUR_TARGET_GROUP_ARN_FROM_TERRAFORM>
  targetType: ip # Ensures direct routing to Pod IPs
  # IMPORTANT for 3 clusters sharing one Target Group
  networking:
    vpcID: <VPC_ID_OF_CLUSTER>
```

## Achieving Round-Robin

AWS ALBs use a round-robin algorithm by default at the Target Group level. Because all pods from Cluster A, Cluster B, and Cluster C are registered as individual IP targets in the same group, the ALB will distribute incoming requests evenly across all pods in all cells.

## Why this is the "Expert" Choice

- **Health-Check Driven Failover**: If "Cell A" (Cluster A) fails or its AZ has an outage, the ALB health checks will detect the failing Pod IPs and immediately stop sending traffic to that cell. The remaining two cells will handle 100% of the traffic automatically.
- **Weighted Routing (Advanced)**: While you asked for round-robin, this setup allows you to easily switch to Weighted Routing. For example, if you are upgrading Cluster B, you can set its weight to 10% to "canary" the new version before giving it full traffic.
- **Zero-Downtime Cluster Migration**: You can bring up a "Cell 4," add it to the Target Group, and then decommission

## Critical Caveat: The "Sticky Session" Trap

If your application requires **Session Stickiness** (Cookie-based affinity), the ALB will try to send the same user to the same Pod. In a Cell-Based Architecture, if that Pod dies and the user is moved to a different Cell, they will lose their session unless you are using a **Global Redis Cache** shared by all three clusters.

## Summary of Component Jobs

| Component | Job |
|-----------|-----|
| **Route 53** | Points your domain to the ALB DNS name. |
| **Shared ALB** | The single entry point; performs the Round-Robin logic. |
| **Target Group** | A unified pool of all healthy Pod IPs from all 3 clusters. |
| **LB Controller** | Keeps the ALB updated as Pods scale up/down in each cluster. |

## Terraform Examples

### Application Load Balancer

```hcl
# Security Group for the ALB
resource "aws_security_group" "alb_sg" {
  name        = "shared-infrastructure-alb-sg"
  description = "Allow HTTP/HTTPS inbound"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# The Load Balancer
resource "aws_lb" "shared_alb" {
  name               = "cell-infrastructure-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  # Spanning 3 Public Subnets (one per AZ)
  subnets            = var.public_subnet_ids 

  enable_deletion_protection = false # Set to true for production
}
```

### Target Group

```hcl
resource "aws_lb_target_group" "app_targets" {
  name        = "cell-app-target-group"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip" # MANDATORY for Direct Pod-IP routing

  health_check {
    enabled             = true
    path                = "/health" # Must match your FastAPI health endpoint
    port                = "traffic-port"
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200"
  }
}
```
### The Listener

This links the ALB to the Target Group

```hcl
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.shared_alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_targets.arn
  }
}
```

### The output (integration value)

Once Terraform applies this, you will have a **Target Group ARN**. You provide this ARN to your GitHub Actions or Helm Values so that each of your 3 EKS clusters can deploy its `TargetGroupBinding`.

```hcl
output "target_group_arn" {
  value = aws_lb_target_group.app_targets.arn
  description = "Use this ARN in your Kubernetes TargetGroupBinding manifests"
}
```

## Global Redis Cache
[AWS Blog: Valkey and Redis Locality](https://aws.amazon.com/blogs/database/reduce-your-amazon-elasticache-costs-by-up-to-60-with-valkey-and-cudos)

A **Global Redis Cache** (specifically the **Amazon ElastiCache Global Datastore**) is a managed architectural pattern used to sync data across different AWS Regions or independent "Cell" clusters.

**Key Technical Comparisons (Valkey vs. Redis OSS)**  

| Feature | Valkey (v8.0/8.1) | Redis OSS (v7.2 and prior) |
|---------|-------------------|---------------------------|
| **Throughput** | Up to 1.2 million queries per second (QPS) on a single node; ~230% higher than predecessor. | Typically lower; Redis 8.0 enhancements claim ~112% improvement but often lag behind Valkey's architecture in high-concurrency. |
| **Threading Model** | Concurrent main and I/O threads; I/O threads handle parsing, writing, and memory deallocation. | Multi-threaded I/O (since v6.0) offloads socket I/O but keeps data manipulation single-threaded. |
| **Memory Efficiency** | ~20–40% lower memory usage due to a new hash table design inspired by "Swiss tables". | Standard dictionary implementation with higher overhead per key-value entry. |
| **Latency** | Up to 70% lower latency. P99 latency remains stable even under aggressive scaling. | Can experience higher latency spikes during cluster leadership changes or scaling. |
| **Pricing on AWS** | 20% lower for node-based clusters; 33% lower for ElastiCache Serverless. | Standard managed service pricing. |

### 1. Key Implementation Patterns

Depending on whether your cells are in the same AWS Region or different ones, the implementation differs:

#### A. Cross-Zone (Single Region) - Shared ElastiCache

Since your 3 EKS clusters are in different AZs within the same region, you don't need the "Global Datastore" feature. Instead, you use a Multi-AZ ElastiCache Cluster. 

- **The Setup**: You deploy one ElastiCache for Redis cluster with Multi-AZ enabled.
- **Connectivity**: All 3 EKS clusters connect to the same Primary Endpoint for writes and Reader Endpoints for reads.
- **Networking**: Use VPC Peering or Transit Gateway if your clusters are in separate VPCs, or a shared Subnet Group if they are in the same VPC. 

#### B. Cross-Region - Global Datastore

If your cells are in different regions (e.g., US-East-1 and EU-West-1), you use Global Datastore. 
- **Architecture**: One Primary Cluster (Active) accepts writes. Up to two Secondary Clusters (Passive) in other regions provide low-latency local reads.
- **Replication**: AWS handles the cross-region replication automatically with sub-second latency.
- **Failover**: If the Primary Region fails, you can manually promote a Secondary Cluster to Primary status. 

### Python code

```python
import redis

# Use the cluster's primary endpoint
r = redis.Redis(
    host='my-cache-primary.abc123.use1.cache.amazonaws.com',
    port=6379,
    decode_responses=True,
    ssl=True # Always use TLS in 2026
)

# Shared session storage
r.set('session:user_123', '{"user": "kuku", "role": "admin"}', ex=3600)
```


### Best Practices

| Feature | Best Practice |
|---------|---------------|
| **Engine Choice** | Use Valkey 7.2+ or Redis OSS 7.1+. Valkey is the 2026 community standard for high performance. |
| **Serverless vs. Node** | Use ElastiCache Serverless for PoCs or variable loads. It auto-scales and eliminates the need to manage shards manually. |
| **Persistence** | Use Data Tiering (r6gd/r7g nodes) if your cache is large (>500GB). It stores less-frequently used data on NVMe SSDs to save 60% on costs. |
| **Security** | Always enable Encryption in Transit (TLS) and Encryption at Rest. Use IAM Authentication instead of static passwords. |

## Zonal Read Isolation

To ensure each cell reads only from its local zone, bypass the shared endpoint and use Direct Node Addressing.

### A: Identify Node Endpoints

Instead of using the cluster's general Reader Endpoint, you find the unique DNS endpoint for each individual replica node.
- **Node 1 (AZ-1)**: `my-cache-001.abc.use1.cache.amazonaws.com`
- **Node 2 (AZ-2)**: `my-cache-002.abc.use1.cache.amazonaws.com`
- **Node 3 (AZ-3)**: `my-cache-003.abc.use1.cache.amazonaws.com`

### B: Configuration Injection per EKS Cluster

In your CI/CD pipeline, inject the specific node endpoint as an Environment Variable unique to each EKS cluster:

- **Cluster 1 (AZ-1) Config**: `REDIS_READER_HOST="my-cache-001..."`
- **Cluster 2 (AZ-2) Config**: `REDIS_READER_HOST="my-cache-002..."`
- **Cluster 3 (AZ-3) Config**: `REDIS_READER_HOST="my-cache-003..."`

### Terraform example

```hcl
# 1. Subnet Group spanning 3 AZs
resource "aws_elasticache_subnet_group" "redis_subnets" {
  name       = "cell-redis-subnets"
  subnet_ids = var.private_subnet_ids # Ensure these IDs cover 3 different AZs
}

# 2. Security Group allowing access from all 3 EKS clusters
resource "aws_security_group" "redis_sg" {
  name   = "cell-redis-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 6379
    to_port     = 6379
    protocol    = "tcp"
    cidr_blocks = var.eks_cluster_cidr_blocks # All 3 cluster IP ranges
  }
}

# 3. The Replication Group (The "Cluster")
resource "aws_elasticache_replication_group" "cell_redis" {
  replication_group_id          = "cell-cache"
  description                   = "Redis for cell-based architecture with zonal isolation"
  node_type                     = "cache.t4g.medium" # Graviton3 is the 2026 value standard
  port                          = 6379
  parameter_group_name          = "default.redis7"
  automatic_failover_enabled    = true
  multi_az_enabled              = true
  num_cache_clusters            = 3 # 1 Primary + 2 Replicas
  subnet_group_name             = aws_elasticache_subnet_group.redis_subnets.name
  security_group_ids            = [aws_security_group.redis_sg.id]
  
  # Ensure nodes are spread across your specific AZs
  preferred_cache_cluster_azs = ["us-east-1a", "us-east-1b", "us-east-1c"]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true # Mandatory for 2026 security compliance
}

# This output provides the specific node endpoints
output "redis_node_endpoints" {
  # This returns a list of maps containing 'address' and 'availability_zone'
  value = [
    for member in aws_elasticache_replication_group.cell_redis.member_clusters : {
      address = member
      # You can use these to map your EKS ENV vars in CI/CD
    }
  ]
}
```
### Output example:

```hcl
redis_node_endpoints = [
  {
    "address" = "cell-cache-001.abc123.0001.use1.cache.amazonaws.com"
    "az"      = "us-east-1a"
  },
  {
    "address" = "cell-cache-002.abc123.0002.use1.cache.amazonaws.com"
    "az"      = "us-east-1b"
  },
  {
    "address" = "cell-cache-003.abc123.0003.use1.cache.amazonaws.com"
    "az"      = "us-east-1c"
  }
]
```
