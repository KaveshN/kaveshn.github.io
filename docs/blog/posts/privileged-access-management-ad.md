---
date: 2026-06-27
authors:
  - kavesh
categories:
  - Identity
  - PowerShell
tags:
  - Active Directory
  - Security
  - Privileged Access
slug: privileged-access-management-ad
description: >-
  Standing admin rights are the gift that keeps on giving — to attackers.
  How time-based group memberships in AD DS work, why they matter, and how to
  start deploying PAM today.
---

# Privileged Access Management in AD DS: stopping the attack path before it starts

Let me describe an attack that has played out, in some variant, in nearly
every major breach I've read about over the last decade.

A user clicks a phishing link. Malware lands on their laptop. The malware
harvests credentials cached in memory — not the user's own password,
necessarily, but the hash of an admin account that ran a maintenance task on
that machine last week. The attacker pivots laterally using that hash, lands
on a server, finds another stored credential, escalates again. Within hours,
they hold a Domain Admin token.

Everything in that chain depends on one thing: somewhere in the environment,
highly privileged credentials were sitting around waiting to be stolen.

<!-- more -->

## The fundamental flaw of standing privilege

In most environments, administrative rights work like this: you decide a
person needs to be a Domain Admin, you add them to the Domain Admins group,
and they stay there — forever, usually. Until they leave the company or change
roles, and even then often not for months.

This is operationally convenient and security-catastrophic. It means that at
any given moment, dozens of accounts have full directory rights. Each of those
accounts logs into workstations, servers, and tools, leaving credential
fragments in memory. Each of those accounts represents a path to total
compromise.

The standard advice — "use fewer admin accounts, use them sparingly" — is
correct but unhelpful. Humans don't reliably restrain themselves under
operational pressure. What's needed is a system that enforces restraint
mechanically.

## What PAM actually does

The core idea is simple: nobody is a member of a privileged group all the
time. When an administrator needs to perform a privileged task, they request
elevation. If approved, they're added to the appropriate group for a limited
time — say, four hours. When the clock runs out, they're automatically removed.

This single change disrupts the attack chain at multiple points. The attacker
who steals a credential mostly steals a credential to a non-privileged account.
The hash sitting in memory on that server isn't usable for domain takeover
because the account it belongs to isn't currently privileged.

Microsoft's implementation uses a dedicated "bastion" forest with a one-way
trust to your production forest. When elevation is needed, group membership is
added as a "shadow principal" in production with a time-to-live. The
technology behind this is **time-based group memberships**, available from
Windows Server 2016 functional level onwards.

## Time-based group memberships in practice

The PowerShell cmdlets make this surprisingly approachable:

```powershell
Add-ADGroupMember -Identity "Domain Admins" `
    -Members "jsmith" `
    -MemberTimeToLive (New-TimeSpan -Hours 4)
```

That's it. Four hours later, the membership evaporates. No human has to
remember to revoke it. No ticket has to be tracked to closure. The directory
itself enforces the boundary.

You can build this into approval workflows using whatever tooling you have — a
help desk system, a custom portal, or a third-party PAM tool. The mechanism in
AD is the same. What varies is how requests are made and approved upstream.

## The cultural problem

The technical implementation is the easy part. The hard part is convincing
your organisation that this is necessary.

Administrators don't like friction. Tickets to elevate, approval workflows,
time limits — these all slow people down. The pushback I hear most often is
"we trust our people, why are we making their lives harder?"

The answer is that the controls aren't about distrust. They're about damage
limitation. You trust your administrators. You don't trust the malware on the
laptop of someone who used that administrator's credential six months ago.

!!! tip "Phase it in, starting with the most powerful groups"
    Start with Enterprise Admins, Schema Admins, Domain Admins. Most
    organisations rarely need anyone in these groups for routine work, so
    removing standing membership has low operational cost. Move outward from
    there to delegated admin roles as the workflow matures.

## Where to start today

If you're running AD DS 2016 or later, you already have the building blocks.
Time-based group memberships work today. You don't need to license anything
additional or build a separate forest to start.

The smallest meaningful step: identify everyone currently in Domain Admins. For
each of them, ask whether they need permanent membership. For almost all of
them, the answer is no. Move them to a process where they request elevation
when needed, with a TTL on the membership.

Then keep going. Schema Admins, Enterprise Admins, Account Operators, Backup
Operators. One group at a time. By the time you've worked through the
privileged groups, you'll have a different organisation — and an attacker
landing on a compromised laptop will find a much less productive trip ahead of
them.
