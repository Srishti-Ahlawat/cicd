# CI/CD Process

## Overview

We use a robust CI/CD pipeline to streamline Terraform and Terragrunt infrastructure management via GitHub Actions. This pipeline automates deployment across multiple environments (nonprod, preprod, prod), handling the full lifecycle of Terraform resources, from planning to applying and destroying. Reusable workflows and actions ensure consistency and efficiency across all deployments.

---

## Folder Structure

The following is an example structure of a environment layout with nonprod, preprod, and prod accounts.
```
.
├── CODEOWNERS                            # Standard Github CODEOWNERS file
├── README.md                             # Generic documentation
├── .github
│   ├── actions                    
│   │   ├── terraform-prerequisites                        
│   │   │   └── actions.yml               # reusable terraform prerequisite tasks for workflows
│   │   ├── terraform-apply                        
│   │   │   └── actions.yml               # reusable terragrunt apply task for workflows
│   │   ├── terraform-plan                        
│   │   │   └── actions.yml               # reusable terragrunt plan task for workflows
│   │   ├── terraform-destroy                        
│   │   │   └── actions.yml               # reusable terragrunt destroy task for workflows
│   └── workflows                    
│           ├── infra-deploy.yml          # workflow to deploy terraform plan and apply to multiple environments (uses terragrunt-plan-apply.yml as jobs)
│           └── terragrunt-plan-apply.yml # Reusable workflow for terraform plan and apply
├── commonvars.yaml                       # Platform-wide variables
├─── nonprod                              # Environment Folder
│   ├── envvars.yaml                      # Environment-specific variables
│   ├── terragrunt.hcl                    # Environment terragrunt config (provider details, remote state etc.)
│   ├── Application-01                    # Application specific folder
│   │     ├── resource-folder-01          # Folder for specific set of resources
│   │     │   └── terragrunt.hcl          # Module config
│   │     ├── resource-folder-02  
│   │     │   └── terragrunt.hcl          # Module config
│   │     ├── resource-folder-03
│   │     │   └── terragrunt.hcl          # Module config
│   │     └── resource-folder-04
│   │         └── terragrunt.hcl          # Module config
│   └── Application-02                    # Application specific folder
│         ├── resource-folder-01          # Folder for specific set of resources
│         │   └── terragrunt.hcl          # Module config
│         └── resource-folder-02  
│              └── terragrunt.hcl         # Module config
├── preprod                               # Same general layout as nonprod
└── prod                                  # Same general layout as nonprod
```

## CI/CD Pipeline Process

### 1. **Terraform Prerequisites**

Before executing any Terraform or Terragrunt commands, the pipeline runs a prerequisite setup to ensure the necessary tools are installed and the environment is configured correctly. This setup is crucial for maintaining code quality and ensuring a smooth deployment process.

- **Set up TFLint**: The pipeline uses the `terraform-linters/setup-tflint@v1` action to install TFLint, a linter that checks for potential issues in Terraform code.
  
- **Initialize TFLint**: After installation, the pipeline initializes TFLint using the command `tflint --init`. This step prepares TFLint for use by setting up any necessary configuration files.

- **Run TFLint**: Finally, the pipeline runs TFLint with the command `tflint` to analyze the Terraform code for potential issues. This ensures that any code quality problems are identified before planning or applying changes.

- **Environment Setup**: The pipeline also ensures that all required authentication and provider setup steps are performed, laying the groundwork for further actions in the CI/CD process.


### 2. **Plan Stage**

The plan stage of the pipeline is crucial for generating a Terraform plan that outlines the proposed changes to the infrastructure. This stage utilizes Terragrunt to perform several key tasks:

- **Initialize Infrastructure**: The pipeline runs `terragrunt init` to set up the necessary backend and provider configurations. This prepares the environment for subsequent operations.

- **Validate Configuration**: It validates the Terraform configuration using the command `terragrunt run-all validate --terragrunt-ignore-dependency-errors`. This step ensures that the configuration files are syntactically correct and adheres to best practices, helping to catch potential issues early.

- **Generate a Detailed Plan**: Finally, the pipeline runs `terragrunt run-all plan -no-color -input=false --terragrunt-out-dir /tmp/tfplan` to create a detailed plan of what changes will be applied to the infrastructure. This plan is stored in a temporary directory and provides a clear overview of the proposed modifications.

