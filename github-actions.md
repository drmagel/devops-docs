# GitHub Actions

## Efficiency & Best Practices

To maximize your efficiency with GitHub Actions, focus on optimizing for speed, security, and the latest platform changes, including significant pricing updates for **self-hosted** runners

### Performance and Cost Optimization

Efficient workflows save both time and developer budget. In 2026, GitHub reduced **hosted-runner** prices by up to 39%, but introduced a $0.002 per-minute cloud platform charge for **self-hosted** runners in private repositories starting March 1, 2026. 
- **Intelligent Caching**: Use actions/cache for dependencies like node_modules or Maven repositories to avoid re-downloading them in every run.
- **Matrix Strategies**: Use strategy: matrix to run tests across multiple OS versions or language runtimes simultaneously rather than sequentially.
- **Path Filtering**: Prevent unnecessary runs by using path filters (e.g., `on: push: paths: ['src/**']`) so that changes to documentation files don't trigger heavy build pipelines.
- **Job Parallelization**: Split your workflow into independent jobs that can run on separate runners simultaneously to reduce total execution time.  

### Security Hardening

Securing your CI/CD pipeline is critical to preventing supply chain attacks.  

- **Pin Actions to Commit SHAs**: Instead of using tags like `@v4`, use the full immutable commit SHA (e.g., `actions/checkout@8ade135...`) to ensure a malicious actor hasn't hijacked a tag.
- **Least Privilege for GITHUB_TOKEN**: Explicitly define granular permissions in your YAML. By default, set permissions to `contents: read` and only add write access where strictly necessary.
- **Use OIDC for Cloud Access**: Replace long-lived secrets (like AWS Access Keys) with **OpenID Connect (OIDC)** to request short-lived, identity-based tokens for cloud providers.
- **Environment Protection**: Set up "Environments" in your repository settings to require manual approval before deploying to sensitive stages like production.  

### Advanced Workflow Management

- **Reusable Workflows**: Avoid duplicating YAML code across repositories. Define a common workflow once and reference it using `uses: my-org/.github/.github/workflows/shared.yml@main`.
- **Concurrency Groups**: Use concurrency to cancel in-progress runs when a new push occurs on the same branch, preventing resource waste and conflicting deployments.
- **Job Summaries**: Enhance developer experience by using `GITHUB_STEP_SUMMARY` to output custom Markdown reports (like test results or linting tables) directly to the workflow run page.
- **Local Testing**: Use tools like `act` to run and debug your GitHub Actions locally before pushing them to the server.  

### Upcoming Features in early 2026

GitHub is introducing several quality-of-life updates in Q1 2026:  
- **Timezone Support**: Scheduled jobs (cron) will finally support specific timezones.
- **Expression Case Function**: A new case operator will simplify complex conditional logic in YAML files.
- **UX Improvements**: Better rendering for massive workflows (300+ jobs) and faster page loads  

## Matrix Strategy

A **Matrix Strategy** in GitHub Actions allows you to run a single job multiple times using different variables. Instead of writing multiple jobs for different environments, you define a "matrix" of configurations, and GitHub automatically generates a job for every possible combination.  

