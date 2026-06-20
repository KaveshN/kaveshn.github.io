---
date: 2026-06-02
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
tags:
  - Entra ID
  - Microsoft Graph
  - Auditing
  - Reporting
slug: audit-entra-users-powershell
description: >-
  A practical Microsoft Graph PowerShell recipe for auditing Entra ID users —
  guests, stale accounts, and disabled identities — exported to CSV.
---

# Auditing Entra ID users with PowerShell

"Who has access?" is the question every security review opens with. With the
Microsoft Graph PowerShell SDK you can answer it in a few lines: list every
user, flag the guests and the dormant accounts, and hand security a CSV.

This builds directly on [connecting to Graph](connect-microsoft-graph-powershell.md) —
make sure you can run `Get-MgContext` first.

<!-- more -->

## Connect with read scopes

Auditing is read-only. Request exactly that — plus `AuditLog.Read.All`, which is
what unlocks the all-important last-sign-in data:

```powershell
Connect-MgGraph -Scopes "User.Read.All", "AuditLog.Read.All"
```

## List users — and only the properties you need

`Get-MgUser` returns a slim default set. Ask explicitly for the fields you'll
report on, or they come back empty:

```powershell
$users = Get-MgUser -All -Property `
  DisplayName, UserPrincipalName, AccountEnabled, UserType,
  CreatedDateTime, SignInActivity

$users |
  Select-Object DisplayName, UserPrincipalName, UserType, AccountEnabled |
  Format-Table -AutoSize
```

!!! warning "`SignInActivity` needs the property AND the scope"
    `signInActivity` is only populated when you (1) request it via `-Property`
    **and** (2) connected with `AuditLog.Read.All`. Omit either and the field is
    silently null — which looks exactly like "never signed in". Don't confuse
    the two.

## Find the guests

External identities are the usual soft spot. Advanced queries (`-ConsistencyLevel
eventual` plus a count) let you filter server-side:

```powershell
$guests = Get-MgUser -All `
  -Filter "userType eq 'Guest'" `
  -ConsistencyLevel eventual `
  -CountVariable guestCount `
  -Property DisplayName, UserPrincipalName, CreatedDateTime

Write-Host "$guestCount guest accounts found"
```

## Flag stale accounts (90+ days dormant)

Combine "enabled" with "hasn't signed in lately". `SignInActivity` can be null
for accounts that have genuinely never signed in — treat that as stale too:

```powershell
$cutoff = (Get-Date).AddDays(-90)

$stale = $users | Where-Object {
    $_.AccountEnabled -and (
        -not $_.SignInActivity.LastSignInDateTime -or
        $_.SignInActivity.LastSignInDateTime -lt $cutoff
    )
}

Write-Host "$($stale.Count) enabled accounts dormant for 90+ days"
```

## Export for the security review

A timestamped CSV is what auditors actually want to receive:

```powershell
$report = $users | Select-Object `
  DisplayName,
  UserPrincipalName,
  UserType,
  AccountEnabled,
  CreatedDateTime,
  @{ Name = "LastSignIn"; Expression = { $_.SignInActivity.LastSignInDateTime } }

$path = "EntraUserAudit_{0:yyyy-MM-dd}.csv" -f (Get-Date)
$report | Sort-Object LastSignIn | Export-Csv -Path $path -NoTypeInformation -Encoding UTF8

Write-Host "Wrote $($report.Count) rows to $path"
```

!!! tip "Calculated properties flatten nested objects"
    `signInActivity` is an object, so it won't land cleanly in a CSV column. The
    `@{ Name = ...; Expression = ... }` hashtable (a *calculated property*) pulls
    the nested value out into a flat column — the standard PowerShell idiom for
    this.

Next in the series: going from read-only auditing to actually granting access —
assigning Entra ID **access packages** with PowerShell.
