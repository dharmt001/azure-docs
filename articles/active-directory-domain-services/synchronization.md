---
title: How synchronization works in Azure AD Domain Services | Microsoft Docs
description: Learn how the synchronization process works for objects and credentials from an Azure AD tenant or on-premises Active Directory Domain Services environment to an Azure Active Directory Domain Services managed domain.
services: active-directory-ds
author: iainfoulds
manager: daveba

ms.assetid: 57cbf436-fc1d-4bab-b991-7d25b6e987ef
ms.service: active-directory
ms.subservice: domain-services
ms.workload: identity
ms.topic: conceptual
ms.date: 08/20/2019
ms.author: iainfou

---
# How objects and credentials are synchronized in an Azure AD Domain Services managed domain

Objects and credentials in an Azure Active Directory Domain Services (AD DS) managed domain can either be created locally within the domain, or synchronized from an Azure Active Directory (AD) tenant. When you first deploy Azure AD DS, an automatic one-way synchronization is configured and started to replicate the objects from Azure AD. This one-way synchronization continues to run in the background to keep the Azure AD DS managed domain up-to-date with any changes from Azure AD.

In a hybrid environment, objects and credentials from an on-premises AD DS domain can be synchronized to Azure AD using Azure AD Connect. Once those objects are successfully synchronized to Azure AD, the automatic background sync then makes those objects and credentials available to applications using the Azure AD DS managed domain.

The following diagram illustrates how synchronization works between Azure AD DS, Azure AD, and an optional on-premises AD DS environment:

![Synchronization overview for an Azure AD Domain Services managed domain](./media/active-directory-domain-services-design-guide/sync-topology.png)

## Synchronization from Azure AD to Azure AD DS

User accounts, group memberships, and credential hashes are synchronized one way from Azure AD to Azure AD DS. This synchronization process is automatic. You don't need to configure, monitor, or manage this synchronization process. The initial synchronization may take a few hours to a couple of days, depending on the number of objects in the Azure AD directory. After the initial synchronization is complete, changes that are made in Azure AD, such as password or attribute changes, take about 20-30 minutes to be updated in Azure AD DS.

The synchronization process is one way / unidirectional by design. There's no reverse synchronization of changes from Azure AD DS back to Azure AD. An Azure AD DS managed domain is largely read-only except for custom OUs that you can create. You can't make changes to user attributes, user passwords, or group memberships within an Azure AD DS managed domain.

## Attribute synchronization and mapping to Azure AD DS

The following table lists some common attributes and how they're synchronized to Azure AD DS.

| Attribute in Azure AD DS | Source | Notes |
|:--- |:--- |:--- |
| UPN | User's *UPN* attribute in Azure AD tenant | The UPN attribute from the Azure AD tenant is synchronized as-is to Azure AD DS. The most reliable way to sign in to an Azure AD DS managed domain is using the UPN. |
| SAMAccountName | User's *mailNickname* attribute in Azure AD tenant or autogenerated | The *SAMAccountName* attribute is sourced from the *mailNickname* attribute in the Azure AD tenant. If multiple user accounts have the same *mailNickname* attribute, the *SAMAccountName* is autogenerated. If the user's *mailNickname* or *UPN* prefix is longer than 20 characters, the *SAMAccountName* is autogenerated to meet the 20 character limit on *SAMAccountName* attributes. |
| Passwords | User's password from the Azure AD tenant | Legacy password hashes required for NTLM or Kerberos authentication are synchronized from the Azure AD tenant. If the Azure AD tenant is configured for hybrid synchronization using Azure AD Connect, these password hashes are sourced from the on-premises AD DS environment. |
| Primary user/group SID | Autogenerated | The primary SID for user/group accounts is autogenerated in Azure AD DS. This attribute doesn't match the primary user/group SID of the object in an on-premises AD DS environment. This mismatch is because the Azure AD DS managed domain has a different SID namespace than the on-premises AD DS domain. |
| SID history for users and groups | On-premises primary user and group SID | The *SidHistory* attribute for users and groups in Azure AD DS is set to match the corresponding primary user or group SID in an on-premises AD DS environment. This feature helps make lift-and-shift of on-premises applications to Azure AD DS easier as you don't need to re-ACL resources. |

