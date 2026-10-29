---
title: Cross-Tenant Message Recall in Exchange Online
---

# Cross-Tenant Message Recall in Exchange Online

!!! info "Originally published on Substack"
    This article was first published on 11 July 2026 at [jpnspace.substack.com](https://jpnspace.substack.com/p/in-exchange-online).

!!! note "TL;DR"
    Microsoft is expanding Exchange Online Message Recall beyond organizational boundaries with **Cross-Tenant Message Recall**. Organizations can choose to allow trusted external Microsoft 365 tenants to recall messages sent to their users. The feature is disabled by default, managed through Exchange Online PowerShell, and requires explicit trust between tenants. This enhancement provides an additional safeguard against accidental email sends to partner and subsidiary organizations, while giving administrators full control over which external tenants are allowed. Rollout starts in August 2026.

Ever sent an email to an external recipient and wished you could take it back?
Microsoft is expanding Message Recall beyond organizational boundaries with **Cross-Tenant Message Recall** in Exchange Online.

Until now, Message Recall only worked within your own tenant. With this enhancement, recall requests can also succeed across organizations—provided both tenants have opted in and trust each other.

The feature rollout begins mid-August 2026 and is expected to complete by early September 2026.

The feature is disabled by default; admins can enable it and manage an allow-list of external tenant IDs using Exchange Online PowerShell.
Recall attempts from tenants not on the allow-list will fail, and each tenant controls which external organizations are allowed.

> A short video walkthrough accompanies this post on [Substack](https://jpnspace.substack.com/p/in-exchange-online).

This feature is associated with Microsoft 365 Message center post **MC1423106** & [Roadmap ID **561330**](https://www.microsoft.com/en-us/microsoft-365/roadmap?filters=&searchterms=561330).

## Who is affected

- Exchange Online administrators
- Organizations collaborating with external Microsoft 365 tenants

## Platforms/Services

- Exchange Online
- Outlook (web, desktop, mobile)

## What will happen

- The feature is **disabled by default.**
- Admins can enable cross-tenant recall using Exchange Online PowerShell.
- Admins can control an **allow-list of external tenant IDs**.
- Once enabled:
    - External senders from allow-listed tenants can recall messages sent to your users.
    - Recipients experience recall the same as intra-tenant recall.
- If recipient recall notifications are enabled, they apply to cross-tenant recalls.
- Recall attempts from tenants **not on the allow-list will fail.**
- Each tenant independently controls which external organizations are allowed.

## Action Required/Recommendations

- No action is required to keep the default behavior (disabled).

If you plan to enable this feature:

- Review with messaging and security teams.
- Enable or disable the feature using Exchange Online PowerShell:

```powershell
Set-CrossTenantRecallConfiguration -CrossTenantRecallEnabled $true
Set-CrossTenantRecallConfiguration -CrossTenantRecallEnabled $false
```

- Manage the allowed tenant list (add or remove):

```powershell
Set-CrossTenantRecallConfiguration -AllowedSenderTenantIds @{Add="tenantId1","tenantId2"}
Set-CrossTenantRecallConfiguration -AllowedSenderTenantIds @{Remove="tenantId1","tenantId2"}
```

- Update internal documentation and notify helpdesk.

As someone who supports Exchange Online environments, this is a feature many customers have been waiting for. This new capability can help reduce the impact of accidental emails sent to trusted external organizations—provided it's implemented with the right governance and controls.

**More information:** [Cross-Tenant Message Recall in Exchange Online | Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/cross-tenant-message-recall-in-exchange-online/4535800)

---

**Jean Paul Nkundabose**
Microsoft 365 Support Specialist, Modern Workplace
[LinkedIn](https://www.linkedin.com/in/jeanpaulnkundabose/)

*Would your organization enable cross-tenant message recall for partners and subsidiaries, or does the security risk outweigh the benefit?*

`#Microsoft365` `#ExchangeOnline` `#EmailSecurity` `#M365Admin` `#ExchangeAdmin` `#Microsoft365Roadmap` `#ITPro` `#Messaging`
