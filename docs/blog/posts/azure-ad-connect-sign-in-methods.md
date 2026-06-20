---
date: 2026-08-08
authors:
  - kavesh
categories:
  - Identity
  - Infrastructure
tags:
  - Entra ID
  - Active Directory
  - Hybrid Identity
  - Authentication
slug: azure-ad-connect-sign-in-methods
description: >-
  Password Hash Sync, Pass-Through Authentication, and Federation — the three
  ways to bridge on-premises AD to Entra ID. What each one does, how it fails,
  and a decision framework for choosing the right one.
---

# Azure AD Connect sign-in methods compared: PHS, PTA, and federation

If you run an on-premises Active Directory environment and also use Microsoft
365, you've made — or somebody before you made — a choice about how
authentication flows between the two. There are three options. Each has
distinct security, availability, and operational properties. And the choice is
not trivially reversible once you've built out around it.

I keep meeting organisations who picked the wrong one years ago and are stuck
with it because changing means migrating thousands of users to new
authentication patterns. So let's think about this properly.

<!-- more -->

## What we're actually choosing between

When a user signs into a Microsoft 365 service, Entra ID needs to verify their
password. The user's authoritative password lives in your on-premises AD. The
three options describe how Entra ID gets satisfied that the user provided the
right one.

**Password Hash Synchronisation** sends a transformation of the on-premises
password hash up to Entra ID. When the user signs in, Entra ID verifies the
password locally against the synchronised hash. Your on-premises infrastructure
doesn't participate in the sign-in at all.

**Pass-Through Authentication** keeps verification on-premises. When the user
signs in, Entra ID forwards the credential to a lightweight agent running in
your environment, which validates it against AD and returns the result.

**Federation** routes authentication entirely through on-premises
infrastructure — typically AD FS. Entra ID redirects the user to your
federation server, which authenticates them and returns a SAML or
WS-Federation token.

## Password Hash Sync: the underestimated option

For years, PHS was treated as the inferior option — "too consumer-grade, not
enterprise enough." That reputation was always undeserved. Microsoft's official
guidance now positions PHS as the **default recommendation** for most
organisations.

The security properties are better than people initially assume. The value
stored in Entra ID is not the AD password hash and cannot be used to
authenticate to AD even if extracted. The transformation is cryptographically
strong. Entra ID never sees the cleartext password.

The operational properties are exceptional:

- Sign-in works whether your on-premises infrastructure is reachable or not.
  If your data centre catches fire, users can still access Microsoft 365.
- Sign-in latency is low — everything is in the Microsoft cloud.
- No extra servers to maintain beyond AD Connect itself.

A useful side benefit: PHS gives Entra ID Identity Protection enough data to
detect credential compromise. Entra ID compares user hashes against
known-compromised credential dumps — this protection doesn't work for federated
or PTA users without PHS also running.

!!! note "The reasons not to use PHS"
    If your organisation has a hard policy that authentication material must
    never leave your premises, PHS doesn't satisfy it. Some regulated industries
    take this view.

    If you need on-premises account state — disabled flags, password expiration,
    logon hour restrictions — enforced in real time during cloud sign-in, PHS by
    itself doesn't do this.

For most organisations without those constraints, PHS is the right answer.

## Pass-Through Authentication: the middle ground

PTA keeps passwords on-premises while avoiding the complexity of full
federation. A lightweight agent runs on a server in your environment (run at
least three for redundancy). When a sign-in arrives, Entra ID encrypts the
credential and posts it to an Azure-managed queue. Your agent picks it up,
validates against AD, and returns success or failure.

The cleartext password is exposed in memory on the agent for the duration of
validation, then discarded. The connection from agent to Azure is outbound only
— no inbound firewall ports needed.

This satisfies the "password material doesn't leave the premises" requirement
with relatively little operational overhead.

Trade-offs:

- Sign-in latency includes a round trip to your environment.
- Sign-in depends on your agents being reachable. This is why you run multiple
  agents in different parts of your environment.

!!! tip "Combine PTA with PHS as a fallback"
    PHS doesn't have to be the active authentication method — it can run in the
    background. If all PTA agents fail, you still have a working fallback. This
    combination is what Microsoft recommends for PTA deployments.

## Federation: the heavyweight option

Federation routes authentication through on-premises federation
infrastructure — typically AD FS. The user's browser is redirected to your
federation server, which authenticates them via AD and returns a signed token.
Entra ID accepts that token.

The reasons federation has historically been chosen:

- Strong support for advanced scenarios: smart cards, certificate-based
  authentication, custom MFA integrations.
- Full control over authentication policy and user experience.
- Compliance with regulations or contractual requirements that explicitly
  mandate on-premises authentication.

The cost is substantial. AD FS is a complete infrastructure: federation servers,
Web Application Proxy servers for external access, a configuration database
(typically SQL Server), certificates, DNS records, monitoring. The blast radius
of an AD FS outage is enormous — if AD FS is down, users cannot sign in to
anything that depends on Entra ID.

!!! danger "Federation is where the most public breaches have happened"
    The SolarWinds attackers used compromised AD FS infrastructure to forge SAML
    tokens — the "Golden SAML" attack — and impersonate any user in the
    federated domain. AD FS sits in a uniquely sensitive position and must be
    defended accordingly.

Microsoft's clear direction is to move organisations off federation where
possible. Most modern Entra capabilities — conditional access, Identity
Protection, passwordless authentication — work best or only with cloud-managed
authentication.

## Seamless SSO: the often-forgotten add-on

One feature worth mentioning: **Seamless SSO** can be layered on top of PHS or
PTA. It uses a domain-joined machine's existing Kerberos ticket to silently
authenticate the user to Entra ID when they're on the corporate network.

Seamless SSO is enabled in AD Connect, requires no additional infrastructure,
and dramatically improves the experience for domain-joined endpoints. If you've
chosen PHS or PTA and users are complaining that "the old AD FS setup felt
smoother," Seamless SSO is usually what's missing.

## A decision framework

Start with Password Hash Sync as the default. Add Seamless SSO for the
domain-joined experience. This gets you fast, resilient, low-maintenance cloud
sign-in with the best support for modern Entra security features.

If your organisation has a documented policy that authentication material must
not leave the premises, switch to Pass-Through Authentication. Keep PHS enabled
as a fallback. Deploy at least three PTA agents for redundancy.

Only choose Federation if you have a specific requirement — typically smart
cards where Windows Hello for Business isn't viable, certain third-party
identity provider integrations, or a regulatory mandate — that the other
options genuinely can't satisfy.

The wrong answer is the one I see most often: federation deployed years ago
because it seemed "more enterprise," now treated as an immovable foundation. It
moved. Microsoft moved. The right answer for most organisations has changed. If
you're running AD FS purely because it's there, it's worth asking whether it
should still be there.
