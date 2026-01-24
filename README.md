# DevOps Documentation Index

This repository contains comprehensive documentation for DevOps practices, focusing on AWS, Kubernetes, Terraform, ArgoCD, GitHub Actions, WAF security, secrets management, and related technologies. This document serves as a navigation guide to help you quickly find information on specific topics.

## Table of Contents

- [Argo](#argo)
- [AWS Services](#aws-services)
- [CNI (Container Network Interface)](#cni-container-network-interface)
- [EKS Cell-Based Architecture](#eks-cell-based-architecture)
- [GitHub Actions](#github-actions)
- [Kubernetes & EKS](#kubernetes--eks)
- [Network Fundamentals](#network-fundamentals)
- [Secrets Management](#secrets-management)
- [Terraform](#terraform)
- [WAF & Security](#waf--security)

---

## Argo

### Core Argo Products
- **[Overview](argo.md#core-argo-products)** - Argo CD, Argo Workflows, Argo Rollouts, Argo Events

### Argo CD
- **[Core Concepts & Patterns](argo.md#core-concepts--patterns)** - App-of-Apps Pattern, ApplicationSets
- **[Best Practices](argo.md#best-practices)** - Repository separation, directory-based environments, secret management, SSO & RBAC, sync waves, drift detection

### ApplicationSet
- **[ApplicationSet Overview](argo.md#argo-applicationset)** - Generator, Template, Controller concepts
- **[Primary Generators](argo.md#primary-generators)** - Git, Cluster, List, SCM Provider, Matrix generators
- **[ApplicationSet vs. App-of-Apps](argo.md#applicationset-vs-app-of-apps)** - Comparison table
- **[ApplicationSet Manifest Example](argo.md#applicationset-manifest-example-matrix-generator-manifest)** - Matrix Generator with Git and Cluster generators

### Cluster Registration
- **[Method 1: Argo CD CLI](argo.md#method-1-argo-cd-cli)** - Command-line cluster registration
- **[Method 2: Declarative Secret](argo.md#method-2-declarative-secret)** - Kubernetes Secret-based registration
- **[Key Considerations](argo.md#key-considerations)** - Security, service account tokens, ApplicationSets, local cluster
- **[Declarative Auto Registration](argo.md#declarative-auto-registration)** - Local cluster discovery setup

---

## AWS Services

### VPC Networking
- **[Connect VPC to Internet](aws.md#connect-vpc-to-internet)** - Essential connectivity resources (Internet Gateway, Public Subnets, Elastic IP, NAT Gateway)
- **[Transit Gateway](aws.md#transit-gateway)** - Hub-and-spoke router with native Network Firewall integration
  - Topics: Centralized traffic inspection, firewall rule types, routing implementation

### Lambda
- **[Lambda's Roles and Policies](aws.md#lambdas-roles-and-policies)** - Execution roles, trust policies, resource-based policies
- **[Lambda's Triggering Options](aws.md#lambdas-triggering-options)** - S3 events, message queues, synchronous triggers, schedules
- **[Asynchronous Invocations](aws.md#asynchronous-invocations)** - Managed retry logic, destinations, error handling, concurrency

### AWS Organizations & IAM
- **[Organization Account](aws.md#organization-account)** - Management account, member accounts, workforce identities
- **[IAM Identities](aws.md#iam-identities)** - Workforce identities, permission sets, IAM roles
- **[Roles vs. Users vs. Groups](aws.md#roles-vs-users-vs-groups)** - Comparison table of IAM resources
- **[Service Control Policies (SCPs)](aws.md#security-governance-service-control-policies-scps)** - Security governance and policy enforcement
- **[Multi-Account Setup](aws.md#multi-account-setup)** - Account types and their purposes (Management, Security, Identity, Log Archive)

---

## CNI (Container Network Interface)

### CNI Overview
- **[CNI Types and Characteristics](cni.md#cnis)** - Overview of CNI plugins and selection criteria
- **[Popular CNI Types & Characteristics](cni.md#popular-cni-types--characteristics)** - Comparison table of major CNI plugins
  - CNIs: Cilium, Calico, Flannel, Canal, Weave Net
- **[Specialized & Cloud-Native CNIs](cni.md#specialized--cloud-native-cnis)** - Cloud provider CNIs and specialized solutions
- **[Core Network Models](cni.md#core-network-models)** - Overlay vs Underlay networking
- **[Selection Criteria](cni.md#selection-criteria)** - Security, performance, scalability, kernel support considerations

### eBPF
- **[What is eBPF?](cni.md#what-is-ebpf)** - Extended Berkeley Packet Filter explained
  - Topics: Safety, efficiency, hook-based architecture
- **[eBPF vs. iptables Comparison](cni.md#ebpf-vs-iptables-comparison)** - Feature comparison table
  - Comparison: Search mechanism, performance, latency, updates, observability, security scope, compatibility
- **[Kubernetes CNI Technology Comparison](cni.md#kubernetes-cni-technology-comparison)** - Comprehensive CNI data plane comparison
  - Technologies: eBPF, iptables, OVS, IPVS, VXLAN, VPC/VNet Native, DPDK/VPP
- **[Technology Summaries](cni.md#technology-summaries)** - Overview of data plane technologies

---

## EKS Cell-Based Architecture

### Architecture Pattern
- **[Shared ALB](eks-cell-based-architecture.md#the-architecture-pattern-shared-alb)** - Central Application Load Balancer for multiple EKS clusters

### Implementation
- **[Central Infrastructure](eks-cell-based-architecture.md#a-central-infrastructure)** - ALB and Target Group setup
- **[Cluster-Level Configuration](eks-cell-based-architecture.md#b-cluster-level-configuration)** - TargetGroupBinding setup
- **[Service-Level Configuration](eks-cell-based-architecture.md#c-service-level-configuration)** - Service and TargetGroupBinding manifests

### Load Balancing
- **[Achieving Round-Robin](eks-cell-based-architecture.md#achieving-round-robin)** - ALB round-robin distribution
- **[Health-Check Driven Failover](eks-cell-based-architecture.md#why-this-is-the-expert-choice)** - Automatic failover capabilities
- **[Critical Caveat: Sticky Sessions](eks-cell-based-architecture.md#critical-caveat-the-sticky-session-trap)** - Session stickiness considerations
- **[Summary of Component Jobs](eks-cell-based-architecture.md#summary-of-component-jobs)** - Component responsibilities table

### Terraform Infrastructure
- **[Terraform Examples](eks-cell-based-architecture.md#terraform-examples)** - Complete infrastructure as code
- **[Application Load Balancer](eks-cell-based-architecture.md#application-load-balancer)** - ALB resource configuration
- **[Target Group](eks-cell-based-architecture.md#target-group)** - Target group with IP targeting
- **[The Listener](eks-cell-based-architecture.md#the-listener)** - ALB listener configuration
- **[The Output](eks-cell-based-architecture.md#the-output-integration-value)** - Target Group ARN output for integration

### Global Redis Cache
- **[Key Implementation Patterns](eks-cell-based-architecture.md#1-key-implementation-patterns)** - Cross-zone and cross-region patterns
  - Topics: Cross-Zone (Single Region), Cross-Region Global Datastore
- **[Valkey vs. Redis OSS Comparison](eks-cell-based-architecture.md#global-redis-cache)** - Technical comparison table
  - Comparison: Throughput, threading model, memory efficiency, latency, pricing
- **[Python Code](eks-cell-based-architecture.md#python-code)** - Redis client implementation example
- **[Best Practices](eks-cell-based-architecture.md#best-practices)** - Engine choice, serverless vs node, persistence, security

### Zonal Read Isolation
- **[Identify Node Endpoints](eks-cell-based-architecture.md#a-identify-node-endpoints)** - Direct node addressing for zone-specific reads
- **[Configuration Injection per EKS Cluster](eks-cell-based-architecture.md#b-configuration-injection-per-eks-cluster)** - Environment variable injection
- **[Terraform Example](eks-cell-based-architecture.md#terraform-example)** - ElastiCache replication group with zonal isolation
- **[Output Example](eks-cell-based-architecture.md#output-example)** - Redis node endpoints output

---

## GitHub Actions

### Efficiency & Best Practices
- **[Performance and Cost Optimization](github-actions.md#performance-and-cost-optimization)** - Intelligent caching, matrix strategies, path filtering, job parallelization
- **[Security Hardening](github-actions.md#security-hardening)** - Pin actions to commit SHAs, least privilege, OIDC for cloud access, environment protection

### Advanced Workflow Management
- **[Reusable Workflows](github-actions.md#reusable-workflows)** - Job-level workflow blueprints, versioning, vs. Composite Actions comparison
  - Topics: Key features and syntax, tagging workflow versions, comparison table
- **[Concurrency Groups](github-actions.md#concurrency-groups)** - Cost reduction and deployment conflict prevention
  - Topics: Core functionality, common use cases, implementation examples (PR builds, sequential deployments)
- **[Job Summaries](github-actions.md#job-summaries)** - Custom Markdown reports on workflow run pages
  - Topics: How to create summaries, technical rules, advanced usage & tools
- **[Local Testing with act](act-tool.md#local-testing-with-act-tool)** - Run GitHub Actions locally for faster iteration
  - Topics: Installation, key features, limitations, usage examples, .actrc configuration
- **[Upcoming Features in early 2026](github-actions.md#upcoming-features-in-early-2026)** - Timezone support, expression case function, UX improvements

### Matrix Strategy
- **[How it Works](github-actions.md#how-it-works)** - Cartesian product of matrix variables
- **[Core Features](github-actions.md#core-features)** - Include & Exclude, Fail-Fast, Max Parallel, Dynamic Matrices
- **[Key Use Cases](github-actions.md#key-use-cases)** - Cross-platform testing, version compatibility, test sharding, multi-arch builds
- **[Limits to Remember](github-actions.md#limits-to-remember)** - Job cap (256 jobs), time limit (6 hours per job)

### Multi-Arch Builds
- **[Multi-Arch Builds](github-actions.md#multi-arch-builds)** - Matrix strategy for parallel platform builds with merge job pattern

### Connection with AWS
- **[Implementation Steps](github-actions.md#implementation-steps)** - OIDC provider setup and workflow configuration
  - Topics: Configure AWS IAM, Update GitHub Workflow
- **[Security Checklist](github-actions.md#security-checklist)** - Least privilege, environment protection, CloudTrail monitoring, self-hosted advantages

---

## Kubernetes & EKS

### Core Kubernetes Concepts
- **[Kubernetes Main Processes](kubernetes.md#kubernetes-main-processes)** - Control plane, worker nodes, helper processes
  - Summary table: [Process Summary Table](kubernetes.md#summary-table)

### AWS EKS Plugins
- **[Core Networking Add-ons](kubernetes.md#1-core-networking-add-ons)** - VPC CNI, CoreDNS, Kube-proxy
- **[Storage and Infrastructure Plugins](kubernetes.md#2-storage-and-infrastructure-plugins)** - Load Balancer Controller, EBS/EFS CSI Drivers, Karpenter
- **[Security and Observability Add-ons](kubernetes.md#3-security-and-observability-add-ons)** - EKS Pod Identity, GuardDuty, ADOT, External Secrets
- **[Community Add-ons Catalog](kubernetes.md#4-community-add-ons-catalog)** - Metrics Server, Cert-Manager, External-DNS
- **[Metrics-Based Autoscalers](kubernetes.md#metrics-based-autoscalers)** - HPA, Metrics Server, KEDA, VPA, Karpenter comparison

### EKS Topology & Architecture
- **[Multi-Zonal (Single Cluster) Topology](kubernetes.md#1-multi-zonal-single-cluster-topology)** - Standard production setup
- **[Multi-Cluster Topology](kubernetes.md#2-multi-cluster-topology-per-environment)** - Per-environment cluster separation
- **[Multi-Account/Multi-Region Topology](kubernetes.md#3-multi-accountmulti-region-topology)** - Advanced isolation and failover
- **[Networking Best Practices](kubernetes.md#4-networking-best-practices-for-topologies)** - IPv6 adoption, Cilium, private endpoints

### Multi-Zonal Topology Limitations
- **[Single Cluster Topology Limitations](kubernetes.md#single-cluster-topology-limitations)** - 8 key limitations and mitigations
  - Topics: Zonal data gravity, inter-AZ costs, network latency, blast radius, scheduling complexity, IPv4 exhaustion, load balancer distribution
- **[Multiple Cluster Topology Limitations](kubernetes.md#multiple-cluster-topology-limitations-cell-based-architecture)** - Comparison table (1 Multi-AZ Cluster vs 3 Independent Clusters)

### AWS IAM Integration with Kubernetes
- **[EKS Pod Identity](kubernetes.md#aws-service-account-for-pods-eks-pod-identity)** - Modern standard for granting Pods IAM permissions
  - Includes: Terraform examples, Kubernetes implementation
- **[IRSA (Legacy)](kubernetes.md#aws-service-account-for-pods-the-legacy-way-irsa-iam-roles-for-service-accounts)** - IAM Roles for Service Accounts
- **[Comparison: EKS Pod Identity vs IRSA](kubernetes.md#comparison)** - Feature comparison table

### Load Balancers
- **[AWS EKS Application and Network Load Balancers](kubernetes.md#aws-eks-application-and-network-load-balancers)** - AWS Load Balancer Controller vs NGINX Ingress
  - Comparison table: [NGINX vs AWS Load Balancer Controller](kubernetes.md#the-architecture-gateway-api)
- **[NGINX Ingress Controller (Legacy)](kubernetes.md#nginx-ingress-controller-legacy)** - Legacy implementation examples
- **[AWS Load Balancer Controller (2026 Standard)](kubernetes.md#aws-load-balancer-controller-2026-standard)** - Modern implementation with Gateway API

### Pod Management
- **[Pod Topology Spread Constraints](kubernetes.md#pod-topology-spread-constraints)** - Distribution control across failure domains
  - Examples: Balanced distribution, zone-sticky StatefulSet
- **[Network Policy](kubernetes.md#network-policy)** - FQDN-based egress rules for EKS
- **[Pod Disruption Budget (PDB)](kubernetes.md#pod-disruption-budget-pdb)** - Availability protection during voluntary disruptions
  - Examples: minAvailable, maxUnavailable

---

## Network Fundamentals

### TCP/IP Protocol
- **[TCP/IP Protocol Overview](network.md#tcpip-protocol-overview)** - TCP and UDP protocols explained
  - Topics: TCP three-way handshake, UDP characteristics, use cases
- **[TCP/IP Model Layers](network.md#tcpip-model-layers)** - Five-layer network model
  - Layers: Application, Transport, Internet (Network), Data Link, Physical
  - Protocol table: [TCP/IP Model Layers Table](network.md#tcpip-model-layers)

---

## Secrets Management

### AWS Secrets Manager Integration

#### Core Integration Methods
- **[Secrets Store CSI Driver](vault-secrets.md#secrets-store-csi-driver-native-aws-recommendation)** - Native AWS recommendation for pod-level secret mounting
  - Topics: How it works, key benefits, best use cases
- **[AWS Secrets Manager Agent](vault-secrets.md#aws-secrets-manager-agent-http-based-access)** - HTTP-based sidecar/DaemonSet access
  - Topics: Agent architecture, caching, dynamic refresh for high-scale apps
- **[External Secrets Operator (ESO)](vault-secrets.md#external-secrets-operator-eso)** - Community-driven Kubernetes operator
  - Topics: Bridge pattern, continuous polling, legacy app support

#### Step-by-Step Integration Guide
- **[CSI Driver Method Setup](vault-secrets.md#step-by-step-integration-guide-csi-driver-method)** - Complete integration walkthrough
  - Topics: Identity & access configuration, driver installation, SecretProviderClass, deployment manifest updates

### HashiCorp Vault (Open Source)

#### Key Features
- **[Vault Overview](vault-secrets.md#external-vault---open-source-project)** - Dynamic secrets, identity-based access, audit logging, leasing & rotation
- **[Key Features of Vault (2026)](vault-secrets.md#key-features-of-vault-2026)** - Secure storage, dynamic secrets, identity-based access, compliance features

#### Integration with AWS EKS
- **[Vault Agent Sidecar Injector](vault-secrets.md#vault-agent-sidecar-injector-most-popular)** - Most popular integration method
  - Topics: Mutating admission webhook, sidecar injection, shared memory volumes, real-time updates
- **[Vault Secrets Operator (VSO)](vault-secrets.md#vault-secrets-operator-native-sync)** - Native Kubernetes Secret synchronization
  - Topics: etcd replication, environment variable support, legacy app compatibility
- **[Secrets Store CSI Driver (Vault Provider)](vault-secrets.md#secrets-store-csi-driver-vault-provider)** - Multi-cloud standardization
  - Topics: Volume mounting, container creation phase, cross-cloud consistency
- **[External Secrets Operator (ESO)](vault-secrets.md#external-secrets-operator-eso)** - Community tool for multi-backend integration
  - Topics: Unified secret management, multiple backend support, ESO ecosystem

### 1Password SaaS

#### Integration Methods
- **[1Password Overview](vault-secrets.md#1password-saas)** - Enterprise password and secrets management
  - Topics: Extended Access Management, SaaS discovery, automated lifecycle, developer tools
- **[1Password Connect Kubernetes Operator](vault-secrets.md#1password-connect-kubernetes-operator-sync-method)** - Most robust 2026 method
  - Topics: Connect Server bridge, OnePasswordItem CRD, auto-restart on secret updates, Kubernetes Secret sync
- **[1Password Secrets Injector](vault-secrets.md#1password-secrets-injector-direct-injection)** - Direct runtime injection
  - Topics: Mutating admission webhook, environment variable injection, etcd bypass, attack surface reduction
- **[External Secrets Operator (ESO)](vault-secrets.md#external-secrets-operator-eso-with-1password)** - ESO backend integration
  - Topics: Connect API, unified secret management, multi-provider aggregation

---

## Terraform

### Locals and Loops
- **[The `locals {}` Block](terraform.md#the-locals--block)** - Internal variable storage and calculations
- **[The `for_each` Meta-Argument](terraform.md#the-for_each-meta-argument)** - Creating multiple resource instances
- **[The `for` Expression](terraform.md#the-for-expression)** - Transforming and filtering collections
- **[Summary Table](terraform.md#summarize)** - Quick reference for locals, for_each, and for loops

### Variable Types
- **[Variable Types Overview](terraform.md#variable-types)** - List, Map, Set, Tuple comparison
  - Detailed comparison table: [Variable Types Comparison](terraform.md#variable-types)
  - Quick reference table: [Variable Types Quick Reference](terraform.md#variable-types)
- **[Examples Snippets](terraform.md#examples-snippets)** - Practical examples for each type
  - Topics: List (ordered), Map (lookup tables), Set (unique values), Tuple (mixed types)
- **[Complex Example](terraform.md#complex-example)** - Security Group Rules with nested structures
- **[Summary of References](terraform.md#summary-of-references)** - Syntax for accessing resource instances

### Built-in Functions
- **[Function Categories](terraform.md#built-in-function)** - Numeric, String, Collection, Filesystem, IP Network, Encoding & Crypto, Type Conversion
  - Category table: [Function Categories Table](terraform.md#documentation-overview)
- **[Terraform Console](terraform.md#documentation-overview)** - Interactive function testing

### Resource Dependencies
- **[Implicit Dependencies](terraform.md#implicit-dependencies)** - Terraform-native dependency handling
- **[Explicit Dependencies (`depends_on`)](terraform.md#explicit-dependencies-depends_on)** - Manual dependency specification
- **[Dependencies with `for_each`](terraform.md#dependencies-with-for_each)** - Handling dependencies in loops
- **[Best Practices](terraform.md#best-practices)** - Guidelines for dependency management

---

## WAF & Security

### F5 Advanced WAF
- **[F5 WAF Overview](waf.md#f5-advanced-waf)** - Enterprise WAF solution with advanced threat protection
  - Topics: Vulnerability protection, advanced defense, data safeguarding, deployment versatility
- **[Programming Languages for F5 WAF](waf.md#programming-and-scripting-languages)** - Tcl, Python, JavaScript, YAML/JSON for WAF customization

### AWS WAF
- **[AWS WAF Overview](waf.md#aws-waf)** - Cloud-native, fully managed Web Application Firewall
  - Topics: Web ACLs, rule language, automation approaches
- **[Key Features for 2026](waf.md#key-features-for-2026)** - Managed Rule Groups, Bot Control, Fraud Control, Real-Time Monitoring
- **[Integration Methods](waf.md#integration)** - ALB, EKS Ingress Controller, Amazon API Gateway
  - Integration details: Direct association, one-click setup, declarative configuration, auto-reconciliation

### Amazon CloudFront
- **[CloudFront Overview](waf.md#amazon-cloudfront)** - Global Content Delivery Network with edge security
  - Topics: Edge caching, regional edge caches, programmable edge, Lambda@Edge
- **[Key Features for 2026](waf.md#key-features-for-2026)** - CloudFront Functions, Lambda@Edge, Viewer mTLS, AWS Shield & WAF integration, Blue/Green deployment

---

## Quick Reference Tables

### Kubernetes Processes
- **[Process Summary Table](kubernetes.md#summary-table)**

### Autoscaling Components
- **[Metrics-Based Autoscalers](kubernetes.md#metrics-based-autoscalers)**

### Topology Comparison
- **[Multi-AZ vs Multi-Cluster](kubernetes.md#multiple-cluster-topology-limitations-cell-based-architecture)**

### IAM Methods Comparison
- **[EKS Pod Identity vs IRSA](kubernetes.md#comparison)**

### Load Balancer Comparison
- **[NGINX vs AWS Load Balancer Controller](kubernetes.md#the-architecture-gateway-api)**

### IAM Resources
- **[Roles vs Users vs Groups](aws.md#roles-vs-users-vs-groups)**

### Multi-Account Setup
- **[Account Types](aws.md#multi-account-setup)**

### ArgoCD Patterns
- **[App-of-Apps vs ApplicationSet](argo.md#applicationset-vs-app-of-apps)**

### Terraform Types
- **[Variable Types Comparison](terraform.md#variable-types)**

### Terraform Functions
- **[Function Categories](terraform.md#documentation-overview)**

### Cell Architecture Components
- **[Component Jobs](eks-cell-based-architecture.md#summary-of-component-jobs)**

### Redis Best Practices
- **[ElastiCache Best Practices](eks-cell-based-architecture.md#best-practices)**

### Network Layers
- **[TCP/IP Model Layers](network.md#tcpip-model-layers)**

### CNI Plugins
- **[Popular CNI Types & Characteristics](cni.md#popular-cni-types--characteristics)**
- **[eBPF vs. iptables Comparison](cni.md#ebpf-vs-iptables-comparison)**
- **[Kubernetes CNI Technology Comparison](cni.md#kubernetes-cni-technology-comparison)**

### Redis/Valkey Comparison
- **[Valkey vs. Redis OSS](eks-cell-based-architecture.md#global-redis-cache)** - Technical comparison table

---

## Contributing

When adding new documentation:
1. Add content to the appropriate markdown file
2. Update this README with the new topic and location
3. Maintain consistent formatting and structure
4. Include code examples where applicable
5. Add comparison tables for related concepts

---

## File Structure

```
devops-docs/
├── README.md                          # This index file
├── act-tool.md                        # Local testing with act tool
├── argo.md                            # Argo CD and related tools
├── aws.md                             # AWS services and best practices
├── cni.md                             # CNI plugins and eBPF
├── eks-cell-based-architecture.md     # Cell-based architecture patterns
├── github-actions.md                  # GitHub Actions workflows and best practices
├── kubernetes.md                      # Kubernetes and EKS documentation
├── network.md                         # Network fundamentals and TCP/IP
├── terraform.md                       # Terraform tips and tricks
├── vault-secrets.md                   # Secrets management (AWS Secrets Manager, Vault, 1Password)
└── waf.md                             # Web Application Firewall (F5, AWS WAF, CloudFront)
```

---

*Last updated: 2026*
