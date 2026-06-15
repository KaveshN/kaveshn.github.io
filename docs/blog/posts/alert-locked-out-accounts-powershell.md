---
date: 2026-06-11
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: alert-locked-out-accounts-powershell
description: >-
  Find currently locked-out AD accounts with Search-ADAccount and email a clean
  report — only when there's actually something to report.
---

# Catching account lockouts before the helpdesk ticket

Locked accounts are noise until they're a pattern. One user fat-fingered their
password — fine. Twenty accounts locking out in ten minutes is either a bad
service-account credential after a password change, or someone spraying
passwords against your directory. Either way you want to see it on a dashboard,
not discover it from a pile of helpdesk tickets.

The data is one cmdlet away. The craft is in reporting it *well*.

<!-- more -->

## The one-liner that does the work

```powershell
Search-ADAccount -UsersOnly -LockedOut
```

That's the entire data-collection step. `Search-ADAccount` has purpose-built
switches — `-LockedOut`, `-AccountDisabled`, `-PasswordExpired`,
`-AccountInactive` — so you never have to hand-roll an LDAP filter against
`userAccountControl` bit flags.

## Turn it into something worth emailing

The mistake in most versions of this script is sending a report every run, even
when nothing's locked, and dumping raw objects into the body. Capture the
results, enrich them, and **only email when the count is non-zero**:

```powershell
Import-Module ActiveDirectory

$locked = Search-ADAccount -UsersOnly -LockedOut |
    Get-ADUser -Properties LockoutTime, LastBadPasswordAttempt, EmailAddress |
    Select-Object Name, SamAccountName, EmailAddress,
        @{ Name = 'LockedAt'; Expression = {
            [DateTime]::FromFileTime($_.LockoutTime)
        }},
        @{ Name = 'LastBadAttempt'; Expression = { $_.LastBadPasswordAttempt }} |
    Sort-Object LockedAt -Descending

if (-not $locked) {
    Write-Host "No locked accounts. Nothing to send."
    return
}
```

!!! warning "`LockoutTime` is a FILETIME, not a date"
    `lockoutTime` comes back as a 64-bit integer (100-nanosecond ticks since
    1601). Print it raw and you get a meaningless number like `133600000000000000`.
    `[DateTime]::FromFileTime()` converts it to a real timestamp. Same trap applies
    to `lastLogonTimestamp` and `pwdLastSet` — any AD date-ish attribute ending in
    "Time" is probably a FILETIME.

## Build an HTML report with `ConvertTo-Html`

You don't have to hand-concatenate `<td>` tags. `ConvertTo-Html` turns objects
straight into a table — split the style and the data so you control both:

```powershell
$style = @'
<style>
  body  { font-family: Segoe UI, Arial; font-size: 10pt; }
  table { border-collapse: collapse; }
  th, td { border: 1px solid #ccc; padding: 6px; text-align: left; }
  th    { background: #2d2d6b; color: #fff; }
</style>
'@

$html = $locked | ConvertTo-Html -Head $style `
    -PreContent "<h2>Locked accounts — $(Get-Date -Format 'yyyy-MM-dd HH:mm')</h2>" `
    -PostContent "<p>$($locked.Count) account(s) currently locked.</p>" |
    Out-String

Send-MailMessage -To 'helpdesk@example.com' -From 'ad-alerts@example.com' `
    -SmtpServer 'smtp.example.com' -BodyAsHtml -Body $html `
    -Subject "AD: $($locked.Count) locked account(s)"
```

!!! tip "`-PreContent` and `-PostContent` are the easy win"
    They inject HTML before and after the auto-generated table — perfect for a
    heading and a summary line. You get a clean report without ever touching a
    `<tr>` by hand.

## Where to go next

A point-in-time list answers "who's locked *right now*". The follow-up question
is "*why*" — and that lives in the **Security event log** on your DCs (event
**4740** records each lockout with the source computer). A natural next step is
to query event 4740 from your PDC emulator to find the machine generating the bad
passwords:

```powershell
Get-WinEvent -ComputerName (Get-ADDomain).PDCEmulator -FilterHashtable @{
    LogName = 'Security'; Id = 4740
} -MaxEvents 50 |
    Select-Object TimeCreated,
        @{ Name = 'LockedUser';   Expression = { $_.Properties[0].Value }},
        @{ Name = 'SourceComputer'; Expression = { $_.Properties[1].Value }}
```

That `SourceComputer` field is usually what cracks the case — it's the machine
with the stale cached credential or the spray tool.
