# JPN365Labs

Hands-on lab notes on Exchange Online, Outlook and Microsoft 365 identity.

!!! info
    All domains, IP addresses and names in these notes are fictional (domain.com, relay.domain.com, 203.0.113.0/24).

## Transport

| Article | Description |
|---|---|
| [Postfix relay as an add-on service](Transport/Postfix-Relay-Add-On-Service.md) | Build a Postfix relay in Azure and wire it into Exchange Online following the "email add-on service" pattern, including troubleshooting 4.4.317 and 5.7.606. |

## Exchange Online

| Article | Description |
|---|---|
| [Cross-Tenant Message Recall in Exchange Online](Exchange-Online/Cross-Tenant-Message-Recall.md) | How Message Recall now works across trusted Microsoft 365 tenants, and how to enable it and manage the allow-list with PowerShell. |
| [Get-GroupSendPermissions](Exchange-Online/Get-GroupSendPermissions.md) | List every DL, DDL and Microsoft 365 group on which a user has Send As or Send on Behalf permission. |

## Tools

| Article | Description |
|---|---|
| [Stellar Converter for EDB Review](Tools/Stellar-Converter-for-EDB-Review.md) | Hands-on review of an EDB-to-PST converter for offline and hosted Exchange databases. |

## Identity

| Article | Description |
|---|---|
| [Sign in to a Microsoft Account with a Passkey](Identity/Passkey-Sign-In-Microsoft-Account.md) | Create a passkey for a personal Microsoft account and sign in with face, fingerprint, PIN or security key instead of a password. |

## Outlook

| Article | Description |
|---|---|
| [Mark Outlook Email as Read with PowerShell](Outlook/Mark-Email-As-Read-PowerShell.md) | Mark every unread item in a folder as read, with classic Outlook (COM) on the PC or server-side with Microsoft Graph. |
| [Share Calendars and Contacts in Microsoft 365](Outlook/Share-Calendar-and-Contacts.md) | Team calendars with a shared mailbox, sharing your own calendar or contacts, PowerShell equivalents and the external sharing policy. |
