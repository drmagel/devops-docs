# Kubernetes tutorial. Short reference.

## Kubernetes main processes

### 1. Control Plane Processes (The Brain)

These usually run on your master nodes and manage the state of the cluster.
- **kube-apiserver**: The central hub. All components (and you, via kubectl) talk to this. It validates and configures data for objects like pods and services.
- **etcd**: The cluster's database. It is a key-value store that holds the "source of truth" for the entire cluster configuration.
- **kube-scheduler**: The matchmaker. It watches for newly created pods with no assigned node and selects the best worker node for them based on resource availability.
- **kube-controller-manager**: The regulator. It runs "control loops" that watch the state of the cluster and make changes to move the current state toward the desired state (e.g., if a pod dies, it starts a new one).
- **cloud-controller-manager**: The cloud link. It interacts with AWS/GCP/Azure to manage external resources like Load Balancers or EBS volumes.

### 2. Worker Node Processes (The Muscle)

These run on every machine where your application containers are actually executed.
- **kubelet**: The captain of the node. It is the primary agent that ensures containers are running in a pod. It takes instructions from the API server and manages the local container runtime.
- **kube-proxy**: The network manager. It maintains network rules on the node, allowing pods to communicate with each other and the outside world via Services.
- **Container Runtime (e.g., `containerd` or `CRI-O`)**: The engine. This is the software that actually pulls images from a registry and starts/stops the containers.

### 3. Essential Helper Processes (Standard Add-ons)

While not "core" binaries, no 2026 cluster functions without these running as pods:
- **CoreDNS**: Handles internal naming (so Service A can find Service B by name).
- **Metrics Server**: Collects CPU/RAM usage so the **Horizontal Pod Autoscaler (HPA)** knows when to scale your workers.

### Summary Table

| Process | Location | Primary Job |
|---------|----------|-------------|
| **API Server** | Control Plane | Communication & Authentication |
| **etcd** | Control Plane | Persistent State Storage |
| **scheduler** | Control Plane | Scheduling Pods to Nodes |
| **kubelet** | Node | Executing & Monitoring Containers |
| **kube-proxy** | Node | Networking & Service Load Balancing |

## Essential AWS EKS Plugins

### 1. Core Networking Add-ons

These three are automatically provisioned with every EKS cluster and are indispensable for basic functionality: 
- **Amazon VPC CNI**: Integrates Pod networking directly with your AWS VPC, assigning each Pod a real VPC IP address. In 2026, it is the primary tool for implementing Kubernetes Network Policies natively.
- **CoreDNS**: Provides naming and service discovery within the cluster, allowing Pods to communicate using hostnames instead of IP addresses.
- **Kube-proxy**: Manages the network rules on each node to enable the Kubernetes Service abstraction and load balancing.

### 2. Storage and Infrastructure Plugins

Essential for stateful applications and cloud-native scaling:
- **AWS Load Balancer Controller**: Manages **AWS Elastic Load Balancers (ALB/NLB)**. It is critical for exposing services to the internet via Ingress (ALB) or Service type: LoadBalancer (NLB).
- **Amazon EBS CSI Driver**: Required for Pods to mount AWS EBS block storage volumes. It is now a mandatory separate installation if you need persistent volumes.
- **Amazon EFS CSI Driver**: Enables shared, multi-AZ file storage for Pods using Amazon EFS.
- **Karpenter**: Karpenter has superseded the legacy Cluster Autoscaler as the standard for just-in-time node provisioning, offering faster scaling and lower costs. 

### 3. Security and Observability Add-ons

- **EKS Pod Identity Agent**: The modern (2026) standard for granting Pods IAM permissions, replacing the more complex IRSA (IAM Roles for Service Accounts) for most use cases.
- **Amazon GuardDuty Agent**: Provides runtime security monitoring to detect threats like crypto-mining or unauthorized access attempts within your cluster.
- **ADOT (AWS Distro for OpenTelemetry)**: The preferred way to collect metrics and traces for Amazon Managed Prometheus and CloudWatch.
- **External Secrets Operator**: Although an open-source tool, it is essential in 2026 for securely syncing AWS Secrets Manager data into Kubernetes

