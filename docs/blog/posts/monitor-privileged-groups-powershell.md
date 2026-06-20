---
date: 2026-06-10
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
tags:
  - Active Directory
  - Security
  - Monitoring
  - Email Alerts
  - Privileged Access
slug: monitor-privileged-groups-powershell
description: >-
  Detect changes to Domain Admins and Enterprise Admins in near real time with a
  PowerShell state-file diff — and get an email the moment membership changes.
---

# Get an email the moment someone joins Domain Admins

A change to **Domain Admins** or **Enterprise Admins** is one of the highest-signal
events in your whole estate. Legitimately, it happens a few times a year. If it
happens and you *didn't* schedule it, you want to know in minutes, not at the next
quarterly access review.

You don't need a SIEM for the first 80% of this. A scheduled PowerShell script
that remembers last run's membership and emails you the delta is a genuinely
useful control — the **state-file diff** pattern.

<!-- more -->

## The pattern: compare now against a saved baseline

The whole thing is three moves:

1. Read current membership.
2. Compare it to a CSV you wrote last time.
3. If anything changed, email the difference and overwrite the baseline.

`Compare-Object` does the heavy lifting. Its `SideIndicator` tells you which way
each difference points.

```powershell
Import-Module ActiveDirectory

$groups    = 'Domain Admins', 'Enterprise Admins'
$stateDir  = 'C:\ProgramData\GroupMonitor'   # not the CWD — see the gotcha
$mailFrom  = 'ad-alerts@example.com'
$mailTo    = 'secops@example.com'
$smtp      = 'smtp.example.com'

New-Item -ItemType Directory -Path $stateDir -Force | Out-Null

foreach ($group in $groups) {
    # -Recursive flattens nested groups — privilege is transitive
    $current = Get-ADGroupMember -Identity $group -Recursive |
        Select-Object Name, SamAccountName, distinguishedName

    $stateFile = Join-Path $stateDir "$group.csv"

    if (-not (Test-Path $stateFile)) {
        # First run: establish the baseline, alert on nothing
        $current | Export-Csv $stateFile -NoTypeInformation
        Write-Host "Baseline written for '$group' — no alert on first run."
        continue
    }

    $previous = Import-Csv $stateFile

    $changes = Compare-Object $previous $current -Property SamAccountName |
        Select-Object SamAccountName,
            @{ Name = 'State'; Expression = {
                if ($_.SideIndicator -eq '=>') { 'ADDED' } else { 'REMOVED' }
            }}

    if ($changes) {
        $body = $changes | Format-Table -AutoSize | Out-String
        Send-MailMessage -To $mailTo -From $mailFrom -SmtpServer $smtp `
            -Subject "ALERT: '$group' membership changed" -Body $body
    }

    # Overwrite the baseline so next run compares against now
    $current | Export-Csv $stateFile -NoTypeInformation
}
```

!!! tip "`SideIndicator`: which set is the value in?"
    `Compare-Object reference difference` marks each row `<=` (only in the
    *reference* / old set → it was **removed**) or `=>` (only in the *difference* /
    new set → it was **added**). Getting this backwards is the classic bug — the
    calculated property above pins it down so you never have to remember.

## Why `-Recursive` matters

`Get-ADGroupMember -Recursive` flattens nested groups. Someone can be a Domain
Admin because they're in a group that's in a group that's in Domain Admins.
Without `-Recursive` you only see the direct members and miss exactly the kind of
indirect privilege an attacker would use to hide.

!!! note "This replaces the old Quest snap-in"
    Older versions of this script used `Get-QADGroupMember` from the Quest
    ActiveRoles snap-in. That dependency is long dead — `Get-ADGroupMember` from
    the built-in `ActiveDirectory` RSAT module does the same job with nothing to
    install.

## Run it often, and mind these traps

Schedule it every 5–15 minutes. A daily run technically works, but a privilege
escalation that's added and removed inside a day slips through.

!!! warning "Three gotchas that bite everyone"
    - **Don't write the state file to the current directory.** If the scheduled
      task's working directory ever changes, the script can't find its baseline,
      treats it as a first run, and silently re-baselines — swallowing the very
      change you wanted to catch. Use an absolute path like `C:\ProgramData\…`.
    - **First run never alerts.** That's deliberate (there's nothing to compare
      against), but it means an attacker who can delete your state files resets
      the baseline. Protect the state directory's ACL.
    - **`Compare-Object` on rich objects compares references, not values.** Always
      pass `-Property` with the fields you care about, or two logically-identical
      members compare as different.

Pair this with a daily [DC health report](dc-health-report-powershell.md) and
you've got the cheap, high-value half of directory monitoring covered before any
SIEM licensing conversation starts.
