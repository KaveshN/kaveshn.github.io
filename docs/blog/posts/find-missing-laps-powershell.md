---
date: 2026-06-12
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: find-missing-laps-powershell
description: >-
  Report on every Windows machine that doesn't have a LAPS-managed local admin
  password — covering both Windows LAPS and the legacy Microsoft LAPS attribute.
---

# Finding the machines that slipped through LAPS

LAPS — the Local Administrator Password Solution — is one of the highest
return-on-effort security controls you can deploy: every machine gets a unique,
rotated, randomly-generated local administrator password, stored in the
directory. The catch is coverage. A control that protects 90% of your estate
leaves the other 10% as the soft targets an attacker pivots through. So the
question that matters isn't "is LAPS deployed?" — it's "*which machines aren't
covered yet?*"

This script answers exactly that.

<!-- more -->

## Two attributes, depending on which LAPS

There are two generations of LAPS, and they store the expiry timestamp in
different attributes:

| LAPS generation | Password-expiry attribute |
|---|---|
| **Windows LAPS** (built into Windows, 2023+) | `msLAPS-PasswordExpirationTime` |
| **Legacy Microsoft LAPS** (the old MSI) | `ms-Mcs-AdmPwdExpirationTime` |

If the relevant attribute is **empty**, that machine has never had a LAPS
password set — it's non-compliant. A migrating estate will have a mix of both,
so check for either:

```powershell
Import-Module ActiveDirectory

$searchBase = 'OU=Servers,DC=corp,DC=example,DC=com'   # scope it — see the gotcha
$props      = 'msLAPS-PasswordExpirationTime',
              'ms-Mcs-AdmPwdExpirationTime',
              'OperatingSystem', 'LastLogonTimeStamp'

$missing = Get-ADComputer -SearchBase $searchBase -Filter 'Enabled -eq $true' -Properties $props |
    Where-Object {
        -not $_.'msLAPS-PasswordExpirationTime' -and
        -not $_.'ms-Mcs-AdmPwdExpirationTime'
    } |
    Select-Object Name, OperatingSystem,
        @{ Name = 'LastLogon'; Expression = {
            [DateTime]::FromFileTime($_.LastLogonTimeStamp)
        }} |
    Sort-Object Name
```

!!! warning "Scope the search — don't `-Filter *` the whole domain"
    The classic version of this script runs `Get-ADComputer -Filter * -Properties *`
    against the entire domain. `-Properties *` pulls **every** attribute for
    **every** computer object — slow, memory-hungry, and mostly wasted. Always:
    name the exact properties you need, filter server-side where you can
    (`-Filter 'Enabled -eq $true'`), and scope with `-SearchBase`.

## Exclude the things that are *meant* to be excluded

Domain controllers don't get LAPS (their local admin account is governed
differently), and you'll usually have a handful of legitimately-exempt boxes.
Keep the exclusion list in a file, not in the script:

```powershell
$exclusions = @()
if (Test-Path 'C:\ProgramData\LAPS\exclusions.txt') {
    $exclusions = Get-Content 'C:\ProgramData\LAPS\exclusions.txt'
}

$missing = $missing | Where-Object {
    $_.Name -notin $exclusions -and
    $_.DistinguishedName -notlike '*OU=Domain Controllers*'
}
```

!!! tip "`-notin` needs a real array on the right"
    `$_.Name -notin $exclusions` only works if `$exclusions` is an array.
    `Get-Content` returns an array of lines, so you're fine — but if the file has a
    single line, some PowerShell versions hand you a string instead. Wrapping with
    `@(Get-Content …)` forces an array every time and saves you a baffling
    debugging session.

## Report only what's broken

```powershell
if (-not $missing) {
    Write-Host "Every in-scope machine has a LAPS password. Nothing to report."
    return
}

$style = '<style>body{font-family:Segoe UI,Arial;font-size:10pt}' +
         'table{border-collapse:collapse}th,td{border:1px solid #ccc;padding:6px}' +
         'th{background:#2d2d6b;color:#fff}</style>'

$html = $missing | ConvertTo-Html -Head $style `
    -PreContent "<h2>Machines missing a LAPS password — $($missing.Count)</h2>" |
    Out-String

Send-MailMessage -To 'secops@example.com' -From 'laps-report@example.com' `
    -SmtpServer 'smtp.example.com' -BodyAsHtml -Body $html `
    -Subject "LAPS coverage gaps: $($missing.Count) machine(s)"
```

Sort the output by `LastLogon`. A non-compliant machine that signed in this
morning is a live remediation target; one that hasn't checked in for six months
is probably a stale computer object you should be *deleting*, not patching — which
is its own useful finding.

This is the kind of weekly report that quietly drives a number from 85% coverage
to 100% — and gives you a clean artefact for the next ISO 27001 or Cyber
Essentials audit.