### 4. Community Add-ons Catalog

AWS now provides a unified management experience for popular open-source tools. You can install these directly from the EKS console with AWS-validated images:  
- **Metrics Server**: Essential for **HPA** (Horizontal Pod Autoscaling) and **KEDA** (Kubernetes Event-driven Autoscaling).
- **Cert-Manager**: Automates the management and issuance of TLS certificates.
- **External-DNS**: Automatically updates your AWS Route 53 records based on Kubernetes Ingress/Service changes.

#### Metrics-Based Autoscalers:

| Component | Job | Scaling Trigger |
|-----------|-----|----------------|
| **HPA** | Scales Pod count | Resource Usage (CPU/RAM) |
| **Metrics Server** | Provides the data | Aggregate Node/Pod usage |
| **KEDA** | Advanced scaling | External events (NATS, Redis, DB count) |
| **VPA** | Scales Pod size | Adjusts CPU/RAM limits for a single pod |
| **Karpenter** | Scales Nodes | Pending pods that need more hardware |

**Architect's Note**:

Use:  
- **HPA**: for API-Service (scaling on CPU)
- **KEDA**: for Workers (scaling on the number of pending tasks in your DB or NATS). Kubernetes HPA Documentation KEDA Official Site


## AWS EKS configurations

### 1. Multi-Zonal (Single Cluster) Topology

This is the standard starting point for production. A single EKS control plane manages worker nodes spread across at least three Availability Zones (AZs). 
- **Best Practice**: Deploy nodes in private subnets and only expose services via an AWS Load Balancer.
- **Resilience**: Use topologySpreadConstraints to ensure pods are evenly distributed across AZs to survive a zone outage.
- **Compute**: Use Karpenter for node autoscaling, which is zone-aware and faster than the legacy Cluster Autoscaler.
- **Limitations**: While highly available, a single cluster remains a single "blast radius" for human error or cluster-wide configuration failures. 

### 2. Multi-Cluster Topology (Per Environment)

To maintain stability, the most common 2026 recommendation is a separate cluster for each environment (e.g., Development, Staging, and Production) rather than using namespaces for environment isolation. 
- **Isolation**: This prevents unstable code in development from accidentally consuming all resources or causing failures that impact production.
- **Staged Rollouts**: It allows you to test Kubernetes version upgrades and system-wide changes (like new security policies) in lower environments before applying them to production. 

### 3. Multi-Account/Multi-Region Topology

For organizations with extreme reliability or regulatory requirements, clusters are distributed across multiple AWS accounts or regions. 
- **Account-level Isolation**: Use a dedicated AWS account per cluster to isolate security boundaries and service quotas.
- **Failover**: Multi-cluster/multi-region setups enable blue-green cluster upgrades and advanced disaster recovery.
- **Secrets**: The standard is to use **AWS Secrets Manager** with the **External Secrets Operator** to sync secrets from AWS securely across these clusters. 

### 4. Networking Best Practices for Topologies

IPv6 Adoption: Move to IPv6 clusters if you face IPv4 address exhaustion in your VPC.
- **Cilium**: For advanced security and performance, consider using Cilium as your CNI for eBPF-based networking and security.
- **Private Endpoints**: Always use private cluster endpoints to ensure your Kubernetes API is not exposed to the public internet.

## Multi-Zonal Topology

### Single Cluster Topology Limitations

#### 1. Zonal Data Gravity (The "Stuck Pod" Problem)

The most common failure in Multi-Zonal clusters involves Persistent Storage.

