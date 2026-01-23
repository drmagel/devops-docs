# GitHub Advanced Usage & Tools

## Local Testing with `act` tool

Using [`act`](https://nektosact.com/) to run GitHub Actions locally allows you to iterate on your CI/CD pipelines without constantly pushing "test" commits to your repository. By simulating the GitHub environment on your machine, you get near-instant feedback while saving runner minutes.  

`act` is a Go-based CLI tool that reads your `.github/workflows/` files and determines the required execution path. It then uses Docker to pull or build images that match GitHub’s hosted runners (like `ubuntu-latest`), executing your jobs inside these containers

### Key Features for Local Debugging

- **Event Simulation**: Trigger workflows as if specific GitHub events occurred (e.g., `act push` or `act pull_request`).
- **Granular Execution**: Run specific jobs instead of the entire workflow to save time using `act -j <job-id>`.
- **Secrets and Env Management**: Load sensitive data from a .secrets file using `--secret-file` or environment variables from a `.env` file with `--env-file`.
- **Visual Studio Code Integration**: Use the GitHub Local Actions extension to manage act runs, view history, and trigger events directly from your editor.

#### Installation:

- **macOS/Linux**: `brew install act`
- **Windows**: `scoop install act` or `winget install nektos.act`

#### Common Limitations

- **Incomplete Contexts**: `act` does not perfectly replicate every GitHub context variable (like some `github.*` or `vars.*` properties).
- **Runner Specifics**: Windows and macOS runners are harder to simulate via Docker; `act` often runs these on your host OS directly using the `-P` flag, which can bypass the container isolation.
- **Cache and Artifacts**: Simulating Actions' built-in cache and artifact storage locally requires additional setup using the --artifact-server-path flag. 

#### Usage examples

```bash
# From project's root directory (containing .github/workflows/)

# List all available jobs: View jobs and their triggers before running them.
act -l

# Simulate a push event: Run all workflows triggered by a standard push.
act
# or explicitly:
act push

# Run a specific job: Target only one job by its ID defined in your YAML.
act -j test-job-name

# Simulate a pull_request event
act pull_request

# Handling Secrets and Variables
# Pass a single secret
act -s MY_SECRET=value

# Use a .secrets file: Create a file (e.g., `.secrets`) with `KEY=VALUE` pairs and load it
act --secret-file .secrets

# Pass environment variables
act --var MY_VAR=data

# Advanced Local Debugging
# Dry-run mode: See the execution steps without actually running the commands
act -n

# Run a job for a specific matrix configuration: Trigger a job for a specific platform or version in your matrix
act push --matrix node:20 --matrix os:ubuntu-latest

# Persistent Artifacts: Save artifacts generated during the run to a local folder
act --artifact-server-path /tmp/artifacts

# Runner Mapping
act -P github-actions-runner=local-registry:3000/github-actions-runner:built

# The Comprehensive Debugging Command
act -j deploy-job \
    --secret-file .secrets \
    --var-file .vars \
    -P github-actions-runner=local-registry:3000/github-actions-runner:built \
    --artifact-server-path ./local-artifacts \
    --container-architecture linux/amd64
```  

**Pro Tip: Use a `.actrc` File**  
The best way to keep your terminal clean is to create a `.actrc` file in your home directory or **project root**.  
Add your preferred flags (like the runner mapping) there, and they will apply automatically every time you run `act`.  
