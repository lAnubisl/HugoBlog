---
title: "Azure API Management to Azure Service Bus"
date: 2023-07-08T10:19:45Z
keywords: "Azure, API Management, Service Bus, Terraform"
description: "How to use Azure API Management to expose an Azure Service Bus"
summary: "![](/images/azure-apimanagement-to-azure-service-bus/apim_to_sb.png)

In my [previous post](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/) I showed how to use [Azure API Management](https://azure.microsoft.com/en-us/products/api-management) to expose an [Azure Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview). In this post I will show how to use Azure API Management to expose an [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview).

This combination is useful when you have a fire and forget HTTP endpoint and you expect irregular traffic. For example, you are designing a mobile application crash reporting system. You want to send the crash report to the server and forget about it. You don't want to wait for the response. You don't want to block the user interface. You don't want to retry if the server is not available. It might happen that a new mobile app version has a significant bug and you get a lot of crash reports. In this case, it is reasonable to use Azure Service Bus to queue the crash reports for peak times and process them later.

As in my [previous post](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/), I will use a scenario with two brands (Adidas and Nike) to show you the flexibility of the combination Azure API Management plus Azure Service Bus.
I will also define everything in [Terraform](https://www.terraform.io/) so the solution deployment is fully automated."
draft: false
---
![](/images/azure-apimanagement-to-azure-service-bus/apim_to_sb.png)


In my [previous post](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/) I showed how to use [Azure API Management](https://azure.microsoft.com/en-us/products/api-management) to expose an [Azure Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview). In this post I will show how to use Azure API Management to expose an [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview).

This combination is useful when you have a fire and forget HTTP endpoint and you expect irregular traffic. For example, you are designing a mobile application crash reporting system. You want to send the crash report to the server and forget about it. You don't want to wait for the response. You don't want to block the user interface. You don't want to retry if the server is not available. It might happen that a new mobile app version has a significant bug and you get a lot of crash reports. In this case, it is reasonable to use Azure Service Bus to queue the crash reports for peak times and process them later.

As in my [previous post](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/), I will use a scenario with two brands (Adidas and Nike) to show you the flexibility of the combination Azure API Management plus Azure Service Bus.
I will also define everything in [Terraform](https://www.terraform.io/) so the solution deployment is fully automated.

Long story short, the solution is here [main.tf](https://gist.github.com/lAnubisl/07bdb00d43a7a98252cdd6da7f748286).

## Infrastructure

First of all, we need to create the resource group where we will deploy all the resources.

``` terraform
resource "azurerm_resource_group" "rg" {
  name     = "rg-apim-to-servicebus"
  location = "westeurope"
}

```

Then we need to create the Azure Service Bus namespace topic and two subscriptions. One subscription for Adidas and one for Nike.

``` terraform
resource "azurerm_servicebus_namespace" "sbns" {
  name                = "sbns-brands-test-2023"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
  local_auth_enabled  = false
}

resource "azurerm_servicebus_topic" "sbt" {
  name                = "brands"
  namespace_id        = azurerm_servicebus_namespace.sbns.id
  enable_partitioning = true
}

resource "azurerm_servicebus_subscription" "sbts_adidas" {
  name               = "adidas"
  topic_id           = azurerm_servicebus_topic.sbt.id
  max_delivery_count = 1
}

resource "azurerm_servicebus_subscription" "sbts_nike" {
  name               = "nike"
  topic_id           = azurerm_servicebus_topic.sbt.id
  max_delivery_count = 1
}

```

For each subscription, we need to create a rule that will filter the messages based on the brand. The rule is defined using SQL syntax. The rule will be applied to the messages sent to the topic. If the rule is not satisfied, the message will be discarded.

``` terraform

resource "azurerm_servicebus_subscription_rule" "sbts_rule_adidas" {
  name            = "adidas"
  subscription_id = azurerm_servicebus_subscription.sbts_adidas.id
  filter_type     = "SqlFilter"
  sql_filter      = "brand = 'adidas'"
}

resource "azurerm_servicebus_subscription_rule" "sbts_rule_nike" {
  name            = "nike"
  subscription_id = azurerm_servicebus_subscription.sbts_nike.id
  filter_type     = "SqlFilter"
  sql_filter      = "brand = 'nike'"
}

```

Now we need to create the Azure API Management instance. We will use the Consumption tier because it is cheaper and it fits the demonstration purposes.

``` terraform
resource "azurerm_api_management" "apim" {
  name                = "apim-test-july-2023"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  publisher_name      = "John Doe"
  publisher_email     = "john.doe@example.com"
  sku_name            = "Consumption_0"
  identity {
    type = "SystemAssigned"
  }
}
```

And the API Product with the API definition:

``` terraform
resource "azurerm_api_management_product" "apim_product" {
  product_id            = "my_product_id"
  api_management_name   = azurerm_api_management.apim.name
  resource_group_name   = azurerm_resource_group.rg.name
  display_name          = "My Product"
  description           = "My Product Description"
  terms                 = "My Product Terms"
  subscription_required = true
  subscriptions_limit   = 1
  approval_required     = true
  published             = true
}

resource "azurerm_api_management_api" "apim_api" {
  name                = "example-api-name"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  revision            = "1"
  display_name        = "Integrations with Azure Managed Services"
  api_type            = "http"
  path                = "path"
  protocols           = ["https"]

  subscription_key_parameter_names {
    header = "subscription"
    query  = "subscription"
  }
}

resource "azurerm_api_management_product_api" "apim_product_api" {
  api_name            = azurerm_api_management_api.apim_api.name
  product_id          = azurerm_api_management_product.apim_product.product_id
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
}

resource "azurerm_api_management_api_operation" "apim_api_opn" {
  operation_id        = "to-service-bus"
  api_name            = azurerm_api_management_api.apim_api.name
  api_management_name = azurerm_api_management_api.apim_api.api_management_name
  resource_group_name = azurerm_api_management_api.apim_api.resource_group_name
  display_name        = "Send data to the service bus queue"
  method              = "POST"
  url_template        = "/servicebus"
  description         = "Send data to the service bus queue"
}
```

Now we need to ling the API Management to the Azure Service Bus.
First, allow the API Management to access the Azure Service Bus Topic:

``` terraform
resource "azurerm_role_assignment" "apim_role_assignment" {
  scope                = azurerm_servicebus_namespace.sbns.id
  role_definition_name = "Azure Service Bus Data Sender"
  principal_id         = azurerm_api_management.apim.identity[0].principal_id
}
```
Then we need to tell API Management how to connect to the Azure Service Bus Topic.
(In my [previous post](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/) I used the concept of Backends, but in this case, I will use the concept of [Named Values](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties?tabs=azure-portal))

``` terraform
resource "azurerm_api_management_named_value" "nv_sb_base_url" {
  name                = "sb-base-url"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  display_name        = "sb-base-url"
  value               = "https://${azurerm_servicebus_namespace.sbns.name}.servicebus.windows.net"
}

resource "azurerm_api_management_named_value" "nv_sb_queue" {
  name                = "sb-queue_or_topic"
  resource_group_name = azurerm_resource_group.rg.name
  api_management_name = azurerm_api_management.apim.name
  display_name        = "sb-queue_or_topic"
  value               = azurerm_servicebus_topic.sbt.name
}
```

Finally, we need to create the API Management Policy that will send the message to the Azure Service Bus Topic.

``` terraform
resource "azurerm_api_management_api_operation_policy" "apim_api_opn_policy_servicebus" {
  api_name            = azurerm_api_management_api_operation.apim_api_opn.api_name
  api_management_name = azurerm_api_management_api_operation.apim_api_opn.api_management_name
  resource_group_name = azurerm_api_management_api_operation.apim_api_opn.resource_group_name
  operation_id        = azurerm_api_management_api_operation.apim_api_opn.operation_id

  xml_content = <<XML
<policies>
  <inbound>
    <base />
    # OUR INBOUND POLICY
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
    # OUR OUTBOUND POLICY
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
XML
}
```

As in my [previous post](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/) I will show you all the policy in parts.

### Inbound

Filter the brand header using [choose policy](https://learn.microsoft.com/en-us/azure/api-management/choose-policy)

In case when the brand header is "adidas" or "nike" set the brand variable to the value of the header using [set-variable policy](https://learn.microsoft.com/en-us/azure/api-management/set-variable-policy).
Othwerwise, return a 400 error with a JSON body using [return-response policy](https://learn.microsoft.com/en-us/azure/api-management/return-response-policy).

``` text
<choose>
  <when condition="@(context.Request.Headers.GetValueOrDefault("brand","") == "adidas")">
    <set-variable name="brand" value="adidas" />
  </when>
  <when condition="@(context.Request.Headers.GetValueOrDefault("brand","") == "nike")">
    <set-variable name="brand" value="nike" />
  </when>
  <otherwise>
    <return-response>
        <set-status code="400" reason="Bad Request" />
        <set-header name="Content-Type" exists-action="override">
          <value>application/json</value>
        </set-header>
        <set-body>@{  
          var json = new JObject() {{"Error", "Invalid brand"}} ;  
          return json.ToString(Newtonsoft.Json.Formatting.None);       
        }</set-body>
    </return-response>
  </otherwise>
</choose>
```

Authenticate using Managed Identity through [authentication-managed-identity policy](https://learn.microsoft.com/en-us/azure/api-management/authentication-managed-identity-policy).
    
``` text
<authentication-managed-identity resource="https://servicebus.azure.net/" />
```

Redirect to the backend using [set-backend-service policy](https://learn.microsoft.com/en-us/azure/api-management/set-backend-service-policy) and [rewrite-uri policy](https://learn.microsoft.com/en-us/azure/api-management/rewrite-uri-policy).
We also are going to use [Named Values](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties?tabs=azure-portal) that we created before.

``` text
<set-backend-service base-url="{{sb-base-url}}" />
<rewrite-uri template="{{sb-queue_or_topic}}/messages" />
```

Set Azure Service Bus Message Headers
    
``` text
<set-header name="BrokerProperties" exists-action="override">
  <value>@{  
    var json = new JObject();  
    json.Add("MessageId", context.RequestId);  
    json.Add("brand", (string)context.Variables["brand"]);  
    return json.ToString(Newtonsoft.Json.Formatting.None);                      
  }</value>
</set-header>
```

### Outbound

``` text
<choose>
  <when condition="@(context.Response.StatusCode == 201)">
    <set-header name="Content-Type" exists-action="override">
      <value>application/json</value>
    </set-header>
    <set-body>@{  
      var json = new JObject() {{"OperationId", context.RequestId}};  
      return json.ToString(Newtonsoft.Json.Formatting.None);       
    }</set-body>
  </when>
</choose>
```

## Testing

To test our API we need to create a new subscription key and get its value.
![](/images/azure-apimanagement-to-azure-service-bus/apim_to_sb_subscription_key.png)
Then we can use a tool like [Postman](https://www.postman.com/) to send a request to our API:
![](/images/azure-apimanagement-to-azure-service-bus/apim_to_sb_postman.png)
After few messages sent with different brands we can check the Azure Service Bus Topic Subscriptions and see that the messages are there:
![](/images/azure-apimanagement-to-azure-service-bus/apim_to_sb_result.png)

## Conclusion

In this post we saw how to send a message to an Azure Service Bus Topic using Azure API Management. We also saw how to use Managed Identity to authenticate to Azure Service Bus.
We also learned how to use Terraform to automate the deployment of the solution. I hope you enjoyed this article and found it useful. If you have any questions please leave a comment below.

## Links
- [Azure Service Bus HTTP API (Send Message)](https://learn.microsoft.com/en-us/rest/api/servicebus/send-message-to-queue)
- [Full solution](https://gist.github.com/lAnubisl/07bdb00d43a7a98252cdd6da7f748286)