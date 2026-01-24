# Vault SaaS - External Secrets Storage

## Integration AWS ASM and AWS EKS

In an AWS context, ASM refers to AWS Secrets Manager, a service for securely storing, managing, and rotating sensitive data like database credentials and API keys.  
Integrating **AWS Secrets Manager** with **Amazon EKS (Elastic Kubernetes Service)** eliminates the need to hardcode secrets or store them in plain-text Kubernetes Secret objects.

### Core Integration Methods
As of 2026, there are three primary ways to integrate AWS Secrets Manager with EKS:

#### Secrets Store CSI Driver (Native AWS Recommendation) 

This method uses the **AWS Secrets and Configuration Provider (ASCP)** to mount secrets directly into pods as files.  

- **How it works**: The **Secrets Store CSI Driver** runs on each node and retrieves secrets from AWS on behalf of the pod during initialization.
- **Key Benefits**: Secrets are mounted as a volume, never touching the disk permanently.
- **Best for**: Applications that expect configuration files on a local file system.  

#### AWS Secrets Manager Agent (HTTP-based Access)
A language-agnostic agent that runs as a **sidecar container** or a **DaemonSet**.  

- **How it works**: The agent pulls and caches secrets in memory, exposing them to your application via a local HTTP endpoint (e.g., localhost:2773).
- **Key Benefits**: Reduces API calls to AWS and avoids pod restarts during secret rotation.
- **Best for**: High-scale applications requiring runtime secret access and dynamic refresh  

#### External Secrets Operator (ESO)

A community-driven Kubernetes operator that synchronizes secrets from AWS Secrets Manager into native Kubernetes Secret objects.

- **How it works**: It acts as a bridge, continuously polling AWS and updating Kubernetes secrets when changes occur.
- **Best for**: Legacy applications that specifically require native Kubernetes Secret environment variables. 

### Step-by-Step Integration Guide (CSI Driver Method)

#### Configure Identity & Access:  

1. Create an IAM Role with a policy allowing `secretsmanager:GetSecretValue` and `secretsmanager:DescribeSecret`.
2. Use **EKS Pod Identity** (recommended for EKS 1.24+) or IAM Roles for Service Accounts (IRSA) to associate the IAM role with a Kubernetes Service Account.  

#### Install the Drivers:  

1. Deploy the **Kubernetes Secrets Store CSI Driver** using Helm.
2. Install the **AWS Secrets and Configuration Provider (ASCP)** to allow the driver to talk to AWS.  

#### Define a SecretProviderClass:

Create a custom resource (`SecretProviderClass`) that specifies which secrets to fetch from AWS.  

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-provider
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "MySecretName"   # Name of the secret in AWS Secrets Manager
        objectType: "secretsmanager"
        objectAlias: "db-config"    # The filename that will appear in the pod
```

#### Update the Deployment Manifest:

Configure your Deployment to mount a volume using the `secrets-store.csi.k8s.io` driver and reference your `SecretProviderClass`. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-eks-service-account # Must have IAM permissions for Secrets Manager
      containers:
      - name: my-container
        image: nginx
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets"  # The path where secrets will be accessible
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "aws-secrets-provider" # Must match the name in Step 1
```

## External Vault - Open Source Project

**HashiCorp Vault** is a popular open-source tool used for centrally managing and protecting sensitive data, such as API keys, passwords, and certificates. Unlike static secret storage, Vault focuses on dynamic secrets, generating credentials on demand (e.g., AWS IAM keys or database logins) that automatically expire after a set time.

#### Key Features of Vault (2026)

- **Secure Storage**: Encrypts data at rest and in transit.
- **Dynamic Secrets**: Generates temporary credentials for systems like AWS, SQL databases, and Azure.
- **Identity-based Access**: Uses trusted identities (like **Kubernetes Service Accounts**) rather than permanent keys for authentication.
- **Audit Logging**: Detailed logs for compliance and security monitoring.
- **Leasing and Rotation**: Automatically revokes and rotates secrets based on defined policies.  

### Integrating Vault with AWS EKS

#### Vault Agent Sidecar Injector (Most Popular)  

The Vault Agent Injector uses a Kubernetes mutating admission webhook to automatically add a sidecar container to your pods.  

- **How it works**: You add specific annotations to your deployment manifest. The injector then adds a sidecar that authenticates with Vault and mounts secrets into a shared memory volume (e.g., `/vault/secrets/`).  
- **Best for**: Applications that need to remain unaware of Vault but require real-time secret updates.  

#### Vault Secrets Operator (Native Sync)  

The **Vault Secrets Operator (VSO)** allows you to synchronize secrets directly from Vault into native Kubernetes `Secret` objects. 
- **How it works**: It watches for changes in Vault and replicates those values into the cluster's `etcd`.
- **Best for**: Legacy applications that must read secrets from environment variables or standard Kubernetes secret volumes  

#### Secrets Store CSI Driver (Vault Provider)  

Similar to the AWS-native method, this uses the **Secrets Store CSI Driver** with a Vault-specific provider.  
- **How it works**: Secrets are mounted as a volume during the pod's container creation phase.
- **Best for**: Standardized secret mounting across multi-cloud environments (e.g., using the same driver for both AWS and Azure).  

#### External Secrets Operator (ESO)  

A community tool that can fetch secrets from Vault and sync them into Kubernetes Secrets. 
- **Best for**: Organizations already using ESO to manage multiple secret backends (e.g., mixing Vault and AWS Secrets Manager).

## 1Password SaaS

**1Password** SaaS (Software as a Service) provides centralized enterprise password and secrets management. Its "Extended Access Management" ecosystem goes beyond simple storage to include SaaS discovery, automated lifecycle management, and dedicated developer tools for infrastructure.   
Integrating **1Password** with Kubernetes (including Amazon EKS) allows you to use **1Password** vaults as the "single source of truth" for application secrets, which are then synchronized or injected directly into your clusters.  

### Primary Integration Methods

#### 1Password Connect Kubernetes Operator (Sync Method)  

The **1Password Operator** is the most robust method for 2026, designed to synchronize vault items directly into native Kubernetes Secret objects. 
How it works: You deploy a 1Password Connect Server (a bridge between 1Password Cloud and your cluster) and the Operator. You then define a OnePasswordItem custom resource that points to a specific 1Password vault item.  

**Key Feature:**  

- **Auto-Restart**: The Operator can automatically trigger a rolling restart of your deployments when the underlying secret is updated in 1Password.
- **Best for**: Teams that want 1Password secrets to behave like standard Kubernetes secrets but stay automatically in sync with the cloud.  

#### 1Password Secrets Injector (Direct Injection)  

The **Secrets Injector** uses a mutating admission webhook to inject secrets directly into a pod's environment variables at runtime.  

- **How it works**: It intercepts pod creation and replaces secret references (formatted as `op://vault/item/field`) with actual values.
- **Key Benefit**: Secrets never hit the Kubernetes etcd database as standard Secret objects, reducing the attack surface.
- **Requirement**: The pod must have a command field defined in its specification for the mutation to work.  

#### External Secrets Operator (ESO) with 1Password  

For organizations already using the **External Secrets Operator**, 1Password can be used as a backend provider.   

- **How it works**: ESO fetches secrets from the 1Password `Connect API` and creates standard Kubernetes `Secrets`.
- **Best for**: Unified management if you are already using ESO to aggregate secrets from multiple providers (e.g., **AWS Secrets Manager** and **1Password**). 
