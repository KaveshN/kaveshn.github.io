---
date: 2026-06-14
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: report-expiring-app-secrets-graph
description: >-
  Scan every Entra app registration for expiring secrets and certificates with
  Microsoft Graph PowerShell — and email a report before something breaks at 2am.
---

# Never get surprised by an expired app secret again

App registration secrets and certificates expire. When one does, whatever it was
authenticating — a SAML app, a scheduled job, an API integration — stops working,
usually at the worst possible time, and usually with an error message that points
nowhere near the real cause. The fix is boring and effective: scan the whole
tenant on a schedule and email a report of everything expiring in the next N days,
while there's still time to rotate.

<!-- more -->

This builds on [connecting to Microsoft Graph](connect-microsoft-graph-powershell.md) —
make sure `Get-MgContext` returns a session before you start.

## Connect with read-only application scope

Reading app registrations needs `Application.Read.All`. That's it — resist the
urge to grab `Directory.Read.All`, which is far broader than this job requires:

```powershell
Connect-MgGraph -Scopes 'Application.Read.All'
```

## Two credential types live on every app

This is the bit people miss. An app registration can hold **both**:

- **`PasswordCredentials`** — client secrets (the "secret" you copy once and never
  see again).
- **`KeyCredentials`** — certificates.

A report that only checks one type gives you false confidence. Pull every app
with just the properties you need, then walk both collections:

```powershell
$thresholdDays = 30
$now    = Get-Date
$cutoff = $now.AddDays($thresholdDays)

$apps = Get-MgApplication -All -Property Id, AppId, DisplayName,
    PasswordCredentials, KeyCredentials

$findings = foreach ($app in $apps) {

    foreach ($secret in $app.PasswordCredentials) {
        if ($secret.EndDateTime -le $cutoff) {
            [pscustomobject]@{
                App        = $app.DisplayName
                AppId      = $app.AppId
                Type       = 'Secret'
                Name       = $secret.DisplayName
                Expires    = $secret.EndDateTime
                DaysLeft   = [math]::Round(($secret.EndDateTime - $now).TotalDays)
            }
        }
    }

    foreach ($cert in $app.KeyCredentials) {
        if ($cert.EndDateTime -le $cutoff) {
            [pscustomobject]@{
                App        = $app.DisplayName
                AppId      = $app.AppId
                Type       = 'Certificate'
                Name       = $cert.DisplayName
                Expires    = $cert.EndDateTime
                DaysLeft   = [math]::Round(($cert.EndDateTime - $now).TotalDays)
            }
        }
    }
}

$findings = $findings | Sort-Object DaysLeft
```

!!! tip "Ask for only the properties you need"
    `Get-MgApplication -Property …` is the difference between a snappy report and
    one that times out on a large tenant. The default object is slim and *won't*
    include `PasswordCredentials`/`KeyCredentials` unless you name them — request
    exactly the fields you'll use, nothing more.

## Don't forget the SAML signing certs

Here's the gap even good versions of this script leave open: **enterprise app
SAML signing certificates don't live on the app registration.** They live on the
matching **service principal**, in `KeyCredentials` with usage `Sign`. If you run
SAML SSO apps, scan service principals too:

```powershell
$spFindings = foreach ($sp in Get-MgServicePrincipal -All -Property Id, DisplayName, KeyCredentials) {
    foreach ($cert in $sp.KeyCredentials | Where-Object Usage -eq 'Sign') {
        if ($cert.EndDateTime -le $cutoff) {
            [pscustomobject]@{
                App      = $sp.DisplayName
                Type     = 'SAML Signing Cert'
                Name     = $cert.DisplayName
                Expires  = $cert.EndDateTime
                DaysLeft = [math]::Round(($cert.EndDateTime - $now).TotalDays)
            }
        }
    }
}
```

!!! warning "An expired SAML signing cert breaks sign-in for everyone"
    Unlike a secret that breaks one integration, a lapsed SAML signing certificate
    takes down SSO for an entire application's user base at once. These belong at
    the top of the report, flagged red.

## Highlight what's already on fire

Colour the rows so urgency is obvious at a glance — already-expired in red,
expiring soon in amber:

```powershell
$all = @($findings) + @($spFindings) | Sort-Object DaysLeft

$rows = foreach ($f in $all) {
    $class = if ($f.DaysLeft -lt 0) { 'expired' }
             elseif ($f.DaysLeft -le 7) { 'soon' } else { 'ok' }
    "<tr class='$class'><td>$($f.App)</td><td>$($f.Type)</td>" +
    "<td>$($f.Expires.ToString('yyyy-MM-dd'))</td><td>$($f.DaysLeft)</td></tr>"
}

$html = @"
<html><head><style>
  body{font-family:Segoe UI,Arial;font-size:10pt}
  table{border-collapse:collapse}th,td{border:1px solid #ccc;padding:6px}
  th{background:#2d2d6b;color:#fff}
  .expired{background:#f7b2b2}.soon{background:#fff3b0}
</style></head><body>
<h2>App credentials expiring within $thresholdDays days</h2>
<table><tr><th>App</th><th>Type</th><th>Expires</th><th>Days left</th></tr>
$($rows -join "`n")
</table></body></html>
"@

Send-MailMessage -To 'identity-team@example.com' -From 'app-secrets@example.com' `
    -SmtpServer 'smtp.example.com' -BodyAsHtml -Body $html `
    -Subject "Entra: $($all.Count) app credential(s) expiring soon"
```

## Make it unattended

Interactive `Connect-MgGraph` is fine while you're building the report, but a
scheduled task can't click a sign-in prompt. For 2 a.m. runs, switch to
**app-only auth** — register an app, grant it `Application.Read.All` as an
*application* permission, and connect with a certificate:

```powershell
Connect-MgGraph -ClientId $appId -TenantId $tenantId -CertificateThumbprint $thumb
```

Certificate-based app-only auth keeps you off stored secrets entirely — which is
a nice irony for the script whose whole job is nagging you about expiring secrets.
