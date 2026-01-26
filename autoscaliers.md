# AWS Autoscaler and Karpenter

## Comparison Summary

| Feature | Cluster Autoscaler (ASG-based) | Karpenter (ASG-less) |
|---------|--------------------------------|----------------------|
| **Orchestration** | Relies on predefined ASGs | Direct EC2 API calls |
| **Instance Choice** | Limited to the ASG's configuration | Dynamically selects any valid EC2 type |
| **Speed** | Slower (minutes) | Very fast (seconds) |
| **Complexity** | High (must manage many ASGs) | Low (group-less provisioning) |

## AWS Autoscaler Groups

In AWS, Kubernetes autoscaling at the infrastructure level is traditionally managed by **AWS Auto Scaling Groups (ASGs)**. These groups serve as the bridge between Kubernetes' need for compute resources and the underlying EC2 instances.

### AWS Auto Scaling Groups (ASGs) and Kubernetes

In an Amazon EKS environment, an ASG contains a collection of EC2 instances treated as a single logical unit.  

- **Role**: While ASGs can scale independently based on EC2 metrics (like CPU usage), they lack awareness of Kubernetes pod requirements. In EKS, the ASG acts as a "dumb" worker pool that launches or terminates nodes only when instructed by an external controller.
- **Managed Node Groups**: These are EKS-specific abstractions that automatically create and manage ASGs for you, handling tasks like node draining and updates gracefully.  

### The Link: Cluster Autoscaler (CA)  

The **Cluster Autoscaler** is the standard tool that connects Kubernetes to ASGs.  

- **How it works**: It monitors the cluster for "pending" pods that cannot be scheduled due to lack of resources. It then increases the **Desired Capacity** of the specific ASG that can satisfy those pods' needs.
- **Discovery**: CA uses `tags` on the ASG (e.g., `k8s.io/cluster-autoscaler/enabled`) to identify which groups it is allowed to manage.
- **Scale-Down**: When nodes are underutilized for a period (default ~10 minutes), CA gracefully drains pods and decreases the ASG size to save costs.  

### Cluster Autoscaler Installation

#### IAM Configuration
The CA requires permissions to modify EC2 Auto Scaling Groups (ASGs).

##### 1. Create IAM Policy

1. Create a policy with necessary permissions for the CA to interact with ASGs, including actions like `DescribeAutoScalingGroups`, `SetDesiredCapacity`, and `TerminateInstanceInAutoScalingGroup`. You can find a complete list of required permissions in the referenced documents.

  - **IAM Role Trust Policy (Trust Relationship)**  
This JSON document allows a specific Kubernetes ServiceAccount (cluster-autoscaler) in a specific namespace (kube-system) to assume the IAM role using the cluster's OIDC provider

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER_URL>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER_URL>:sub": "system:serviceaccount:kube-system:cluster-autoscaler",
          "<OIDC_PROVIDER_URL>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

  - **IAM Permission Policy**  
This policy provides the minimum permissions required for the Cluster Autoscaler to manage your Auto Scaling Groups (ASGs).  

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions",
                "ec2:DescribeInstanceTypes"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

2. Create **IAM Role for Service Account (IRSA)**: Create an IAM role and link it to a Kubernetes ServiceAccount in the kube-system namespace. You can use `eksctl` with a command similar to the one provided in the original response, making sure to replace placeholder values with your cluster details and the ARN of the policy created in the previous step.  

  - **Kubernetes ServiceAccount Annotation**  
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    ://eks.amazonaws.com: arn:aws:iam::<ACCOUNT_ID>:role/<YOUR_ROLE_NAME>
```

  - **Quick Setup with `eksctl`**
If you use `eksctl`, these JSON steps are handled automatically with one command  

```bash
eksctl create iamserviceaccount \
  --cluster=<YOUR_CLUSTER_NAME> \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --role-name=AmazonEKSClusterAutoscalerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/<YOUR_POLICY_NAME> \
  --approve
```

3. Tag your ASGs: To enable **Auto-Discovery**, tag your EC2 Auto Scaling Groups  `k8s.io/cluster-autoscaler/enabled: true` and `k8s.io/cluster-autoscaler/<YOUR-CLUSTER-NAME>: owned`

```hcl
variable "cluster_name" {
  description = "Name of your EKS Cluster"
  type        = string
  default     = "my-eks-cluster"
}

