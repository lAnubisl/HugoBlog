---
title: "Azure API Management to Azure Cosmosdb"
date: 2023-11-07T20:12:52Z
keywords: "Azure, API Management, CosmosDB, Terraform"
description: "How to use Azure API Management to expose Azure CosmosDB"
summary: "![](/images/azure-apimanagement-to-azure-cosmosdb/apim_cosmosdb_logo.jpeg)

In my previous posts I have showed you how to connect [Azure API Management to Azure Service Bus](https://byalexblog.net/article/azure-apimanagement-to-azure-service-bus/) and [Azure API Management to Azure Storage account](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/). In this post I will show you how to connect Azure API Management to Azure Cosmosdb."
draft: false
---

![](/images/azure-apimanagement-to-azure-cosmosdb/apim_cosmosdb_logo.jpeg)
In my previous posts I have showed you how to connect [Azure API Management to Azure Service Bus](https://byalexblog.net/article/azure-apimanagement-to-azure-service-bus/) and [Azure API Management to Azure Storage account](https://byalexblog.net/article/azure-apimanagement-to-azure-storage-account/). In this post I will show you how to connect Azure API Management to Azure Cosmosdb.
> At this point of time (7 Now 2023) Azure API Management has the policy [cosmosdb-data-source](https://learn.microsoft.com/en-us/azure/api-management/cosmosdb-data-source-policy) in preview and it is not supported in the Consumption Plan. So, you need to have Developer, Basic, Standard or Premium plan to use this policy. 
And even then I would not recommend to use this policy in production environment yet.

Before we start, let's have a look at the architecture of our solution:
![](/images/azure-apimanagement-to-azure-cosmosdb/architecture.jpeg)

There is a significant limitation in this solution: [NOT ALL TYPES OF QUERIES ARE SUPPORTED](https://learn.microsoft.com/en-us/rest/api/cosmos-db/querying-cosmosdb-resources-using-the-rest-api#queries-that-cannot-be-served-by-gateway) by CosmosDB HTTP API.
So if you want to use queries that include TOP, ORDER BY, OFFSET LIMIT, DISTINCT, GROUP BY, and aggregate functions like MAX, MIN, AVG, SUM, COUNT then **this solution will not work for you**. 
There is a workaround for the OFFSET LIMIT limitation, which is covered by this post. For other cases you will have to run compute resource behind the API Management to do the job and that compute resource will use native library to talk to Azure CosmosDB without such limitations.

If ALL your queries look like 
```SELECT {something} WHERE {condition}```then let's get started.

## 1. Create Azure Cosmosdb

I will use [Azure Cosmos DB for NoSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/) database with Capacity mode: "serverless" for the purpose of this post. You can use any other plan as well.

![](/images/azure-apimanagement-to-azure-cosmosdb/cosmosdb_type.png)

In our solution we are going to use [role-based access control](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac) to access Azure Cosmosdb. 
Please note that this is **NOT** the same as [Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) and cannot be configured in the Azure portal.
You either need to use [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/cosmosdb/sql/role?view=azure-cli-latest) or [Azure PowerShell](https://learn.microsoft.com/en-us/powershell/module/az.cosmosdb/?view=azps-10.4.1) to configure it.
I will use [Terraform](https://www.terraform.io/) instead (which uses Azure CLI under the hood).

``` hcl
# Resource Group
resource "azurerm_resource_group" "rg" {
  name     = "rg_samples_azure_cosmosdb_apim"
  location = "westeurope"
}

# CosmosDB Account
resource "azurerm_cosmosdb_account" "account" {
  name                = "cosmon-samples-234f1f22a3"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"
  
  consistency_policy {
    consistency_level = "Session"
  }

  geo_location {
    location          = azurerm_resource_group.rg.location
    failover_priority = 0
  }

  capabilities {
    name = "EnableServerless"
  }
}

# CosmosDB Database
resource "azurerm_cosmosdb_sql_database" "database" {
  name                = "mydatabase"
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_cosmosdb_account.account.name
}

# CosmosDB Container
resource "azurerm_cosmosdb_sql_container" "container" {
  name                  = "mycontainer"
  resource_group_name   = azurerm_resource_group.rg.name
  account_name          = azurerm_cosmosdb_account.account.name
  database_name         = azurerm_cosmosdb_sql_database.database.name
  partition_key_path    = "/id"
  partition_key_version = 1
  indexing_policy {
    indexing_mode = "consistent"
    included_path {
      path = "/*"
    }
  }
}
```

## 2. Create Azure API Management

I will create Azure API Management using [Terraform](https://www.terraform.io/) as well.

``` hcl
resource "azurerm_api_management" "apim" {
  name                = "apim-samples-234f1f22a3"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  publisher_name      = "Alex"
  publisher_email     = "alexander.panfilenok@gmail.com"
  sku_name            = "Consumption_0"
  identity {
    type = "SystemAssigned"
  }
}
```

## 3. Link Azure API Management to Azure Cosmosdb

Now we need to tell Azure CosmosDB that Azure API Management is allowed to access it.

``` hcl
# CosmosDB SQL Role Definition that allows to read data from CosmosDB
resource "azurerm_cosmosdb_sql_role_definition" "role_definition" {
  name                = "roledefinition"
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_cosmosdb_account.account.name
  type                = "CustomRole"
  assignable_scopes   = [azurerm_cosmosdb_account.account.id]

  permissions {
    data_actions = [
      "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed",
      "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery",
      "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read",
      "Microsoft.DocumentDB/databaseAccounts/readMetadata"
    ]
  }
}

# CosmosDB SQL Role Assignment that assigns the role to Azure API Management
resource "azurerm_cosmosdb_sql_role_assignment" "am_cosmosdb_sql_assignment" {
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_cosmosdb_account.account.name
  role_definition_id  = azurerm_cosmosdb_sql_role_definition.role_definition.id
  principal_id        = azurerm_api_management.apim.identity[0].principal_id
  scope               = azurerm_cosmosdb_account.account.id
}
```

Now CosmosDB is ready to accept requests from Azure API Management.

## 4. Create API endpoints in Azure API Management

In order to demonstrate you an example of such endpoint I will skip the terraform code for API and will show you the screenshot of the Azure Portal instead.

I have created 2 API endpoints: one is to get a document by id and another one is to get list of documents by some condition.
An important part is an Inbound policy that is used to connect to Azure CosmosDB.

### 4.1. Get document by id

![](/images/azure-apimanagement-to-azure-cosmosdb/endp1.png)
```
<inbound>
  <base />
  <set-backend-service 
    base-url="https://cosmon-samples-234f1f22a3.documents.azure.com:443/" />
  <authentication-managed-identity 
    resource="https://cosmon-samples-234f1f22a3.documents.azure.com" 
    output-token-variable-name="msi-access-token" 
    ignore-error="false" />
  <set-header name="Authorization" exists-action="override">
    <value>@("type=aad&ver=1.0&sig=" + context.Variables["msi-access-token"])</value>
  </set-header>
  <set-header name="x-ms-date" exists-action="override">
    <value>@(DateTime.UtcNow.ToString("r"))</value>
  </set-header>
  <set-header name="x-ms-version" exists-action="override">
    <value>2018-12-31</value>
  </set-header>
  <rewrite-uri 
    template="/dbs/mydatabase/colls/mycontainer/docs/" 
    copy-unmatched-params="false" />
  <set-header name="x-ms-documentdb-partitionkey" exists-action="override">
    <value>@("[\"" + context.Request.MatchedParameters["id"] + "\"]")</value>
  </set-header>
</inbound>
```

There are several important parts in this policy:

```
<set-backend-service base-url="https://cosmon-samples-234f1f22a3.documents.azure.com:443/" />
```
this is the URL of Azure CosmosDB account. You can find it in Azure Portal in the Overview section of your Azure CosmosDB account.

```
<authentication-managed-identity 
    resource="https://cosmon-samples-234f1f22a3.documents.azure.com" 
    output-token-variable-name="msi-access-token" ignore-error="false" />
```
this is the part that tells Azure API Management to use Managed Identity to authenticate to Azure CosmosDB. You can find more details about it [here](https://learn.microsoft.com/en-us/azure/api-management/authentication-managed-identity-policy).

```
<set-header name="Authorization" exists-action="override">
    <value>@("type=aad&ver=1.0&sig=" + context.Variables["msi-access-token"])</value>
</set-header>
```
this is the authorization header that is used to authenticate to Azure CosmosDB. You can find more details about [set-header](https://learn.microsoft.com/en-us/azure/api-management/set-header-policy) and [Authorization header](https://learn.microsoft.com/en-us/rest/api/cosmos-db/access-control-on-cosmosdb-resources?redirectedfrom=MSDN).

```
<rewrite-uri 
    template="/dbs/mydatabase/colls/mycontainer/docs/" 
    copy-unmatched-params="false" />
```
This is the policy that modifies the request URI to match CosmosDB HTTP API endpoint. You can find more details about [rewrite-uri](https://learn.microsoft.com/en-us/azure/api-management/rewrite-uri-policy) it [CosmosDB HTTP Endpoints](https://learn.microsoft.com/en-us/rest/api/cosmos-db/cosmosdb-resource-uri-syntax-for-rest).

```
<set-header name="x-ms-documentdb-partitionkey" exists-action="override">
    <value>@("[\"" + context.Request.MatchedParameters["id"] + "\"]")</value>
</set-header>
```
this header is used to specify the partition key. You can find more details about it [here](https://learn.microsoft.com/en-us/rest/api/cosmos-db/common-cosmosdb-rest-request-headers).

### 4.2. Get list of documents by condition with pagination

The second endpoint is very similar to the first one. The only difference is that it uses [query parameter](https://learn.microsoft.com/en-us/rest/api/cosmos-db/querying-cosmosdb-resources-using-the-rest-api#queries-that-can-be-served-by-gateway) instead of path parameter to specify the partition key.

![](/images/azure-apimanagement-to-azure-cosmosdb/endp2.png)
```
<inbound>
    <base />
    <choose>
        <when condition="@{
            var takeStr = context.Request.MatchedParameters['take'];
            var takeInt = 0;
            if (!int.TryParse(takeStr, out takeInt)) {
                return true;
            }

            if (takeInt > 0 && takeInt <= 1000) {
                return false;
            }

            return true;
        }">
            <return-response>
                <set-status code="400" reason="Bad Request" />
                <set-header name="Context-Type" exists-action="override">
                    <value>text/plain</value>
                </set-header>
                <set-body>parameter 'take' is invalid</set-body>
            </return-response>
        </when>
    </choose>
    <choose>
        <when condition="@(context.Request.Headers.GetValueOrDefault("from", "") != "")">
            <set-header name="x-ms-continuation" exists-action="override">
                <value>@(context.Request.Headers.GetValueOrDefault("from", ""))</value>
            </set-header>
        </when>
    </choose>
    <set-backend-service
      base-url="https://cosmon-samples-234f1f22a3.documents.azure.com:443/" />
    <set-method>POST</set-method>
    <authentication-managed-identity
      resource="https://cosmon-samples-234f1f22a3.documents.azure.com"
      output-token-variable-name="msi-access-token"
      ignore-error="false" />
    <set-header name="Authorization" exists-action="override">
        <value>@("type=aad&ver=1.0&sig=" + context.Variables["msi-access-token"])</value>
    </set-header>
    <set-header name="x-ms-date" exists-action="override">
        <value>@(DateTime.UtcNow.ToString("r"))</value>
    </set-header>
    <set-header name="x-ms-version" exists-action="override">
        <value>2018-12-31</value>
    </set-header>
    <set-header name="x-ms-documentdb-isquery" exists-action="override">
        <value>True</value>
    </set-header>
    <set-header name="x-ms-documentdb-query-enablecrosspartition" exists-action="override">
        <value>True</value>
    </set-header>
    <rewrite-uri 
      template="/dbs/mydatabase/colls/mycontainer/docs"
      copy-unmatched-params="false" />
    <set-header name="Content-Type" exists-action="override">
        <value>application/query+json</value>
    </set-header>
    <set-header name="x-ms-max-item-count" exists-action="override">
        <value>@(context.Request.MatchedParameters["take"])</value>
    </set-header>
    <set-body>@("{\"query\": \"SELECT * FROM c\" }")</set-body>
</inbound>
```

I will comment some parts of this policy that are different from the previous one.

```
<choose>
    <when condition="@{
        var takeStr = context.Request.MatchedParameters['take'];
        var takeInt = 0;
        if (!int.TryParse(takeStr, out takeInt)) {
            return true;
        }

        if (takeInt > 0 && takeInt <= 1000) {
            return false;
        }

        return true;
    }">
        <return-response>
            <set-status code="400" reason="Bad Request" />
            <set-header name="Context-Type" exists-action="override">
                <value>text/plain</value>
            </set-header>
            <set-body>parameter 'take' is invalid</set-body>
        </return-response>
    </when>
</choose>
```
this part is responsible for validation of the query parameter "take". It should be a number between 1 and 1000. If it is not then the request will be rejected with 400 Bad Request.
later in the policy we will use this parameter to specify the number of documents to return:

```
<set-header name="x-ms-max-item-count" exists-action="override">
    <value>@(context.Request.MatchedParameters["take"])</value>
</set-header>
```

you can learn more about [choole](https://learn.microsoft.com/en-us/azure/api-management/choose-policy) policy, [return-response](https://learn.microsoft.com/en-us/azure/api-management/return-response-policy) policy and x-ms-max-item-count header [here](https://learn.microsoft.com/en-us/rest/api/cosmos-db/querying-cosmosdb-resources-using-the-rest-api#pagination-of-query-results) and [here](https://learn.microsoft.com/en-us/rest/api/cosmos-db/query-documents#headers)

```
<choose>
    <when condition="@(context.Request.Headers.GetValueOrDefault("from", "") != "")">
        <set-header name="x-ms-continuation" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("from", ""))</value>
        </set-header>
    </when>
</choose>
```
this part is responsible for the second part of the pagination. For my case it is expected that the [continuation token](https://learn.microsoft.com/en-us/rest/api/cosmos-db/querying-cosmosdb-resources-using-the-rest-api#pagination-of-query-results) that is returned by CosmosDB is passed back to the API Management in the header "from". If it is present then it will be passed to CosmosDB in the header "x-ms-continuation".

```
<set-body>@("{\"query\": \"SELECT * FROM c\" }")</set-body>
```
this is the query that is passed to CosmosDB. For reasons of simplicity I do not use "where" clause here, but you can do that easily and pass the condition arguments as a query parameters. You can read about the syntax of the query [here](https://learn.microsoft.com/en-us/rest/api/cosmos-db/query-documents#body).

## 5. Test the solution

First, let's add couple of documents to CosmosDB.
``` json
{
    "id": "123",
    "name": "Alex",
    "_rid": "YVEzAK5GDSwBAAAAAAAAAA==",
    "_self": "dbs/YVEzAA==/colls/YVEzAK5GDSw=/docs/YVEzAK5GDSwBAAAAAAAAAA==/",
    "_etag": "\"7a002c05-0000-0d00-0000-65494dc10000\"",
    "_attachments": "attachments/",
    "_ts": 1699302849
},
{
    "id": "122",
    "name": "Elena",
    "_rid": "YVEzAK5GDSwCAAAAAAAAAA==",
    "_self": "dbs/YVEzAA==/colls/YVEzAK5GDSw=/docs/YVEzAK5GDSwCAAAAAAAAAA==/",
    "_etag": "\"7a00430a-0000-0d00-0000-65494ec70000\"",
    "_attachments": "attachments/",
    "_ts": 1699303111
}
```
Then let's test the first endpoint to get document by id.

``` http
GET https://apim-samples-234f1f22a3.azure-api.net/document/123 HTTP/1.1
Host: apim-samples-234f1f22a3.azure-api.net
```

the response is:
``` http
HTTP/1.1 200 Ok
cache-control: no-store, no-cache
content-encoding: gzip
content-location: https://cosmon-samples-234f1f22a3.documents.azure.com/dbs/mydatabase/colls/mycontainer/docs/
content-type: application/json
date: Wed, 08 Nov 2023 21:47:24 GMT
lsn: 33
pragma: no-cache
strict-transport-security: max-age=31536000
transfer-encoding: chunked
vary: Accept-Encoding,Origin
x-ms-aad-applied-role-assignment: ,a654236b-8ba1-0dd2-7a87-43dd83a89fa5
x-ms-activity-id: 27d187fc-8a4a-422d-a517-956bdd94ca4f
x-ms-alt-content-path: dbs/mydatabase/colls/mycontainer
x-ms-content-path: YVEzAK5GDSw=
x-ms-cosmos-is-partition-key-delete-pending: false
x-ms-cosmos-llsn: 33
x-ms-cosmos-physical-partition-id: 0
x-ms-cosmos-query-execution-info: {"reverseRidEnabled":false,"reverseIndexScan":false}
x-ms-cosmos-quorum-acked-llsn: 33
x-ms-current-replica-set-size: 4
x-ms-current-write-quorum: 3
x-ms-documentdb-partitionkeyrangeid: 0
x-ms-gatewayversion: version=2.14.0
x-ms-global-committed-lsn: 33
x-ms-item-count: 1
x-ms-last-state-change-utc: Mon,06 Nov 2023 20:05:32.452 GMT
x-ms-number-of-read-regions: 0
x-ms-quorum-acked-lsn: 33
x-ms-request-charge: 2.82
x-ms-request-duration-ms: 20.375
x-ms-resource-quota: documentSize=51200;documentsSize=52428800;documentsCount=-1;collectionSize=52428800;
x-ms-resource-usage: documentSize=0;documentsSize=0;documentsCount=2;collectionSize=0;
x-ms-schemaversion: 1.16
x-ms-serviceversion: version=2.14.0.0
x-ms-session-token: 0:-1#33
x-ms-transport-request-id: 1
x-ms-xp-role: 1
    
{
    "_rid": "YVEzAK5GDSw=",
    "Documents": [{
        "id": "123",
        "name": "Alex",
        "_rid": "YVEzAK5GDSwBAAAAAAAAAA==",
        "_self": "dbs\/YVEzAA==\/colls\/YVEzAK5GDSw=\/docs\/YVEzAK5GDSwBAAAAAAAAAA==\/",
        "_etag": "\"7a002c05-0000-0d00-0000-65494dc10000\"",
        "_attachments": "attachments\/",
        "_ts": 1699302849
    }],
    "_count": 1
}
```

I'm sure you can figure out how to remove unnecessary headers from the response ;)

Now, let's test the second endpoint to get list of documents by condition with pagination.
Since I have only 2 documents in my database I will set the parameter "take" to 1 to demonstrate the pagination.

``` http
GET https://apim-samples-234f1f22a3.azure-api.net/document/take/1 HTTP/1.1
Host: apim-samples-234f1f22a3.azure-api.net
```

the response is:
``` http
HTTP/1.1 200 Ok
cache-control: no-store, no-cache
content-encoding: gzip
content-type: application/json
date: Wed, 08 Nov 2023 21:51:17 GMT
lsn: 34
pragma: no-cache
strict-transport-security: max-age=31536000
transfer-encoding: chunked
vary: Accept-Encoding,Origin
x-ms-aad-applied-role-assignment: ,a654236b-8ba1-0dd2-7a87-43dd83a89fa5
x-ms-activity-id: b1d57e0e-7467-461f-971b-c6e02cf8097b
x-ms-alt-content-path: dbs/mydatabase/colls/mycontainer
x-ms-content-path: YVEzAK5GDSw=
x-ms-continuation: {"token":"-RID:~YVEzAK5GDSwBAAAAAAAAAA==#RT:1#TRC:1#ISV:2#IEO:65567#QCF:8","range":{"min":"","max":"FF"}}
x-ms-cosmos-is-partition-key-delete-pending: false
x-ms-cosmos-llsn: 34
x-ms-cosmos-physical-partition-id: 0
x-ms-cosmos-query-execution-info: {"reverseRidEnabled":false,"reverseIndexScan":false}
x-ms-cosmos-quorum-acked-llsn: 34
x-ms-current-replica-set-size: 4
x-ms-current-write-quorum: 3
x-ms-documentdb-partitionkeyrangeid: 0
x-ms-gatewayversion: version=2.14.0
x-ms-global-committed-lsn: 34
x-ms-item-count: 1
x-ms-last-state-change-utc: Mon,06 Nov 2023 20:05:32.452 GMT
x-ms-number-of-read-regions: 0
x-ms-quorum-acked-lsn: 34
x-ms-request-charge: 2.26
x-ms-request-duration-ms: 20.615
x-ms-resource-quota: documentSize=51200;documentsSize=52428800;documentsCount=-1;collectionSize=52428800;
x-ms-resource-usage: documentSize=0;documentsSize=0;documentsCount=2;collectionSize=0;
x-ms-schemaversion: 1.16
x-ms-serviceversion: version=2.14.0.0
x-ms-session-token: 0:-1#34
x-ms-transport-request-id: 1
x-ms-xp-role: 1
    
{
    "_rid": "YVEzAK5GDSw=",
    "Documents": [{
        "id": "123",
        "name": "Alex",
        "_rid": "YVEzAK5GDSwBAAAAAAAAAA==",
        "_self": "dbs\/YVEzAA==\/colls\/YVEzAK5GDSw=\/docs\/YVEzAK5GDSwBAAAAAAAAAA==\/",
        "_etag": "\"7a002c05-0000-0d00-0000-65494dc10000\"",
        "_attachments": "attachments\/",
        "_ts": 1699302849
    }],
    "_count": 1
}
```

The important part of the response is the ```x-ms-continuation: {"token":"-RID:~YVEzAK5GDSwBAAAAAAAAAA==#RT:1#TRC:1#ISV:2#IEO:65567#QCF:8","range":{"min":"","max":"FF"}}``` header. 
The consumer should pass it back to the API Management in the header "from" to get the next page of the results.

``` http
GET https://apim-samples-234f1f22a3.azure-api.net/document/take/1 HTTP/1.1
Host: apim-samples-234f1f22a3.azure-api.net
from: {"token":"-RID:~YVEzAK5GDSwBAAAAAAAAAA==#RT:1#TRC:1#ISV:2#IEO:65567#QCF:8","range":{"min":"","max":"FF"}}
```
and the response is the second document:
``` http
HTTP/1.1 200 Ok
cache-control: no-store, no-cache
content-encoding: gzip
content-type: application/json
date: Wed, 08 Nov 2023 21:55:17 GMT
lsn: 34
pragma: no-cache
strict-transport-security: max-age=31536000
transfer-encoding: chunked
vary: Accept-Encoding,Origin
x-ms-aad-applied-role-assignment: ,a654236b-8ba1-0dd2-7a87-43dd83a89fa5
x-ms-activity-id: df0943e0-42ce-43c9-90e5-b4642dfeab64
x-ms-alt-content-path: dbs/mydatabase/colls/mycontainer
x-ms-content-path: YVEzAK5GDSw=
x-ms-continuation: [{"token":"-RID:~YVEzAK5GDSwCAAAAAAAAAA==#RT:2#TRC:2#ISV:2#IEO:65567#QCF:8","range":{"min":"","max":"FF"}}]
x-ms-cosmos-is-partition-key-delete-pending: false
x-ms-cosmos-llsn: 34
x-ms-cosmos-physical-partition-id: 0
x-ms-cosmos-query-execution-info: {"reverseRidEnabled":false,"reverseIndexScan":false}
x-ms-documentdb-partitionkeyrangeid: 0
x-ms-gatewayversion: version=2.14.0
x-ms-global-committed-lsn: 33
x-ms-item-count: 1
x-ms-last-state-change-utc: Mon,06 Nov 2023 20:05:55.349 GMT
x-ms-number-of-read-regions: 0
x-ms-request-charge: 2.26
x-ms-request-duration-ms: 21.668
x-ms-resource-quota: documentSize=51200;documentsSize=52428800;documentsCount=-1;collectionSize=52428800;
x-ms-resource-usage: documentSize=0;documentsSize=0;documentsCount=2;collectionSize=0;
x-ms-schemaversion: 1.16
x-ms-serviceversion: version=2.14.0.0
x-ms-session-token: 0:-1#34
x-ms-transport-request-id: 1
x-ms-xp-role: 2
    
{
    "_rid": "YVEzAK5GDSw=",
    "Documents": [{
        "id": "122",
        "name": "Elena",
        "_rid": "YVEzAK5GDSwCAAAAAAAAAA==",
        "_self": "dbs\/YVEzAA==\/colls\/YVEzAK5GDSw=\/docs\/YVEzAK5GDSwCAAAAAAAAAA==\/",
        "_etag": "\"7a00430a-0000-0d00-0000-65494ec70000\"",
        "_attachments": "attachments\/",
        "_ts": 1699303111
    }],
    "_count": 1
}
```

As you can see the x-ms-continuation header is here again but there is no nore documents to be returned for the query.
Yes I see that this time it is an array with an object in it. Don't ask me why. I don't know. Just pass it back to the API Management in the header "from" and you will get an empty response.

``` http
So, when the consumer will pass it back to the API Management in the header "from" it will get an empty response.

``` http
GET https://apim-samples-234f1f22a3.azure-api.net/document/take/1 HTTP/1.1
Host: apim-samples-234f1f22a3.azure-api.net
from: [{"token":"-RID:~YVEzAK5GDSwCAAAAAAAAAA==#RT:2#TRC:2#ISV:2#IEO:65567#QCF:8","range":{"min":"","max":"FF"}}]
```

the response is empty:
``` http
HTTP/1.1 200 Ok
cache-control: no-store, no-cache
content-encoding: gzip
content-type: application/json
date: Wed, 08 Nov 2023 21:59:16 GMT
lsn: 34
pragma: no-cache
strict-transport-security: max-age=31536000
transfer-encoding: chunked
vary: Accept-Encoding,Origin
x-ms-aad-applied-role-assignment: ,a654236b-8ba1-0dd2-7a87-43dd83a89fa5
x-ms-activity-id: 4a929ad5-08fc-4200-a5b5-427288d56b22
x-ms-alt-content-path: dbs/mydatabase/colls/mycontainer
x-ms-content-path: YVEzAK5GDSw=
x-ms-cosmos-is-partition-key-delete-pending: false
x-ms-cosmos-llsn: 34
x-ms-cosmos-physical-partition-id: 0
x-ms-cosmos-query-execution-info: {"reverseRidEnabled":false,"reverseIndexScan":false}
x-ms-documentdb-partitionkeyrangeid: 0
x-ms-gatewayversion: version=2.14.0
x-ms-global-committed-lsn: 33
x-ms-item-count: 0
x-ms-last-state-change-utc: Mon,06 Nov 2023 20:05:55.349 GMT
x-ms-number-of-read-regions: 0
x-ms-request-charge: 2.25
x-ms-request-duration-ms: 20.272
x-ms-resource-quota: documentSize=51200;documentsSize=52428800;documentsCount=-1;collectionSize=52428800;
x-ms-resource-usage: documentSize=0;documentsSize=0;documentsCount=2;collectionSize=0;
x-ms-schemaversion: 1.16
x-ms-serviceversion: version=2.14.0.0
x-ms-session-token: 0:-1#34
x-ms-transport-request-id: 1
x-ms-xp-role: 2
    
{
    "_rid": "YVEzAK5GDSw=",
    "Documents": [],
    "_count": 0
}
```

Now the consumer can stop calling the API.

## 6. Conclusion

In this post I have showed you how to connect Azure API Management to Azure Cosmosdb. This solution can be wery useful if you need to expose your CosmosDB data to the external consumers. Creating custom application that sits Azure API Management and CosmosDB will take much more time and effort.

Let me know if you have any questions or comments.