- **EBS Locking**: Standard AWS EBS volumes are zonal. A volume created in us-east-1a cannot be attached to a node in us-east-1b.
- **The Limitation**: If a node in Zone A fails and Kubernetes tries to restart your PostgreSQL pod in Zone B, the pod will stay in ContainerCreating or Pending forever because it cannot "reach" its disk across the zone boundary.
- **Best Practice**: You must use **Volume Binding Mode: `WaitForFirstConsumer`** in your StorageClass and implement **Topology Spread Constraints** to keep pods "pinned" to their data's zone.

#### 2. Inter-AZ Data Transfer Costs

AWS charges for all data that leaves one AZ and enters another.
- **The Cost**: As of 2026, this is typically $0.01 per GB in each direction.
- **The Limitation**: If your API-Service (Zone A) frequently queries your Database (Zone B), or your Worker pods pull large images from a registry in a different zone, your "Data Transfer" bill can eventually exceed your compute costs.
- **Best Practice**: Enable **Service Topology** / **Locality-Aware Routing** to keep traffic within the same zone whenever possible.

#### 3. Increased Network Latency

- **The Delay**: Cross-AZ communication adds approximately **1ms** to **2ms** of round-trip latency.
- **The Limitation**: While negligible for a single request, "chatty" microservice architectures that make 10-20 internal calls to fulfill one user request will see a cumulative performance hit of 20-40ms, which is visible to the end user.

#### 4. The Single "Control Plane" Blast Radius

Even though the worker nodes are spread out, the **Kubernetes API Server (Control Plane)** is a single logical entity for that cluster.
- **The Limitation**: If a human operator applies a broken Network Policy, a buggy Admission Controller, or a corrupt Global ConfigMap, the entire cluster (across all zones) will fail simultaneously. Multi-zonal clusters protect against hardware failure, but they do not protect against human or configuration failure.

#### 5. Scheduling Complexity (Skew)

Kubernetes does not perfectly balance pods across zones by default.
- **The Limitation**: Without explicit configuration, the scheduler might place 80% of your pods in Zone A simply because that node was the first to scale up. If Zone A then has a power outage, your application loses 80% of its capacity instantly.
- **Best Practice**: Manually define `topologySpreadConstraints` in every deployment to force an even distribution.

#### 6. Node Group & Autoscaler Synchronization

- **The Limitation**: If using the legacy Cluster Autoscaler with a single Auto Scaling Group (ASG) spanning 3 zones, it may struggle to scale up. It might try to add a node in Zone A to satisfy a pod that requires a disk in Zone B, leading to a "scaling loop" where nodes appear but pods stay pending.
- **2026 Mitigation**: Use Karpenter, which is natively zone-aware and handles multi-zonal constraints much more intelligently than the legacy autoscaler.

#### 7. IPv4 Address Exhaustion

- **The Limitation**: In a Multi-AZ VPC, the Amazon VPC CNI assigns a real VPC IP address to every Pod. If you have many small pods across 3 zones, you can quickly run out of IPs in your subnets.
- **2026 Mitigation**: Use IPv6-based EKS clusters, which provide a virtually infinite address space and are the recommended 2026 standard for large-scale multi-zonal deployments.

#### 8. Load Balancer Distribution
- **The Limitation**: An AWS **Application Load Balancer (ALB)** may not distribute traffic evenly if the number of pods per zone is uneven.
- **The Fix**: You must ensure **Cross-Zone Load Balancing** is enabled on the Load Balancer, which can add a slight additional cost and latency.
- **AWS EKS Best Practices**: Reliability Kubernetes: Topology Spread Constraints

### Multiple Cluster Topology Limitations (Cell-Based Architecture)

| Feature | 1 Multi-AZ Cluster | 3 Independent Clusters |
|---------|-------------------|------------------------|
| **Control Plane Cost** | $73 / month | $219 / month |
| **Service Discovery** | Native (CoreDNS) | Requires Service Mesh |
| **Upgrade Risk** | Impact all 3 zones | Staggered (Safer) |
| **Isolation** | Soft (Namespaces) | Hard (Physical) |
| **Resource Sharing** | High Efficiency | Low Efficiency |



