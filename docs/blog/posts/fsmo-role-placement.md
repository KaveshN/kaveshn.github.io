---
date: 2026-07-11
authors:
  - kavesh
categories:
  - Identity
  - Infrastructure
slug: fsmo-role-placement
description: >-
  Five FSMO roles. Most admins can't name them all. All of them matter when
  something goes wrong. A practical guide to placement, transferring vs seizing,
  and the decommissioning trap that takes down environments.
---

# FSMO role placement: the underrated decision that breaks AD environments

A few years ago I was called into an environment where logins were failing
intermittently across half the company. Users in the head office worked fine.
Users in three regional sites were getting locked out or seeing password change
errors. The on-call engineer had restarted everything that could be restarted
and was beginning to panic.

The cause turned out to be embarrassingly simple. Six months earlier, someone
had decommissioned an old domain controller without realising it held the PDC
emulator role. Time synchronisation drifted. Account lockout processing got
confused. Password changes propagated unpredictably. The whole environment had
been slowly degrading and nobody had noticed because nothing failed
dramatically.

This is FSMO roles in a nutshell: they don't matter until they do, and when
they do, they matter very badly.

<!-- more -->

## What the five roles actually do

Active Directory uses a multi-master replication model — most operations can
happen on any domain controller and replicate out. But five specific operations
require a single authoritative source. These are the Flexible Single Master
Operations roles.

**Two roles exist once per forest:**

**Schema Master.** Owns changes to the AD schema — adding new attributes,
classes, or modifying existing ones. Schema changes are rare but critical. If
the Schema Master is unavailable, you cannot make schema changes. Everything
else keeps working.

**Domain Naming Master.** Owns the addition and removal of domains from the
forest. Also rare to use, also non-critical for day-to-day operations.

**Three roles exist once per domain:**

**RID Master.** Hands out pools of Relative Identifiers to other DCs, which
they use to construct SIDs for new security principals. If the RID Master is
down long enough, DCs exhaust their pools and cannot create new objects. "Long
enough" is usually weeks, not hours.

**Infrastructure Master.** Maintains references to objects in other domains
within the same forest. Important in multi-domain forests with cross-domain
group memberships. Has a famous gotcha: in a multi-domain forest, the
Infrastructure Master should **not** be on a global catalog server — except
when every DC in the domain is a GC, which is increasingly the default.

**PDC Emulator.** This is the one that matters most. It's the authoritative
time source for the domain, the focal point for password changes, and the
processor for account lockouts. If the PDC emulator misbehaves, you'll see
strange, intermittent, hard-to-diagnose problems across the entire domain.

## Where to put the roles

The default placement is all five roles on the first DC you install. Fine for
a tiny environment; not fine for anything larger.

Standard guidance:

- **Forest-level roles** (Schema Master, Domain Naming Master) belong in the
  forest root domain on a well-connected, stable DC. They're rarely used, so
  performance doesn't matter much, but they need to be available when needed.
- **PDC Emulator** should be on a beefy DC with excellent network connectivity
  to the rest of the domain. Because of its time and password roles, it
  generates more traffic than other DCs.
- **RID Master and Infrastructure Master** should be on reliable, well-connected
  DCs, but their placement is less critical than the PDC emulator.

!!! warning "Two rules that apply to all of them"
    Never place all FSMO roles on the same DC in a multi-DC environment. You
    want roles distributed so the failure of any single DC takes out at most
    one or two roles, not all five.

    **Document where they live.** Run `netdom query fsmo` and write it down.
    Put it somewhere your successors will find it. The amount of grief caused
    by undocumented FSMO placement is enormous.

## Transferring vs seizing

**Transferring** is the polite move. The current role holder is alive and
reachable. Both DCs cooperate, the role moves cleanly, and there's no risk of
inconsistency:

```powershell
Move-ADDirectoryServerOperationMasterRole `
    -Identity "DC02" `
    -OperationMasterRole PDCEmulator
```

**Seizing** is the emergency move. The current role holder is dead or
unreachable. You forcibly assign the role to another DC.

!!! danger "Seizing is destructive"
    If you seize a role because a DC seemed down but was actually just a
    network issue, and that DC comes back online, you'll have two DCs both
    believing they hold the same role — "duelling FSMO" — which corrupts the
    directory. Seized roles are gone from the original DC permanently.

    Only seize when you are certain the original DC will **never** return.

```powershell
Move-ADDirectoryServerOperationMasterRole `
    -Identity "DC02" `
    -OperationMasterRole PDCEmulator `
    -Force
```

The `-Force` flag is what turns a transfer into a seizure. Use it carefully.

## The decommissioning trap

The story I opened with — admin decommissions a DC without realising it held
FSMO roles — is so common it deserves its own warning.

Before you demote or remove **any** domain controller, run `netdom query fsmo`
and confirm what roles that DC holds. If it holds any, transfer them first.
Then demote.

Newer Windows Server versions handle this more gracefully than old ones, but
you should not rely on the tooling to save you. Build the FSMO check into your
decommissioning runbook and follow it every time.

## The discipline that prevents most FSMO problems

Keep role placement documented somewhere your team actually looks at — pinned to
a wiki or architecture diagram, wherever your team's eyes go.

Review placement during any architectural change: adding sites, decommissioning
DCs, expanding domains. Check whether the placement still makes sense for the
new topology.

Monitor for role availability. Most monitoring tools can alert when an FSMO
role holder is unreachable. If you don't have a SIEM, even a simple scheduled
PowerShell script that emails on failure is better than nothing.

Practice role transfers in your test environment. The first time you do a
transfer should not be at 2am with a real outage in progress.

The five FSMO roles are not glamorous. They're plumbing. But like plumbing, the
time to think about them is before something leaks.
