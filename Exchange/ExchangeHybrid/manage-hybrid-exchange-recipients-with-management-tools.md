---
ms.localizationpriority: medium
description: 'Manage on-premises recipients in Exchange Server 2019 Hybrid Environments using Exchange Server Management tools.'
ms.topic: how-to
author: JoanneHendrickson
ms.author: jhendr
ms.assetid: 928a4a0b-0082-4d50-a696-bfaf2782f42d
ms.reviewer: 
title: "Manage recipients in Exchange Server 2019 Hybrid environments"
ms.collection:
- Strat_EX_Admin
- exchange-server
f1.keywords:
- NOCSH
audience: ITPro
ms.prod: exchange-server-it-pro
manager: serdars

---
# Manage recipients in Exchange Hybrid environments using Management tools

In Exchange Hybrid environments, you must have an active Exchange Server to manage recipients attributes. First, attributes are edited using an Exchange Server in on-premises Active Directory (AD), that are then copied to Azure AD using directory synchronization. On-premises recipients can’t be modified directly in Azure Active Directory (Azure AD) or Exchange Online. Therefore you have to keep an Exchange Server running using directory synchronization via Azure Active Directory (AAD) Connect tool.

If you keep an Exchange server running just for recipient management, you may be able to shut down your last Exchange server and manage recipients using Windows PowerShell. 

>[!Important]
>You can still use this method to manage your recipients and leave your server running if you only run the Exchange server for recipient management. Shutting down the server is optional.

Install the latest Management tools provided through Exchange Server 2019 Setup on any domain-joined computer (client or server).  [Learn how to install the latest Management tools](/exchange/plan-and-deploy/post-installation-tasks/install-management-tools?view=exchserver-2019&preserve-view=true).

>[!Important]
>This feature is only available for Exchange Server 2019 Cumulative Update 12 or later.
 
## Will this work for me?

An updated version of the Exchange Management Tools can eliminate the need for running Exchange server if all of the following are true:

- You have migrated all mailboxes and public folders to Exchange Online 
- Use AD for recipient management and Azure AD Connect for synchronization
- You don't use/require the on-premises Exchange admin center or Exchange Role-Based Access Control (RBAC)
- Are comfortable with using only Windows PowerShell for recipient management
- You don't require auditing or logging of recipient management activity
- You are running only one Exchange server and only for recipient management purposes
- Want to manage recipients without running any Exchange servers.


