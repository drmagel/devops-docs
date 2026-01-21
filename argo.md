# Argo CD, Argo Workflow, Argo Rollouts, Argo Events

## Core Argo Products

- **Argo CD (Continuous Delivery)**: A declarative, GitOps tool used for automated application deployment. It uses Git repositories as the "single source of truth" and ensures the live state of a Kubernetes cluster matches the desired state defined in Git.
- **Argo Workflows (Workflow Automation)**: A container-native engine used to orchestrate parallel jobs on Kubernetes. It is widely used for CI/CD pipelines, complex data processing, and machine learning (ML) tasks.
- **Argo Rollouts** (Progressive Delivery): An advanced deployment controller that provides capabilities beyond standard Kubernetes rolling updates, such as Blue-Green and Canary deployments. It integrates with service meshes and ingress controllers to manage traffic shifting during updates.
- **Argo Events (Event-Driven Automation)**: A framework for managing event-based dependencies in Kubernetes. It can trigger actions (like starting an Argo Workflow) in response to external events such as webhooks, file changes, or schedules.

## Argo CD
Argo CD is a declarative GitOps continuous delivery tool for Kubernetes that uses Git as the "single source of truth" for application state. By 2026, it remains the industry standard for managing containerized workloads through automated synchronization between Git and live clusters.

#### Core Concepts & Patterns

- **App-of-Apps Pattern**: A deployment strategy where a single "parent" Argo CD Application manages and deploys multiple "child" Applications.
  - **Purpose**: Ideal for cluster bootstrapping or managing environment-wide sets of tools (e.g., Ingress, Cert-Manager, Monitoring).
  - **Scaling**: Recommended for smaller stacks (roughly 10 or fewer apps); larger environments often transition to ApplicationSets for dynamic, template-driven management.
- **ApplicationSets**: An evolution of the App-of-Apps pattern, allowing you to use a single manifest to target multiple clusters or deploy multiple apps from different repositories simultaneously using "Generators"

#### Best Practices

To maintain a production-ready environment, follow these architectural and security standards:  
- **Repository Separation**: Maintain distinct Git repositories for application source code and Kubernetes configuration. This prevents infinite CI build loops and allows for granular access control.
- **Directory-Based Environments**: Use folders (e.g., /prod, /staging) within your configuration repo rather than long-lived Git branches to model environments.
- **Secure Secret Management**: Do not store plaintext secrets in Git. Use tools like Sealed Secrets, SOPS, or the Argo CD Vault Plugin to manage credentials externally or encrypt them in-repo.
- **Harden Access (SSO & RBAC)**: Disable the local "admin" user immediately. Integrate with OIDC/SSO (e.g., Okta, Google) and use Argo CD AppProjects to restrict which teams can deploy to specific namespaces or clusters.
- **Declarative Infrastructure**: Use **Sync Waves** (annotations) to control the order of application deployment, ensuring dependencies like databases are ready before applications.
- **Automated Drift Detection**: Enable **Self-Heal** and **Auto-Prune** in your sync policies to automatically reconcile the cluster when it drifts from the Git definition or when resources are deleted in Git. 


### Argo ApplicationSet

Think of the **ApplicationSet** as a factory that takes a template and some data (from a Generator) to mass-produce Argo CD Applications.  
- **The Generator**: Collects data (parameters like cluster name, URL, or Git folder path).
- **The Template**: A parameterized version of a standard Argo CD Application manifest.
- **The Controller**: Automatically creates, updates, or deletes the resulting Application resources whenever the source (Git, cluster list, etc.) changes.  

#### Primary Generators

**ApplicationSets** utilize several key generators to drive automation:
- **Git Generator**: Discovers directories or files within a Git repository. It is perfect for monorepos where each folder represents a different app or environment.
- **Cluster Generator**: Automatically targets all clusters (or a filtered list based on labels) registered in Argo CD.
- **List Generator**: Uses a fixed, hard-coded list of parameters (like cluster names and URLs) directly in the manifest.
- **SCM Provider Generator**: Scans your entire GitHub or GitLab organization to find repositories matching a specific pattern (e.g., all repos with a production tag).
- **Matrix Generator**: Combines two generators to create a "cross-product." For example, combining a Git Generator (for 5 apps) with a Cluster Generator (for 10 clusters) will automatically generate 50 unique applications. 


