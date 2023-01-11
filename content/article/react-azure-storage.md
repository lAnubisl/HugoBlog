---
title: "Host React frontend app on Azure Storage Account"
date: 2023-01-10T22:03:40Z
draft: false
summary: "Running a React frontend web application on an Azure Storage account is a great way to host your app with a low cost and high scalability. In this post, we'll go through the steps of setting up a new Azure Storage account and deploying your React app to it."
---

Running a React frontend web application on an Azure Storage account is a great way to host your app with a low cost and high scalability. In this post, we'll go through the steps of setting up a new Azure Storage account and deploying your React app to it.

First, create a new Azure Storage account by going to the Azure portal and selecting "Create a resource" and then "Storage account." Fill in the necessary details and select a subscription and resource group.

Once the storage account is created, go to the "Data management" → "Static website" menu item, and enable the "Static Website" feature. This will create a container called $web and you'll need to upload your files to this container. This feature also provides endpoint for serving content. Make sute that you select "Index document name" as index.html and you have it in your build directory.

Next, build and upload your React app to the $web container. You can do this using tools like Azure Storage Explorer, Azure Cli or Azure powershell.
Here is an example of how to do this using Azure Cli:

``` bash
az storage blob upload-batch 
    --source $REACT_APP_BUILD_DIR 
    --destination '$web' 
    --overwrite 
    --connection-string $AZURE_STORAGE_CONNECTION_STRING 
```
You can get the connection string for your storage account by going to the "Access keys" menu item.

Once the files are uploaded, you can access your app by going to the endpoint provided by the static website feature. You may notice that the app is not working as expected. This is because the app is using the browser's history API to navigate between pages. The browser is trying to access the pages directly and not going through the index.html file which is located in the root of your $web blob container. To fix this we need to configure the hosting platform to redirect all such requests to that index.html file. This can be done by using Azure CDN profile.

Create a new CDN profile by going to the Azure portal and selecting "Create a resource" and then "CDN profile." Fill in the necessary details and select a "Microsoft Standard" pricing tier, subscription and resource group. Once the profile is created, go to the "Endpoints" menu item and create a new endpoint. Select the storage account you created earlier as the origin. Once profile is created, go to "Rule Engine" menu item and add the following rule:
![azure-cdn-react-rule](/images/react-azure-storage/azure-cdn-react-rule.png)

Once the rule is created, go to the "Overview" menu item and copy the "Endpoint hostname" value. This is the endpoint for your app. You can now access your app by going to this endpoint.