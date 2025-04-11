# GitHub Actions Retry Workflow

This repository contains a GitHub Actions workflow that demonstrates how to implement a job retry mechanism with matrix strategy. The workflow allows jobs to be retried up to 3 times, skipping remaining attempts if any previous one succeeds.

## How It Works

The workflow consists of two main jobs:

1. **`retry` job**: Uses a matrix strategy to attempt a task up to 3 times
2. **`follow-up` job**: Runs only if at least one of the retry attempts succeeds

### Key Features

- Automatically retries failed jobs up to 3 times
- Skips remaining attempts if any attempt succeeds
- Ensures sequential execution of retry attempts with max-parallel: 1
- Configurable via parameters to control which attempt should succeed (for testing)
- Uses artifact-based communication between job attempts
- Follow-up jobs only run if at least one attempt succeeded

## Workflow Structure

```yml
jobs:
    retry:
        # Matrix job with 3 attempts
        strategy:
            max-parallel: 1  # Ensures sequential execution
            matrix:
                attempt: [1, 2, 3]
        continue-on-error: true
        
    follow-up:
        # Only runs if at least one retry succeeded
        needs: retry
```

### How the Retry Logic Works

1. **Sequential Execution**: Matrix jobs run one at a time (not in parallel) thanks to `max-parallel: 1`
2. **Skip Detection**: Each attempt checks if any previous attempt has succeeded by looking for a "success-artifact"
3. **Early Exit**: If a success artifact is found, the job exits with code 78 (neutral/skipped status)
4. **Task Execution**: If no previous success, the job attempts to run the actual task
5. **Success Recording**: If the job succeeds, it creates and uploads a success-artifact
6. **Failure Handling**: If a job fails, the next matrix attempt will run automatically

### How the Follow-up Job Works

The follow-up job:
1. Depends on the retry job (`needs: retry`)
2. Checks for the existence of the success-artifact
3. Runs its tasks only if the artifact exists (indicating at least one success)
4. Fails with exit code 1 if no attempt succeeded

## How to Run/Test

### Local Testing with `act`

You can test this workflow locally using [act](https://github.com/nektos/act):

```bash
# Run the entire workflow
act

# Run with a specific passing attempt
act -e event.json
```

Create an `event.json` file to simulate workflow dispatch with parameters:

```json
{
  "inputs": {
    "passing_attempt": "1"
  }
}
```

### Running on GitHub

1. Push the workflow to your repository
2. Go to Actions tab in your GitHub repository
3. Select "Retry Matrix Workflow"
4. Click "Run workflow"
5. Choose which attempt should succeed (1, 2, or 3)
6. Click "Run workflow" button

## Customizing the Workflow

### Replacing the Task Logic

To replace the task logic with your own:

1. Edit the `.github/workflows/retry-jobs.yml` file
2. Locate the "Do stuff" step:

```yml
- name: Do stuff
  id: do-stuff
  run: |
    echo "doing stuff, attempt number ${{ matrix.attempt }}"
    # Replace this with your own logic
    if [ "${{ matrix.attempt }}" != "${{ github.event.inputs.passing_attempt || '2' }}" ]; then
        echo "Attempt ${{ matrix.attempt }} fails because it's not the designated passing attempt"
        exit 1
    fi
```

3. Replace the script content with your own task logic
4. Keep the success/failure pattern or implement your own retry criteria

### Adding More Steps

You can add additional steps to both the retry job and follow-up job:

```yml
- name: Your Custom Step
  run: |
    # Your custom code here
```

### Success Criteria

The current workflow records success when the configured attempt number runs. To change the success criteria:

1. Modify the `Do stuff` step to use your own success/failure conditions
2. Update the `record-success` step to create the success.txt file based on your criteria

## Best Practices

1. Keep retry attempts reasonable (3-5 max)
2. Use `continue-on-error: true` for retry jobs
3. Set appropriate timeouts for each attempt
4. Consider exponential backoff between retries for network-related issues
5. Use clear success criteria for determining when to stop retrying

## Troubleshooting

- **All attempts fail**: Check your task logic and failure conditions
- **Subsequent attempts aren't skipped**: Ensure artifact upload/download is working
- **Follow-up job doesn't run**: Check permissions and artifact existence
- **GitHub API rate limits**: Add authentication token if hitting rate limits