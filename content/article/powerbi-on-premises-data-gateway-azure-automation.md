---
title: "Automate Power BI On-Premises Data Gateway (Self-Hosted) on Azure VM"
date: 2025-04-05T06:39:47Z
draft: false
keywords: "powerbi, powerbi-gateway, azzure, automation, terraform"
description: "How to automate the installation of On-Premises Data Gateway on Azure VM"
summary: "![](/images/powerbi-on-premises-data-gateway-azure-automation/gateway-connectivity.png)
Microsoft Power BI Pro (the cloud service) often needs to query data that resides in secure, private networks. In our scenario, the data source is an Azure SQL Database deployed inside a private Azure Virtual Network with no public internet access. By default, Power BI’s cloud service cannot reach such isolated data sources directly​.
In this article, we will focus on the first solution, which is more flexible and allows for a wider range of data sources."
---

## Background and Challenges

Microsoft Power BI Pro (the cloud service) often needs to query data that resides in secure, private networks. In our scenario, the data source is an Azure SQL Database deployed inside a private Azure Virtual Network with no public internet access. By default, Power BI’s cloud service cannot reach such isolated data sources directly​. [learn.microsoft.com](https://learn.microsoft.com/en-us/power-bi/guidance/powerbi-implementation-planning-data-gateways). 
![](/images/powerbi-on-premises-data-gateway-azure-automation/no-connectivity-1.png)
Enabling broad access (for example, by turning on “Allow Azure Services…” on the SQL firewall) would expose the database to all Azure services and is not recommended for production​. Therefore, we need a solution that preserves security (no public endpoints) while allowing Power BI to query the data.

Key challenges and requirements include:
- **No Public Exposure**: The Azure SQL must remain inaccessible from the public internet (e.g. using Private Endpoints and disabling public network access)​
- **Power BI Service Connectivity**: Power BI (a SaaS cloud service) must be able to reach the database through a secure channel despite the network isolation.
- **Security First**: All data transit should occur over secure, internal channels (e.g. Azure backbone)​, with minimal attack surface (no inbound ports opened to random addresses).
- **Automation**: The entire setup should be deployable and repeatable.
- **Deployment and Recovery**: The solution should account for high availability, easy re-deployment (e.g. updates or changes), and disaster recovery (e.g. if the gateway VM or service fails, how to recover access).

Microsoft provides two solutions to this problem:
1. **On-Premises Data Gateway (Self-Hosted)**: running the standard Power BI Enterprise Gateway on a VM inside the VNet.
2. **Virtual Network Data Gateway (Managed)**: using Power BI’s VNet integration feature (managed gateway service injected into your VNet, available with Premium capacities).

In this article, we will focus on the first solution, which is more flexible and allows for a wider range of data sources.

## Azure VM with On-Premises Data Gateway (Self-Hosted)

In this pattern, we deploy an On-Premises Data Gateway (standard mode) on a Windows VM that resides inside the same private VNet as the Azure SQL Database (or one that has network access to it). Despite the name, this gateway can be used not only for on-premises networks but also for private cloud networks​ [learn.microsoft.com](https://learn.microsoft.com/en-us/power-bi/guidance/powerbi-implementation-planning-data-gateways). The gateway acts as a bridge between Power BI cloud and the private database:
![](/images/powerbi-on-premises-data-gateway-azure-automation/gateway-connectivity.png)
This approach keeps the database fully isolated. The only “pipe” into the VNet is the gateway’s outgoing relay to Power BI. Azure SQL accepts connections only from within the VNet (the gateway VM’s IP). All data in transit is encrypted via TLS. We avoid opening any broad firewall rules (like “Allow Azure services…”) which would otherwise allow unknown Azure IPs​. We also avoid exposing a public IP for the database at all. DNS resolution of the database name is handled via the private DNS zone so that even the gateway uses the internal address.

## Automation

### Install the On-Premises Data Gateway on Azure VM

There are two options to install the On-Premises Data Gateway on the Azure VM:
1. Using the [Installer executable](https://learn.microsoft.com/en-us/data-integration/gateway/service-gateway-monthly-updates)
2. Using the [DataGateway PowerShell module](https://learn.microsoft.com/en-us/powershell/module/datagateway/?view=datagateway-ps) (Public Preview as of 2025-04-05)

The first option is more straightforward, however, it requires manual installation and configuration of the gateway. The second option allows for a more automated approach, but it requires additional setup and configuration.

#### Using the DataGateway PowerShell module

This approach includes the installation of the [PowerShell Cmdlets for On-premises data gateway management](https://learn.microsoft.com/en-us/powershell/gateway/overview?view=datagateway-ps) which requires the [PowerShell 7.0.6 or higher](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.5&viewFallbackFrom=powershell-7&WT.mc_id=THOMASMAURER-blog-thmaure)

[Microsoft recommends installing the PowerShell using WinGet](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.5&WT.mc_id=THOMASMAURER-blog-thmaure#install-powershell-using-winget-recommended). However, [this method also works well](https://github.com/PowerShell/PowerShell/blob/master/tools/install-powershell.ps1-README.md):

``` powershell
# Download PowerShell 7
Invoke-Expression -Command "&{$(Invoke-RestMethod https://aka.ms/install-powershell.ps1)} -UseMSI -Quiet"
."C:\Program Files\PowerShell\7\pwsh.exe"
```

After installing PowerShell, we can install the DataGateway module using the following command:

``` powershell
# Install DataGateway module
Start-Process pwsh -ArgumentList "-Command Install-Module -Name DataGateway -Force" -Wait
```

After installing the DataGateway module, we need to do the following:

``` powershell
# Connect to the Data Gateway service (https://learn.microsoft.com/en-us/powershell/module/datagateway.profile/connect-datagatewayserviceaccount?view=datagateway-ps)
Connect-DataGatewayServiceAccount `
    -ApplicationId "YOUR APPLICATION ID" `
    -Tenant "YOUR TENANT ID" `
    -ClientSecret "YOUR SERVICE PRINCIPAL SECRET"

# Install the Data Gateway (https://learn.microsoft.com/en-us/powershell/module/datagateway/install-datagateway?view=datagateway-ps)
Install-DataGateway -AcceptConditions

# Create a cluster(https://learn.microsoft.com/en-us/powershell/module/datagateway/add-datagatewaycluster?view=datagateway-ps)
$recoveryKey = ConvertTo-SecureString "YOUR RECOVERY KEY HERE" `
    -AsPlainText `
    -Force;
$cluster = Add-DataGatewayCluster `
    -Name "YOUR CLUSTER NAME HERE" `
    -RecoveryKey $recoveryKey
```

## Recovery / Re-Deployment

If you look at the [DataGateway module documentation](https://learn.microsoft.com/en-us/powershell/module/datagateway/?view=datagateway-ps), you will not find any command to add newly installed gateways to the cluster (as of 2025-04-05). However, the **[Add-DataGatewayClusterMember command was announced in October 2023](https://powerbi.microsoft.com/en-in/blog/on-premises-data-gateway-october-2023-release/)** and the Microsoft documentation is not updated yet (2025-04-05).

The **Add-DataGatewayClusterMember** will register a new gateway to the local machine and add it as a new member to the gateway cluster you specify. The recovery key and gateway cluster id in the command must pertain to the primary gateway node. In addition, the gateway must already be installed, but not registered to the local machine. The command takes three inputs:
- **GatewayName** (string): This is the name of the gateway that will be created on the local machine and added as a member to the cluster. It cannot conflict with any existing gateways on the same tenant.
- **RecoveryKey** (Secure string): The recovery key of the primary gateway member in the existing cluster. The recovery key is used by the gateway to encrypt/decrypt on-prem credentials. This is also required to restore the gateway or add a new member to the gateway cluster.
- **GatewayClusterId** (Guid): The gateway object id of the primary gateway node.

## Terraform

Having all the above in mind, we can now create a powershell script to automate the installation of the On-Premises Data Gateway on the Azure VM.

Here is a sample Terraform code to attach the installation script to the VM and run it after the VM is created:

``` hcl
resource "azurerm_windows_virtual_machine" "vm" {
    ...
}

data "template_file" "ps5script" {
  template = file("${path.module}/installation-scripts/ps5script.ps1")
}

resource "azurerm_virtual_machine_extension" "script" {
  name                 = "install_data_gateway"
  virtual_machine_id   = azurerm_windows_virtual_machine.vm.id
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.9"
  protected_settings   = <<SETTINGS
  {
    "commandToExecute": "powershell -command \"[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('${base64encode(data.template_file.ps5script.rendered)}')) | Out-File -filepath ps5script.ps1\" && powershell -ExecutionPolicy Unrestricted -File ps5script.ps1 > ps5script.log"
  }
  SETTINGS
}
```

## Conclusion
In this article, we have discussed how to set up an On-Premises Data Gateway (Self-Hosted) on an Azure VM using DataGateway module. We have also discussed how to automate the installation and configuration of the gateway using PowerShell and Terraform. This solution allows us to securely connect Power BI to our private Azure SQL Database without exposing it to the public internet.
This approach also allows for easy re-deployment and disaster recovery in case of VM or service failures. The use of the DataGateway PowerShell module allows for a more automated approach to managing the gateway, while the Terraform code allows for easy deployment and management of the Azure resources.