## AWS IAM <--> Kubernetes Service Account

### AWS Service Account for pods: `EKS Pod Identity`

The industry standard for connecting AWS permissions to a Kubernetes Pod is **EKS Pod Identity**. It is simpler and more scalable than the older **IRSA** (IAM Roles for Service Accounts) method because it eliminates the need to manage OIDC providers

#### 1. EKS Pod Identity Agent

```bash
aws eks create-addon --cluster-name <cluster-name> --addon-name eks-pod-identity-agent
```

#### 2. IAM Role with a Trust Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

#### 3. Kubernetes Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: dev
```

#### Terraform example

```hcl
# The Trust Policy allowing EKS Pod Identity to use this role
data "aws_iam_policy_document" "pod_identity_trust" {
  statement {
    actions = ["sts:AssumeRole", "sts:TagSession"]
    effect  = "Allow"
    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }
  }
}

# The IAM Role
resource "aws_iam_role" "pod_role" {
  name               = "eks-app-role"
  assume_role_policy = data.aws_iam_policy_document.pod_identity_trust.json
}

# Example Policy: Allow access to a specific S3 bucket
resource "aws_iam_role_policy" "app_permissions" {
  name   = "app-permissions"
  role   = aws_iam_role.pod_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Effect   = "Allow"
        Resource = ["arn:aws:s3:::my-app-data", "arn:aws:s3:::my-app-data/*"]
      }
    ]
  })
}

resource "aws_eks_pod_identity_association" "app_association" {
  cluster_name    = var.eks_cluster_name
  namespace       = "dev"
  service_account = "my-app-sa" # This name must match your K8s manifest
  role_arn        = aws_iam_role.pod_role.arn
}
```

#### Kubernetes Implementation

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa # Matches Terraform 'service_account'
  namespace: dev

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: dev
spec:
  template:
    spec:
      serviceAccountName: my-app-sa # Connects the Pod to the IAM Role
      containers:
      - name: my-app
        image: my-image:latest
```

### AWS Service Account for pods: The Legacy Way: IRSA (IAM Roles for Service Accounts)

This method uses an annotation directly on the Kubernetes ServiceAccount resource. 
- **Logic**: The EKS Pod Identity Webhook watches for a specific annotation. When it sees it, it injects the necessary AWS credentials into any pod using that service account.
- **Infrastructure Requirements**: Requires an OIDC Identity Provider configured in IAM for your specific cluster.

#### Terraform Examples

```hcl
## 1. OIDC Provider Configuration
# Fetch the TLS certificate data for the OIDC URL
# This is required to generate the "Thumbprint" for IAM
data "tls_certificate" "eks" {
  url = aws_eks_cluster.cluster.identity[0].oidc[0].issuer
}

# Create the IAM OIDC Identity Provider
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  # Use the hex thumbprint from the TLS data
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.cluster.identity[0].oidc[0].issuer
}

## 2. Creating an IRSA-Compatible IAM Role
# Define the Trust Policy using a Data Source for clarity
data "aws_iam_policy_document" "eks_oidc_assume_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      # The 'sub' field restricts the role to a specific ServiceAccount in a Namespace
      variable = "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub"
      values   = ["system:serviceaccount:dev:my-app-sa"]
    }

    principals {
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "eks_irsa_role" {
  name               = "eks-irsa-role"
  assume_role_policy = data.aws_iam_policy_document.eks_oidc_assume_role.json
}
```

#### Deployment Kubernetes annotation
```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>

```

#### Service Account Kubernetes annotation

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: dev
  annotations:
    # Key must be exactly this string
    ://eks.amazonaws.com: arn:aws:iam::1234567890:role/eks-irsa-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: dev
spec:
  template:
    spec:
      serviceAccountName: my-app-sa # The webhook sees the annotation and injects credentials
      containers:
      - name: worker
        image: my-registry/worker:latest
