---
date: 2026-07-18
authors:
  - kavesh
categories:
  - Identity
  - Infrastructure
tags:
  - PKI
  - Security
  - Architecture
slug: pki-design-two-tier-three-tier
description: >-
  Internal certificate authorities are forever. A practical guide to two-tier
  vs three-tier PKI hierarchies, offline root CAs, validity periods that won't
  trap you, and the CRL/AIA mistakes that cause mysterious outages months later.
---

# Designing a PKI that won't haunt you: two-tier vs three-tier hierarchies

The decision to deploy an internal Public Key Infrastructure is one of those
choices that seems modest at the time and turns out to be effectively
permanent. The root CA you stand up today will issue certificates that live in
your environment for years. The trust relationships you build will become
foundational. And the design mistakes you make at the beginning will follow you
long after the original architect has moved on.

I have watched organisations live with bad PKI designs for over a decade
because the cost of replacing them is so high. So this is one of the few areas
in IT where it genuinely pays to slow down, read the documentation properly,
and think carefully before you click "Install Role."

<!-- more -->

## Why you need PKI at all

Active Directory Certificate Services gives you the ability to issue, manage,
and revoke digital certificates inside your organisation. Those certificates
underpin a lot of modern infrastructure: secure LDAP, Windows Hello for
Business authentication, EFS file encryption, certificate-based VPN, IPsec,
Wi-Fi authentication via 802.1X, code signing, and the certificates that
secure traffic between internal services.

For external-facing services, buy certificates from a public CA. For internal
services — where the relying parties are all machines you control — an internal
PKI is dramatically cheaper, faster to issue, and more flexible. The cost is
that you become responsible for running it properly, and "properly" is a higher
bar than people expect.

## The three hierarchy models

**The single-tier model** has one CA that does everything — root of trust and
direct issuer of all certificates. Appropriate for labs and very small test
environments.

!!! danger "Do not use single-tier in production"
    If that single CA is compromised, every certificate it ever issued must be
    considered compromised, and you have no way to bring up a replacement trust
    hierarchy without retrofitting trust into every device.

**The two-tier model** has an offline root CA that issues certificates only to
subordinate issuing CAs, which then issue certificates to end entities. The
root sits powered off most of the time. The issuing CAs do the day-to-day
work. **This is the right design for most organisations.**

**The three-tier model** adds an intermediate policy CA layer between the root
and the issuing CAs. Each tier signs the next. The benefit is granular
separation: you can revoke an entire branch of the tree by revoking the policy
CA that signed it, without affecting the rest. Appropriate for very large
organisations or those issuing certificates across many distinct business units.

## Why offline roots matter

The root CA is the anchor of trust. Every certificate your PKI issues chains
back to it. If the root's private key is stolen, an attacker can issue
arbitrary certificates that everything in your environment will trust. The only
remediation is removing trust in the root from every device — a catastrophic,
multi-week emergency at any reasonable organisation size.

The defence is to keep the root offline almost all the time. It is not joined
to the domain. It has no network connectivity. Many organisations keep it as a
VM exported to encrypted storage, powered on in an isolated network only when
needed.

!!! note "How often does the root actually need to come online?"
    A few times a year — to renew its CRL, to issue certificates for new
    subordinate CAs, to handle key recovery. Standing it up for an afternoon
    twice a year is a small operational cost for a substantial security gain.

## Validity periods that won't trap you

A common pattern that works well: a root CA with a **20-year** certificate,
issuing or policy CAs with **10-year** certificates, end-entity certificates
with **one or two-year** validity and automated renewal.

Two traps to avoid:

**Making end-entity validity too short without renewal automation.** If
certificates expire annually and you issue each one manually, you'll spend a
noticeable percentage of your operational capacity on certificate hygiene. Plan
for automated enrolment via Group Policy, Certificate Enrollment Web Services,
or NDES for non-domain-joined devices.

**Making root validity so long you forget to plan for renewal.** If your root
certificate expires in 18 years, what happens in year 16? Diary the root
renewal project now, not in year 17. It's a real project involving a new key
pair and propagation to every machine in the environment.

## CDP, AIA, and the things that get forgotten

Two pieces of CA configuration cause more outages than the rest of the PKI
combined: **CRL Distribution Points** and **Authority Information Access**
locations.

When a certificate is presented for validation, the relying party retrieves a
Certificate Revocation List from one of the URLs the certificate itself
advertises — its CDP. If those URLs are unreachable, validation fails.

Similarly, AIA URLs point relying parties to the issuer's certificate so they
can build a chain to the root.

!!! warning "The failure mode is delayed"
    Certificates seem to work fine immediately after issuance, then
    mysteriously fail months later when something tries to use them in a context
    where the CDP isn't reachable.

Standard practice: publish CDPs and AIAs to **HTTP locations** — not LDAP,
which only works inside the directory — at URLs reachable both internally and
externally if your certificates will ever be used outside the corporate network.
Plan this on day one. It costs almost nothing and saves substantial pain later.

## Disaster recovery, before you need it

The CA database — records of every certificate issued, keys for key archival if
enabled — needs to be backed up reliably. Test the recovery. Build a parallel
test CA, restore a backup to it, confirm the restored CA can issue certificates
that chain correctly.

```powershell
# Back up the CA database and private key
Backup-CARoleService -Path "D:\CABackup" -Password (Read-Host -AsSecureString)
```

The first time you do a real CA recovery should not be during an actual
disaster.

## A practical roadmap for most organisations

A two-tier hierarchy. One offline standalone root CA, kept in a safe or
encrypted storage, brought online a few times a year. One or two issuing CAs
joined to the domain, doing the day-to-day work. CRLs and AIAs published to
internal HTTP, with consideration for whether any certificates need external
validation.

Reasonable validity periods: 20 years on the root, 10 on the issuing CAs, one
or two years on most end-entity certificates with automated renewal via
autoenrolment for domain-joined devices.

A documented runbook for taking the root online, issuing new subordinate CAs,
renewing the root's own certificate, and disaster recovery — stored somewhere
your future colleagues will find it.

This is not exciting work. Nobody will applaud you for getting PKI right. But
a PKI that traps you in operational pain, or exposes you to a wholesale trust
compromise, is one of the worst architectural legacies you can leave behind.
