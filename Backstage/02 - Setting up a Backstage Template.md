# Setting up a Backstage Template to Trigger Github Actions Workflow & Provision Terraform Resources from the Developer Portal

## Introduction

This documentation outlines a process for enabling developers to trigger CI/CD pipelines directly from the Backstage developer portal. By leveraging Backstage templates, GitHub Actions, and Terraform, we create a streamlined workflow that allows team members to initiate infrastructure provisioning or destruction with minimal friction. This solution is designed for internal use, providing a user-friendly interface for managing CI/CD processes without requiring deep knowledge of the underlying infrastructure.

## Prerequisites

To implement this solution, you'll need:

1. A running **Backstage** installation as your developer portal
2. **Terraform** for defining infrastructure as code
3. **GitHub** for source control and running GitHub Actions

## Repository Structure

```sh
.
├── infrastructure
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
├── template.yaml
└── .github
    └── workflows
        └── random-tf.yml
```

## Backstage Template Breakdown

Let's examine the `template.yaml` file in detail:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: random-tf
  title: Provision a random tf resource
  description: Provision a random tf resource
spec:
  owner: group:devops
  type: service
```

This section defines the template's metadata. It specifies the API version, declares this as a Template kind, and provides basic information like name, title, and description. The `spec` section begins by defining the owner and type of the service.

```yaml
  parameters:
    - title: Project Details
      required:
        - name
        - description
      properties:
        name:
          title: Project Name
          type: string
          description: Unique name. No spaces. No symbols.
          maxLength: 50
          ui:autofocus: true
        description:
          title: Project Description
          type: string
          description: A short description
```

This part defines the user inputs for project details. It requires a name and description, with specific constraints on the name (no spaces or symbols, maximum 50 characters).

```yaml
    - title: Action
      required:
        - action
      properties:
        action:
          title: Action
          type: string
          description: Action to perform (apply/destroy)
          enum:
            - apply
            - destroy
```

This section defines another user input for selecting the action to perform (either 'apply' or 'destroy').

```yaml
  steps:
    - id: github-action
      name: Trigger GitHub Action
      action: github:actions:dispatch
      input:
        workflowId: random-tf.yml
        repoUrl: "github.com?repo=backstage-templates&owner=LearnWithHomer"
        branchOrTagName: "main"
        workflowInputs:
          action: ${{ parameters.action }}
```

The `steps` section defines the actions to be taken when the template is used. In this case, it triggers a GitHub Action. The `github:actions:dispatch` action is used to start a workflow in the specified GitHub repository. The `action` parameter from the user input is passed to the workflow. `workflowId` is the name of the workflow file in the `.github/workflows` directory. `repoUrl` is the URL of the GitHub repository. `branchOrTagName` is the branch or tag to use. `workflowInputs` is a map of inputs to pass to the workflow.

## GitHub Actions Workflow

```yaml
# .github/workflows/random-tf.yml

name: Run Simple Random TF
on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Action to perform (apply/destroy)'
        required: true
env:
  TFPATH: 'simple-ec2-template/infrastructure'
jobs:
  apply_instance:
    runs-on: ubuntu-latest
    if: ${{ github.event.inputs.action == 'apply' }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
    - name: Terraform Init
      run: terraform init
      working-directory: ${{env.TFPATH}}
    - name: Terraform Format
      run: terraform fmt
      working-directory: ${{env.TFPATH}}
    - name: Terraform Validate
      run: terraform validate
      working-directory: ${{env.TFPATH}}
    - name: Terraform Apply
      run: terraform apply -auto-approve
      working-directory: ${{env.TFPATH}}
  destroy_instance:
    runs-on: ubuntu-latest
    if: ${{ github.event.inputs.action == 'destroy' }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
    - name: Terraform Init
      run: terraform init
      working-directory: ${{env.TFPATH}}
    - name: Terraform FMT
      run: terraform fmt
      working-directory: ${{env.TFPATH}}
    - name: Terraform Validate
      run: terraform validate
      working-directory: ${{env.TFPATH}}
    - name: Terraform Destroy
      run: terraform destroy -auto-approve
      working-directory: ${{env.TFPATH}}
```

The GitHub Actions workflow (`random-tf.yml`) is set up to run Terraform commands based on the action specified (apply or destroy). It uses the `workflow_dispatch` event to allow manual triggering and accepts an `action` input.

Key points of the workflow:

1. It defines two jobs: `apply_instance` and `destroy_instance`, which run based on the `action` input.
2. Both jobs follow similar steps: checkout code, setup Terraform, initialize, format, and validate.
3. The `apply_instance` job runs `terraform apply`, while the `destroy_instance` job runs `terraform destroy`.
4. All Terraform commands are run in the directory specified by the `TFPATH` environment variable.

## Terraform Configuration

```hcl
# infrastructure/main.tf

provider "random" {}

resource "random_string" "example" {
  length  = 16
  special = true
}

resource "random_integer" "example" {
  min = 1
  max = 100
}

resource "random_pet" "example" {
  length = 3
}

output "random_string" {
  value = random_string.example.result
}

output "random_integer" {
  value = random_integer.example.result
}

output "random_pet_name" {
  value = random_pet.example.id
}
```

This example Terraform configuration (`main.tf`) in the `infrastructure/` directory defines random resources:

1. A random string of 16 characters
2. A random integer between 1 and 100
3. A random pet name with 3 words

These resources are used as examples and can be replaced with actual infrastructure resources as needed for your specific use case.

## Conclusion

This setup allows developers to trigger infrastructure provisioning or destruction directly from the Backstage UI. By selecting the template and providing basic inputs, they can initiate a GitHub Actions workflow that runs Terraform commands. This approach simplifies infrastructure management, integrates it into the developer portal, and maintains the benefits of infrastructure-as-code practices.