## How to create a pre-commit for terraform locally 

Pre-commit hooks are essential scripts that automatically execute before committing your code changes. They serve a critical role in identifying various types of issues, including linting errors, security vulnerabilities, and formatting inconsistencies. This robust pre-commit process ensures the highest quality and safety of your code, making it ready for deployment.


## Now, let’s explore some of the most popular pre-commit hooks tailored for Terraform:

1. terraform-docs: This hook meticulously inspects your Terraform configuration files, detecting and correcting documentation errors.
(Follow documentation for installation)[https://github.com/terraform-docs/terraform-docs]

1. tflint: With this hook, your Terraform configurations undergo a thorough linting process to identify and rectify errors.
(https://github.com/terraform-linters/tflint)[Follow documentation for installation]

1. tfsec: Ensuring security is paramount, and tfsec specializes in scanning Terraform configurations for potential vulnerabilities.
(https://github.com/aquasecurity/tfsec)[Follow documentation for installation]

1. Terraform Fmt: is a Terraform command that is available natively that is used to rewrite Terraform configuration files to a canonical format and style. This command applies a subset of the Terraform language style conventions, along with other minor adjustments for readability.
(https://developer.hashicorp.com/terraform/cli/commands/fmt)[Follow documentation for installation]

1. Terraform Trivy: Trivy is a comprehensive and versatile security scanner. Trivy has scanners that look for security issues, and targets where it can find those issues.
(https://github.com/aquasecurity/trivy)[Documentation of Trivy]

1. checkov: This hook evaluates your Terraform configurations against a predefined set of security best practices, ensuring robust security posture.
(https://github.com/bridgecrewio/checkov)[Documentation of checkov]

## To globally install the pre-commit hook and configure it for use with Terraform, follow these steps:
 
 1. Install Pre-Commit Globally (Not Needed if Using Docker Image):

    ```bash
      DIR=~/.git-template
      git config --global init.templateDir ${DIR}
      pre-commit init-templatedir -t pre-commit ${DIR}
    ```
2. Add Configurations and Hooks:

    Navigate to the repository where you want to set up the pre-commit hooks, and perform the following steps:    
    - create a file named .pre-commit-config.yaml
    - Add this 

    ```yaml
         repos:
             - repo: https://github.com/antonbabenko/pre-commit-terraform
               rev: v1.89.0
               hooks:
                - id: terraform_fmt
                - id: terraform_tflint
                - id: terraform_trivy
                - id: terraform_docs
                  args:
                    - --hook-config=--path-to-file=tf-readme\README.md       # Valid UNIX path. I.e. ../TFDOC.md or docs/README.md etc.
                    - --hook-config=--add-to-existing-file=true     # Boolean. true or false
                    - --hook-config=--create-file-if-not-exist=true # Boolean. true or false
                    - --hook-config=--use-standard-markers=true
    ```        
3. run `pre-commit run -a` 