>[!Warning]
>**DO NOT** uninstall the last server. You can choose to shut down the server, and use the script to clean up, but DO NOT uninstall. Uninstalling the server removes critical information out of Active Directory that results in breaking the management tool package to manage Exchange attributes. Learn more here: [Important: Be Aware](#important-be-aware)

With the updated Exchange Management Tools, domain admins and members of the Recipient Management EMT group (created through step 6 below) can use Windows PowerShell to run the following cmdlets without a running Exchange server:

- Set-MailUser, Get-MailUser, New-MailUser, Remove-MailUser, Disable-MailUser and Enable-MailUser
- Set-MailContact, Get-MailContact, New-MailContact, Remove-MailContact, Disable-MailContact and Enable-MailContact
- Set-RemoteMailbox, Get-RemoteMailbox, New-RemoteMailbox, Remove-RemoteMailbox, Disable-RemoteMailbox and Enable-RemoteMailbox
- Set-DistributionGroup, Get-DistributionGroup, New-DistributionGroup, Remove-DistributionGroup, Disable-DistributionGroup and Enable-DistributionGroup (excluding Upgrade-DistributionGroup)
- Get-DistributionGroupMember, Add-DistributionGroupMember, Remove-DistributionGroupMember and Update-DistributionGroupMember
- Set-EmailAddressPolicy, Get-EmailAddressPolicy, New-EmailAddressPolicy, Remove-EmailAddressPolicy and Update-EmailAddressPolicy
- Set-User and Get-User 

>[!Note]
> You can't modify on-premises recipients directly in Azure AD or Exchange Online. 

## Verify that Management Tools can run without Exchange Server

If your environment includes only a single Exchange server running solely because of recipient management requirements, use the steps below to test the Management Tools update.

### Prepare the Exchange Environment

1. Verify that all mailboxes are in the cloud by running the following commands in the Exchange Management Shell:

```powershell
Set-AdServerSettings -ViewEntireForest $true
Get-Mailbox
```
>[!Note]
>Built-in admin mailboxes are not synced to cloud through Azure AD Connect by default. Before you proceed, you should disable these mailboxes using Disable-Mailbox.

2. Verify that the Exchange Online tenant coexistence domain (usually something like “contoso.mail.onmicrosoft.com”) is configured as target delivery domain by running the following command:

```powershell
Get-RemoteDomain Hybrid* | fl DomainName,TargetDeliveryDomain
```

If the coexistence domain isn't added as a remote domain, you can add it using New-RemoteDomain. For example:

```powershell
New-RemoteDomain -Name 'Hybrid Domain - M365B434489.mail.onmicrosoft.com' -DomainName 'M365B434489.mail.onmicrosoft.com'
```

If it isn't set as the Target Delivery Domain, you can set it using *Set-RemoteDomain*. For example:

```powershell
Set-RemoteDomain -TargetDeliveryDomain: $true -Identity 'Hybrid Domain - M365B434489.mail.onmicrosoft.com'
```
>[!Note]
>In you have already removed Exchange Server or never had an Exchange Server to start with, Set-Remotedomain and New-RemoteDomain cmdlets can be accessed via Microsoft.Exchange.Management.PowerShell.E2010 snapin. Add the snapin before using the Set-RemoteDomain or New-RemoteDomain cmdlets.

3. [Install the Exchange Management Tools](/exchange/plan-and-deploy/post-installation-tasks/install-management-tools) role using the Exchange Server 2019 April 2022 Cumulative Update Setup. The updated tools can be installed on any domain-joined computer in an Exchange organization. It can be used in organizations running Exchange Server 2013, Exchange Server 2016, and/or Exchange Server 2019. 

>[!Note]
>Installing the updated Exchange Management Tools in an environment with only Exchange 2013 and/or Exchange 2016 will upgrade the Exchange organization to Exchange Server 2019, and it will perform an AD schema update. If you have a large AD deployment, or if a separate team manages AD, use the steps here: [Prepare Active Directory and domains for Exchange Serve](/Exchange/plan-and-deploy/prepare-ad-and-domains) to perform the schema update.

4. Install the Windows Remote Server Administration Tools using the steps in this article: [Install, uninstall and turn off/on RSAT tools](/windows-server/remote/remote-server-administration-tools#install-uninstall-and-turn-offon-rsat-tools).

5. If you have the Scripting Agent enabled, copy **ScriptingAgentConfig.xml** from *$env:ExchangeInstallPath\Bin\CmdletExtensionAgents* on the Exchange Server, to the *$env:ExchangeInstallPath\Bin\CmdletExtensionAgents* folder on the computer with the Management Tools update installed.

6. Run the provided script to create the Recipient Management EMT security group that grants users without Domain admin rights to manage recipients.
   
   a)	Sign-in to the computer with the Management Tools update as a Domain Admin and open Windows PowerShell.
   
   b)	Load the Recipient Management snap-in by running "Add-PSSnapin *RecipientManagement".
   
   c)	Run Add-PermissionForEMT.ps1 from the $env:ExchangeInstallPath\Scripts folder. The script creates a security group called Recipient Management EMT. Members of this group have recipient management permissions. All admins without domain admin rights need to perform recipient management should be added to this security group.

7.	Sign in to the computer with the Management Tools update with the appropriate permissions (domain admin or member of Recipient Management EMT) and load the Recipient Management snap-in by running 
```powershell
Add-PSSnapin *RecipientManagement 
```
This step must be performed every time you manage recipients.

8.	Test all recipient management cmdlets and verify that you see expected results.

9.	Shut down your last Exchange server and verify that all recipient management cmdlets still work as expected.

## Permanently shutting down your last Exchange Server

If you intend to permanently shut down your last Exchange Server, we recommend that you use the following steps to clean up aspects of your Exchange configuration to improve the security posture of your environment.

>[!Important]
>If you are using your last Exchange server for any purpose other than recipient management (e.g., for SMTP relay), then do not shut it down.

