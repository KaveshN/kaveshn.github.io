---
date: 2026-05-20
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: connect-microsoft-graph-powershell
description: >-
  The foundation for every Entra ID automation: installing the Microsoft Graph
  PowerShell SDK, connecting with the right scopes, and the difference between
  delegated and app-only access.
---

# Connecting to Microsoft Graph with PowerShell

Almost every Entra ID automation I write starts the same way: connect to
Microsoft Graph, request the least privilege I need, do the work, disconnect.
Get this foundation right and everything else — users, groups, access packages —
is just cmdlets on top.

This post covers installing the SDK, the two auth models (delegated vs
app-only), and the scope gotcha that trips everyone up the first time.

<!-- more -->

## Install the SDK

The Microsoft Graph PowerShell SDK ships as a meta-module (`Microsoft.Graph`)
that pulls in every sub-module — handy but heavy. For day-to-day identity work I
install only what I use:

```powershell
# Everything (large; ~40 sub-modules)
Install-Module Microsoft.Graph -Scope CurrentUser

# Or just the bits this series uses (faster)
Install-Module Microsoft.Graph.Authentication -Scope CurrentUser
Install-Module Microsoft.Graph.Users -Scope CurrentUser
Install-Module Microsoft.Graph.Groups -Scope CurrentUser
Install-Module Microsoft.Graph.Identity.Governance -Scope CurrentUser
```

!!! tip "Scope CurrentUser avoids the admin prompt"
    `-Scope CurrentUser` installs into your profile, so you don't need an
    elevated shell. On a locked-down work machine that's often the only option.

## Delegated sign-in (you, interactively)

The quickest way in — a browser pops up, you sign in as yourself, and Graph acts
**as you** with the scopes you request:

```powershell
Connect-MgGraph -Scopes "User.Read.All", "Group.Read.All"
```

Check what you actually got — the token may have fewer scopes than you asked for
if you lack consent:

```powershell
Get-MgContext | Select-Object Account, TenantId, Scopes
(Get-MgContext).Scopes
```

!!! warning "Scopes are additive across a session — but only after reconnect"
    Calling `Connect-MgGraph` again with new scopes **replaces** the token; it
    doesn't merge. If a cmdlet fails with `Insufficient privileges`, reconnect
    with the full set you need:

    ```powershell
    Connect-MgGraph -Scopes "User.Read.All", "Group.ReadWrite.All", "AuditLog.Read.All"
    ```

## App-only (unattended automation)

For scheduled tasks and runbooks there's no human to click "sign in". You
register an app in Entra, grant it **application** permissions (admin-consented),
and authenticate with a certificate. Prefer a certificate over a client
secret — secrets expire and leak; cert private keys can stay in the machine
store.

```powershell
Connect-MgGraph `
  -ClientId  "00000000-0000-0000-0000-000000000000" `
  -TenantId  "contoso.onmicrosoft.com" `
  -CertificateThumbprint "A1B2C3D4E5F6...."
```

!!! note "Delegated vs application scopes are different permissions"
    `User.Read.All` as a **delegated** scope lets the signed-in admin read
    users. As an **application** permission it lets the app read all users with
    no user present. They're granted separately in the app registration —
    don't assume consenting one covers the other.

## Find the cmdlet (and the permission) you need

The SDK has thousands of cmdlets. Two helpers save a lot of portal-digging:

```powershell
# Which cmdlet maps to which Graph endpoint?
Find-MgGraphCommand -Command Get-MgUser | Select-Object Command, Method, Uri

# Which permissions does an endpoint require?
Find-MgGraphPermission user -PermissionType Delegated
```

## Always disconnect in automation

In a long-running session or a shared runbook host, leave no token behind:

```powershell
Disconnect-MgGraph
```

That's the whole pattern: install → connect with least privilege → verify scopes
→ work → disconnect. The next post puts it to use auditing every user in the
tenant.
