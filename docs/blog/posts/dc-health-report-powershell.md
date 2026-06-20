---
date: 2026-06-09
authors:
  - kavesh
categories:
  - PowerShell
  - Infrastructure
tags:
  - Active Directory
  - Domain Controllers
  - Monitoring
  - Reporting
slug: dc-health-report-powershell
description: >-
  Build a colour-coded HTML domain controller health report in PowerShell —
  ping, DNS, uptime, disk, services, and DCDiag — across the whole forest.
---

# A domain controller health report you'll actually read

Every morning someone should be able to glance at one page and know whether the
directory is healthy. `dcdiag` and `repadmin` give you the raw truth, but their
output is a wall of text — nobody reads it daily. The fix is a scheduled script
that polls every DC in the forest, scores each check, and emails a colour-coded
HTML table: green is fine, yellow needs a look, red is on fire.

<!-- more -->

This is a sanitised, CIM-based rewrite of a pattern that's been floating around
the PowerShell community for years (the original
[Test-DomainControllerHealth](https://powershellneedfulthings.com) by Jean Louw,
itself building on Paul Cunningham's HTML helpers). Swap the placeholders for
your own domain and relay.

## Enumerate the targets

Let the forest tell you what to check — don't hardcode a server list that goes
stale the moment someone promotes a new DC:

```powershell
Import-Module ActiveDirectory

$domains = (Get-ADForest).Domains
$allDCs  = foreach ($domain in $domains) {
    Get-ADDomainController -Filter * -Server $domain
}
```

## Check one DC, return a flat object

The core idea: one function per DC that returns a `[PSCustomObject]` of named
checks. A flat object slots straight into a table later.

```powershell
function Get-DCHealth {
    param([string]$ComputerName)

    $reachable = Test-Connection $ComputerName -Count 1 -Quiet

    # CIM, not WMI — Get-WmiObject is gone in PowerShell 7
    $os = if ($reachable) {
        Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName -ErrorAction SilentlyContinue
    }

    $uptimeHours = if ($os) {
        [math]::Round(((Get-Date) - $os.LastBootUpTime).TotalHours)
    } else { $null }

    $services = 'DNS', 'NTDS', 'Netlogon' | ForEach-Object {
        $svc = Get-Service -ComputerName $ComputerName -Name $_ -ErrorAction SilentlyContinue
        [pscustomobject]@{ Name = $_; Status = if ($svc.Status -eq 'Running') { 'Pass' } else { 'Fail' } }
    }

    [pscustomobject]@{
        Server     = $ComputerName.ToLower()
        Ping       = if ($reachable) { 'Pass' } else { 'Fail' }
        Uptime     = $uptimeHours
        DNSService = ($services | Where-Object Name -eq 'DNS').Status
        NTDS       = ($services | Where-Object Name -eq 'NTDS').Status
        Netlogon   = ($services | Where-Object Name -eq 'Netlogon').Status
        DCDiag     = Get-DCDiagSummary -ComputerName $ComputerName -Reachable $reachable
    }
}
```

!!! warning "Use CIM, not WMI"
    The classic versions of this script call `Get-WmiObject` everywhere. It works
    on Windows PowerShell 5.1 but is **removed** in PowerShell 7+. `Get-CimInstance`
    is the drop-in replacement and uses WS-MAN instead of DCOM, so it's friendlier
    to firewalls too.

## Parse DCDiag without losing your mind

`dcdiag.exe` is text-only, but its output is regular enough to parse. The trick
is a `switch -Regex` that captures the test name on the "Starting test" line and
the verdict on the "passed/failed test" line:

```powershell
function Get-DCDiagSummary {
    param([string]$ComputerName, [bool]$Reachable)

    if (-not $Reachable) { return 'Fail' }

    $raw = dcdiag.exe /s:$ComputerName /test:Services /test:Advertising /test:KnowsOfRoleHolders
    $results = @{}
    $testName = $null

    switch -Regex ($raw) {
        'Starting test: (\w+)'        { $testName = $Matches[1]; continue }
        '(passed|failed) test (\w+)'  { $results[$Matches[2]] = $Matches[1] }
    }

    if ($results.Values -contains 'failed') { 'Fail' } else { 'Pass' }
}
```

!!! tip "`$Matches` is populated by `-match` and `switch -Regex`"
    Every successful regex match fills the automatic `$Matches` hashtable —
    `$Matches[0]` is the whole match, `$Matches[1]` the first capture group. It's
    the idiomatic way to pull fields out of text in PowerShell, no `Select-String`
    needed.

## Score it into colour-coded HTML

Map each verdict to a CSS class. A tiny dispatcher keeps the table-building loop
readable:

```powershell
$style = @'
<style>
  body  { font-family: Segoe UI, Arial; font-size: 10pt; }
  table { border-collapse: collapse; }
  th, td { border: 1px solid #ccc; padding: 6px; }
  th    { background: #2d2d6b; color: #fff; }
  .pass { background: #c8f7c5; }
  .warn { background: #fff3b0; }
  .fail { background: #f7b2b2; }
</style>
'@

function Format-Cell ($value) {
    switch ($value) {
        'Pass'  { "<td class='pass'>$value</td>" }
        'Fail'  { "<td class='fail'>$value</td>" }
        default {
            # Uptime: yellow if a DC rebooted in the last 24h
            if ($value -is [int] -and $value -le 24) { "<td class='warn'>$value</td>" }
            else { "<td>$value</td>" }
        }
    }
}

$report = $allDCs | ForEach-Object { Get-DCHealth -ComputerName $_.HostName }

$rows = foreach ($dc in $report) {
    "<tr><td>$($dc.Server)</td>" +
    (Format-Cell $dc.Ping) +
    (Format-Cell $dc.Uptime) +
    (Format-Cell $dc.DNSService) +
    (Format-Cell $dc.NTDS) +
    (Format-Cell $dc.Netlogon) +
    (Format-Cell $dc.DCDiag) + "</tr>"
}

$html = @"
<html><head>$style</head><body>
<h2>Domain Controller Health — $(Get-Date -Format 'yyyy-MM-dd HH:mm')</h2>
<table>
<tr><th>Server</th><th>Ping</th><th>Uptime (h)</th><th>DNS</th><th>NTDS</th><th>Netlogon</th><th>DCDiag</th></tr>
$($rows -join "`n")
</table></body></html>
"@
```

## Deliver it

```powershell
$mail = @{
    To         = 'it-team@example.com'
    From       = 'dc-health@example.com'
    Subject    = "AD: Domain Controller Health — $(Get-Date -Format 'yyyy-MM-dd')"
    SmtpServer = 'smtp.example.com'
    BodyAsHtml = $true
    Body       = $html
}
Send-MailMessage @mail
```

!!! danger "`Send-MailMessage` is officially deprecated"
    Microsoft marked it obsolete because it doesn't support modern TLS or OAuth
    cleanly. It still works on an internal relay, but for anything internet-facing
    use [MailKit](https://github.com/jstedfast/MailKit) or a Graph
    `sendMail` call instead. I'm keeping `Send-MailMessage` here because the whole
    point is a 2 a.m. scheduled task hitting an internal smart host — but don't
    reach for it in new, externally-routed code.

Schedule it every morning with Task Scheduler running as a service account that
has read access to the DCs, and you've turned a wall of `dcdiag` text into a
report people actually open.

Next in this series: turning the same scheduled-script pattern toward security —
[getting an email the moment someone joins Domain Admins](monitor-privileged-groups-powershell.md).