The plan stage is automatically triggered when a pull request is created. This allows reviewers to see exactly what changes will be made once the pull request is merged, ensuring transparency and facilitating informed decision-making.

### 3. **Apply Stage**

After a pull request has been reviewed and merged into the main branch, the apply stage of the pipeline executes (if configured to do so). This stage takes the approved Terraform plan and applies it to the specified environment, making the infrastructure changes live.

The apply stage consists of the following key steps:

- **Initialize Infrastructure**: The pipeline begins by running `terragrunt init --terragrunt-non-interactive`, which sets up the necessary backend and provider configurations in a non-interactive manner. This step is essential for preparing the environment for the apply operation.

- **Validate Configuration**: Before applying the changes, the pipeline validates the Terraform configuration again using `terragrunt run-all validate --terragrunt-ignore-dependency-errors`. This step ensures that the configuration remains correct and ready for deployment.

- **Generate Plan**: The pipeline runs `terragrunt run-all plan -no-color -input=false --terragrunt-out-dir /tmp/tfplan` to create the plan file, which outlines the specific changes to be made.

- **Apply Changes**: Finally, if the workflow input `do_apply` is set to `true`, the pipeline executes `terragrunt run-all apply --terragrunt-non-interactive --terragrunt-out-dir /tmp/tfplan`. This command applies the changes to the infrastructure as defined in the plan.

The flexibility of the apply stage allows it to be conditionally triggered based on the `do_apply` input, enabling teams to decide whether to automatically apply changes or review the plan before proceeding. This ensures that infrastructure changes are made with the necessary oversight and control.

### 4. **Environment-Specific Deployments**

The pipeline is designed to work across multiple environments (e.g., `nonprod`, `preprod`, and `prod`). Each environment is isolated in its own folder with specific configuration files (e.g., `envvars.yaml` and `terragrunt.hcl`). These files allow each environment to be uniquely configured while following the same process for planning and applying changes.

This structure enables seamless promotion of infrastructure changes from non-production environments to production environments, ensuring consistency and reducing the risk of environment-specific issues.

### Triggering the Pipeline

The CI/CD pipeline is triggered automatically in the following scenarios:

- **Pull Requests**: A Terraform plan is generated whenever a pull request is opened, providing visibility into the proposed infrastructure changes.
- **Merges to Main**: After a pull request is approved and merged into the main branch, the apply stage can be triggered to deploy the approved changes.
- **Manual Triggers**: The pipeline allows for manual triggering of apply or destroy stages when needed, providing flexibility in managing the infrastructure lifecycle.

---

## Repository Structure

The repository follows a modular structure to organize infrastructure code by environment and application:

- **.github/actions**: Contains reusable actions for different stages of the pipeline (e.g., prerequisites, planning, applying).
- **.github/workflows**: Contains workflows for deploying infrastructure (e.g., plan, apply, and destroy).
- **commonvars.yaml**: Contains platform-wide variables that are shared across all environments.
- **Environment folders** (`nonprod`, `preprod`, `prod`): Each environment has its own set of configuration files (`envvars.yaml`, `terragrunt.hcl`) and application-specific folders. This allows for fine-grained control of infrastructure across different stages.

---

## Key Features

- **Reusable GitHub Actions**: The pipeline uses modular and reusable GitHub Actions, making it easy to maintain and scale. These actions handle everything from setting up prerequisites to planning and applying infrastructure.
  
- **Environment-Specific Configuration**: Each environment (nonprod, preprod, and prod) has its own configuration files, ensuring that infrastructure changes are appropriately scoped and managed across different stages of deployment.

- **Plan and Apply Separation**: The CI/CD process is designed to separate the planning and applying stages. Plans are automatically generated on pull requests, and changes are only applied after approval and merging, adding a layer of control and review.

- **Manual or Automated Apply**: The `do_apply` input in the pipeline allows for flexibility in how infrastructure changes are applied. You can choose to run the apply stage manually or automate it upon merge, depending on your team's workflow.

- **Support for Destroying Resources**: In addition to deploying infrastructure, the pipeline supports destroying resources when necessary, giving you full lifecycle management of your Terraform infrastructure.

---

## Workflow Summary

1. **Pull Request**: A plan is generated when a pull request is opened, showing the proposed changes to the infrastructure.
2. **Review and Merge**: The plan is reviewed, and once approved, the pull request is merged into the main branch.
3. **Apply Changes**: Depending on the configuration, the changes are automatically applied to the target environment after the merge.
---
