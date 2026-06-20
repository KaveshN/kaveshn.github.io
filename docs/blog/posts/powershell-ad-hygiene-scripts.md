---
date: 2026-08-22
authors:
  - kavesh
categories:
  - Identity
  - PowerShell
tags:
  - Active Directory
  - Automation
  - Reporting
  - Security
slug: powershell-ad-hygiene-scripts
description: >-
  Active Directory environments accumulate cruft. Ten PowerShell scripts every
  admin should have on hand — dormant accounts, password audits, privileged
  group reviews, replication health checks, and more. Run them regularly and
  push the entropy back.
---

# PowerShell for AD hygiene: 10 scripts every admin should have in their back pocket

Active Directory environments accumulate cruft. Accounts of people who left
two years ago. Service accounts whose owners forgot they existed. Group
memberships granted for a project that ended in 2019. Computer objects for
laptops that were stolen, decommissioned, or simply forgotten. Without active
hygiene, your directory becomes a fossil record of every decision anyone made
for the last decade.

The fix is not heroic. It's PowerShell, run regularly, with a handful of
scripts that surface the rot so you can deal with it.

<!-- more -->

## Setup

All of these assume you have the Active Directory PowerShell module available.
On Windows Server, install the AD DS Tools or RSAT. On Windows 10/11, install
the RSAT optional feature. PowerShell 7 works fully with the AD module — older
guidance that "you must use Windows PowerShell 5.1 for AD" is no longer
correct.

```powershell
Import-Module ActiveDirectory
```

Some scripts use the Microsoft Graph module for Entra ID work:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
```

## Script 1: find dormant user accounts

Accounts that haven't been used in a long time should be disabled or deleted.

```powershell
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
    -Properties LastLogonDate, Department, Manager |
    Select-Object Name, SamAccountName, LastLogonDate, Department |
    Sort-Object LastLogonDate |
    Export-Csv -Path "C:\Reports\Dormant-Users.csv" -NoTypeInformation
```

!!! note "On LastLogonDate accuracy"
    This attribute is replicated and updated periodically — reasonably accurate
    for hygiene purposes. For precision, query `lastLogon` against each
    individual DC and take the most recent. Slower, but exact.

## Script 2: find dormant computer accounts

Devices that haven't authenticated to the domain in months are usually
decommissioned or lost.

```powershell
$cutoff = (Get-Date).AddDays(-180)
Get-ADComputer -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
    -Properties LastLogonDate, OperatingSystem |
    Select-Object Name, OperatingSystem, LastLogonDate |
    Sort-Object LastLogonDate |
    Export-Csv -Path "C:\Reports\Dormant-Computers.csv" -NoTypeInformation
```

Six months is a reasonable starting threshold for computers. Tune to your
environment — devices for travelling staff might legitimately be quiet longer.

## Script 3: accounts with Password Never Expires

These are almost always service accounts, but every once in a while you find a
user account with this set "because it was easier."

```powershell
Get-ADUser -Filter {PasswordNeverExpires -eq $true -and Enabled -eq $true} `
    -Properties PasswordNeverExpires, PasswordLastSet, Description |
    Select-Object Name, SamAccountName, PasswordLastSet, Description |
    Sort-Object PasswordLastSet |
    Export-Csv -Path "C:\Reports\PasswordNeverExpires.csv" -NoTypeInformation
```

For each service account in this list, ask whether it could be migrated to a
Group Managed Service Account (gMSA), which handles password rotation
automatically.

## Script 4: privileged group membership audit

A regular review of who's in the most powerful groups is foundational security
hygiene.

```powershell
$privGroups = @(
    'Domain Admins',
    'Enterprise Admins',
    'Schema Admins',
    'Account Operators',
    'Backup Operators',
    'Server Operators',
    'Print Operators'
)

foreach ($group in $privGroups) {
    $members = Get-ADGroupMember -Identity $group -Recursive -ErrorAction SilentlyContinue
    foreach ($m in $members) {
        [PSCustomObject]@{
            Group   = $group
            Member  = $m.Name
            SamName = $m.SamAccountName
            ObjType = $m.objectClass
        }
    }
} | Export-Csv -Path "C:\Reports\PrivilegedGroups.csv" -NoTypeInformation
```

Run this monthly. The Domain Admins membership in particular should be small
enough to fit on a sticky note.

## Script 5: recent account lockouts

Account lockouts can be ordinary — a user mistyping their password — or they
can be a sign of an attack. Knowing about them in something close to real time
is useful.

```powershell
$pdc = (Get-ADDomain).PDCEmulator
Get-WinEvent -ComputerName $pdc -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4740
    StartTime = (Get-Date).AddHours(-24)
} | ForEach-Object {
    [PSCustomObject]@{
        Time    = $_.TimeCreated
        Account = $_.Properties[0].Value
        Caller  = $_.Properties[1].Value
    }
} | Format-Table -AutoSize
```