### How it Works
When you define variables in a matrix, GitHub creates a Cartesian product of those values.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node-version: [18, 20, 22]
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}
```

### Core Features

- **Include & Exclude**: You can add specific extra configurations using `include`, or skip certain invalid combinations (e.g., a specific legacy version that doesn't run on Windows) using `exclude`.
- **Fail-Fast**: By default, if any job in the matrix fails, GitHub cancels all other in-progress jobs to save resources. You can disable this by setting `fail-fast: false` to ensure all tests complete.
- **Max Parallel**: Use `max-parallel` to limit how many jobs run at once, which is helpful if you are using limited self-hosted runners or need to avoid hitting API rate limits.
- **Dynamic Matrices**: Advanced users can generate a matrix dynamically at runtime using a previous job's JSON output, allowing the workflow to adapt to the specific files changed in a PR  

### Key Use Cases

- **Cross-Platform Testing**: Ensure your code works on Linux, macOS, and Windows simultaneously.
- **Version Compatibility**: Test against multiple versions of languages (Python 3.10-3.13) or databases (PostgreSQL 14-17).
- **Test Sharding**: Split a massive test suite into smaller "shards" that run in parallel to drastically reduce total CI time.
- **Multi-Arch Builds**: Simultaneously build Docker images for `amd64` and `arm64` architectures.  

### Limits to Remember

- **Job Cap**: A single workflow run is limited to a maximum of **256** jobs generated by a matrix.
- **Time Limit**: Each individual job in the matrix can run for up to **6** hours before it is automatically terminated. 


## Multi-Arch Builds

The recommended pattern for multi-architecture builds is to use a matrix strategy to build individual platform images in parallel, followed by a merge job to create a single multi-arch manifest. This approach is significantly faster than building all platforms on a single runner using emulation.  

An example of a GitHub Actions workflow using a matrix strategy for multi-arch builds can be found on [Docker Docs](https://docs.docker.com/build/ci/github-actions/multi-platform/). This approach uses parallel builds for different architectures and pushes images by digest for merging.  

```yaml
name: ci

on:
  push:

