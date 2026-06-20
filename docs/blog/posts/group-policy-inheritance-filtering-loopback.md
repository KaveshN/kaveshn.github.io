---
date: 2026-07-25
authors:
  - kavesh
categories:
  - Infrastructure
  - PowerShell
tags:
  - Active Directory
  - Group Policy
  - Troubleshooting
slug: group-policy-inheritance-filtering-loopback
description: >-
  Most "Group Policy isn't working" tickets come down to one of three things:
  inheritance order, security or WMI filtering, and loopback processing. A
  clear explanation of each — plus the one diagnostic command that solves most
  problems in thirty seconds.
---

# Group Policy inheritance, filtering, and loopback: the three concepts that trip everyone up

Group Policy feels simple at first. You create a GPO, link it to an OU, set a
thing, and the thing happens on the machines in that OU. Everyone follows along
for about twenty minutes. Then someone asks "but what if a setting is
configured here AND here?" and "what if I want this to apply to laptops but
not desktops?" and "what if a user logs into a server — which settings apply?"
and suddenly nobody can answer reliably.

The good news is that all of Group Policy's complexity reduces to three
concepts. The bad news is that all three are subtle in ways that bite when you
stop paying attention.

<!-- more -->

## Concept 1: inheritance and the LSDOU order

Group Policy is applied based on where the object sits in the directory
hierarchy. The famous ordering is **LSDOU**: Local, Site, Domain, OU.

In practice: the local policy applies first, then GPOs linked to the AD site
the machine is in, then GPOs linked at the domain root, then GPOs linked to
OUs from the OU closest to the domain root down to the OU containing the
object. Later applications win, so lower OUs generally override higher ones.

When multiple GPOs are linked at the same level, they apply in link order —
and here's where most people get confused: **the GPO at the bottom of the link
order applies first**, the one at the top applies last. Since later wins, the
top-of-list GPO is the most authoritative at that level. This is the opposite
of what most people guess.

The **Resultant Set of Policy** is what actually applies. The fastest way to
debug anything: run this on the affected machine and open the HTML report:

```powershell
gpresult /h C:\Temp\gp-report.html /f
```

The report shows every setting, which GPO it came from, and — if it's being
overridden — which GPO won.

**Two exceptions that disrupt inheritance:**

**Enforced links.** Right-clicking a GPO link and setting Enforced makes it
override conflicting GPOs at lower levels. Useful for genuinely mandatory
settings (security baselines), but use sparingly — it makes the inheritance
order surprising.

**Block Inheritance.** An OU can have its inheritance blocked, causing GPOs
linked higher up to be ignored — unless those higher-up GPOs are Enforced.
A red flag when overused because it makes the policy environment fragmented
and hard to reason about.

## Concept 2: filtering — security and WMI

Linking a GPO to an OU applies it to everything in that OU. Filtering gives
you finer control.

**Security filtering** uses the GPO's Access Control List. By default,
"Authenticated Users" has Read and Apply Group Policy permissions. Change the
security filter to only include a specific group — say, "Marketing Laptops" —
and only members of that group will have the GPO applied, even if the link is
on a broader OU.

!!! warning "The MS16-072 gotcha"
    Microsoft's MS16-072 update requires GPOs to have at least Read access for
    the computer account to be evaluated and applied, even if the Apply
    permission is filtered to a user group. If you remove Authenticated Users
    entirely and only grant rights to a specific user group, computer-scoped
    settings can stop applying silently.

    **Fix:** add "Domain Computers" with Read permission (not Apply) to any GPO
    that filters on user groups. This is the kind of detail that turns "I tested
    it and it works" into "it stopped working three months later for reasons no
    one understands."

**WMI filtering** is more powerful and more dangerous. You attach a WMI query
to the GPO; the GPO only applies to machines where the query returns true. The
classic example — Windows 10/11 workstations only:

```
SELECT Version FROM Win32_OperatingSystem 
WHERE Version LIKE "10.0%" AND ProductType = "1"
```

WMI filtering is flexible but slow. Every WMI filter is evaluated at every
Group Policy refresh. Use it where you genuinely need it, but prefer OU
placement and security filtering for performance-critical scenarios.

## Concept 3: loopback processing

This is the one nobody understands until they have to use it.

By default: user settings apply based on where the **user account** sits in
the directory; computer settings apply based on where the **computer account**
sits. So a marketing user logging into a marketing laptop gets marketing user
policies — clean and obvious.

The trouble starts when users log into shared computers in unusual scopes.
Imagine a kiosk computer in a reception area. Anyone in the company might log
into it. By default, each user brings their own user policies — desktop
wallpapers, drive mappings, software shortcuts — which is not what you want on
a kiosk. You want a consistent experience regardless of who logs in.

Loopback processing tells Group Policy to apply user policies based on the
**computer's** location, not the user's. Two modes:

**Merge mode:** applies the user's normal policies first, then overlays the
policies from the computer's location. Computer-scoped OU wins on conflicts.
The user still gets some of their normal experience, with computer-specific
overrides.

**Replace mode:** ignores the user's normal policies entirely and applies only
the user policies from the computer's location. Right for kiosks, terminal
servers, VDI, and other tightly controlled environments.

Loopback is enabled per-computer via a setting in Computer Configuration
→ Policies → Administrative Templates → System → Group Policy → "Configure
user Group Policy loopback processing mode."

!!! note "Classic legitimate uses"
    Kiosks, RDS session hosts, jump boxes, and lab computers. If you're
    using loopback for general-purpose user workstations, something is probably
    wrong with your OU design.

## The diagnostic shortcut

When a GPO setting isn't applying as expected, the investigation that takes
hours without `gpresult` takes thirty seconds with it. Look for:

- The GPO is linked but security or WMI filtering is excluding the target
- A different GPO at a different level is winning on the same setting
- The GPO is linked to the wrong scope (user setting on a computer OU or vice versa)
- The user or computer hasn't refreshed policies (`gpupdate /force` to test immediately)
- Loopback is enabled where it shouldn't be, or not enabled where it should be

`gpresult` will tell you which one of these is happening.

## Keeping GPOs manageable over time

Use descriptive names. "Marketing Laptops — Drive Mappings" is better than
"GPO-MKT-001." You will thank yourself in three years.

Keep GPOs small and focused. One GPO that does one thing is easier to reason
about than one mega-GPO that does fifteen unrelated things.

Document why a GPO exists. A short description on the GPO itself, or a wiki
entry, costs a minute and saves whoever inherits the environment days.

Don't enable settings just because you can. Every setting is technical debt.
Audit your GPOs periodically and disable anything that no longer serves a
purpose.

Group Policy is one of the most powerful tools in a Windows administrator's
kit. The complexity is in the corners — inheritance edge cases, filtering
subtleties, loopback. Once those three concepts click, the rest becomes
much less mysterious.
