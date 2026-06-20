---
date: 2026-08-01
authors:
  - kavesh
categories:
  - Identity
  - PowerShell
tags:
  - Active Directory
  - Security
  - Authentication
slug: pass-the-hash-defence
description: >-
  Protected Users, Authentication Policies, and Authentication Policy Silos —
  the three Active Directory features that would have stopped most of the
  breaches you've read about. What they do, how they layer, and the order
  to deploy them.
---

# Pass-the-Hash defence: Protected Users, Authentication Policies, and Authentication Policy Silos

If you've followed cybersecurity news at all over the last decade, you've seen
variations of the same headline play out repeatedly. Major company hit by
ransomware. Attackers moved laterally for weeks before triggering encryption.
Initial compromise was a single phished credential; final compromise was Domain
Admin.

The middle part of that story — the lateral movement — is almost always powered
by credential theft from compromised machines. The most famous technique is
Pass-the-Hash, but it has cousins: Pass-the-Ticket, Overpass-the-Hash, Silver
Tickets, Golden Tickets. They all work on the same basic principle: Windows
caches authentication material in memory to provide single sign-on, and an
attacker with sufficient privilege can extract that material and reuse it
elsewhere.

The good news: Microsoft has built three increasingly powerful features into
Active Directory specifically to defeat these attacks. Almost nobody deploys
them properly.

<!-- more -->

## How the attack actually works

When a user logs into a Windows machine, Windows stores the authentication
material — the NTLM hash, the Kerberos TGT, or both — in the memory of the
LSASS process. This is how the user can access network resources later without
being prompted again. Single sign-on requires the credential to be available
somewhere.

If an attacker gains administrative access to that machine, they can extract
those credentials from LSASS. Tools like Mimikatz have made this trivially
easy. The extracted hash can then be used to authenticate to other systems as
the user, without ever knowing the actual password. The hash *is* the
credential.

The problem compounds because high-privilege accounts log into many machines.
A Domain Admin who runs a maintenance task on a server leaves their credentials
in that server's LSASS memory. An attacker landing on that server can steal
those credentials and become Domain Admin. The blast radius of one compromised
workstation can be enormous.

## Defence 1: the Protected Users group

The Protected Users security group was introduced with Windows Server 2012 R2.
It's a built-in group in every modern AD environment. Add an account to it, and
a number of additional security restrictions kick in automatically.

Accounts in Protected Users:

- **Cannot authenticate using NTLM.** They must use Kerberos. This eliminates
  the entire class of NTLM-based pass-the-hash attacks against those accounts.
- **Cannot use DES or RC4 in Kerberos.** Only AES is permitted.
- **Cannot delegate credentials,** including via CredSSP (what RDP uses for
  credential forwarding).
- **Cannot have a long-lived TGT.** Lifetime is capped at four hours and
  cannot be renewed.
- **Cannot have credentials cached for offline logon.** The user must
  authenticate to a DC every time.

The effect: an account in Protected Users is dramatically less useful to an
attacker who manages to steal anything from a machine the account has touched.

!!! warning "Test before deploying broadly"
    The same restrictions can break legitimate workflows — applications that
    rely on NTLM, environments that need delegation, users who travel and depend
    on cached credentials. The right starting point is your most sensitive
    accounts only: Domain Admins, Enterprise Admins, Schema Admins, and
    privileged service accounts. Regular user accounts should not be in this
    group.

## Defence 2: Authentication Policies

Authentication Policies let you put specific restrictions on individual
accounts: where they can log in from, what kinds of access they can perform,
and how long their tickets last.

Typical use case: ensuring Domain Admin accounts can only authenticate from a
specific set of Privileged Access Workstations. You create an Authentication
Policy that says "this user's TGT can only be issued if the request comes from
one of these machines." Anyone trying to log in as that admin from a regular
workstation fails authentication outright.

Authentication Policies are deployed via Active Directory Administrative Center
→ Authentication. Create the policy, configure the constraints, then apply it
to specific accounts. This is fine-grained — one policy can target one account
if needed.

Requires Windows Server 2012 R2 or later domain functional level, and Kerberos
for the controlled accounts — which dovetails neatly with Protected Users
membership.

## Defence 3: Authentication Policy Silos

Silos group Authentication Policies together. A silo is a container that holds
users, computers, and service accounts that all share authentication policies.
The canonical use case is implementing a tier model.

In Microsoft's enterprise access model:

- **Tier 0** — the most sensitive identities: Domain Admins, Enterprise Admins,
  credentials that can take over the directory.
- **Tier 1** — server administrators and service accounts for enterprise
  applications.
- **Tier 2** — user workstations and end-user accounts.

The rule: credentials never flow downward. A Tier 0 admin never logs into a
Tier 2 workstation. A Tier 1 server admin never authenticates from a Tier 2
endpoint.

You implement this with Authentication Policy Silos. Create three silos, assign
accounts to them, and configure the silo's authentication policy so that Tier 0
accounts can only authenticate from machines that are also in the Tier 0 silo.
Anything attempting authentication from outside that silo fails.

!!! note "This is significantly more effort than Protected Users"
    You have to define your tier boundaries clearly, classify every account and
    machine, ensure your operational processes work within the tier constraints,
    and accept the friction of administrators using dedicated PAWs for admin
    work. The payoff is the architectural pattern that actually defeats lateral
    movement at scale.

## Microsoft LAPS as a complement

While not strictly part of the same feature family, LAPS solves a related
problem: shared local administrator passwords.

In many environments, every workstation has the same local Administrator
password — set during imaging and never changed. An attacker who extracts that
password from one machine has it for every machine.

LAPS randomises the local administrator password on every machine and stores
the current password in Active Directory, accessible only to authorised
accounts. Deploying LAPS is well-documented and relatively low-effort. There's
an older "Microsoft LAPS" and a newer "Windows LAPS" built into recent Windows
versions. Either is dramatically better than shared local admin passwords.

## The deployment order

Each control is useful on its own. Together, they're a defensive system:

1. **Start with LAPS.** Easiest win, lowest deployment cost, broadest immediate
   benefit.
2. **Add highest-value accounts to Protected Users.** Test carefully — there
   will be applications that break. Address those, then expand membership
   cautiously.
3. **Introduce Authentication Policies** for your most sensitive accounts,
   restricting them to specific privileged access workstations.
4. **Formalise your tier model with Authentication Policy Silos** when the
   cultural and operational groundwork is in place.

This is a journey, not a weekend project. But every step forward closes a class
of attack. Take the first step this quarter.
