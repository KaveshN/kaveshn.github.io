---
date: 2026-06-15
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: find-admins-without-mfa-graph
description: >-
  Find privileged accounts with no MFA registered using Microsoft Graph
  PowerShell — the modern replacement for the dead MSOnline/AzureAD approach.
---

# Which of your admins still don't have MFA?

Of all the gaps a security review can surface, a **privileged account with no
second factor** is the one that should keep you up at night. A Global Admin
protected by a password alone is a single phish away from total tenant compromise.
You want a report that does exactly one thing: list the accounts that hold a
privileged role *and* have no MFA registered.

If you've done this before with `Get-MsolUser` and `StrongAuthenticationMethods`,
that approach is dead — let's do it the way that still works.

<!-- more -->

!!! warning "MSOnline and AzureAD are retired — stop writing new scripts against them"
    The old recipe read `StrongAuthenticationMethods` via the **MSOnline** module
    (`Connect-MsolService`). Microsoft retired MSOnline and AzureAD in 2024–2025;
    `Connect-MsolService` now fails outright in many tenants. Everything below uses
    **Microsoft Graph PowerShell**, which is the only supported path forward. If you
    inherit an old MFA script, treat it as a rewrite, not a patch.

This builds on [connecting to Microsoft Graph](connect-microsoft-graph-powershell.md)
and pairs naturally with [auditing Entra users](audit-entra-users-powershell.md).

## The clean shortcut: the registration report

Graph exposes a purpose-built report —
`userRegistrationDetails` — that already knows, per user, whether MFA is
registered and whether they're privileged. No per-user method enumeration
required. It needs `AuditLog.Read.All`:

```powershell
Connect-MgGraph -Scopes 'AuditLog.Read.All', 'Directory.Read.All'

$details = Get-MgReportAuthenticationMethodUserRegistrationDetail -All

$adminsNoMfa = $details |
    Where-Object { $_.IsAdmin -and -not $_.IsMfaRegistered } |
    Select-Object UserDisplayName, UserPrincipalName,
        @{ Name = 'Methods'; Expression = { $_.MethodsRegistered -join ', ' }}

$adminsNoMfa | Format-Table -AutoSize
```

`IsAdmin` is true when the account holds *any* privileged directory role, and
`IsMfaRegistered` is the authoritative "can this user actually do MFA" flag. Two
booleans and you have your answer.

!!! tip "`MethodsRegistered` shows what they *do* have"
    An admin with `IsMfaRegistered = $false` might still have a phone number on
    file for SSPR — `MethodsRegistered` tells you what's actually there, which
    helps you tell "never set anything up" apart from "set up the wrong thing".

## Want it broken down by role?

The report's `IsAdmin` flag doesn't tell you *which* role. If you need "no MFA,
and specifically a Global Admin", join against the active role members. This is
the modern equivalent of the old `Get-AzureADDirectoryRoleMember` loop:

```powershell
$sensitiveRoles = 'Global Administrator', 'Privileged Role Administrator',
    'Security Administrator', 'Exchange Administrator', 'SharePoint Administrator'

# Map each user's UPN -> the sensitive roles they hold
$roleMap = @{}
foreach ($role in Get-MgDirectoryRole -All) {
    if ($role.DisplayName -notin $sensitiveRoles) { continue }
    foreach ($member in Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id) {
        $upn = $member.AdditionalProperties.userPrincipalName
        if ($upn) {
            if (-not $roleMap.ContainsKey($upn)) { $roleMap[$upn] = @() }
            $roleMap[$upn] += $role.DisplayName
        }
    }
}

$report = foreach ($a in $adminsNoMfa) {
    $roles = $roleMap[$a.UserPrincipalName]
    if ($roles) {
        [pscustomobject]@{
            User  = $a.UserDisplayName
            UPN   = $a.UserPrincipalName
            Roles = $roles -join '; '
            MFA   = 'NOT registered'
        }
    }
}
```

!!! note "`Get-MgDirectoryRole` only returns *activated* roles"
    A role that's never had a member assigned doesn't show up. That's usually what
    you want (you only care about roles in use), but if you're hunting for eligible
    PIM assignments rather than active ones, you need the
    `roleManagement/directory` endpoints instead — a different, more involved query.

## Ship it

```powershell
if (-not $report) {
    Write-Host "Every privileged account has MFA registered. Nice."
    return
}

$report | Export-Csv -Path ("AdminsNoMFA_{0:yyyy-MM-dd}.csv" -f (Get-Date)) `
    -NoTypeInformation -Encoding UTF8

Write-Warning "$($report.Count) privileged account(s) without MFA — see CSV."
```

!!! danger "Don't dump admin UPNs to the console on a shared host"
    The old script wrote every admin's sign-in name to the terminal with
    `Write-Host`. On a shared jump box that lands in the PowerShell transcript and
    console history — effectively a list of your highest-value targets, missing
    their second factor. Write to a CSV with a restrictive ACL and skip the
    console spew.

A report like this is the kind of thing you run weekly and want to see return
*zero rows*. The first time it doesn't, you've found the most important fifteen
minutes of remediation work in your tenant.
