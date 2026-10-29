# Mark Outlook Email as Read with PowerShell

Mark every unread message in a folder as read: on the user's PC with **classic Outlook**, or server-side with **Microsoft Graph PowerShell**.

## Which method to use

| | Option 1: Outlook (COM) | Option 2: Microsoft Graph |
|---|---|---|
| Runs where | User's Windows PC with classic Outlook | Any machine with PowerShell 7 or 5.1 |
| Works with new Outlook | No (no COM support) | Yes, it changes the mailbox itself |
| Needs admin consent | No | Yes, `Mail.ReadWrite` |
| Good for | One user, one-off cleanup | Admins, other mailboxes, automation |

## Option 1: Classic Outlook (COM)

Run in **Windows PowerShell** on the PC where the Outlook profile is configured. Outlook may be open or closed.

```powershell
<#
.SYNOPSIS
    Marks all unread items in an Outlook folder as read.
.EXAMPLE
    .\Set-OutlookFolderRead.ps1 -Mailbox user@domain.com -Folder "Deleted Items"
.EXAMPLE
    .\Set-OutlookFolderRead.ps1 -Mailbox user@domain.com -Folder "Inbox\Projects" -Recurse -WhatIf
#>
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)] [string]$Mailbox,   # display name of the store, usually the SMTP address
    [string]$Folder = "Inbox",                 # use \ for subfolders, e.g. "Inbox\Projects"
    [switch]$Recurse
)

$outlook   = New-Object -ComObject Outlook.Application
$namespace = $outlook.GetNamespace("MAPI")

# Resolve the folder path
$target = $namespace.Folders.Item($Mailbox)
foreach ($part in $Folder.Split('\')) { $target = $target.Folders.Item($part) }

function Set-FolderRead($f) {
    # Restrict returns only unread items; copy to an array so the
    # collection does not shrink while we change it
    $unread = @($f.Items.Restrict("[UnRead] = True"))
    if ($unread.Count -and $PSCmdlet.ShouldProcess($f.FolderPath, "Mark $($unread.Count) item(s) as read")) {
        foreach ($item in $unread) { $item.UnRead = $false; $item.Save() }
    }
    Write-Host ("{0,-50} {1,6} marked read" -f $f.FolderPath, $unread.Count)
    if ($Recurse) { foreach ($sub in $f.Folders) { Set-FolderRead $sub } }
}

Set-FolderRead $target
```

!!! tip
    `-WhatIf` shows how many items would change without touching anything. The original version also loaded `Microsoft.Office.Interop.Outlook`; that line isn't needed with `New-Object -ComObject`.

## Option 2: Microsoft Graph PowerShell (server-side)

Works for any mailbox you have rights to, and for users on new Outlook, OWA or mobile.

```powershell
Install-Module Microsoft.Graph.Mail -Scope CurrentUser   # once
Connect-MgGraph -Scopes "Mail.ReadWrite"                 # add Mail.ReadWrite.Shared for shared mailboxes

$user   = "user@domain.com"
$folder = "deleteditems"    # well-known name (inbox, deleteditems, junkemail, ...) or a folder ID

$unread = Get-MgUserMailFolderMessage -UserId $user -MailFolderId $folder `
            -Filter "isRead eq false" -Property Id -All

foreach ($m in $unread) {
    Update-MgUserMessage -UserId $user -MessageId $m.Id -BodyParameter @{ isRead = $true }
}
"{0} message(s) marked as read" -f $unread.Count
```

!!! note
    To run this unattended for many mailboxes, use an app registration with the **Mail.ReadWrite** application permission and limit it with an [application access policy / RBAC for Applications](https://learn.microsoft.com/exchange/permissions-exo/application-rbac) so it can't touch every mailbox.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Retrieving the COM class factory ... failed` | Classic Outlook isn't installed, or you're on new Outlook. Use Option 2. |
| `The attempted operation failed. An object could not be found.` | Wrong `-Mailbox` or `-Folder` name. Mailbox is the name shown at the top of the folder pane. |
| Some items stay unread (Option 1) | Iterating `Items` directly while changing it skips items. The script copies the restricted set first. |
| Outlook shows a security prompt | Antivirus isn't recognised as up to date, so Outlook's object-model guard kicks in. |
| `403 Forbidden` / `ErrorAccessDenied` (Option 2) | Missing `Mail.ReadWrite` consent, or no access to that mailbox. |
| Read state comes back on another device | Wait for sync. Option 2 changes the server copy directly, so it wins. |

## References

- [Items.Restrict method (Outlook)](https://learn.microsoft.com/office/vba/api/outlook.items.restrict)
- [Update-MgUserMessage](https://learn.microsoft.com/powershell/module/microsoft.graph.mail/update-mgusermessage)
- [List messages - Microsoft Graph](https://learn.microsoft.com/graph/api/user-list-messages)