#### ApplicationSet vs. App-of-Apps

While both patterns manage multiple apps, they serve different needs:  

| Feature | App-of-Apps Pattern | ApplicationSet |
|---------|---------------------|----------------|
| Mechanism | A "Parent" app pointing to a folder of "Child" apps. | A dedicated controller that "renders" apps from a template. |
| Scalability | Manual creation of each child manifest in Git. | Fully automated; new clusters or folders are detected instantly. |
| Multi-Cluster | Requires unique manifests for each cluster destination. | One manifest can target N clusters using the Cluster Generator. |
| Self-Service | Difficult to restrict what developers can change. | High security; admins can lock the "template" and only let devs change Git paths. |

#### ApplicationSet Manifest Example (Matrix Generator Manifest)

This example combines a **Git Directory Generator** (to find applications in a repo) with a **Cluster Generator** (to find all clusters labeled as "production").

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: prod-cluster-addons
  namespace: argocd
spec:
  goTemplate: true      # Uses advanced Go templating (2026 standard)
  generators:
    - matrix:
        generators:
          # 1. Discover every folder in the 'apps/' directory of the Git repo
          - git:
              repoURL: https://github.com
              revision: HEAD
              directories:
                - path: apps/*
          # 2. Select every cluster registered in Argo CD with the label 'env: production'
          - clusters:
              selector:
                matchLabels:
                  env: production
  template:
    metadata:
      # Dynamically creates names like 'guestbook-us-east-1'
      name: '{{.path.basename}}-{{.name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com
        targetRevision: HEAD
        path: '{{.path}}'  # Uses the path discovered by the Git generator
      destination:
        server: '{{.server}}' # Uses the server URL from the Cluster generator
        namespace: '{{.path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

#### Register cluster to Argo CD

Let's assume we have external Argo CD server that servers multiple clusters

##### Method 1: Argo CD CLI
1. Login to Argo CD:
```bash
argocd login <ARGOCD_SERVER_URL> --username admin --password <YOUR_PASSWORD>
```
2. Get your Kubernetes context
```bash
kubectl config get-contexts -o name
```

3. Add the cluster
```bash
argocd cluster add <CONTEXT_NAME> --name <DESIRED_CLUSTER_NAME>
```

##### Method 2: Declarative Secret

Argo CD identifies clusters by looking for Kubernetes Secrets in its own namespace with a specific label

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-self-registration-secret
  labels:
    argocd.argoproj.io/secret-type: cluster # Required label
    env: production # Optional: Used by ApplicationSet selectors
type: Opaque
stringData:
  name: my-production-cluster
  server: https://<CLUSTER_API_URL>
  config: |
    {
      "bearerToken": "<SERVICE_ACCOUNT_TOKEN>",
      "tlsClientConfig": {
        "caData": "<BASE64_ENCODED_CA_CERT>"
      }
    }
```

**Key Considerations**
- **Security**: In 2026, it is recommended to use short-lived tokens or IAM-based authentication (e.g., via `--aws-role-arn` for EKS) instead of static long-lived ServiceAccount tokens where possible.
- **Service Account Tokens**: For Kubernetes 1.24+ and into 2026, ServiceAccount secrets are no longer auto-generated. When using the CLI, Argo CD handles this, but for manual declarative setup, you must manually create the `kubernetes.io/service-account-token` secret.
- **ApplicationSets**: Once registered, your cluster will be automatically picked up by any ApplicationSet using a Cluster generator that matches its labels.
- **Local Cluster**: To deploy to the same cluster where Argo CD is running, use the built-in server address `https://kubernetes.default.svc`

**Declarative Auto Registration**  

To make your local cluster discoverable by **ApplicationSets** (e.g., to label it as env: local), you must create a specific Secret in the `argocd` namespace.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: local-cluster-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster # Marks it for Argo CD auto-discovery
    env: local                              # Custom label for ApplicationSets
type: Opaque
stringData:
  name: in-cluster
  server: https://kubernetes.default.svc
  # No 'config' field is needed for the local cluster as it uses its own service account
```
