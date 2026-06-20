---
date: 2026-06-10
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
tags:
  - Entra ID
  - Microsoft Graph
  - Entitlement Management
  - Automation
slug: assign-entra-access-packages-powershell
description: >-
  Use Microsoft Graph PowerShell to grant Entra ID Entitlement Management
  access packages programmatically — the admin-add assignment request, end to
  end.
---

# Assigning Entra ID access packages with PowerShell

Entitlement Management is how Entra ID does access-at-scale: bundle the groups,
apps, and roles a role needs into an **access package**, then assign people to
it through a policy. The portal is fine for one person. For onboarding a cohort,
or wiring it into a joiner workflow, you want PowerShell.

This post walks the full admin-driven assignment: find the package, find the
policy, submit the request, confirm it landed.

<!-- more -->

!!! note "Permissions"
    You need `EntitlementManagement.ReadWrite.All` (delegated, as an Identity
    Governance admin) or the equivalent application permission for unattended
    runs. Access packages are a Microsoft Entra ID Governance (premium) feature.

```powershell
Connect-MgGraph -Scopes "EntitlementManagement.ReadWrite.All"
```

## 1. Find the access package

```powershell
$pkg = Get-MgEntitlementManagementAccessPackage `
  -Filter "displayName eq 'Engineering — Standard Access'"

$pkg.Id
```

## 2. Find the assignment policy

A package can have several policies (different approval rules, different eligible
populations). An admin-add must target a specific one:

```powershell
$policy = Get-MgEntitlementManagementAssignmentPolicy `
  -Filter "accessPackageId eq '$($pkg.Id)'" |
  Select-Object -First 1

$policy.Id
```

## 3. Resolve the target user

```powershell
$user = Get-MgUser -UserId "ada.lovelace@contoso.com"
$user.Id
```

## 4. Submit the assignment request

The request body is the fiddly part. `requestType = "adminAdd"` is the admin
granting access directly (no end-user request, no approval round-trip):

```powershell
$params = @{
    requestType = "adminAdd"
    assignment = @{
        targetId           = $user.Id
        assignmentPolicyId = $policy.Id
        accessPackageId    = $pkg.Id
    }
}

$request = New-MgEntitlementManagementAssignmentRequest -BodyParameter $params
$request.Id
$request.State   # e.g. "submitted" / "delivering"
```

!!! warning "Cmdlet and property names track the Graph version"
    The SDK is generated from the Graph metadata, so names shift between v1.0 and
    beta and across SDK releases. If `New-MgEntitlementManagement...` errors,
    confirm the current shape rather than trusting this verbatim:

    ```powershell
    Find-MgGraphCommand -Command "New-MgEntitlementManagementAssignmentRequest" |
      Select-Object Command, Method, Uri, ApiVersion
    Get-Help New-MgEntitlementManagementAssignmentRequest -Full
    ```

## 5. Confirm it delivered

Assignment is asynchronous — the request goes `submitted → delivering →
delivered`. Poll the request, not the assignment:

```powershell
do {
    Start-Sleep -Seconds 5
    $state = (Get-MgEntitlementManagementAssignmentRequest -AccessPackageAssignmentRequestId $request.Id).State
    Write-Host "State: $state"
} while ($state -in @("submitted", "delivering"))
```

## Bulk: onboard a whole cohort

The point of scripting this is doing it 50 times. Loop a list of UPNs, and don't
let one failure abort the batch:

```powershell
$upns = Get-Content .\new-hires.txt

foreach ($upn in $upns) {
    try {
        $u = Get-MgUser -UserId $upn -ErrorAction Stop
        $body = @{
            requestType = "adminAdd"
            assignment  = @{
                targetId           = $u.Id
                assignmentPolicyId = $policy.Id
                accessPackageId    = $pkg.Id
            }
        }
        New-MgEntitlementManagementAssignmentRequest -BodyParameter $body | Out-Null
        Write-Host "[ok]   $upn"
    }
    catch {
        Write-Warning "[fail] $upn — $($_.Exception.Message)"
    }
}
```

!!! tip "Let the platform dedupe"
    Re-running an `adminAdd` for someone who already has the package is harmless —
    Entitlement Management won't double-assign. That makes the loop safe to retry,
    which matters when item 37 of 50 fails on a transient error.

That closes the loop on this PowerShell series: [connect](connect-microsoft-graph-powershell.md)
→ [audit](audit-entra-users-powershell.md) → grant. From here it's the same
pattern for any Graph resource — find the cmdlet, check the scope, script it.
