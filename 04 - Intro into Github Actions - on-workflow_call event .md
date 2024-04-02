
# Create a Simple workflow Using On Workflow_call Event

For a workflow to be reusable, the values for on must include workflow_call.

## Using inputs and secrets in a reusable workflow
You can define `inputs` and `secrets`, which can be passed from the caller workflow and then used within the called workflow. There are three stages to using an `input` or a `secret` in a reusable workflow.

1. In the reusable workflow, use the `inputs` and `secrets` keywords to define inputs or secrets that will be passed from a caller workflow.

2. In the reusable workflow, reference the `input` or `secret` that you defined in the on key in the previous step.

3. Pass the input or secret from the caller workflow.

   To provide named inputs to a called workflow, utilize the `with` keyword within a job. Use the `secrets` keyword to pass named secrets. For `inputs`, the data type of the input value must match the type specified in the called workflow, which can be boolean, number, or string.

## Example reusable workflow

    ```yml
    name: Reusable Workflow

    on:
      workflow_call:
        inputs:
          message:
            required: true
            description: 'Message to display'

    jobs:
      display_message:
        runs-on: ubuntu-latest
        
        steps:
          - 
            name: Display Message
            run: echo "${{ inputs.message }}"
    ```        

## Calling a reusable workflow
You call a reusable workflow by using the uses keyword. Unlike when you are using actions within a workflow, you call reusable workflows directly within a job, and not from within job steps.

``jobs.<job_id>.uses``

Reusable workflow files are referenced using one of the following formats:

- `{owner}/{repo}/.github/workflows/{filename}@{ref}` for reusable workflows in public and private repositories. Here, `{ref}` can be a SHA, a release tag, or a branch name.
- `./.github/workflows/{filename}` for reusable workflows in the same repository.
_Congratulations! You have successfully created a simple Github Actions using the on workflow_call event._