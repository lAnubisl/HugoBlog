---
title: "Azure Function Serverless Deployment Python"
date: 2023-05-26T12:03:58Z
draft: false
Summary: "In my previous article, I have explained how to deploy a simple Dotnet Azure Function using ZipDeploy. In this article, I will explain how to do the same for Python Azure Function."
---
Consumption plan is the cheapest way to run your Azure Function. However, it has some limitations. For example, you can not use Web Deploy, Docker Container, Source Control, FTP, Cloud sync or Local Git. You can use [External package URL](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies#external-package-url) or [Zip deploy](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies#zip-deploy) instead. In this article I will show you how to deploy Python Azure Function [Programming Model v2](https://techcommunity.microsoft.com/t5/azure-compute-blog/azure-functions-v2-python-programming-model/ba-p/3665168) App to Linux Consumption Azure Function resource using Zip Deployment.

## Create azure function resource
I will use [terraform](https://www.terraform.io) to create the azure function resource. The terraform code is shown below. The code is also available on [github](https://github.com/lAnubisl/AzureFunctionPythonLinuxConsumptionZipDeployment/blob/main/terraform_infrastructure/main.tf).

```terraform
# Here we define the resource group where the function will be deployed. https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group
resource "azurerm_resource_group" "rg" {
  name     = "rg-func-python-deployment-test"
  location = "westeurope"
}

# Here we define the storage account that will be used by the function. Any azure function needs a storage account. https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account
resource "azurerm_storage_account" "st_func" {
  name                            = "stfuncpythondeployment"
  resource_group_name             = azurerm_resource_group.rg.name
  location                        = azurerm_resource_group.rg.location
  account_tier                    = "Standard"
  account_replication_type        = "LRS"
}

# Here we define the service plan that will be used by the function. The service plan defines the location, the operating system and the pricing tier. https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/service_plan
resource "azurerm_service_plan" "func_plan" {
  name                = "plan-func-python-deployment-test"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "Y1"
}

# Here we define the function app. The function app is the actual function that will be deployed. It is linked to the service plan and the storage account. https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/linux_function_app
resource "azurerm_linux_function_app" "func" {
  name                       = "func-python-deployment-test"
  location                   = azurerm_resource_group.rg.location
  resource_group_name        = azurerm_resource_group.rg.name
  service_plan_id            = azurerm_service_plan.func_plan.id
  storage_account_name       = azurerm_storage_account.st_func.name
  storage_account_access_key = azurerm_storage_account.st_func.primary_access_key
  app_settings = {
    AzureWebJobsFeatureFlags = "EnableWorkerIndexing" # Here is the reason why you need this value: https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-decorators#update-app-settings
    FUNCTIONS_WORKER_RUNTIME = "python"
  }
  site_config {
    application_stack {
      python_version = "3.10"
    }
  }
}
```

## Create azure function python application

You can create azure function in a different ways. Here is the Microsoft documentation about how to do this in [Visual Studio Code](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-decorators) or [Command Line](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-cli-python?tabs=azure-cli%2Cbash&pivots=python-mode-decorators). I will use the simplified function that is created by Visual Studio Code. You can find the source code on my [github](https://github.com/lAnubisl/AzureFunctionPythonLinuxConsumptionZipDeployment/tree/main/azure_function). Here is the function HTTP trigger code.

```python
@app.route(route="Health")
def Health(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse("Healthy", status_code=200)
```

## Deployment

There are [many ways](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies) you can use to deploy your azure funtion. However, for Linux Consumption Plan there are only two ways: you can use [External package URL](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies#external-package-url) or [Zip deploy](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies#zip-deploy).
![](/images/azure-function-serverless-deployment-dotnet/deployment_options.png)
I will use Zip deploy option for this example.

All you need is to create deployment package. You can find more information about deployment package structure [here](https://learn.microsoft.com/en-us/azure/azure-functions/deployment-zip-push#deployment-zip-file-requirements). In short, you need to create a zip file with all the files from the publish folder but not the publish folder itself!
``` bash
cd azure_function
zip -r ../publish.zip .
cd ..
```
Double check that you generate the zip file correctly. If you don't then you can face an issue when Azure says 'Deployment successful' but the function app is not working.
The archive should not contain the folder with the function app it shoukd contain the files from the folder.

Here is the example of the incorrect archive structure.
``` bash
unzip -l publish.zip
Archive:  publish.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2023-06-03 12:30   azure_function/
       98  2023-06-03 12:10   azure_function/.funcignore
      259  2023-06-03 13:01   azure_function/function_app.py
      203  2023-06-03 12:09   azure_function/requirements.txt
      288  2023-06-03 12:09   azure_function/host.json
      199  2023-06-03 12:13   azure_function/local.settings.json
---------                     -------
     1047                     6 files
```
Here is the example of the correct archive structure.
``` bash
$ unzip -l publish.zip
Archive:  publish.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
       98  2023-06-03 12:10   .funcignore
      259  2023-06-03 13:01   function_app.py
      203  2023-06-03 12:09   requirements.txt
      288  2023-06-03 12:09   host.json
      199  2023-06-03 12:13   local.settings.json
---------                     -------
     1047                     5 files
```

  Then you need to deploy the package to the function app. You can find credentials on Azure Portal or use Terraform Outputs.
``` bash
# deploy the artifact to the function app. You can find credentials on Azure Portal or use Terraform Outputs.
DEPLOYMENT_USER='$func-python-deployment-test'
DEPLOYMENT_PASSWORD='***************************'
DEPLOYMENT_APP='func-python-deployment-test'
CREDENTIALS=$DEPLOYMENT_USER:$DEPLOYMENT_PASSWORD
curl -v -X POST --user $CREDENTIALS --data-binary @"publish.zip" https://$DEPLOYMENT_APP.scm.azurewebsites.net:443/api/zipdeploy
```

After deployment you can check the logs.
``` bash
curl -X GET --user $CREDENTIALS https://$DEPLOYMENT_APP.scm.azurewebsites.net:443/deployments
```

After successful deployment you can check that the archive is locates in the storage account of the function app inside the 'scm-releases' container.
![](/images/azure-function-serverless-deployment-python/deployment_package_in_storage.png)

Another interesting thing is [trigger syncing](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies#trigger-syncing). The Microsoft documentation says that the [Zip deploy will perform this synchronization for you](https://github.com/projectkudu/kudu/wiki/Deploying-from-a-zip-file-or-url#comparison-with-zip-api:~:text=Zipdeploy%20will%20perform%20this%20synchronization%20for%20you). Hovewer, for for some reason the Python Azure function model v2 requires additional parameter to be set in the app settings. You can find more information about this parameter [here](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-decorators#update-app-settings). You need to set the value of the 'AzureWebJobsFeatureFlags' to 'EnableWorkerIndexing'. You can do this in the Azure Portal or using Terraform (as we did in this article).

After the deployment you can check that the function is working.

![](/images/azure-function-serverless-deployment-python/functions_list.png)

## Conclusion
Zip Deploy can be used to deploy python azure function to Linux Consumption Plan. It is simple and does not require additional software installed on the CD runner or shared credentials like [Azure Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals?tabs=browser).