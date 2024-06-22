---
title: "Azure DevOps Terraform Pipeline"
date: 2024-06-22T07:38:40Z
draft: false
keywords: "azure-devops, terraform, iac, azure, devops"
description: "How to provision cloud infrastructure with Terraform and Azure DevOps."
Summary: "
![](/images/azure-devops-terraform-pipeline/logo.png)
I was recently working on a project that required me to create a Terraform pipeline in Azure DevOps. I had never done this before, so I had to do some research to figure out how to set it up. In this article, I will share the final pipeline that I created, as well as some of the resources that I found helpful along the way."
---
![](/images/azure-devops-terraform-pipeline/logo.png)

I was recently working on a project that required me to create a Terraform pipeline in Azure DevOps. I had never done this before, so I had to do some research to figure out how to set it up. In this article, I will share the final pipeline that I created, as well as some of the resources that I found helpful along the way.

## Preconditions
- I will use __'ubuntu-latest'__ as the agent pool for this pipeline. Because I use Ubuntu in other CI/CD tool like GitHub Actions and GitLab CI/CD, I am familiar with the commands and tools available in the Ubuntu environment. If you are using a different agent pool, you may need to adjust the commands accordingly.
- Terraform installation script can be found [here](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli#install-terraform). I will use this script to install Terraform on the agent machine.
- My Azure DevOps uses [Service Connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml) to authenticate with Azure. In my case the Service Connection uses [Identity Federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) as the authentication method.
- Terraform pipeline steps from [marketplace](https://marketplace.visualstudio.com/search?term=terraform&target=AzureDevOps&category=Azure%20Pipelines&sortBy=Relevance) are not available for my use case.

## Terraform Pipeline
``` yaml
stages:
  - stage: Terraform
    jobs:
    - job: Terraform
      pool:
        vmImage: 'ubuntu-latest'
      steps:
      - task: CmdLine@2
        displayName: 'Terraform Install'
        inputs:
          script: |
# https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli#install-terraform
            sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
            wget -O- https://apt.releases.hashicorp.com/gpg | \
              gpg --dearmor | \
              sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
            echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
              https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
              sudo tee /etc/apt/sources.list.d/hashicorp.list
            sudo apt update
            sudo apt-get install terraform
            terraform --version

      - task: AzureCLI@2
        displayName: 'Terraform Plan'
        inputs:
          addSpnToEnvironment: true
          azureSubscription: MyServiceConnection
          scriptType: bash
          scriptLocation: inlineScript
          workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure'
          inlineScript: |
# https://devblogs.microsoft.com/devops/public-preview-of-workload-identity-federation-for-azure-pipelines/#azure-cli-task-support-for-inline-authentication
            export ARM_CLIENT_ID=$servicePrincipalId
            export ARM_OIDC_TOKEN=$idToken
            export ARM_TENANT_ID=$tenantId
            export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
            export ARM_USE_OIDC=true

            terraform init \
              -var-file="./env/dev/input.tfvars" \
              -backend-config="./env/dev/backend.tfvars"

            terraform plan \
              -var-file="./env/dev/input.tfvars" \
              -input=false
```

## Links
- Microsoft documentation: [AzureCLI@2 - Azure CLI v2 task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-cli-v2?view=azure-pipelines) Azure CLI task documentation. In particular __addSpnToEnvironment__ parameter that allows to get azure connection details from the service connection.
- Terraform documentation: [Azure Provider: Authenticating using a Service Principal with Open ID Connect](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/service_principal_oidc)
- Medium article: [Terraform in Azure Devops without additional tasks](https://medium.com/@maciej.skorupka/terraform-in-azure-devops-without-additional-tasks-8f29434e6e2)
- Microsoft blog post: [Public preview of Workload identity federation for Azure Pipelines](https://devblogs.microsoft.com/devops/public-preview-of-workload-identity-federation-for-azure-pipelines/)