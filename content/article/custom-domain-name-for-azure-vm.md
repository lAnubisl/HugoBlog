---
title: "Custom domain name for Azure VM"
date: 2015-10-04T23:33:03+03:00
draft: false
Summary: "If you are running VM on Azure you know that your default domain name looks like this: `yourservicename.cloudapp.net`. Also public IP address is not static that means you can't use 'A' DNS record on it."
---

If you are running VM on Azure you know that your default domain name looks like this: "yourservicename.cloudapp.net". Also public IP address is not static that means you can't use 'A' DNS record on it. If you want to use different domain name like "john.com" then you have 2 options:

*   Use 'CNAME' DNS record
*   Use reserved IP record on Azure and 'A' DNS record

Here are the details about both of them.

## Use CNAME DNS record

Azure VM gets dynamic public IP by default. This means that you cannot use "A" DNS record to address it. Your VM gets default domain name like "yourservicename.cloudapp.net" and you can use it to create canonical name reference from your "john.com" to "yourservicename.cloudapp.net". BUT! your domain name provider can not support CNAME to different domain (my doesn't ;( ).

## Use reserved IP record on Azure and 'A' DNS record

As I said your VM has dynamic IP by default. Nevertheless you can create "[reserved IP address](https://azure.microsoft.com/en-us/documentation/articles/virtual-networks-reserved-public-ip/)" to your subscription. As described on "[pricing page](https://azure.microsoft.com/en-us/pricing/details/ip-addresses/)" you can have up to 5 reserved IP addresses for free.

Download and install [Azure PowerShell](https://github.com/Azure/azure-powershell/releases)

Open it and authorize using the following command

``` powershell 
Add-AzureAccount 
```

Get available locations using the following command

``` powershell
Get-AzureLocation | Select DisplayName
```

you will see something like

``` powershell
 DisplayName
 -----------
 Central US
 South Central US
 East US 2
 North Europe
 Southeast Asia
 East Asia
 Japan West
 ```

Next you need to add reserved IP address to our subscription and specify location of your choice

``` powershell
New-AzureReservedIP "MyReservedIP" -Label "MyReservedIpLabel" -Location "North Europe"
```

Take your VM name. You can find it on Azure web portal in VMs listing table.

Assign your reserved IP address to your vm

``` powershell
Set-AzureReservedIPAssociation -ReservedIPName MyReservedIP -ServiceName MyServiceName
```

Now your VM's public IP should be changed. Also it should be static and you can use it for 'A' DNS record.

Profit!