```

### Comparison
| Feature | EKS Pod Identity (New) | IRSA (Legacy) |
|---------|------------------------|---------------|
| **Connection Method** | EKS API Association | K8s Annotation (role-arn) |
| **IAM Trust** | `pods.eks.amazonaws.com`| Cluster-specific OIDC URL |
| **Reusability** | One role can serve multiple clusters | One role per OIDC provider (usually) |
| **Configuration** | Simpler (No OIDC wiring) | Complex (Manual OIDC setup) |

## AWS EKS Application and Network Load Balancers

Consequently, AWS is pushing users toward the native **AWS Load Balancer Controller** as the standard for both ALBs and NLBs

- **Replacing NGINX Logic**: Instead of running NGINX pods inside your cluster to handle routing, the AWS Load Balancer Controller offloads this logic to the cloud infrastructure (ALB/NLB). This reduces "hop" latency and management overhead.
- **Direct-to-Pod Routing**: The "new" way to connect NLBs to Kubernetes is using IP Targeting Mode. The NLB routes traffic directly to Pod IPs rather than NodePorts, bypassing the NGINX ingress layer and the kube-proxy entirely for better performance

**The Architecture: Gateway API**  
The industry-wide successor to the "Ingress" resource is the Kubernetes Gateway API, which AWS fully supports in 2026. 
| Feature | NGINX-based Ingress (Legacy) | AWS Load Balancer Controller (2026) |
|---------|------------------------------|-------------------------------------|
| **Routing Engine** | NGINX pods in the cluster | AWS ALB/NLB infrastructure |
| **Connection Method** | NLB → NGINX Pod → App Pod | NLB/ALB → App Pod (Direct) |
| **Maintenance** | Reaching EOL March 2026 | Fully managed by AWS |
| **API Standard** | Ingress API (Frozen) | Gateway API (Modern) |

**The "New" NLB Capability: ALB-as-a-Target**  
A key 2026 architectural pattern uses NLB in front of ALB. This is often used to replace custom NGINX setups that needed both static IPs and complex routing: 
- **NLB (Entry)**: Provides a single static IP or PrivateLink support.
- **ALB (Router)**: Acts as the target for the NLB, handling path-based routing and WAF security.
- **Kubernetes**: The controller automatically wires these together when you define a Gateway or Ingress resource.  

**Ingress To API Gateway Migration Procedure**  
Read [paper](https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress) and use [`ingress2gateway`](https://github.com/kubernetes-sigs/ingress2gateway) for automatic conversion.  

### NGINX Ingress Controller (Legacy)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: legacy-nginx-ingress
  annotations:
    # NGINX-specific logic
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

### AWS Load Balancer Controller (2026 Standard)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: modern-aws-ingress
  annotations:
    # Use native AWS ALB features
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Direct routing to Pod IPs (avoids NodePort/kube-proxy latency)
    alb.ingress.kubernetes.io/target-type: ip 
    # Force SSL redirection via ALB action
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # Grouping: Multiple Ingresses can share ONE ALB to save cost
    alb.ingress.kubernetes.io/group.name: "my-app-group"
spec:
  ingressClassName: alb
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80

```