resource "aws_autoscaling_group" "eks_nodes" {
  name                = "eks-worker-nodes"
  max_size            = 10
  min_size            = 2
  desired_capacity    = 2
  vpc_zone_identifier = ["subnet-12345", "subnet-67890"]

  launch_template {
    id      = aws_launch_template.eks_lt.id
    version = "$Latest"
  }

  # Cluster Autoscaler Tags
  tag {
    key                 = "k8s.io/cluster-autoscaler/enabled"
    value               = "true"
    propagate_at_launch = true
  }

  tag {
    key                 = "k8s.io/cluster-autoscaler/${var.cluster_name}"
    value               = "owned"
    propagate_at_launch = true
  }

  # Lifecycle to prevent Terraform from fighting with Cluster Autoscaler
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

## Karpenter

**Karpenter** is a high-performance autoscaler built by AWS.  

- **Bypassing ASGs**: Unlike CA, Karpenter does not use Auto Scaling Groups. It talks directly to the **EC2 Fleet API** to provision the exact instance types needed for pending pods.
- **Speed**: Karpenter can provision nodes in seconds, whereas CA/ASG setups often take several minutes due to the multiple layers of orchestration  

### Installation
  Use [Karpenter Documentation](https://karpenter.sh/docs/getting-started/migrating-from-cas/).

```bash
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

### Connection between EC2NodeClass, NodePool and Deployment

In Kubernetes with **Karpenter**, the connection between these three components works like a chain of references and labels:

- **Deployment → NodePool**: The Deployment uses nodeSelector or tolerations to signal that it wants a specific type of node.  

- **NodePool → EC2NodeClass**: The NodePool has a nodeClassRef that points to the specific AWS infrastructure settings (Subnets, AMIs).  

- **Karpenter Logic**: When the Deployment has "Pending" pods, Karpenter finds the NodePool that matches the pod's requirements and uses the linked EC2NodeClass to launch the EC2.


### EC2NodeClass (Infrastructure Settings)

Define your AMI family, node role, and network selectors.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: extreme-performance-class # The "ID" for these AWS settings
spec:
  amiFamily: AL2023
  role: KarpenterNodeRole-my-cluster
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
```

### NodePool (Scheduling Logic)
Specify node requirements (like capacity type and architecture) and disruption settings

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    metadata:
      labels:
        intent: intensive-processing # Custom label to connect to Deployment
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: extreme-performance-class # Matches the metadata.name above
      requirements:
        # 1. SPECIFIC INSTANCE TYPES LIST
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["t3.medium", "m5.large", "m5.xlarge", "c5.large"]
        
        # 2. ARCHITECTURE (Optional but recommended)
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64", "arm64"]
        
        # 3. CAPACITY TYPE (Spot is cheaper, On-Demand is more stable)
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand", "spot"]

  # Disruption settings (how Karpenter manages node lifecycle)
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

### Deployment matches the NodePool name

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: heavy-worker
spec:
  replicas: 0 # Scale this up to trigger Karpenter
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      # THE CONNECTION POINT
      nodeSelector:
        intent: intensive-processing # Matches the label in the NodePool template
      containers:
      - name: processor
        image: busybox
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
```

### Other Scaling Types in Kubernetes

To achieve full automation, infrastructure autoscaling is usually combined with workload autoscaling:  

- **Horizontal Pod Autoscaler (HPA)**: Increases the number of pod replicas based on CPU/RAM usage.  

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```  

- **Vertical Pod Autoscaler (VPA)**: Adjusts the CPU/RAM requests of existing pods to match actual usage.  

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto" # "Auto" restarts pods to apply new limits; "Off" only provides recommendations
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 4Gi
```

- **Event-Driven (KEDA)**: Scales pods based on external events, such as messages in an SQS queue  

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-sqs-queue-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: worker-deployment # The deployment to scale
  minReplicaCount: 0        # Scale to zero when queue is empty
  maxReplicaCount: 20
  pollingInterval: 30       # Check SQS every 30 seconds
  cooldownPeriod:  300      # Wait 5 mins before scaling back to zero
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com
      queueLength: "5"      # Target 1 pod for every 5 messages
      awsRegion: "us-east-1"
      identityOwner: pod    # Use EKS Pod Identity / IRSA
```

#### KEDA Installation Procedure

##### Terraform: IAM Role for KEDA
This configuration creates the IAM role, attaches a trust policy for your cluster's OIDC provider, and adds a policy allowing KEDA to read AWS SQS metrics (a common use case)

```hcl
# 1. Retrieve information about your existing EKS cluster
data "aws_eks_cluster" "this" {
  name = "my-eks-cluster"
}

# 2. Extract the OIDC Provider URL (stripping the https://)
locals {
  oidc_url = replace(data.aws_eks_cluster.this.identity[0].oidc[0].issuer, "https://", "")
}

# 3. Create the IAM Role with the Trust Relationship
resource "aws_iam_role" "keda_operator" {
  name = "keda-operator-irsa-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${local.oidc_url}"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            # Limits the role to only be assumed by the KEDA ServiceAccount in the keda namespace
            "${local.oidc_url}:sub" = "system:serviceaccount:keda:keda-operator",
            "${local.oidc_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

# 4. Create a Policy giving KEDA access to monitor resources (e.g., SQS)
resource "aws_iam_policy" "keda_monitoring" {
  name        = "KEDAMonitoringPolicy"
  description = "Allows KEDA to read SQS and Cloudwatch metrics"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "sqs:GetQueueAttributes",
          "sqs:GetQueueUrl",
          "cloudwatch:GetMetricData",
          "cloudwatch:GetMetricStatistics"
        ]
        Resource = "*"
      }
    ]
  })
}

# 5. Attach the Policy to the Role
resource "aws_iam_role_policy_attachment" "keda_attach" {
  role       = aws_iam_role.keda_operator.name
  policy_arn = aws_iam_policy.keda_monitoring.arn
}

# Output the ARN to use in your Helm command
output "keda_role_arn" {
  value = aws_iam_role.keda_operator.arn
}

data "aws_caller_identity" "current" {}
```

##### Helm install
Once you run terraform apply, take the outputted `keda_role_arn` and pass it to your Helm installation:  

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

KEDA_ROLE_ARN=$(terraform output -raw keda_role_arn)

helm upgrade -i keda kedacore/keda \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=$KEDA_ROLE_ARN \
  --namespace keda --create-namespace 
```
