---
date: 2026-06-13
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: rotate-krbtgt-password-safely
description: >-
  The krbtgt account signs every Kerberos ticket in your domain. Here's how to
  rotate its password safely — and the double-reset mistake that breaks Kerberos
  domain-wide.
---

# Rotating the krbtgt password without breaking Kerberos

There's one account in every Active Directory domain whose password signs *every
Kerberos ticket ever issued*: `krbtgt`. It's disabled, nobody logs in as it, and
most people never touch it — which is exactly why attackers love it. Steal the
krbtgt hash and you can forge **Golden Tickets**: valid-looking TGTs for any user,
any group, with a lifetime you choose. The only cure is to change the password.

But krbtgt is also the one password you can get *catastrophically* wrong. Rotate
it carelessly and you break authentication for the entire domain. Let's do it
properly.

<!-- more -->

## Why there's a right way and a very wrong way

Active Directory keeps the **current and previous** krbtgt password (N and N-1)
so that tickets signed just before a change still validate. That's the safety
net. It's also the trap:

!!! danger "The double-reset that breaks every domain"
    If you reset krbtgt **twice in quick succession** — before all existing
    tickets have expired — you invalidate the N-1 key that those in-flight tickets
    were signed with. Every ticket issued under the old key is instantly
    untrustable, and you get domain-wide authentication failures until things
    expire and re-issue. **Reset once, then wait at least one maximum TGT lifetime
    (default 10 hours, plus clock skew) before resetting again.**

So the procedure is: reset once → force replication → verify convergence →
**wait** → only then consider a second reset.

## Step 1 — know your ticket lifetime

The safe waiting window comes from your Kerberos policy. Read it before you touch
anything:

```powershell
Import-Module ActiveDirectory

$domain = Get-ADDomain
$krbtgt = Get-ADUser krbtgt -Properties PasswordLastSet -Server $domain.PDCEmulator

# Default domain policy: max TGT lifetime (hrs) and max clock skew (mins)
[xml]$gpo = Get-GPOReport -Guid '{31B2F340-016D-11D2-945F-00C04FB984F9}' -ReportType Xml
$security = ($gpo.gpo.Computer.ExtensionData |
    Where-Object { $_.Name -eq 'Security' }).Extension.ChildNodes

$maxTgtHours  = ($security | Where-Object Name -eq 'MaxTicketAge').SettingNumber
$maxSkewMins  = ($security | Where-Object Name -eq 'MaxClockSkew').SettingNumber
if (-not $maxTgtHours) { $maxTgtHours = 10 }   # AD defaults
if (-not $maxSkewMins) { $maxSkewMins = 5 }

$safeAfter = $krbtgt.PasswordLastSet.AddHours($maxTgtHours).AddMinutes($maxSkewMins * 2)

Write-Host "krbtgt last set: $($krbtgt.PasswordLastSet)"
Write-Host "Earliest SAFE time for the NEXT reset: $safeAfter"
```

If `$safeAfter` is in the future, **stop** — a previous reset is still inside its
window. Doubling the clock skew accounts for drift in both directions.

## Step 2 — reset once, on the PDC emulator

`Set-ADAccountPassword -Reset` against krbtgt forces AD to generate a brand-new
random key (you don't supply a real password — krbtgt's is never used to log in,
so the actual string is throwaway):

```powershell
# Generate throwaway complexity-satisfying bytes; AD replaces the key anyway
$bytes = [byte[]]::new(32)
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$dummy = [Convert]::ToBase64String($bytes)

Set-ADAccountPassword -Identity $krbtgt.DistinguishedName `
    -Server $domain.PDCEmulator -Reset `
    -NewPassword (ConvertTo-SecureString $dummy -AsPlainText -Force)
```

!!! note "Always target the PDC emulator"
    Reset on the PDC emulator so there's a single authoritative origin to
    replicate *out* from. Resetting on some random DC means you have to wait for it
    to reach the PDC first, complicating the convergence check.

## Step 3 — force replication and verify convergence

Don't trust normal replication latency for something this important. Push the
single object to every writable DC, then confirm `PasswordLastSet` matches
everywhere:

```powershell
$rwdcs = Get-ADDomainController -Filter { IsReadOnly -eq $false }
$pdc   = $domain.PDCEmulator
$krbDN = $krbtgt.DistinguishedName

foreach ($dc in $rwdcs | Where-Object HostName -ne $pdc) {
    repadmin.exe /replsingleobj $dc.HostName $pdc $krbDN | Out-Null
}

$pdcSet = (Get-ADUser krbtgt -Properties PasswordLastSet -Server $pdc).PasswordLastSet
foreach ($dc in $rwdcs) {
    $dcSet = (Get-ADUser krbtgt -Properties PasswordLastSet -Server $dc.HostName).PasswordLastSet
    $state = if ($dcSet -eq $pdcSet) { 'IN SYNC' } else { 'OUT OF SYNC' }
    Write-Host ("{0,-30} {1}" -f $dc.HostName, $state)
}
```

When every DC reports **IN SYNC**, the rotation is complete and safe.

## Don't write this from scratch

Everything above is the *mechanism* so you understand what's happening. For
production, use Microsoft's battle-tested script — Jared Poeppelman's
**New-KrbtgtKeys.ps1** (the v2.x rewrite on GitHub). It wraps all of this in
three modes:

- **Informational** — checks prerequisites and tells you the safe reset window.
  Run it any time; it changes nothing.
- **Simulation** — triggers the replication path *without* resetting, so you can
  measure how long convergence actually takes in your environment.
- **Reset** — does the real thing, with guardrails and the N-1 warning built in.

!!! tip "Run Informational → Simulation → Reset, in that order"
    The first two modes are completely safe and tell you whether Reset will go
    cleanly. There is no reason to ever run Reset cold.

## When to actually do this

- **Routinely**, twice with the safe gap between, if you suspect compromise or as
  part of incident recovery (this is a standard step in evicting an attacker).
- **Periodically** (e.g. annually) as hygiene, especially if krbtgt hasn't changed
  since the domain was built — a password from 2014 is a 2014-era key sitting in
  every backup you've ever taken.

After a confirmed compromise, you reset it *twice* (with the wait between) to flush
both the N and N-1 keys — that's the only way to fully invalidate forged tickets.