1. Turn on your last Exchange server.
2. Clean up your hybrid configuration by performing [Steps 1 to 8](/exchange/decommission-on-premises-exchange#to-keep-ad-fs-and-directory-synchronization-and-decommission-most-of-the-exchange-servers) for Scenario 2 in How and when to decommission your on-premises Exchange servers in a hybrid deployment.

3. Remove the Federation Trust by running the following command in the Exchange Management Shell:
```powershell
Remove-FederationTrust "Microsoft Federation Gateway" 
```

4. Remove the Federation Certificate:  To find the certificate thumbprint, run: 
```powershell
$fedThumbprint = (Get-ExchangeCertificate | ?{$_.Subject -eq "CN=Federation"}).Thumbprint  
``` 
To remove the certificate thumbprint, run:
```powershell
Remove-ExchangeCertificate –Thumbprint $fedThumbprint
```
5. Remove the service principal credentials created for OAuth. To do this, you need to determine which KeyId matches the key value of the OAuth certificate. To find the KeyId that matches, follow these steps:

   a. Run these commands in the Exchange Management Shell to get the OAuth credValue:
```powershell
$thumbprint = (Get-AuthConfig).CurrentCertificateThumbprint 
$oAuthCert = (dir Cert:\LocalMachine\My) | where {$_.Thumbprint -match $thumbprint} 
$certType = [System.Security.Cryptography.X509Certificates.X509ContentType]::Cert 
$certBytes = $oAuthCert.Export($certType) 
$credValue = [System.Convert]::ToBase64String($certBytes) 
```
  b. Find the KeyId that is same as the $credValue found above, run the following commands as a tenant admin using the Azure Active Directory Module for Windows PowerShell.

```powershell
Install-Module -Name MSOnline
Connect-MsolService
$ServiceName = "00000002-0000-0ff1-ce00-000000000000" 
$p = Get-MsolServicePrincipal -ServicePrincipalName $ServiceName 
$keyId = (Get-MsolServicePrincipalCredential -AppPrincipalId $p.AppPrincipalId -ReturnKeyValues $true | ?{$_.Value -eq $credValue}).KeyId 
``` 
This gives the KeyId of the key whose value matches the $credValue found above.

   c. To remove the service principal credential, run the following command: 

```powershell
Remove-MsolServicePrincipalCredential –KeyIds @($keyId) -AppPrincipalId $p.AppPrincipalId
```
6.	Uninstall the Hybrid agent. If your environment has a Modern Hybrid configuration, follow the steps below to remove it.

   a.	In the computer where Hybrid Agent is installed, open the Exchange Management Shell and change directory to the location of the script **C:\Program Files\Microsoft Hybrid Service\HybridManagement.psm1** and then import the Hybrid Agent PowerShell Module.
  ```PowerShell
      Import-Module .\HybridManagement.psm1
  ```
  
   b.	To remove the App, an AppId is required. Use any of the following cmdlets in Exchange Online PowerShell to find the AppId.  

```powershell
Get-OrganizationRelationship ((Get-OnPremisesOrganization).OrganizationRelationship) | Select-Object TargetSharingEpr 
```
The output looks like this:

```powershell
TargetSharingEpr
----------------
https://6ca7c832-49a2-4a5d-aeae-a616f6d4b8e7.resource.mailboxmigration.his.msappproxy.net/EWS/Exchange.asmx
----------------
https://6ca7c832-49a2-4a5d-aeae-a616f6d4b8e7.resource.mailboxmigration.his.msappproxy.net/EWS/Exchange.asmx
```

Or, run:

```powershell
Get-MigrationEndpoint "Hybrid Migration Endpoint - EWS (Default Web Site)" | Select-Object RemoteServer
```
The output looks like this:

```powershell 
RemoteServer
------------
6ca7c832-49a2-4a5d-aeae-a616f6d4b8e7.resource.mailboxmigration.his.msappproxy.net
```
In this example, 6ca7c832-49a2-4a5d-aeae-a616f6d4b8e7 is the AppId to be used in the next step.

   c. Remove the App by running:

```powershell
Remove-HybridApplication -appId 6ca7c832-49a2-4a5d-aeae-a616f6d4b8e7 -Credential (Get-Credential)
```

>[!Note]
>The AppId is 6ca7c832-49a2-4a5d-aeae-a616f6d4b8e7 only for this example; your value will be different.

   d.	Uninstall the Hybrid agent using the steps here: [Uninstall the hybrid agent](/exchange/hybrid-deployment/hybrid-agent#uninstall-the-hybrid-agent). 
   
7.	If not already done, point your MX and Autodiscover DNS records to Exchange Online. This is important to ensure mail flow isn't affected. For more information, see External Domain Name System records for Office 365. 

8.	Shut down your last Exchange server.

## Active Directory clean up

If you plan to never run Exchange Server on-premises, we recommend that you clean up your Active Directory by removing unnecessary Exchange objects. 

>[!Important]
>This step can’t be undone, so only proceed if you **never** want to run Exchange Server again.
 
AD cleanup can be done by running the CleanupActiveDirectoryEMT script shipped with Management tools. The script removes system mailboxes, unnecessary Exchange containers, permissions for Exchange Security Groups on the domain and configuration partitions, and the Exchange Security Groups. You need to run this script with domain admin credentials.

This script is available at: *$env:ExchangeInstallPath\Scripts\CleanupActiveDirectoryEMT.ps1*

### Important: Be Aware

>[!Warning]
>**Once you shut down the last Exchange server, Exchange RBAC will no longer function**. Users who were a part of Exchange Recipient groups or had custom Exchange roles allowing for recipient management will no longer have permission. Only domain admins and users who are assigned permission using Add-PermissionForEMT.ps1 script will be able to perform recipient management.
>
>Once you shut down your last Exchange server and perform the Exchange hybrid and Active Directory cleanup steps listed above, you should **erase and reformat your last Exchange server**. **Do Not Uninstall the Exchange Server**.