> [!TIP]
> **Sign in to the managed domain using the UPN format** The *SAMAccountName* attribute, such as `CONTOSO\driley`, may be auto-generated for some user accounts in an Azure AD DS managed domain. Users' auto-generated *SAMAccountName* may differ from their UPN prefix, so isn't always a reliable way to sign in. For example, if multiple users have the same *mailNickname* attribute or users have overly long UPN prefixes, the *SAMAccountName* for these users may be auto-generated. Use the UPN format, such as `driley@contoso.com`, to reliably sign in to an Azure AD DS managed domain.

### Attribute mapping for user accounts

The following table illustrates how specific attributes for user objects in Azure AD are synchronized to corresponding attributes in Azure AD DS.

| User attribute in Azure AD | User attribute in Azure AD DS |
|:--- |:--- |
| accountEnabled |userAccountControl (sets or clears the ACCOUNT_DISABLED bit) |
| city |l |
| country |co |
| department |department |
| displayName |displayName |
| facsimileTelephoneNumber |facsimileTelephoneNumber |
| givenName |givenName |
| jobTitle |title |
| mail |mail |
| mailNickname |msDS-AzureADMailNickname |
| mailNickname |SAMAccountName (may sometimes be autogenerated) |
| mobile |mobile |
| objectid |msDS-AzureADObjectId |
| onPremiseSecurityIdentifier |sidHistory |
| passwordPolicies |userAccountControl (sets or clears the DONT_EXPIRE_PASSWORD bit) |
| physicalDeliveryOfficeName |physicalDeliveryOfficeName |
| postalCode |postalCode |
| preferredLanguage |preferredLanguage |
| state |st |
| streetAddress |streetAddress |
| surname |sn |
| telephoneNumber |telephoneNumber |
| userPrincipalName |userPrincipalName |

### Attribute mapping for groups

The following table illustrates how specific attributes for group objects in Azure AD are synchronized to corresponding attributes in Azure AD DS.

| Group attribute in Azure AD | Group attribute in Azure AD DS |
|:--- |:--- |
| displayName |displayName |
| displayName |SAMAccountName (may sometimes be autogenerated) |
| mail |mail |
| mailNickname |msDS-AzureADMailNickname |
| objectid |msDS-AzureADObjectId |
| onPremiseSecurityIdentifier |sidHistory |
| securityEnabled |groupType |

## Synchronization from on-premises AD DS to Azure AD and Azure AD DS

Azure AD Connect is used to synchronize user accounts, group memberships, and credential hashes from an on-premises AD DS environment to Azure AD. Attributes of user accounts such as the UPN and on-premises security identifier (SID) are synchronized. To sign in using Azure AD Domain Services, legacy password hashes required for NTLM and Kerberos authentication are also synchronized to Azure AD.

If you configure write-back, changes from Azure AD are synchronized back to the on-premises AD DS environment. For example, if a user changes their password using Azure AD self-service password management, the password is updated back in the on-premises AD DS environment.

> [!NOTE]
> Always use the latest version of Azure AD Connect to ensure you have fixes for all known bugs.

### Synchronization from a multi-forest on-premises environment

Many organizations have a fairly complex on-premises AD DS environment that includes multiple forests. Azure AD Connect supports synchronizing users, groups, and credential hashes from multi-forest environments to Azure AD.

Azure AD has a much simpler and flat namespace. To enable users to reliably access applications secured by Azure AD, resolve UPN conflicts across user accounts in different forests. Azure AD DS managed domains use a flat OU structure, similar to Azure AD. All user accounts and groups are stored in the *AADDC Users* container, despite being synchronized from different on-premises domains or forests, even if you've configured a hierarchical OU structure on-premises. The Azure AD DS managed domain flattens any hierarchical OU structures.

