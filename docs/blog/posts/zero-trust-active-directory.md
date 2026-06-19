---
date: 2026-06-20
authors:
  - kavesh
categories:
  - Identity
  - Infrastructure
slug: zero-trust-active-directory
description: >-
  The perimeter is gone. Your domain controllers haven't caught up yet. How
  Zero Trust principles apply to an on-premises Active Directory environment —
  and where most organisations are actually stuck.
---

# Zero Trust meets Active Directory: why your on-prem identity strategy needs a rethink

For roughly two decades, Active Directory has been the quiet workhorse of
enterprise IT. It authenticated your users, applied your group policies, and
managed your service accounts. And for most of that time, it operated on a
comfortable assumption: if you were inside the corporate network, you were
probably trustworthy.

That assumption is dead. It died somewhere between the rise of remote work, the
explosion of SaaS applications, and the steady drumbeat of ransomware attacks
that begin with one phished credential and end with a domain admin token.

<!-- more -->

## The death of the network perimeter

Traditional AD security was built on castle-and-moat thinking. The corporate
network was the castle. The firewall was the moat. Anything inside the moat —
workstations, servers, domain controllers — was assumed friendly.

This worked when work happened in offices, applications lived on internal
servers, and users carried company laptops on the corporate LAN. None of those
assumptions hold anymore. Your users are at home, in coffee shops, on airline
Wi-Fi. Your applications live in Azure, AWS, and a hundred SaaS providers. And
your attackers are very good at getting inside the moat — they no longer need
to breach it from outside.

Zero Trust flips the model. Instead of trusting based on network location, it
trusts based on continuously verified identity, device posture, and contextual
signals. The mantra is "never trust, always verify."

## What this actually means for AD

Zero Trust isn't a product you buy. It's a set of design principles, and
applying them to an AD environment looks like this:

**Identity becomes the new perimeter.** Every access decision — to a file
share, an application, a server — should be authenticated and authorised based
on who the user is and what state they're in, not just whether they sit on a
particular subnet.

**Least privilege becomes non-negotiable.** Standing administrative access is
the single biggest gift you can give an attacker. The Domain Admins group
should be nearly empty, with privileged access granted just-in-time through
tools like Microsoft's Privileged Access Management or third-party PAM
solutions.

**Lateral movement gets harder.** Once an attacker compromises one account,
they typically pivot to others — collecting credentials, hopping between
machines, escalating privileges. Segmenting AD into tiers (Tier 0 for domain
controllers and identity infrastructure, Tier 1 for servers, Tier 2 for
workstations) and preventing credential exposure across tiers is foundational.

**Authentication is layered, not single-factor.** Passwords alone are not
credentials. MFA, conditional access policies, and increasingly passwordless
authentication using Windows Hello for Business or FIDO2 keys should be the
default.

**Every action is logged, monitored, and analysed.** You cannot detect what
you do not see. Advanced audit policies, event forwarding to a SIEM, and
behavioural analytics tools like Microsoft Defender for Identity are no longer
optional for organisations of any size.

## Where most organisations are stuck

Here's the uncomfortable truth: the AD itself is usually fine. It's the
surrounding habits that are broken.

There are too many people in privileged groups — often inherited from
migrations a decade ago and never cleaned up. Service accounts have domain
admin rights for reasons nobody can explain. Group policies have accreted like
geological strata, with no one quite sure what's still doing anything useful.
Audit logging is enabled but goes nowhere. MFA is deployed but with so many
exceptions that it's functionally optional.

You don't fix this by ripping out AD and starting over. You fix it
incrementally, with discipline. Start with an audit of privileged group
memberships. Document your tier model, even if you have to invent it. Get MFA
truly universal. Stand up proper logging. Then, and only then, start thinking
about more sophisticated controls.

## The hybrid reality

Most organisations aren't choosing between on-prem AD and cloud identity.
They're running both. Azure AD Connect, pass-through authentication, federation
— these are no longer migration stepping stones but permanent architectural
realities. Your Zero Trust strategy has to account for identity living in two
places simultaneously.

This is actually an opportunity. Cloud identity platforms like Microsoft Entra
ID offer Zero Trust capabilities — conditional access, risk-based
authentication, identity protection — that are difficult or impossible to
replicate purely on-prem. The right move for most organisations is to make
Entra ID the centre of identity decision-making while keeping AD for the
workloads that genuinely need it.

## Starting where you are

!!! tip "One decision at a time"
    Zero Trust isn't a project with an end date. It's a posture, and you adopt
    it one decision at a time. The next account you provision, provision with
    least privilege. The next GPO you write, scope it tightly. The next
    administrative task, do it from a properly hardened PAW — not your
    daily-driver laptop.

Your AD environment will not become a Zero Trust shop on a Tuesday afternoon.
But it can become slightly better every week, and over a year that adds up to
a transformation. The attackers, after all, are not slowing down.