The PDC emulator is the authoritative source for lockout events (Event 4740).
If the same account is locking out repeatedly from different sources, that's
worth investigating.

## Script 6: find accounts with adminCount set

Accounts placed in protected groups get `adminCount` set to 1. This attribute
persists even after the user is removed from the group, and it disables
permission inheritance on the account.

```powershell
Get-ADUser -Filter {adminCount -eq 1} -Properties adminCount, MemberOf |
    Select-Object Name, SamAccountName, @{
        Name       = 'MemberOf'
        Expression = {
            ($_.MemberOf | ForEach-Object {
                ($_ -split ',')[0] -replace 'CN='
            }) -join ', '
        }
    } |
    Export-Csv -Path "C:\Reports\AdminCount.csv" -NoTypeInformation
```

Cross-reference with your privileged group memberships. Any account with
`adminCount=1` that's not currently in a privileged group is a candidate for
cleanup — the attribute should be removed and inheritance re-enabled.

## Script 7: password age report

Knowing how old user passwords are tells you whether your password policy is
actually being honoured.

```powershell
$users = Get-ADUser -Filter {Enabled -eq $true} -Properties PasswordLastSet
$users | ForEach-Object {
    $age = if ($_.PasswordLastSet) {
        ((Get-Date) - $_.PasswordLastSet).Days
    } else {
        'Never set'
    }
    [PSCustomObject]@{
        Name           = $_.Name
        SamAccountName = $_.SamAccountName
        PasswordAgeDays = $age
    }
} | Sort-Object PasswordAgeDays -Descending |
    Export-Csv -Path "C:\Reports\PasswordAge.csv" -NoTypeInformation
```

If your max password age is 90 days, anyone older than that has either a
"password never expires" account or has somehow escaped the policy.

## Script 8: replication health check

A quick scan of replication status across all domain controllers. If anything
is failing, you want to know before users start noticing.

```powershell
$dcs = (Get-ADDomainController -Filter *).HostName
foreach ($dc in $dcs) {
    $result = repadmin /showrepl $dc /csv | ConvertFrom-Csv |
        Where-Object { $_.'Number of Failures' -gt 0 }
    if ($result) {
        Write-Host "Replication issues on $dc" -ForegroundColor Red
        $result | Format-Table -AutoSize
    } else {
        Write-Host "$dc — OK" -ForegroundColor Green
    }
}
```

If you have a SIEM, replication failures should be alerts in there. If you
don't, running this daily and reviewing the output is reasonable.

## Script 9: disabled accounts still in groups

A common issue: an account gets disabled when someone leaves, but their group
memberships aren't cleaned up.

```powershell
Get-ADUser -Filter {Enabled -eq $false} -Properties MemberOf |
    Where-Object { $_.MemberOf.Count -gt 0 } |
    ForEach-Object {
        [PSCustomObject]@{
            Account = $_.SamAccountName
            Name    = $_.Name
            Groups  = $_.MemberOf.Count
        }
    } | Sort-Object Groups -Descending |
    Export-Csv -Path "C:\Reports\DisabledWithGroups.csv" -NoTypeInformation
```

Cleanup pattern: for each account that's been disabled for more than 90 days,
strip its group memberships, document what was removed, then either delete the
account or leave it in a holding OU for archival.

## Script 10: stale group membership review

Groups whose membership hasn't changed in years — probably fine, but worth
reviewing that the membership still reflects current reality.

```powershell
$cutoff = (Get-Date).AddYears(-2)
Get-ADGroup -Filter * -Properties Modified |
    Where-Object { $_.Modified -lt $cutoff } |
    Select-Object Name, GroupCategory, GroupScope, Modified |
    Sort-Object Modified |
    Export-Csv -Path "C:\Reports\StaleGroups.csv" -NoTypeInformation
```

Stale isn't bad by itself — many groups have stable membership for good
reasons. But it's a useful surfacing for review. A "Project X Contributors"
group that hasn't been modified in three years probably belongs to a project
nobody works on anymore.

## Running these regularly

The trick to AD hygiene is doing the work continuously, not once.

Schedule the scripts weekly or monthly via a scheduled task on a management
server. Email the output to whoever cares about directory health. Build the
review into a regular operational rhythm — a 30-minute monthly meeting where
someone walks through what's changed and what action is needed.

!!! tip "When in doubt, disable rather than delete"
    Disabled accounts are easy to re-enable if you got it wrong. Deleted
    accounts require restoration from the AD Recycle Bin (and you *do* have
    that enabled, right?) or a more painful restore from backup.

Each of these scripts can be expanded with more attributes, conditional logic,
or different output formats. Treat them as starting points. The point isn't
the specific PowerShell — it's the discipline of regularly looking at your
directory and pushing it back toward a known-good state.

The directory will not maintain itself. But thirty minutes with PowerShell,
every week, is genuinely enough to keep most of the entropy at bay.