env:
  REGISTRY_IMAGE: user/app

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
        - platform: linux/amd64
          runner: action-runner-dind-amd64
        - platform: linux/arm64
          runner: action-runner-dind-arm64
    runs-on: ${{ matrix.runner }}
    steps:
      - name: Prepare
        run: |
          platform=${{ matrix.platform }}
          echo "PLATFORM_PAIR=${platform//\//-}" >> $GITHUB_ENV

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push by digest
        id: build
        uses: docker/build-push-action@v6
        with:
          platforms: ${{ matrix.platform }}
          labels: ${{ steps.meta.outputs.labels }}
          tags: ${{ env.REGISTRY_IMAGE }}
          outputs: type=image,push-by-digest=true,name-canonical=true,push=true

      - name: Export digest
        run: |
          mkdir -p ${{ runner.temp }}/digests
          digest="${{ steps.build.outputs.digest }}"
          touch "${{ runner.temp }}/digests/${digest#sha256:}"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digests-${{ env.PLATFORM_PAIR }}
          path: ${{ runner.temp }}/digests/*
          if-no-files-found: error
          retention-days: 1

  merge:
    runs-on: action-runner-dind-arm64
    needs:
      - build
    steps:
      - name: Download digests
        uses: actions/download-artifact@v4
        with:
          path: ${{ runner.temp }}/digests
          pattern: digests-*
          merge-multiple: true

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Create manifest list and push
        working-directory: ${{ runner.temp }}/digests
        run: |
          docker buildx imagetools create $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
            $(printf '${{ env.REGISTRY_IMAGE }}@sha256:%s ' *)

      - name: Inspect image
        run: |
          docker buildx imagetools inspect ${{ env.REGISTRY_IMAGE }}:${{ steps.meta.outputs.version }}
```  

## Connection with AWS

The industry-standard best practice for connecting GitHub Actions to AWS is using **OpenID Connect (OIDC)**. This method eliminates the need for long-lived IAM access keys and secret keys, which are prone to exposure and require manual rotation.

### Implementation Steps

#### 1. Configure AWS IAM

- **Create an OIDC Provider**: In the IAM console, add a new provider with the URL `https://token.actions.githubusercontent.com` and audience `sts.amazonaws.com`.
- **Create a Trust Role**: Create an IAM role with a "Web Identity" trust policy. Scope the `Condition` to your specific GitHub `organization` and `repository` to prevent unauthorized access.  

#### 2. Update GitHub Workflow

The workflow must have specific permissions to request the OIDC token and use the official AWS action to assume the role.  

```yaml
permissions:
  id-token: write   # Required for requesting the JWT
  contents: read    # Required for actions/checkout

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
          aws-region: us-east-1
```

### Security Checklist

- **Enforce Least Privilege**: Attach only the specific IAM policies required for the job (e.g., only S3 access for static site uploads).
- **Environment Protection**: In GitHub settings, use "Environments" to require manual approval before a workflow can assume a production-level AWS role.
- **Monitor with CloudTrail**: Enable **AWS CloudTrail** to audit all `AssumeRoleWithWebIdentity` events and detect any anomalous activity.
- **Self-Hosted Advantage**: If using self-hosted runners on EC2 or AWS CodeBuild, you can use **IAM Instance Profiles** instead of OIDC, as the runner already has a secure identity within the AWS environment.  

## Advanced Workflow Management

### Reusable Workflows

**Reusable Workflows** are the standard for scaling CI/CD across multiple repositories, effectively serving as "blueprints" for entire pipelines. They differ from composite actions by allowing you to reuse entire jobs and distinct runner configurations.  

#### Key Features and Syntax

- **The Trigger**: A workflow becomes reusable by defining the `on: workflow_call` trigger.
- **Calling a Workflow**: Use the uses keyword at the job level

```yaml
jobs:
  call-reusable:
    uses: octo-org/shared-workflows/.github/workflows/ci.yml@v1
    with:
      environment: production
    secrets: inherit  # Automatically passes all caller secrets

```

- **Parameters**: You can define `inputs` (string, boolean, number) and `secrets` that the calling workflow must provide

#### Tagging Reusable Workflow Version

To set the `v1` version for your reusable workflow, you need to create and push a Git tag in your shared-workflows repository. GitHub Actions uses these tags as references when a caller workflow specifies `@v1`.  

```bash
# Create the Tag Manually (CLI)
git tag v1
git push origin v1

# Best Practice: "Moving" Tags
# The standard for GitHub Actions is to use Semantic Versioning (SemVer) alongside moving major tags.
# This allows users to stay on `@v1` while you push non-breaking bug fixes.
# a. Tag a specific release (e.g., `v1.0.1`).
# b. Update the `v1` tag to point to the same commit as `v1.0.1`
git tag -fa v1 -m "Update v1 to point to v1.0.1"
git push origin v1 --force
```

#### Reusable Workflows vs. Composite Actions

| Feature | Reusable Workflows | Composite Actions |
|---------|-------------------|-------------------|
| **Level** | Job-level (orchestrates jobs) | Step-level (groups steps) |
| **Runners** | Can specify different runners per job | Runs on the caller's runner |
| **Logging** | Detailed real-time logs for every step | Groups all steps into one log entry |
| **Secrets** | Native secret support and inheritance | Secrets must be passed as inputs |
| **Marketplace** | Cannot be published to Marketplace | Can be published to Marketplace |

### Concurrency Groups

**Concurrency Groups** remain one of the most effective ways to reduce GitHub Actions costs and prevent deployment conflicts. By default, GitHub runs multiple instances of the same workflow or job in parallel. Concurrency groups override this behavior, ensuring that only one run per group is active at a time  

#### Core Functionality

When a new workflow or job is triggered within a defined concurrency group:  

- **The Lock**: If a run is already in progress, any new runs with the same group key are put into a "pending" state.
- **The Queue**: GitHub allows at most **one running and one pending job in a group**. If a third run is triggered, it cancels any existing pending run and takes its place.
- **Canceled in Progress**: By setting `cancel-in-progress: true`, any currently running job is immediately terminated to start the latest one. 

#### Common Use Cases

- **Protecting Deployments**: Prevent two developers from deploying to the same staging or production environment simultaneously, which could cause inconsistent states.
- **Cost Savings**: Stop wasting minutes on outdated PR commits. If a developer pushes three times in quick succession, only the latest commit needs to be tested. 

#### Implementation Examples

##### Cancel Outdated Pull Request Builds

Using a dynamic group name that includes the branch name ensures you only cancel runs on the same branch, not affecting other developers' work. 

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

##### Sequential Production Deployments

For sensitive deployments where you never want to cancel an active run but must ensure they happen one at a time, use a hardcoded group name and set `cancel-in-progress: false`.

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: false
```

##### GitHub Actions Workflow: CI & Deployment

This example demonstrates a professional-grade CI/CD pipeline. It uses **Concurrency Groups** to ensure that while tests on a Pull Request are "fail-fast" (canceling old runs), the deployment to Production is sequential and never interrupted.

```yaml
name: CI and Deployment

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

# 1. Workflow-level Concurrency:
# On Pull Requests: Cancel any previous runs for this specific PR to save minutes.
# On Main: Ensure only one deployment process runs at a time.
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run Tests
        run: npm test

  deploy:
    name: Deploy to Production
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    # 2. Job-level Concurrency Override:
    # We want deployments to be sequential (FIFO), NOT canceled halfway through.
    concurrency:
      group: production-release
      cancel-in-progress: false 

    steps:
      - name: Deploying Application
        run: |
          echo "Connecting to AWS via OIDC..."
          echo "Executing zero-downtime deployment..."
          sleep 30 # Simulating a long deployment process
```

- **Dynamic Group Naming**: The group name uses `${{ github.event.pull_request.number || github.ref }}`. This ensures that a build on `branch-a` does not cancel a build on `branch-b`.  
- **Strategic `cancel-in-progress`**:
  1. In the **CI phase**, it is set to `true`. If you push three times in one minute, GitHub kills the first two runs immediately, saving you money and runner availability.  
  2. In the **Deploy phase**, it is set to `false`. If two merges to main happen back-to-back, the second deployment will wait for the first to finish successfully rather than killing it mid-deploy, which could leave your production environment in a broken state.  
- **Dependency Handling**: The `needs: test` keyword ensures that even if the concurrency group allows the deployment to start, it will wait for the testing job to pass first.  

#### Critical Limitations

- **Order of Execution**: GitHub does not guarantee that queued runs will be handled in chronological order; they are processed as resources become available.
- **Matrix Conflicts**: If you apply concurrency at the job level within a matrix, each matrix item might compete for the same group, causing parallel test shards to cancel each other. Wrap the **matrix** in a **reusable workflow** to avoid this.
- **Organization-Wide Scope**: Concurrency group names are scoped to the repository. If multiple workflows across different repositories use the same group name, they do not affect each other

### Job Summaries

**Job Summaries** in GitHub Actions allow you to generate custom Markdown-formatted reports that appear directly on the workflow run's landing page. This feature is essential for visualizing complex data—like test results, code coverage, or deployment links—without forcing developers to dig through thousands of lines of raw console logs.

#### How to Create a Summary

To add content to a job summary, you append Markdown text to the special environment file located at `$GITHUB_STEP_SUMMARY`

```yaml
steps:
  - name: Generate Summary
    run: |
      cat <<EOF >> $GITHUB_STEP_SUMMARY
      ## Build Report 🚀
      | Component | Status |
      |-----------|--------|
      | API       | ✅ Pass |
      | Frontend  | ❌ Fail |
      EOF
```

#### Key Technical Rules

- **Aggregation**: All steps in a single job share the same summary file. When the job completes, GitHub combines all content appended to `$GITHUB_STEP_SUMMARY` into one rendered display.
- **Visibility**: Summaries only appear on the main Actions run summary page after a job has finished.
- **Limits**: Summaries are limited to **1MB per job**. If you exceed this, GitHub will truncate the content.
- **Markdown Support**: It supports standard GitHub Flavored Markdown (GFM), including `tables`, `emojis`, `links`, and `even Mermaid diagrams` for workflow visualizations.  

#### Advanced Usage & Tools

- **Javascript Actions**: If you are writing custom JavaScript actions, use the official `@actions/core` library's `summary` utility to programmatically build tables and headings.
- **Dynamic Templates**: Tools like `Markdown Builder` allow you to use Handlebars templates to generate complex, data-driven summaries from JSON files.
- **External Integration**: Many third-party actions, such as Docker's build-push action, now automatically generate job summaries showing details like cache utilization and Dockerfile contents.  
