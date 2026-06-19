---
date: 2026-06-19
authors:
  - kavesh
categories:
  - PowerShell
  - Identity
slug: room-mailbox-external-booking-powershell
description: >-
  External guests trying to book your meeting rooms get silently rejected and
  nobody knows why. Two scripts — one to audit every room mailbox in the tenant,
  one to fix them all in a single run.
---

# Fixing room mailboxes that silently reject external meeting invites

Someone outside your organisation tries to book a meeting room. They send the
invite, get no error, and assume it worked. Then the meeting happens and the
room is already taken by someone else who booked it through the internal system.
The external guest never got a rejection — the room just quietly discarded their
invite.

The culprit is almost always `ProcessExternalMeetingMessages` set to `$false`
on the room mailbox. It's the default in many tenants and it trips people up
constantly.

<!-- more -->

## Why it happens

When Exchange Online provisions a room mailbox, the calendar processing defaults
are conservative. `ProcessExternalMeetingMessages` controls whether the room's
calendar assistant will accept or decline meeting requests that originate outside
your organisation. When it's `$false`, external invites are silently dropped —
no bounce, no rejection notice, nothing.

A few other settings compound the problem. If `DeleteSubject` or `DeleteComments`
are set to `$true`, the room strips out the meeting details when it accepts, so
attendees see a blank entry on the room calendar. And if `AddOrganizerToSubject`
is `$true`, the subject gets replaced with the organiser's name, which looks odd
in a shared room display.

## Step 1: audit every room in the tenant

Before changing anything, get a picture of where you stand. This script pulls
every room mailbox and checks the settings that matter, then emails you an
HTML report.

```powershell
### Audit all room mailboxes for external booking capability
$rooms = Get-Mailbox -RecipientTypeDetails RoomMailbox -ResultSize Unlimited

$report = foreach ($room in $rooms) {
    $cal = Get-CalendarProcessing -Identity $room.Identity
    [PSCustomObject]@{
        DisplayName                    = $room.DisplayName
        EmailAddress                   = $room.PrimarySmtpAddress
        ProcessExternalMeetingMessages = $cal.ProcessExternalMeetingMessages
        DeleteComments                 = $cal.DeleteComments
        DeleteSubject                  = $cal.DeleteSubject
        AddOrganizerToSubject          = $cal.AddOrganizerToSubject
        RemovePrivateProperty          = $cal.RemovePrivateProperty
    }
}

$sorted  = $report | Sort-Object ProcessExternalMeetingMessages, DisplayName
$blocked = $report | Where-Object { -not $_.ProcessExternalMeetingMessages }

### Build HTML email
$rows = foreach ($r in $sorted) {
    $colour = if ($r.ProcessExternalMeetingMessages) { '#d4edda' } else { '#f8d7da' }
    "<tr style='background:$colour'>
        <td>$($r.DisplayName)</td>
        <td>$($r.EmailAddress)</td>
        <td>$($r.ProcessExternalMeetingMessages)</td>
        <td>$($r.DeleteComments)</td>
        <td>$($r.DeleteSubject)</td>
        <td>$($r.AddOrganizerToSubject)</td>
        <td>$($r.RemovePrivateProperty)</td>
    </tr>"
}

$html = @"
<html><body style='font-family:Segoe UI,Arial,sans-serif;font-size:13px;'>
<h2>Room Mailbox External Booking Audit</h2>
<p>Run: $(Get-Date -Format 'dd MMM yyyy HH:mm') &nbsp;|&nbsp;
   <strong>$($report.Count)</strong> rooms total &nbsp;|&nbsp;
   <strong style='color:#c0392b'>$($blocked.Count) blocking external messages</strong></p>
<table border='1' cellpadding='6' cellspacing='0' style='border-collapse:collapse;width:100%;'>
  <thead style='background:#343a40;color:#fff;'>
    <tr>
      <th>Display Name</th><th>Email Address</th><th>External Booking</th>
      <th>Delete Comments</th><th>Delete Subject</th>
      <th>Add Organiser to Subject</th><th>Remove Private</th>
    </tr>
  </thead>
  <tbody>
    $($rows -join "`n")
  </tbody>
</table>
<p style='color:#888;font-size:11px;'>Green = external booking enabled &nbsp;|&nbsp; Red = blocked</p>
</body></html>
"@

Send-MailMessage `
    -To         'you@yourdomain.com' `
    -From       'it-reports@yourdomain.com' `
    -Subject    "Room Mailbox External Booking Audit - $(Get-Date -Format 'dd MMM yyyy')" `
    -Body       $html `
    -BodyAsHtml `
    -SmtpServer '10.x.x.x'

Write-Host "Report sent. $($blocked.Count) of $($report.Count) rooms are blocking external messages." -ForegroundColor Yellow
```

The report sorts blocked rooms to the top and colour-codes them red, so you can see the problem rooms immediately without reading through the whole list.

## What each setting does

| Setting | Recommended | Why |
|---|---|---|
| `ProcessExternalMeetingMessages` | `$true` | Accepts invites from outside the org |
| `DeleteComments` | `$false` | Preserves the meeting body so the room calendar shows context |
| `DeleteSubject` | `$false` | Keeps the original meeting title |
| `AddOrganizerToSubject` | `$false` | Stops the subject being replaced with the organiser's name |
| `OrganizerInfo` | `$true` | Includes organiser details in the acceptance |
| `DeleteAttachments` | `$true` | Strips attachments from accepted invites — keeps the mailbox clean |
| `RemovePrivateProperty` | `$false` | Respects the Private flag on sensitive meetings |

## Step 2: fix every room in one run

Once you've seen the audit report, run this to apply the correct settings across
all room mailboxes at once. Each room prints `OK` or `FAIL` as it processes, and
you get a summary at the end.

```powershell
### Apply standard calendar processing settings to all room mailboxes
$rooms = Get-Mailbox -RecipientTypeDetails RoomMailbox -ResultSize Unlimited

$total   = $rooms.Count
$success = 0
$failed  = @()

Write-Host "Applying calendar processing settings to $total room mailboxes..." -ForegroundColor Cyan

foreach ($room in $rooms) {
    try {
        Set-CalendarProcessing -Identity $room.Identity `
            -AddOrganizerToSubject          $false `
            -OrganizerInfo                  $true  `
            -DeleteAttachments              $true  `
            -DeleteComments                 $false `
            -DeleteSubject                  $false `
            -RemovePrivateProperty          $false `
            -ProcessExternalMeetingMessages $true  `
            -ErrorAction Stop

        Write-Host "  OK  $($room.PrimarySmtpAddress)" -ForegroundColor Green
        $success++
    }
    catch {
        Write-Host "  FAIL $($room.PrimarySmtpAddress) -- $($_.Exception.Message)" -ForegroundColor Red
        $failed += $room.PrimarySmtpAddress
    }
}

Write-Host "`nDone. $success of $total updated successfully." -ForegroundColor Cyan

if ($failed.Count -gt 0) {
    Write-Host "`nFailed rooms:" -ForegroundColor Red
    $failed | ForEach-Object { Write-Host "  $_" -ForegroundColor Red }
}
```

After it finishes, run the audit script again to confirm everything is green.

## Prerequisites

Both scripts need an active Exchange Online session:

```powershell
Connect-ExchangeOnline -UserPrincipalName you@yourdomain.com
```

!!! note "Tenant size"
    On large tenants with many room mailboxes, the audit script can take a few
    minutes — `Get-CalendarProcessing` makes one API call per room. The set
    script is the same. Let it run; don't interrupt it mid-way or you'll end up
    with some rooms updated and some not.