[**Note**](https://repost.aws/questions/QU46JUVWI-Q8O7TMvgsE6HFA/how-to-rewrite-paths-in-amazon-application-load-balancer): Unfortunately, the Application Load Balancer (ALB) does not have a built-in feature to directly rewrite or remove path segments from incoming requests before forwarding them to the target group. The ALB primarily focuses on routing based on the existing path rather than modifying it.

However, there are a few alternative approaches you could consider:

- **Use CloudFront Functions**: Instead of Lambda@Edge, you can use CloudFront Functions to modify the request path. CloudFront Functions are lightweight and can perform simple path manipulations more efficiently than Lambda@Edge.

- **Modify your application logic**: If possible, adjust your backend application to handle requests with the "/hi" segment and process them as if the segment wasn't there.

- **Use a reverse proxy**: You could set up a reverse proxy (like NGINX) between your ALB and your application servers. This proxy can handle the path rewriting before forwarding requests to your application.

- **Content-based routing**: While this doesn't remove the path segment, you could use the ALB's content-based routing feature to route requests with "/hi" to the same target group as requests without it. This way, your application would receive both types of requests and could handle them accordingly.

### The Choice Between Ingress and TargetGroupBinding (TGB)

The choice between **Ingress** and **TargetGroupBinding (TGB)** when using **an existing Target Group** depends on where you want to manage your routing logic (Layer 7 rules like paths and host headers).

**Summary: Which one to use?**  

- Use **Ingress** if you want to manage routing rules inside Kubernetes using standard manifests.  
- Use **TargetGroupBinding** if you want to manage routing rules outside Kubernetes (e.g., via Terraform or AWS Console).

#### When to use Ingress (Internal Routing Management)

Use an **Ingress** resource when you want the **AWS Load Balancer Controller** to manage the **ALB Listeners** and **Rules**, even if the **Target Group** itself was created externally.  

- **Scenario**: You have a shared ALB managed by Terraform, but you want developers to define their own URL paths (e.g., /api or /app) in their Kubernetes manifests.  
- **How it works**: You point the Ingress backend to an existing Target Group ARN using an annotation.
- **Benefit**: Centralizes routing logic within the cluster while maintaining an external lifecycle for the physical ALB  

#### When to use TargetGroupBinding (External Routing Management)  

Use a **TargetGroupBinding** when you want to provision the entire load balancer infrastructure (ALB, Listeners, Rules, and Target Groups) completely outside of Kubernetes.  

- **Scenario**: Your Infrastructure-as-Code (Terraform) has already defined that example.com/login goes to TargetGroup-A. You just need Kubernetes to register your Pods into that specific TargetGroup-A.
- **How it works**: You create a TGB resource that maps a Kubernetes Service to the pre-existing Target Group ARN.
- **Benefit**: Decouples infrastructure from the application. The ALB and routing rules remain intact even if you delete your Kubernetes Ingress or Service resources  

#### Key Comparison Table  

| Feature | Ingress (with existing TG) | TargetGroupBinding |
|---------|---------------------------|-------------------|
| **Who manages ALB Rules?** | Kubernetes (via Ingress rules) | AWS Console/Terraform |
| **Who manages Pod Registration?** | ALB Controller (internally uses TGB) | ALB Controller (via explicit TGB) |
| **Primary Use Case** | Path/Host-based routing managed in K8s | Multi-cluster traffic or pure IaC management |
| **Protocol Support** | Only Layer 7 (ALB) | Layer 4 (NLB) and Layer 7 (ALB) |  

### Port and Hostname for TargetGroup

When creating an AWS Target Group for use with an Application Load Balancer (ALB) and Kubernetes, the Port and Hostname are configured at different stages and in different locations.  

#### Where to set the Port  

You set the port in the Target Group configuration. 
- **Port during creation**: When you create a Target Group (via AWS Console, CLI, or Terraform), you define a "default" port (e.g., port 80 or 8080).
- **Target-specific override**: When the **AWS Load Balancer Controller** registers your Pods as targets, it typically overrides the target group's default port to match the `containerPort` or Service port defined in your Kubernetes manifests.
- **Important**: For IP-mode (the modern standard for EKS), the Target Group must be set to `target-type: ip`  

#### Where to set the Hostname  

A Target Group itself does not have a hostname. Instead, the hostname is used to route traffic to the Target Group. You set this in the ALB Listener Rules.  
- **If using Ingress**: You define the `hostname` (e.g., `api.example.com`) in the `rules.host` section of your Kubernetes Ingress manifest. The AWS Load Balancer Controller will then automatically create a Host Header rule on the ALB to route traffic for that hostname to your Target Group.
- **If using TargetGroupBinding only**: You must manually (or via Terraform/CLI) create a `Listener Rule` on the AWS ALB. This rule specifies:
  - **Condition**: Host-header = `myapp.example.com`
  - **Action**: Forward to `Target Group ARN` 

## Pod Topology Spread Constraints

**Pod Topology Spread Constraints** (PTSC) allow you to control how Pods are distributed across your cluster's failure domains, such as nodes, availability zones (AZs), or regions. 
Unlike **Pod Anti-Affinity**, which is binary (a Pod either can or cannot be co-located with another), PTSC focuses on **distribution ratios** (skew) to ensure high availability and efficient resource utilization  

### Key Fields in a Constraint

A PTSC is defined in the spec.topologySpreadConstraints section of a Pod manifest using these primary fields:  
- **maxSkew**: The maximum allowed difference in the number of matching Pods between any two topology domains. For example, a `maxSkew: 1` ensures a nearly equal distribution.
- **topologyKey**: The node label that defines the domain (e.g., `kubernetes.io/hostname` for nodes or `topology.kubernetes.io/zone` for zones).
- **whenUnsatisfiable**: Determines the scheduler's behavior if the constraint cannot be met:
  - **DoNotSchedule**: The Pod remains pending.
  - **ScheduleAnyway**: The Pod is scheduled but the scheduler prioritizes nodes that minimize the skew.
- **labelSelector**: Identifies which Pods are counted when calculating the current skew.
- **minDomains**: (Added in recent versions) Ensures that Pods are spread across at least a certain number of domains, even if some domains have zero pods.
- **matchLabelKeys**: Allows the scheduler to distinguish between different revisions of a workload (e.g., during a rolling update) to maintain even distribution across all versions

### Example

To achieve a perfectly balanced distribution across both Availability Zones (AZs) and individual nodes, you must define two separate constraints within your Pod spec. 
Setting `maxSkew: 1` ensures that the difference in Pod count between any two domains (zones or nodes) never exceeds one. Using `whenUnsatisfiable: DoNotSchedule` makes these "hard" requirements that the scheduler must satisfy before placing a Pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: balanced-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      topologySpreadConstraints:
        # Constraint 1: Equal distribution across Availability Zones
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web-server
        
        # Constraint 2: Equal distribution across individual Nodes
        - maxSkew: 1
          topologyKey: "kubernetes.io/hostname"
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web-server
      containers:
      - name: nginx
        image: nginx:latest
```

### Example: Zone-Sticky StatefulSet

The **Storage Topology** is the standard way to ensure a StatefulSet Pod returns to its original Availability Zone (AZ).  
For a `StatefulSet Pod` to be "sticky" to its AZ upon recreation, the scheduler must align the Pod placement with the location of its **Persistent Volume (PV)**.  

This example uses a `maxSkew: 1` to initially spread Pods across zones, but relies on `volumeBindingMode: WaitForFirstConsumer` in the `StorageClass` to "pin" each Pod's PV (and thus the recreated Pod) to the specific zone where it was first scheduled. 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zone-aware-storage
provisioner: pd.csi.storage.gke.io # Example for GKE; use your provider's CSI
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer # ESSENTIAL: Pins volume to the Pod's initial zone

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sticky-az-app
spec:
  serviceName: "sticky-service"
  replicas: 3
  selector:
    matchLabels:
      app: stateful-db
  template:
    metadata:
      labels:
        app: stateful-db
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: DoNotSchedule # Strictly enforces AZ distribution on first deploy
          labelSelector:
            matchLabels:
              app: stateful-db
      containers:
      - name: database
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "zone-aware-storage"
      resources:
        requests:
          storage: 10Gi
```

### Routing: Ensuring Traffic Stays Local 
Even if Pods are in the same AZ, Kubernetes Services may distribute traffic across zones by default. To force local communication, enable these routing features:  
- **Topology Aware Routing (TAR)**: When enabled for a Service, the control plane adds hints to EndpointSlices. kube-proxy uses these hints to prioritize routing traffic to endpoints in the same zone as the source Pod.
- **Traffic Distribution (PreferClose)**: Introduced as a simpler, more predictable alternative to TAR in Kubernetes 1.30+ (GA in 1.33). It directs traffic to the nearest endpoints first (typically same-zone) based on zonal hints.  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-service
spec:
  selector:
    app: pool-a-app
  trafficDistribution: PreferClose  # Keeps traffic local to us-east-1a
  ports:
    - protocol: TCP
      port: 80
```

### For Kubernetes Version less than v1.33

For older cluster version (<v1.33), use Topology Aware Routing via an annotation:  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  # ... rest of service spec
```

### The Strategy: "Zone-Matched" DNS Discovery
Instead of using one global service, create a separate Service for the database in each Availability Zone. Your application then uses the Downward API to "discover" its own zone at runtime and connects to the service matching that zone name.

#### 1. Define Zone-Specific Database Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rds-us-east-1a
  namespace: default
spec:
  type: ExternalName
  externalName: db-instance.xxxx.us-east-1a.rds.amazonaws.com
---
apiVersion: v1
kind: Service
metadata:
  name: rds-us-east-1b
  namespace: default
spec:
  type: ExternalName
  externalName: db-instance.xxxx.us-east-1b.rds.amazonaws.com
---
apiVersion: v1
kind: Service
metadata:
  name: rds-us-east-1c
  namespace: default
spec:
  type: ExternalName
  externalName: db-instance.xxxx.us-east-1c.rds.amazonaws.com
```

#### 2. Inject the Zone into your Application Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
          # Injects the AZ name (e.g., "us-east-1a") into the pod
          - name: POD_ZONE
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['topology.kubernetes.io/zone']
          # Dynamically constructs the DB endpoint
          - name: DB_ENDPOINT
            value: "rds-$(POD_ZONE).default.svc.cluster.local"
```

## Network Policy

To allow outbound traffic only to `*.microsoft.com` URLs from your AWS EKS cluster, you can use **AWS Network Firewall** for a VPC-wide solution or the **Enhanced EKS VPC CNI Network Policies** for a Pod-specific solution.  
**Requirement**: EKS v1.29+ and VPC CNI v1.21.0+.  

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-microsoft-wildcard
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  - to:
    - fqdn: "*.microsoft.com"
```

## Pod Disruption Budget (PDB). 

A PDB is a critical policy resource used to protect application availability during voluntary disruptions—intentional actions such as node draining for maintenance, cluster upgrades, or scaling operations. 

- **Purpose**: It ensures that a minimum number (or percentage) of pods remain available at all times, preventing an automated system or administrator from accidentally taking down too many replicas of an application simultaneously.
- **How it Works with Autoscaling**: The Cluster Autoscaler respects PDBs. If a node needs to be removed to save costs, the autoscaler will only evict pods if the PDB's requirements are met.
- **Main Parameters**:
  - `minAvailable`: The minimum number or percentage of pods that must always be healthy.
  - `maxUnavailable`: The maximum number or percentage of pods that can be evicted at any given time.
- **Limitations**: It only protects against voluntary disruptions; it cannot prevent involuntary disruptions like a hardware failure or a kernel panic. 

### Example 1: Using minAvailable (Absolute Number)

This configuration is ideal for quorum-based applications (like ZooKeeper or Etcd) that require a specific number of healthy replicas to maintain state.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
  namespace: production
spec:
  # Ensures at least 2 pods matching the selector are always running
  minAvailable: 2 
  selector:
    matchLabels:
      app: critical-backend
```

### Example 2: Using maxUnavailable (Percentage)
This is best for stateless web applications that scale frequently. Using a percentage allows the budget to scale automatically as you adjust the number of replicas in your Deployment. 

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-frontend-pdb
spec:
  # Allows no more than 20% of your pods to be offline at once
  maxUnavailable: 20% 
  selector:
    matchLabels:
      tier: frontend
```