As previously detailed, there's no synchronization from Azure AD DS back to Azure AD. You can [create a custom Organizational Unit (OU)](create-ou.md) in Azure AD DS and then users, groups, or service accounts within those custom OUs. None of the objects created in custom OUs are synchronized back to Azure AD. These objects are available only within the Azure AD DS managed domain, and aren't visible using Azure AD PowerShell cmdlets, Azure AD Graph API, or using the Azure AD management UI.

## What isn't synchronized to Azure AD DS

The following objects or attributes aren't synchronized to Azure AD or to Azure AD DS:

* **Excluded attributes:** You can choose to exclude certain attributes from synchronizing to Azure AD from an on-premises AD DS environment using Azure AD Connect. These excluded attributes aren't then available in Azure AD DS.
* **Group Policies:** Group Policies configured in an on-premises AD DS environment aren't synchronized to Azure AD DS.
* **Sysvol folder:** The contents of the *Sysvol* folder in an on-premises AD DS environment aren't synchronized to Azure AD DS.
* **Computer objects:** Computer objects for computers joined to an on-premises AD DS environment aren't synchronized to Azure AD DS. These computers don't have a trust relationship with the Azure AD DS managed domain and only belong to the on-premises AD DS environment. In Azure AD DS, only computer objects for computers that have explicitly domain-joined to the managed domain are shown.
* **SidHistory attributes for users and groups:** The primary user and primary group SIDs from an on-premises AD DS environment are synchronized to Azure AD DS. However, existing *SidHistory* attributes for users and groups aren't synchronized from the on-premises AD DS environment to Azure AD DS.
* **Organization Units (OU) structures:** Organizational Units defined in an on-premises AD DS environment don't synchronize to Azure AD DS. There are two built-in OUs in Azure AD DS - one for users, and one for computers. The Azure AD DS managed domain has a flat OU structure. You can choose to [create a custom OU in your managed domain](create-ou.md).

## Password hash synchronization and security considerations

When you enable Azure AD DS, legacy password hashes for NTLM + Kerberos authentication are required. Azure AD doesn't store clear-text passwords, so these hashes can't automatically be generated for existing user accounts. Once generated and stored, NTLM and Kerberos compatible password hashes are always stored in an encrypted manner in Azure AD. The encryption keys are unique to each Azure AD tenant. These hashes are encrypted such that only Azure AD DS has access to the decryption keys. No other service or component in Azure AD has access to the decryption keys. Legacy password hashes are then synchronized from Azure AD into the domain controllers for an Azure AD DS managed domain. The disks for these managed domain controllers in Azure AD DS are encrypted at rest. These password hashes are stored and secured on these domain controllers similar to how passwords are stored and secured in an on-premises AD DS environment.

For cloud-only Azure AD environments, [users must reset/change their password](tutorial-create-instance.md#enable-user-accounts-for-azure-ad-ds) in order for the required password hashes to be generated and stored in Azure AD. For any cloud user account created in Azure AD after enabling Azure AD Domain Services, the password hashes are generated and stored in the NTLM and Kerberos compatible formats. Those new accounts don't need to reset/change their password generate the legacy password hashes.

For hybrid user accounts synced from on-premises AD DS environment using Azure AD Connect, you must [configure Azure AD Connect to synchronize password hashes in the NTLM and Kerberos compatible formats](active-directory-ds-getting-started-password-sync-synced-tenant.md).

## Next steps

For more information on the specifics of password synchronization, see [How password hash synchronization works with Azure AD Connect](../active-directory/hybrid/how-to-connect-password-hash-synchronization.md?context=/azure/active-directory-domain-services/context/azure-ad-ds-context).

To get started with Azure AD DS, [create a managed domain](tutorial-create-instance.md).
