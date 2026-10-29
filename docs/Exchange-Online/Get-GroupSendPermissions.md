# Get-GroupSendPermissions

Lists every Distribution List (DL), Dynamic Distribution List (DDL) and Microsoft 365 group on which a given user holds **Send As** or **Send on Behalf** permission in Exchange Online.

!!! note "Originally published on GitHub"
    This script was first shared in [PowerShell-script-for-listing-group-permissions](https://github.com/jeanpaulnk/PowerShell-script-for-listing-group-permissions). That version has been reworked here into a single script.

## Requirements

- The [ExchangeOnlineManagement](https://learn.microsoft.com/powershell/exchange/exchange-online-powershell-v2) module.
- An account with rights to read recipients and recipient permissions (for example **View-Only Organization Management** or **Recipient Management**).

## Where the permissions live

| Permission | Group type | Where it is stored | How to read it |
|---|---|---|---|
| Send As | DL, DDL, M365 group | Recipient ACL | `Get-RecipientPermission -Trustee <user>` |
| Send on Behalf | DL, DDL | `GrantSendOnBehalfTo` | `Get-DistributionGroup` / `Get-DynamicDistributionGroup` |
| Send on Behalf | M365 group | `GrantSendOnBehalfTo` | `Get-UnifiedGroup` |

!!! warning
    Send on Behalf for groups is **not** in a mailbox's `DelegateList`. Checking mailboxes only finds mailbox delegates, not group permissions.

## Usage

```powershell
Connect-ExchangeOnline

.\Get-GroupSendPermissions.ps1 -User user@domain.com

# Export to CSV
.\Get-GroupSendPermissions.ps1 -User user@domain.com |
    Export-Csv .\GroupSendPermissions.csv -NoTypeInformation
```

## Parameters

| Parameter | Description |
|---|---|
| `-User` | UPN, primary SMTP address or alias of the user (trustee) to check. Required. |

## Output

| Column | Description |
|---|---|
| `Group` | Display name of the group. |
| `PrimarySmtpAddress` | Group's primary address. |
| `GroupType` | `DL`, `DDL` or `M365Group`. |
| `Permission` | `SendAs` or `SendOnBehalf`. |

## Script

```powershell
<#
.SYNOPSIS
    Lists DLs, DDLs and Microsoft 365 groups on which a user has
    Send As or Send on Behalf permission in Exchange Online.
.EXAMPLE
    .\Get-GroupSendPermissions.ps1 -User user@domain.com
#>
[CmdletBinding()]
param(
    [Parameter(Mandatory = $true)]
    [string]$User
)

if (-not (Get-Command Get-RecipientPermission -ErrorAction SilentlyContinue)) {
    throw "Not connected. Run Connect-ExchangeOnline first."
}

$trustee = Get-Recipient -Identity $User -ErrorAction Stop
# Values GrantSendOnBehalfTo may hold for this user
$ids = @($trustee.Name, $trustee.DisplayName, $trustee.Alias,
         $trustee.DistinguishedName, $trustee.Identity.ToString(),
         $trustee.PrimarySmtpAddress.ToString()) | Where-Object { $_ } | Select-Object -Unique

$typeMap = @{
    MailUniversalDistributionGroup = 'DL'
    MailUniversalSecurityGroup     = 'DL'
    MailNonUniversalGroup          = 'DL'
    RoomList                       = 'DL'
    DynamicDistributionGroup       = 'DDL'
    GroupMailbox                   = 'M365Group'
}

$results = [System.Collections.Generic.List[object]]::new()

# --- Send As (one call covers DLs, DDLs and M365 groups) ---
Write-Verbose "Checking Send As..."
$sendAs = Get-RecipientPermission -Trustee $trustee.Identity -AccessRights SendAs -ResultSize Unlimited
foreach ($p in $sendAs) {
    $r = Get-Recipient -Identity $p.Identity -ErrorAction SilentlyContinue | Select-Object -First 1
    if ($r -and $typeMap.ContainsKey([string]$r.RecipientTypeDetails)) {
        $results.Add([PSCustomObject]@{
            Group              = $r.DisplayName
            PrimarySmtpAddress = [string]$r.PrimarySmtpAddress
            GroupType          = $typeMap[[string]$r.RecipientTypeDetails]
            Permission         = 'SendAs'
        })
    }
}

# --- Send on Behalf (GrantSendOnBehalfTo) ---
function Test-Delegate($list) {
    foreach ($d in $list) { if ($ids -contains $d.ToString()) { return $true } }
    return $false
}

Write-Verbose "Checking Send on Behalf..."
$sources = @(
    @{ Type = 'DL';        Items = { Get-DistributionGroup -ResultSize Unlimited } }
    @{ Type = 'DDL';       Items = { Get-DynamicDistributionGroup -ResultSize Unlimited } }
    @{ Type = 'M365Group'; Items = { Get-UnifiedGroup -ResultSize Unlimited } }
)
foreach ($s in $sources) {
    foreach ($g in (& $s.Items)) {
        if ($g.GrantSendOnBehalfTo -and (Test-Delegate $g.GrantSendOnBehalfTo)) {
            $results.Add([PSCustomObject]@{
                Group              = $g.DisplayName
                PrimarySmtpAddress = [string]$g.PrimarySmtpAddress
                GroupType          = $s.Type
                Permission         = 'SendOnBehalf'
            })
        }
    }
}

if ($results.Count -eq 0) {
    Write-Host "No Send As or Send on Behalf permissions found on groups for $User."
}
$results | Sort-Object GroupType, Group
```

## Quick one-liners

If you only need a fast check, these cmdlets give the same answer per group type:

```powershell
# Send As
Get-RecipientPermission -Trustee user@domain.com -AccessRights SendAs -ResultSize Unlimited

# Send on Behalf
Get-DistributionGroup -ResultSize Unlimited        | Where-Object { $_.GrantSendOnBehalfTo -match "User Name" }
Get-DynamicDistributionGroup -ResultSize Unlimited | Where-Object { $_.GrantSendOnBehalfTo -match "User Name" }
Get-UnifiedGroup -ResultSize Unlimited             | Where-Object { $_.GrantSendOnBehalfTo -match "User Name" }
```

!!! tip
    Test in a lab tenant first. Large tenants can take several minutes because every group is read once.
