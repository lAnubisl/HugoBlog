---
title: "Host React frontend app on Azure Storage Account"
date: 2023-08-19T11:56:40Z
draft: false
summary: "![](/images/react-azure-storage/react-azure-storage-logo.jpg) Running a React frontend web application on an Azure Storage account is a great way to host your app with a low cost and high scalability. In this post, we'll go through the steps of setting up a new Azure Storage account and deploying your React app to it."
---

![](/images/react-azure-storage/react-azure-storage-logo.jpg)

Running a [React](https://react.dev/) frontend web application on an [Azure Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview) is a great way to host your app with a low cost and high scalability.
Azure Storage Account can handle [up to 20,000](https://learn.microsoft.com/en-us/azure/storage/tables/scalability-targets) requests per second out of the box.
It has a lot of features that make it a great choice for hosting static websites including:
- [Public anonymous access](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure?tabs=portal) over HTTP/HTTPS;
- [Custom domain names](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-custom-domain-name?tabs=azure-portal);
- [Static Website](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website) feature;

In this post, we'll go through the steps of setting up a new Azure Storage account and deploying your React app to it.

In the end of this post you also can find the links to the [terraform](https://www.terraform.io) template and [GitLab CI/CD pipeline](https://docs.gitlab.com/ee/ci/) that will automate the infrastructure creation and deployment processes.

#### Plan

We are not going to use 'Static Website' feature of Azure Storage Account because it does not meet the requirements of the React apps.
The thing is that most of the react apps use [react-router](https://reactrouter.com/) to handle routing. 
In these cases, the index.html file is served for all requests. 
However, the static website feature of Azure Storage Account does not support this. 
This can be done by using [Azure CDN profile](https://learn.microsoft.com/en-us/azure/cdn/cdn-add-to-web-app?toc=%2Fazure%2Ffrontdoor%2FTOC.json).

So the plan is to:
 - Create a new Azure Storage account. 
 - Then create Azure CDN profile and configure it to redirect all requests to the index.html file.

#### Step 1: Resource Group
First, we need to create an [Azure Resource Group](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal). The resource group is a logical container for the resources that we will create later.

![azure-resource-group-creation](/images/react-azure-storage/azure-resource-group-creation.png)
The rest of settings can be left as default.

#### Step 2: Storage Account

Next, create a new Azure Storage account that will keep our react application ([here is how to do this using Azure Portal](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal)).
Fill in the necessary details and select a subscription and resource group name created in the previous step.

![azure-storage-account-creation](/images/react-azure-storage/azure-storage-account-creation.png)
The rest of settings can be left as default.

#### Step 3: Storage Account Container

Files in Storage Account are stored in [Containers](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction#containers) so we need to create one.
To do that go to the created Azure Storage account and click on the "+ Container" button.

![azure-storage-container-creation](/images/react-azure-storage/azure-storage-container-creation.png)
The name of the container does not matter. But it is important to set the access level to "Blob (anonymous read access for blobs only)".

#### Step 4: Azure CDN Profile

Azure CDN profile is required to manage the routing of the requests to the index.html file.
There are different types of CDN profiles. We will use the "Standard Microsoft" profile.
When you create the CDN profile, make sure you selected "Explore other offers" and then "Azure CDN Standard Microsoft (classic)" as on the screenshot below.
![azure-cdn-creation-1](/images/react-azure-storage/azure-cdn-creation-1.png)
Then fill in the fields as on the screenshot below.
![azure-cdn-creation-2](/images/react-azure-storage/azure-cdn-creation-2.png)
The rest of settings can be left as default.

#### Step 5: Azure CDN Profile Endpoint

After the CDN profile is created, we need to create a new endpoint.
Go to the created CDN profile and click "+ Endpoint" button.
![azure-cdn-endpoint](/images/react-azure-storage/azure-cdn-endpoint.png)
You need to fill in the following fields:
- **Name**: this will be the default hostname of your website. For example, if you set it to "myapp", then the default hostname will be "myapp.azureedge.net".
- **Origin type**: "Storage" (this will be the Azure Storage account that we created earlier).
- **Origin hostname**: the name of the Azure Storage account that we created earlier.
- **Origin path**: the name of the container that we created earlier.
- **Origin host header**: will be populated automatically once you select the 'Origin hostname' with the value of the name of the Azure Storage account that we created earlier.

#### Step 6: Rules Engine

Next we need to configure the rules for the endpoint.
Click on the "Rules engine" menu item under the "Settings" section.
Then click on the "Add rule" button.
The new rule block will appear. Click on the "Add condition" button and select "If URL file extension" condition.
Then fill in the fields as on the screenshot below.
Then click on the "Add action" button and select "URL rewrite" action.
Then fill in the fields as on the screenshot below.
![azure-cdn-endpoint-rules](/images/react-azure-storage/azure-cdn-endpoint-rules.png)
It is also useful to add the global action with the "Cache expiration" action type and set the expiration time to some reasonable value. For development environment, you can set it to 5 minute.

Don't forget to click on the "Save" button to save the changes.
The changes need to be propagated to the CDN nodes. It can take up to 15 minutes.

## Deploying React app to Azure Storage Account

If you are reading this, then you probably already have a React app. 
For the sake of simplicity, we will use the React app from the [React tutorial](https://react.dev/learn/tutorial-tic-tac-toe#what-are-you-building).
Save it to the local folder as "index.html" file.
Then open the Azure Storage account that we created earlier and go to the container. 
Then click on the "Upload" button and select the "index.html" file. 
Then click on the "Upload" button to upload the file to the container.

![azure-storage-container-file-upload](/images/react-azure-storage/azure-storage-container-file-upload.png)

## Verifying the deployment

Now we can verify that the React app is deployed to the Azure Storage account.
To do that, go to the Azure CDN profile Endpoint that we created earlier and click on the link under the "Endpoint hostname" label.
![zure-cdn-endpoint-address](/images/react-azure-storage/azure-cdn-endpoint-address.png)
You should see the react app from the tutorial.

![website-index](/images/react-azure-storage/website-index.png)

And if you add something to the URL, you should see the same page. 
That means that the rules are configured correctly and your React app should work as expected.

![website-routing](/images/react-azure-storage/website-routing.png)

## Automation

If you want to automate the infrastructure creation process then take a look at [this gist](https://gist.github.com/lAnubisl/a8b95a2391669a5c943271f937dd6752), where you can find the [terraform](https://www.terraform.io) template that will create the Azure Storage account and Azure CDN profile with the endpoint.

If you want to automate the deployment process then take a look at [this gist](https://gist.github.com/lAnubisl/817dc46b63905340ad44fd9a85798fd2), where you can find the GitLab CI/CD pipeline that will deploy your React app to the Azure Storage account.