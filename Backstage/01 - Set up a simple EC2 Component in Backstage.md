# Simple Backstage POC - Enable Developers Provision EC2 Instances

## Motivation

Providing a good developer experience (DevEx) is crucial for enabling productive software teams. DevEx refers to the systems, processes, tools, and culture that impact a developer's effectiveness. A key aspect of DevEx is minimizing toil and giving developers seamless access to the resources they need to build and ship software.

This documentation, will walk you through leveraging Backstage as a developer portal, Terraform for infrastructure as code, and GitHub Actions for automating environment provisioning. By combining these tools, we can create a self-service workflow for developers to spin up EC2 instances through a simple UI, without needing to manually provision infrastructure.

## Prereqs

To begin, we'll need:

1. A running **Backstage** installation as the developer portal (<https://backstage.io>)
2. **Terraform** for defining infrastructure as code
3. The **GitHub** ecosystem (repositories, Actions) for source control and automation

## Recipe

Our goal is to offer one or more templates in the Backstage developer portal. When a developer selects a template, they provide basic inputs about the project. This will then trigger:

- Creation of a new GitHub repository with a predefined layout and starter content
- Provisioning of the required infrastructure resources (e.g. EC2 instances) via Terraform
- Registration of the new project in the Backstage catalog with links to the source repo

Let's break this down into steps:

### GitHub Setup

We'll create a GitHub repository containing templates for different project setups. This repo will include:

- A sample project structure
- Terraform configurations for provisioning infrastructure
- GitHub Actions workflows for CI/CD and governance

The specifics like project structure, frameworks, and tooling will align with your specific needs and best practices.

Here's the final file structure for this walk-through:

```sh
repo/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── manage-ec2.yaml
├── docs/
│   └── index.md
├── infrastructure/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── src/
│   └── ... 
├── catalog-info.yaml
├── mkdocs.yaml
├── README.md
└── template.yaml
```

### Backstage Setup

Next, we'll configure a software template in Backstage to represent the EC2 provisioning workflow. The template definition includes:

- Metadata (name, description, tags)
- User input parameters (e.g. project name, EC2 instance type)
- Scaffolding actions to execute when the template is instantiated

Here's an example `templates.yaml` file defining the template:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: ec2-instance
  title: Provision an AWS EC2 Instance
  description: Provision AWS EC2 Instance using Terraform
spec:
  owner: user:mrdankuta
  type: service

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
          pattern: '^[a-zA-Z0-9_-]+$'
        description:
          title: Project Description
          type: string
          description: A short description
    - title: EC2 Configuration
      required:
        - instanceName
        - instanceType
        - region
        - action
      properties:
        instanceName:
          title: Instance Name
          type: string
          description: The name of your EC2 instance
          ui:autofocus: true
        instanceType:
          title: EC2 Instance Type
          type: string
          enum:
            - 't2-micro'
            - 't2-small'
        region:
          title: AWS Region
          type: string
          description: The AWS region where the instance will be deployed
          enum:
            - us-east-1
            - us-west-2
        action:
          title: Action
          type: string
          description: Action to perform (apply/destroy)
          enum:
            - apply
            - destroy

  steps:
    - id: fetch-base
      name: Fetch Base
      action: fetch:template
      input:
        url: https://github.com/Makinates/smpls-ec2-tmplt
        copyWithoutTemplating:
          - .github/**/*.yaml
          - infrastructure/*.tf
          - src/*.*
        values:
          name: ${{ parameters.name }}
    # This step publishes the contents of the working directory to GitHub.
    - id: publish
      name: Publish
      action: publish:github
      input:
        allowedHosts: ['github.com']
        repoVisibility: public
        defaultBranch: 'main'
        description: This is ${{ parameters.name }}
        repoUrl: 'github.com?repo=${{ parameters.name }}&owner=Makinates'
    # Start a GitHub Action to build an EC2 instance with Terraform
    - id: github-action
      name: Trigger GitHub Action
      action: github:actions:dispatch
      input:
        workflowId: manage-ec2.yaml
        repoUrl: 'github.com?repo=${{ parameters.name }}&owner=Makinates'
        branchOrTagName: 'main'
        workflowInputs:
          instanceName: ${{ parameters.instanceName }}
          instanceType: ${{ parameters.instanceType }}
          awsRegion: ${{ parameters.region }}
          action: ${{ parameters.action }}
    # The final step is to register our new component in the catalog.
    - id: register
      name: Register
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

  # Outputs are displayed to the user after a successful execution of the template.
  output:
    links:
      - title: Repository
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: Open in catalog
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}

```

To make the template repository register each instance created a component we must add a `catalog-info.yaml` file to our GitHub repository. It will contain the metatdata and annotations necessary for Backstage to create the component, view CI/CD pipelines, display documentation, and much more:  

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name | dump }}
  annotations:
    github.com/project-slug: Makinates/${{ values.name }}
    github.com/actions-workflow: manage-ec2.yaml
    backstage.io/techdocs-ref: dir:.
  description: ${{ values.name }} running on AWS EC2 instance managed by Terraform
spec:
  type: service
  owner: user:mrdankuta
  lifecycle: experimental
```

We'll add this template configuration to the app-config.yaml file to make it visible in the Backstage portal.

```yaml
catalog:
  ...
  locations: 
  ...
    - type: url
      target: https://github.com/Makinates/smpls-ec2-tmplt/blob/main/template.yaml
      rules:
        - allow: [Template]
```

### GitHub Actions & Terraform  

The GitHub Actions workflow triggered by Backstage will handle the Terraform execution for provisioning the EC2 instances.

```yaml
# .github/workflows/manage-ec2.yaml

name: Manage EC2 Instance

on:
  workflow_dispatch:
    inputs:
      instanceName:
        description: 'Name of the EC2 Instance'
        required: true
      instanceType:
        description: 'Type of the EC2 Instance'
        required: true
      awsRegion:
        description: 'AWS Region for the instance'
        required: true
      action:
        description: 'Action to perform (apply/destroy)'
        required: true

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
      working-directory: infrastructure

    - name: Terraform Format
      run: terraform fmt
      working-directory: infrastructure

    - name: Terraform Validate
      run: terraform validate
      working-directory: infrastructure

    - name: Terraform Apply
      run: terraform apply -var instanceName=${{ github.event.inputs.instanceName }} -var instanceType=${{ github.event.inputs.instanceType }} -var awsRegion=${{ github.event.inputs.awsRegion }} -auto-approve
      working-directory: infrastructure

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
      working-directory: infrastructure

    - name: Terraform FMT
      run: terraform fmt
      working-directory: infrastructure

    - name: Terraform Validate
      run: terraform validate
      working-directory: infrastructure
      
    - name: Terraform Destroy
      run: terraform destroy -var instanceName=${{ github.event.inputs.instanceName }} -var instanceType=${{ github.event.inputs.instanceType }} -var awsRegion=${{ github.event.inputs.awsRegion }} -auto-approve
      working-directory: infrastructure
```

The Terraform configuration in `infrastructure/` will define the resources to provision:

```hcl
# infrastructure/main.tf

provider "aws" {
  region = var.awsRegion
}

resource "aws_instance" "example" {
  ami           = "ami-0c94855ba95c71c99" # Amazon Linux 2 AMI (HVM), SSD Volume Type
  instance_type = var.instanceType

  vpc_security_group_ids = [
    aws_security_group.example.id
  ]

  user_data = <<-EOF
    #!/bin/bash
    # Install Docker
    amazon-linux-extras install docker -y
    systemctl start docker
    usermod -a -G docker ec2-user

    # Pull and run the Docker container
    docker run -d -p 80:3000 YOUR_DOCKER_IMAGE
  EOF

  tags = {
    Name = "example-instance"
  }
}

resource "aws_security_group" "example" {
  name        = "example-security-group"
  description = "Allow inbound traffic on port 80"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}


# infrastructure/variables.tf 

variable "awsRegion" {}
variable "instanceName" {}
variable "instanceType" {}


# infrastructure/output.tf

output "public_ip" {
  value       = aws_instance.example.public_ip
  description = "The public IP address of the EC2 instance"
}
```

## Putting it Together

With this setup, a developer can access the Backstage portal, select the "Provision EC2" template, and provide the required inputs like:

- Project name
- Project description  
- EC2 instance name
- EC2 instance type
- AWS region

Upon submission, Backstage will:

1. Create a new GitHub repo with the starter content
2. Trigger the `manage-ec2.yaml` GitHub Actions workflow
3. Register the new project in the Backstage catalog

The GitHub Actions workflow will:

1. Check out the new repo
2. Setup Terraform
3. Run `terraform init`
4. Run `terraform apply` with the user inputs to create the EC2 instances

Finally, the developer has a configured repo with running EC2 instances, able to start developing immediately.

## Conclusion

By combining Backstage, Terraform, and GitHub Actions, we've established a self-service model for provisioning cloud resources like EC2 instances. Developers can rapidly instantiate new projects built
