# Share Calendars and Contacts in Microsoft 365

How to share calendars and contacts in Exchange Online, either through **a shared mailbox for a team** or by **one person sharing their own folders**, with PowerShell equivalents and the external sharing policy.

## Pick the right approach

| You want... | Use | Who sets it up |
|---|---|---|
| A team calendar / contact list everyone can edit | **Shared mailbox** + Full Access | Admin |
| To let a colleague see or edit *your* calendar | **Calendar sharing** (folder permission) | User |
| To share *your* contacts folder | **Contacts sharing** (classic Outlook only) | User |
| To share with people in another company | Calendar/contacts sharing + **sharing policy** | User + Admin |

!!! note
    A shared mailbox also receives email. Anyone given access to its calendar can also read and reply to its mail.

## Option A: Team calendar and contacts with a shared mailbox

### 1. Create the shared mailbox

**Microsoft 365 admin center:** **Teams & groups > Shared mailboxes > Add a shared mailbox**, enter a name and address, then add members.

**PowerShell:**

```powershell
Connect-ExchangeOnline

New-Mailbox -Shared -Name "Team Calendar" -PrimarySmtpAddress teamcal@domain.com
```

### 2. Grant access

For a few people, add them directly. For a large or changing group, grant a **mail-enabled security group** so you only manage membership in one place.

```powershell
# Individual user (AutoMapping adds it to their classic Outlook automatically)
Add-MailboxPermission -Identity teamcal@domain.com -User alice@domain.com -AccessRights FullAccess -AutoMapping $true

# Security group (members get access; AutoMapping does not apply to groups)
Add-MailboxPermission -Identity teamcal@domain.com -User "SG-TeamCalendar" -AccessRights FullAccess
```

!!! tip
    Only need the calendar, not the mailbox? Skip Full Access and grant folder rights instead:
    `Add-MailboxFolderPermission -Identity teamcal@domain.com:\Calendar -User SG-TeamCalendar -AccessRights Editor`

### 3. Open it

| Client | Calendar | Contacts |
|---|---|---|
| Classic Outlook | Appears under **My Calendars** automatically (AutoMapping) | Appears under **My Contacts** |
| New Outlook | **Calendar > Add calendar > Add from directory** | Not supported |
| Outlook on the web | **Calendar > Add calendar > Add from directory** | Not supported |

For mail in new Outlook / OWA: right-click your account in the folder pane > **Add shared folder or mailbox**. It can take up to an hour after access is granted before it shows.

## Option B: Share your own calendar

**Outlook (classic, new or web):** open **Calendar**, select **Share** (or right-click the calendar > **Sharing and permissions**), add the person and choose a level, then **Send / Share**. The recipient clicks **Accept** / **Add calendar** in the invitation.

| Permission | What they see / do |
|---|---|
| Can view when I'm busy | Free/busy only |
| Can view titles and locations | Time, subject, location |
| Can view all details | Everything except private items |
| Can edit | Create, change and delete items |
| Delegate | Edit + receives meeting requests on your behalf |

**Change or remove access** in the same **Sharing and permissions** pane (or **Calendar Permissions** in classic Outlook).

**Admin equivalent:**

```powershell
# Grant / change
Add-MailboxFolderPermission -Identity user@domain.com:\Calendar -User colleague@domain.com -AccessRights Reviewer
Set-MailboxFolderPermission -Identity user@domain.com:\Calendar -User colleague@domain.com -AccessRights Editor

# Review / remove
Get-MailboxFolderPermission -Identity user@domain.com:\Calendar
Remove-MailboxFolderPermission -Identity user@domain.com:\Calendar -User colleague@domain.com
```

!!! warning
    Folder names are localised. If `:\Calendar` fails, check `Get-MailboxFolderStatistics user@domain.com -FolderScope Calendar`.

## Option C: Share your contacts folder

Only **classic Outlook for Windows** can send a contacts sharing invitation.

1. Go to **People**, select the contacts folder.
2. **Home > Share Contacts**, enter the recipient, and **Send**.
3. To change or remove access: right-click the folder > **Properties > Permissions**.

PowerShell works for any client: `Add-MailboxFolderPermission -Identity user@domain.com:\Contacts -User colleague@domain.com -AccessRights Reviewer`

## Sharing with people outside your organization

External calendar and contacts sharing is controlled by the **sharing policy**. If users get an error like *"This calendar can't be shared with ..."* or *"Folder sharing is not available"*, the policy doesn't allow that level of detail.

**Exchange admin center:** **Organization > Sharing > Individual sharing**, edit the **Default Sharing Policy**, and set the rule for `*` (all domains) or a specific partner domain.

**PowerShell:**

```powershell
Get-SharingPolicy | Format-List Name, Domains, Enabled, Default

# Allow full calendar details and contacts for all external domains
Set-SharingPolicy -Identity "Default Sharing Policy" -Domains "*:CalendarSharingFreeBusyReviewer, ContactsSharing"

# Safer: full details only for a partner domain, free/busy for everyone else
Set-SharingPolicy -Identity "Default Sharing Policy" -Domains "partner.com:CalendarSharingFreeBusyReviewer", "*:CalendarSharingFreeBusySimple"
```

| Value | External recipients can see |
|---|---|
| `CalendarSharingFreeBusySimple` | Free/busy only |
| `CalendarSharingFreeBusyDetail` | Free/busy, subject and location |
| `CalendarSharingFreeBusyReviewer` | All details |
| `ContactsSharing` | Contacts folder |

## Troubleshooting

| Symptom | Check |
|---|---|
| Shared mailbox doesn't appear in classic Outlook | Wait up to an hour; AutoMapping only works for direct user grants, not groups. Add it manually via **Account Settings > More Settings > Advanced**. |
| "You don't have permission" on a shared calendar | `Get-MailboxFolderPermission mailbox:\Calendar` and confirm the user or group is listed. |
| External invite fails | Sharing policy for that domain is missing or too restrictive (see above). |
| Contacts option missing | Contacts sharing needs classic Outlook for Windows. |

## References

- [Share calendar and contacts in Microsoft 365 - Microsoft Learn](https://learn.microsoft.com/microsoft-365-apps/outlook/calendaring/share-calendar-and-contacts)
- [Add-MailboxFolderPermission](https://learn.microsoft.com/powershell/module/exchange/add-mailboxfolderpermission)
- [Set-SharingPolicy](https://learn.microsoft.com/powershell/module/exchange/set-sharingpolicy)
