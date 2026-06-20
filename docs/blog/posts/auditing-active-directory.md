---
date: 2026-08-15
authors:
  - kavesh
categories:
  - Identity
  - PowerShell
tags:
  - Active Directory
  - Auditing
  - Security
  - Monitoring
slug: auditing-active-directory
description: >-
  You can't detect what you don't see. A layered guide to AD auditing — from
  enabling Advanced Audit Policies and centralising logs, through to what to
  actually alert on and what Microsoft Defender for Identity adds on top.
---

# Auditing Active Directory: from Event Viewer to Microsoft Defender for Identity

Every breach report I read tells the same story in different words: the
attackers were in the environment for weeks or months before anyone noticed.
The signs were there. The logs recorded them. But nobody was looking, or the
logging wasn't capturing what mattered, or the volume was too high to extract
signal from noise.

Auditing Active Directory is one of those areas where the default setup is
mostly useless and the proper setup is more involved than people expect. But you
can build it up in stages, starting with what's free and built-in, and
graduating to genuinely sophisticated detection.

<!-- more -->

## The starting point: Event Viewer

Every domain controller is generating events constantly about what's happening
in the directory. Most of these events are recorded and then never looked at,
because there's no good way to look at them at scale.

The relevant logs on a DC:

- **Security log** — authentication events, account management, directory
  changes, and a great deal else if you've enabled the right audit policies.
- **Directory Service log** — replication issues, schema operations, and
  AD-internal events.
- **DNS Server log** — DNS activity, relevant because so much of AD depends on
  DNS being healthy.

Event Viewer is fundamentally an interactive tool for looking at one machine at
a time. It is not a monitoring solution. You need to move logs somewhere useful.

## Advanced Audit Policies — the part you must enable

Here's the part that catches people: the default audit settings in Active
Directory do not record most of what you'd want to know. To get useful auditing,
you need to enable the **Advanced Audit Policies** via Group Policy:

*Computer Configuration → Policies → Windows Settings → Security Settings
→ Advanced Audit Policy Configuration*

Also enable: *"Audit: Force audit policy subcategory settings to override audit
policy category settings"* — otherwise the granular settings won't take effect.

The categories worth enabling on domain controllers:

| Category | What it captures |
|---|---|
| Account Logon → Kerberos Authentication Service | Who's getting TGTs and from where |
| Account Logon → Kerberos Service Ticket Operations | Service tickets issued (Kerberoasting detection) |
| Account Management → Security Group Management | Group membership changes |
| Account Management → User Account Management | Account creation, modification, deletion |
| DS Access → Directory Service Changes | Attribute-level before/after on directory object changes |
| Logon/Logoff → Special Logon | Use of sensitive privileges on logon |
| Policy Change → Audit Policy Change | Changes to the audit policy itself |

!!! note "Enabling all these categories will significantly increase event volume"
    A busy DC can produce tens of thousands of audit events per minute. This is
    a feature, not a bug — but it means you can't reasonably hope to read these
    logs interactively. You need to collect them somewhere and search them.

## Centralising the logs

Logs sitting on individual DCs have several problems: they roll over and
disappear, they can be tampered with by an attacker who gains DC access, and
they can only be searched one machine at a time.

**Windows Event Forwarding** is free and built in. Configure source machines
(DCs) to forward specific events to a collector using a subscription. This
works adequately at small scale.

At larger scale, forward events into a proper SIEM — Microsoft Sentinel, Splunk,
Elastic, QRadar. The SIEM gives you searchable storage, dashboards, alerts on
specific patterns, and correlation across multiple data sources.

!!! warning "Protect your log store"
    Make sure collected logs are not writable from compromised hosts. An attacker
    who breaks into a DC should not be able to erase the evidence from your
    central log store.

## What to alert on

A SIEM full of events that nobody triages is hardly better than no SIEM. High-
value alerts worth building first:

**Membership changes to highly privileged groups** — Domain Admins, Enterprise
Admins, Schema Admins, Account Operators. Any change should be alerted on, with
confirmation the change was authorised.

**Service account behaviour anomalies** — service accounts interactively logging
in (they shouldn't), authenticating from unexpected sources, password changes.

**Disabled-account authentication attempts** — probing leftover accounts is
common early-reconnaissance behaviour.

**Replication failures between DCs** — can indicate hardware issues but can also
indicate an attacker disrupting normal operation.

**Kerberos pre-authentication failures in patterns** — concentrated failures
suggest Kerberoasting or password spraying.

**Account creation outside normal working hours**, particularly in privileged OUs.

**Object permission changes on sensitive directory objects** — granting yourself
rights to a privileged group is a classic persistence move.

## Microsoft Defender for Identity

For organisations licensed for it — included in Microsoft 365 E5 — Defender for
Identity is the modern answer to AD-focused detection.

It works by installing lightweight sensors on your domain controllers. These
sensors monitor network traffic to and from the DC, watch the security event
log, and feed both into a cloud analytics engine trained to recognise the
behaviours of common attack tools: pass-the-hash, pass-the-ticket,
Kerberoasting, DCSync, Golden Tickets, Silver Tickets, suspicious replication
requests, brute-force patterns.

The detection quality is genuinely good. I've seen Defender for Identity detect
Mimikatz usage in a domain within minutes of the tool running.

Beyond detection:

- **Lateral movement path mapping** — given a compromised user, where could an
  attacker reach next, given credentials currently cached around the
  environment?
- **Defender XDR integration** — pivot from a directory alert to endpoint
  telemetry to email content in a single investigation.
- **Honeytoken accounts** — create a decoy account that should never be used.
  Any authentication attempt against it triggers an alert. Attackers reliably
  trip honeytokens during reconnaissance.

## Azure AD Connect Health

If you're running hybrid identity, Azure AD Connect Health monitors the health
of your synchronisation infrastructure. This is operational monitoring rather
than security monitoring, but it overlaps: a failing AD Connect can mask
security-relevant changes in your environment.

Deploy agents on the relevant servers, point them at your Entra tenant, and view
the resulting dashboards in the Entra admin centre. Low operational overhead.

## A practical roadmap

1. **Enable Advanced Audit Policies on all domain controllers.** Free and
   produces dramatically richer logs than the default.
2. **Centralise logs somewhere** — WEF to a collector at minimum, a SIEM if
   you have one. Protect the central store against tampering.
3. **Build a small set of high-value alerts** and make sure someone triages
   them. Start with privileged group changes and service account anomalies.
4. **Deploy Microsoft Defender for Identity** if you have the licensing. The
   detection capability it adds is hard to replicate any other way.
5. **Iterate** — tune false positives, add detections, practise response.

Detection alone doesn't stop attacks, but it shortens the window between
intrusion and discovery from months to hours. That difference is often the
difference between a contained incident and a public breach.
