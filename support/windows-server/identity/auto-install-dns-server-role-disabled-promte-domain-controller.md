---
title: Unable to select DNS Server role when adding a domain controller into an existing Active Directory domain
description: Fixes an issue where the option to auto-install the DNS Server role is disabled or grayed out in the Active Directory Installation Wizard (DCPROMO) when promoting a Windows Server 2008 or Windows Server 2008 R2 replica domain controller.
ms.date: 09/07/2020
author: Deland-Han
ms.author: delhan
manager: dscontentpm
audience: itpro
ms.topic: troubleshooting
ms.prod: windows-server
localization_priority: medium
ms.reviewer: arrenc, kaushika
ms.prod-support-area-path: DCPromo and the installation of domain controllers
ms.technology: ActiveDirectory
---
# Unable to select DNS Server role when adding a domain controller into an existing Active Directory domain

This article helps fix an issue where the option to auto-install the DNS Server role is disabled or grayed out in the Active Directory Installation Wizard (DCPROMO) when promoting a Windows Server 2008 or Windows Server 2008 R2 replica domain controller.
 
_Original product version:_ &nbsp;Windows Server 2012 R2  
_Original KB number:_ &nbsp;2002584

## Symptoms

When promoting a Windows Server 2008 or Windows Server 2008 R2 replica domain controller, the option to auto-install the DNS Server role is disabled or grayed out in the Active Directory Installation Wizard (DCPROMO).

Text in the **Additional information** field states:

DNS cannot be installed on this domain controller because this domain does not host DNS.

A screenshot of this condition is shown below:

![DCPROMODNS checkbox greyed out during replica promotion.](./media/auto-install-dns-server-role-disabled-promte-domain-controller/dcpromo-dns-check-box-greyed.jpg)

The %windir%\debug\dcpromoui.log file on the replica domain controller being promoted shows the following: 

> Enter DoesDomainHostDns SLD  
dcpromoui A74.A78 046C 14:07:18.800                 Dns_DoesDomainHostDns testing domain name SLD  
dcpromoui A74.A78 046D 14:07:19.113                 SOA query returned 9003 so the domain does not host DNS  
dcpromoui A74.A78 046E 14:07:19.113                 Dns_DoesDomainHostDns returning false  
dcpromoui A74.A78 046F 14:07:19.113                 HRESULT = 0x00000000  
dcpromoui A74.A78 0470 14:07:19.113                 The domain does not host DNS.  

## Cause


1. A code defect prevents the DNS Server checkbox from being enabled when promoting replica domain controllers into existing domains with single-label DNS names like "contoso" instead of best-practice fully qualified DNS name like "`contoso.com`" or "`corp.contoso.com`". This condition exists even when Microsoft DNS is installed on a domain controller and hosts Active Directory-integrated forward lookup zones for the target domain.

    For more information about single label domains, visit the following Microsoft web site:  

     [Microsoft DNS Namespace Planning Solution Center](https://support.microsoft.com/gp/gp_namespace_master#tab4) 
    
OR

2. DCPromo checks to see if the DNS zone for the target Active Directory forest is hosted in Active Directory. If the DNS zone for the target domain isn't hosted on an existing domain controller in the target forest, DCPROMO doesn't allow the user to install DNS during the replica promotion.

    The goal of this behavior is to prevent administrators from creating duplicate copies of DNS zones with different replication scopes (that is, file-based zones on Microsoft or third-party DNS Servers and Active Directory integrated DNS zones on domain controllers on the newly promoted domain controller).
    
## Resolution

For the first root cause, continue the promotion and install the DNS Server role after it's promoted.

For the second root cause, the DNS client and server configuration on the replica domain controller being promoted was sufficient to discover a helper domain controller in the target domain but DCPROMO has determined that the DNS zone for the domain wasn't Active Directory integrated.  

Determine which DNS servers are going to host the zone for your Active Directory domain and what replication scopes those zones will use (Microsoft DNS versus third-party DNS, forest-wide application partition, domain-wide application partition, file-based primary, and so on.)

Don't let the inability to auto-install the DNS Server role during DCPROMO block the promotion of Windows Server 2008 replica domain controllers in the domain. Server Manager can be used to install the Microsoft DNS Server role on existing domain controllers, as well as computers functioning as member or workgroup computers. DNS zones and their records can be replicated or copied between DNS servers.

 **Specific workarounds include:** 

1. If the DNS zones exist on DNS servers outside the domain, consider moving the zones to an existing domain controller in the domain that hosts the DNS Server role.

2. If zone data needs to be moved, configure the Microsoft DNS server to host a secondary copy of the zone, then convert that zone to be a file-based primary, then transition the zone to be Active Directory integrated as required. You can ignore this step if you have no interest in saving the DNS zone data.

3. Configure the new replica domain controller being promoted to point exclusively to DNS servers hosting Active Directory integrated copies of the zone.

4. Use the following command to force Windows 2000, Windows XP, Windows Server 2003, Windows Vista, and Windows Server 2008 computers to dynamically register Host A or AAAA records:

    ```console
    ipconfig /registerdns  
    ```
5. Use the following command to force Windows 2000, Windows Server 2003, and Windows Server 2008 domain controllers to dynamically register SRV records
     ```console
    net stop netlogon & net start netlogon 
    ```
6. Restart DCPROMO on the replica domain controller.

## More information

**NOTE: The following information is MSInternal and shouldn't be shared with customers.**
  
Explanation from Jeff Westhead via email on 2008.10.10.

Installing DNS during DCPROMO autocreates DNS application partitions

If the first DC in the domain does not host DNS, then neither should any replica. The time to decide to rearchitect your DNS deployment is not while you are promoting a replica!

Informal testing has shown that AD-integrated DNS may be installed on DCs other than the first DC in the domain and replicas will STILL have the option to install DNS Server role during DCPROMO. \<end Arrenc edit>

I don't believe this is actually a check for a third-party DNS server. This might not be stringent enough but it most likely works well most of the time. I believe dcpromo calls Dns_DoesDomainHostDns, which performs a SOA query for the domain name and ensures that the response asserts that the start of authority for the name is the name itself rather than some parent name. This is testing that there is, on some DNS server somewhere in the infrastructure, a DNS zone that exactly matches the name of the domain.

For example, if I'm promoting a replica for "`child.corp.contoso.com`" but the only zone that exists is "`corp.contoso.com`" then this check will fail and dcpromo will assume that installing DNS on this replica is the wrong thing to do.

What dcpromo is trying to determine is whether or not DNS is required to support AD. If it's not, then dcpromo shouldn't offer the option to install DNS. The admin should be forced to install the DNS server role manually after dcpromo if the admin intends to run a DNS server on this DC for some purpose that's not related to serving DNS for the Active Directory domain.

 **Sample Customer Experience:**  

SRX0808XXXXX495. 19 logs in 7.52 hours of labor. Customer type: Partner. Customer name: \<removed>. Case Title: DNS Server option is grayed out during DCPROMO. L7: Customer calls when DCPROMO doesn't give the option to install DNS. L8: EasyAssist. PSS finds customer using single-label DNS domain name. PSS says promote DC and add DNS after the fact.

SRX0809XXXXX240. 6.63 hours in 16 logs. Partner. \<customer name removed>. Title: DNS Role can't be installed. Problem statement IS that checkbox to install DNS can't be enabled in DCPROMO. PSS says DNS zone for child domain resides on parent domain DNS Servers vs. delegating to DNS Servers in child domain.

Keywords:

gray grey grayed greyed unselected deselected unchecked checked enabled enable disable disabled checkbox check checkmark