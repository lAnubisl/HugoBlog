---
title: "Is Azure Resource Group just a Logical Container?"
date: 2024-07-11T13:19:52Z
draft: false
keywords: "azure, azure-resource-group"
summary: "![](/images/azure-resource-group/logo.png)
I was always thinking about Azure Resource Groups as a Logical container for resources that helps in managing and organizing resources in Azure. But it appears that there are some limitations that I was not aware of."
---
![](/images/azure-resource-group/logo.png)

I was always thinking about Azure Resource Groups as a Logical container for resources that helps in managing and organizing resources in Azure. But it appears that there are some limitations that I was not aware of.

Here is the [official definition from Microsoft](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal#what-is-a-resource-group):

> A resource group is a container that holds related resources for an Azure solution. The resource group can include all the resources for the solution, or only those resources that you want to manage as a group. You decide how you want to allocate resources to resource groups based on what makes the most sense for your organization. Generally, add resources that share the same lifecycle to the same resource group so you can easily deploy, update, and delete them as a group.

However, recently I have faced a new error that I have never seen before:
```Code="BadRequest" Message="Requested features 'Dynamic SKU, Linux Worker' not available in resource group rg-my-resource-group. Please try using a different resource group or create a new one.```

After some research, I found [this](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale#limitations-for-creating-new-function-apps-in-an-existing-resource-group):

> In some cases, __when trying to create a new hosting plan for your function app__ in an existing resource group you might receive one of the following errors ...
> 
> The reason this happens is due to how function app and web app plans are mapped to different pools of resources when being created. 
> Different SKUs require a different set of infrastructure capabilities. When you create an app in a resource group, that __resource group is mapped and assigned__ to a specific pool of resources. If you try to create another plan in that resource group and the mapped pool does not have the required resources, this error occurs.

## Conclusion

The Azure Resource Group can still be considered as a logical container. But now I should keep in mind that it is also mapped and assigned to a specific pool of physical resources. I'm not sure why Microsoft has this limitation. Maybe it is related to some legacy reasons or some technical limitations. But I will keep this in mind for my future projects.
