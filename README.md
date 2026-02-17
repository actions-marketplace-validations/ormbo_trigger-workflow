# GitHub Custom Action: trigger-workflow

[![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/codex-/return-dispatch/test.yml?style=flat-square)](https://github.com/Codex-/return-dispatch/actions/workflows/test.yml)


A custom GitHub Action that triggers a workflow (in the same or a different repository) and—unlike the native API—waits for its completion.

### 💡 The Problem
The standard GitHub workflow_dispatch API has two major limitations:

1. **Fire and Forget**: It returns a 204 No Content status immediately. It does not return the ID of the run it just created.

2. **No Waiting**: You cannot natively make one workflow wait for another to finish before proceeding (essential for CD pipelines where you need the build to finish before deploying).

### ✨ The Solution
This custom action solves these issues by:

1. **Injecting a Correlation ID**: It passes a unique ID to the target workflow to identify the specific run.

2. **Smart Polling**: It scans the target repo to find the run matching that ID.

3. **Tracking Status**: It monitors the run in real-time and reports the final status (Success/Failure) back to the caller.


## Usage

1. Dispatching Action (The Caller)
Use this step in the workflow that initiates the trigger.

```yaml
steps:
  - name: Trigger CD and wait for it to complete
    # Update the path below to point to where this action resides
    uses: ./cloud-infrastructure/.github/actions/trigger-workflow 
    with:
      # Note: This requires a PAT (Personal Access Token), GITHUB_TOKEN is often insufficient
      token: ${{ secrets.PAT_TOKEN }}
      
      # Target Repository Details
      owner: 'my-org'
      repo: 'cloud-infrastructure'
      branch: 'main'
      workflow_file: 'deploy.yml'
      
      # Optional: Inputs to pass to the target workflow (Must be JSON string)
      inputs: '{"environment": "prod", "version": "1.0.0"}'
      
      # Optional: Enable waiting behavior - default false
      wait_until_complete: true 
      
      # Optional: Manually provide an ID (otherwise one is auto-generated)
      correlation_id: ${{ github.run_id }}
```

### Receiving Repository Action

To allow the action to track the run, you must add the correlation_id input and map it to the run-name

```yaml
name: Trigger by
run-name: Workflow ${{ inputs.correlation_id}}

on:
  workflow_dispatch:
      correlation_id:
        description: 'Unique identifier for this run to track the workflow'
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: echo distinct ID ${{ github.event.inputs.correlation_id }}
        run: echo ${{ github.event.inputs.correlation_id }}
```
## ⚙️ Inputs

| Input | Description | Required | Default |
| :--- | :--- | :---: | :--- |
| `token` | A Personal Access Token (PAT) with `repo` scope. | **Yes** | |
| `owner` | The organization or user owning the target repository. | **Yes** | |
| `repo` | The name of the target repository. | **Yes** | |
| `workflow_file` | The filename of the workflow (e.g., `main.yml`). | **Yes** | |
| `branch` | The branch reference to run the workflow on. | No | `main` |
| `inputs` | A JSON string of inputs to pass to the workflow. | No | `{}` |
| `wait_until_complete`| If `true`, the action waits for the target workflow to finish. | No | `false` |
| `correlation_id` | A unique string to track the run. If not provided, the action generates a random ID automatically. | No | *(Auto)* |

## Token Requirements

To trigger workflows in other repositories, you need a Personal Access Token (PAT). The default GITHUB_TOKEN usually does not have permission to access other repositories.

Required Scopes:
- **repo** (Full control of private repositories)

  - If the target is public, public_repo might suffice.

  - If the target is private, the full repo scope is mandatory.

- **actions:read** (To poll for status)

- **actions:write** (To dispatch the workflow)

## APIs Used

For transparency, this action utilizes the following GitHub REST API endpoints:

1.  **[Create a workflow dispatch event](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event)**
    * `POST /repos/{owner}/{repo}/actions/workflows/{workflow_id}/dispatches`
2.  **[List workflow runs](https://docs.github.com/en/rest/actions/workflow-runs#list-workflow-runs-for-a-repository)**
    * `GET /repos/{owner}/{repo}/actions/workflows/{workflow_id}/runs`
3.  **[Get a workflow run](https://docs.github.com/en/rest/actions/workflow-runs#get-a-workflow-run)**
    * `GET /repos/{owner}/{repo}/actions/runs/{run_id}`

## Flow

```ascii
┌─────────────────┐
│                 │
│ Dispatch Action │
│                 │
│ with unique ID  │
│                 │
└───────┬─────────┘
        │
        │
        ▼                          ┌───────────────┐
┌────────────────┐                 │               │
│                │                 │ Check Run Name│
│ Request recent ├────────────────►│               │
│                │                 │ for ID match  │
│ workflow runs  │                 │               │
│                │◄────────────────┤               │
└───────┬────────┘     Retry       │               │
        │                          └───────┬───────┘
        │                                  │
Timeout │                                  │ Match Found!
        │                                  │
        ▼                                  ▼
     ┌──────┐                      ┌───────────────┐
     │ Fail │                      │ Poll Status   │
     └──────┘                      │ until done    │
                                   └───────────────┘
```