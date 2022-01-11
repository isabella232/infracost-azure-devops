# Terragrunt

This example shows how to run Infracost Azure Devops tasks with Terragrunt.

[//]: <> (BEGIN EXAMPLE)
```yml
pr:
  - master

variables:
  API_KEY: $(apiKey)
  
jobs:
  - job: terragrunt
    displayName: Terragrunt
    pool: 
      vmImage: ubuntu-latest

    steps:
      - task: TerraformInstaller@0
        displayName: Install Terraform

      - bash: |
          INSTALL_LOCATION="$(Build.SourcesDirectory)/bin"
          mkdir -p ${INSTALL_LOCATION}

          curl -sL "https://github.com/gruntwork-io/terragrunt/releases/latest/download/terragrunt_linux_amd64" > ${INSTALL_LOCATION}/terragrunt
          chmod +x ${INSTALL_LOCATION}/terragrunt

          echo "##vso[task.setvariable variable=PATH;]$(PATH):${INSTALL_LOCATION}"

        displayName: Setup Terragrunt

      - task: InfracostSetup@0
        displayName: Setup Infracost
        inputs:
          apiKey: $(API_KEY)

      - bash: infracost breakdown --path=examples/terragrunt/code --format=json --out-file=/tmp/infracost.json
        displayName: Run Infracost
        
      - task: InfracostComment@0
        displayName: Post the comment
        inputs:
          path: /tmp/infracost.json
          behavior: update # Create a single comment and update it. See https://github.com/infracost/infracost-azure-devops#comment-options for other options
```
[//]: <> (END EXAMPLE)