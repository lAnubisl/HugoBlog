---
title: "How to Provision Azure OpenAI with Pulumi in C#"
date: 2025-04-17T18:50:13Z
draft: false
keywords: "azure, azure-openai, pulumi"
summary: "![](/images/azure-openai-pulumi/logo.png)
With the rise of AI-powered applications, developers and organizations are increasingly looking to integrate large language models into their solutions. Azure OpenAI Service provides access to powerful models like GPT-4, Codex, and DALL·E, backed by Azure’s infrastructure. In this guide, we'll walk you through provisioning Azure OpenAI resources using Pulumi, a modern Infrastructure as Code (IaC) tool that supports multiple languages including TypeScript, Python, Go, and C#."
---
![](/images/azure-openai-pulumi/logo.png)

With the rise of AI-powered applications, developers and organizations are increasingly looking to integrate large language models into their solutions. Azure OpenAI Service provides access to powerful models like GPT-4, Codex, and DALL·E, backed by Azure’s infrastructure. In this guide, we'll walk you through provisioning Azure OpenAI resources using Pulumi, a modern Infrastructure as Code (IaC) tool that supports multiple languages including TypeScript, Python, Go, and C#.

## Prerequisites
To get started, ensure you have:

- An active Azure subscription
- Azure CLI installed and logged in (az login)
- .NET SDK installed
- Pulumi CLI installed
- A Pulumi C# project set up (run pulumi new azure-csharp)

## Step-by-Step: Provision Azure OpenAI with Pulumi in C#

### Step 1: Add Azure Native NuGet Package

In your project directory, install the Azure Native Pulumi package:

``` console
dotnet add package Pulumi.AzureNative
```

### Step 2: Create a Resource Group

In your MyStack.cs file (or wherever your stack logic lives), define a new [resource group](https://www.pulumi.com/registry/packages/azure-native/api-docs/resources/resourcegroup/):
``` csharp
Pulumi.Config config = new Pulumi.Config();

Azure.Resources.ResourceGroup rg = new Azure.Resources.ResourceGroup(
    "rg", 
    new Azure.Resources.ResourceGroupArgs
    {
        ResourceGroupName = "rg-my-resource-group",
        Location = config.Require("location"),
    }
);
```

### Step 3: Create a Cognitive Services Account

The Azure OpenAI resource is provisioned as a type of [Cognitive Services Account](https://www.pulumi.com/registry/packages/azure-native/api-docs/cognitiveservices/account/) with the kind set to "OpenAI".

``` csharp
Azure.CognitiveServices.Account account = new Azure.CognitiveServices.Account(
    "openai",
    new()
    {
        ResourceGroupName = rg.Name,
        Location = rg.Location,
        AccountName = "openai-my-openai-account",
        Kind = "OpenAI",
        Sku = new Azure.CognitiveServices.Inputs.SkuArgs
        {
            Name = "S0",
        }
    }
);
```

### Step 4: Create Model Dployment

We need to define a [deployment for the OpenAI model](https://www.pulumi.com/registry/packages/azure-native/api-docs/cognitiveservices/deployment/). This is where you specify the model type, version, and other properties.

``` csharp
Azure.CognitiveServices.Deployment deployment = new Azure.CognitiveServices.Deployment(
    "openai-deployment", 
    new()
    {
        ResourceGroupName = rg.Name,
        AccountName = account.Name,
        Sku = new Azure.CognitiveServices.Inputs.SkuArgs
        {
            Name = "GlobalStandard",
            Capacity = 50
        },
        Properties = new Azure.CognitiveServices.Inputs.DeploymentPropertiesArgs
        {
            Model = new Azure.CognitiveServices.Inputs.DeploymentModelArgs
            {
                Format = "OpenAI",
                Name = "gpt-4o",
                Version = "2024-11-20"
            },
            RaiPolicyName = "Microsoft.Default"
        }
    }
); 
```

### Step 5: Extract API Key
To interact with the OpenAI API, you need to extract the API key from the Cognitive Services account.

``` csharp
Output<Azure.CognitiveServices.ListAccountKeysResult> keys = 
    Azure.CognitiveServices.ListAccountKeys.Invoke(
        new Azure.CognitiveServices.ListAccountKeysInvokeArgs
        {
            ResourceGroupName = rg.Name,
            AccountName = account.Name
        }
    );

Output<string?> key = keys.Apply(keys => keys.Key1);

return key;
```

you can now use the `key` variable to specify an application environment variable or better a Key Vault secret.

### Step 6: Deploy the Stack
Run the following command to deploy your stack:

``` console
pulumi up
```
This command will show you a preview of the resources that will be created. Review the changes and confirm to proceed with the deployment.

## Conclusion
Using Pulumi to manage Azure resources offers the power of traditional IaC with the flexibility of modern programming. As AI services become a key part of the developer toolkit, tools like Pulumi make it easier to automate and scale responsibly.

If you're curious to learn more, check out:
- [Azure OpenAI Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [Pulumi Azure Native Docs](https://www.pulumi.com/registry/packages/azure-native/)
- [Pulumi Examples](https://github.com/pulumi/examples)