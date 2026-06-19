---
hide:
  - navigation
  - toc
---

# Kavesh Naidoo

I'm an IT guy at heart. I've spent years in the weeds of Active Directory,
firewalls, cloud infrastructure, and everything that keeps an organisation
running — and I genuinely enjoy it. This is where I write about the things I'm
figuring out: identity, security, PowerShell, and increasingly AI and how it
changes the way we work.

Just honest notes from someone who spends a lot of time at the keyboard.

[Browse all posts →](blog/index.md){ .md-button .md-button--primary }

---

## Latest writing

<div class="grid cards" markdown>

-   :material-shield-check:{ .lg .middle } &nbsp; __[PowerShell for AD hygiene: 10 scripts every admin should have](blog/posts/powershell-ad-hygiene-scripts.md)__

    ---

    Dormant accounts, password audits, privileged group reviews, replication
    health, and more — the scripts that push entropy back.

    `PowerShell` · `Identity`

-   :material-magnify-scan:{ .lg .middle } &nbsp; __[Auditing Active Directory: from Event Viewer to Defender for Identity](blog/posts/auditing-active-directory.md)__

    ---

    Advanced Audit Policies, centralising logs, what to alert on, and what
    Microsoft Defender for Identity adds on top.

    `PowerShell` · `Identity`

-   :material-cloud-sync:{ .lg .middle } &nbsp; __[Azure AD Connect sign-in methods compared: PHS, PTA, and federation](blog/posts/azure-ad-connect-sign-in-methods.md)__

    ---

    Three ways to bridge on-premises AD to Entra ID, how each one fails, and a
    decision framework for choosing the right one.

    `Identity` · `Infrastructure`

-   :material-lock-reset:{ .lg .middle } &nbsp; __[Pass-the-Hash defence: Protected Users, Authentication Policies, and Silos](blog/posts/pass-the-hash-defence.md)__

    ---

    The three AD features that would have stopped most of the breaches you've
    read about — and the order to deploy them.

    `PowerShell` · `Identity`

-   :material-file-document-multiple:{ .lg .middle } &nbsp; __[Group Policy inheritance, filtering, and loopback](blog/posts/group-policy-inheritance-filtering-loopback.md)__

    ---

    The three concepts behind every "Group Policy isn't working" ticket —
    plus the one diagnostic command that solves most problems in thirty seconds.

    `Infrastructure` · `PowerShell`

-   :material-certificate:{ .lg .middle } &nbsp; __[Designing a PKI that won't haunt you: two-tier vs three-tier](blog/posts/pki-design-two-tier-three-tier.md)__

    ---

    Offline root CAs, validity periods that won't trap you, and the CRL/AIA
    mistakes that cause mysterious outages months later.

    `Identity` · `Infrastructure`

-   :material-sitemap:{ .lg .middle } &nbsp; __[FSMO role placement: the underrated decision that breaks AD environments](blog/posts/fsmo-role-placement.md)__

    ---

    Five roles, what they actually do, where to put them, and the
    decommissioning trap that takes down environments.

    `Identity` · `Infrastructure`

-   :material-family-tree:{ .lg .middle } &nbsp; __[Single forest vs. multi-forest: a decision framework for AD architects](blog/posts/single-vs-multi-forest-ad.md)__

    ---

    Most multi-forest deployments shouldn't exist. A framework for when a
    second forest is genuinely justified — and what goes wrong when it isn't.

    `Identity` · `Infrastructure`

-   :material-timer-play:{ .lg .middle } &nbsp; __[Privileged Access Management in AD DS: stopping the attack path before it starts](blog/posts/privileged-access-management-ad.md)__

    ---

    Time-based group memberships, why standing privilege is security-catastrophic,
    and how to start deploying PAM today without a separate forest.

    `PowerShell` · `Identity`

-   :material-security-network:{ .lg .middle } &nbsp; __[Zero Trust meets Active Directory: why your on-prem identity strategy needs a rethink](blog/posts/zero-trust-active-directory.md)__

    ---

    The perimeter is gone. How Zero Trust principles apply to an on-premises
    AD environment — and where most organisations are actually stuck.

    `Identity` · `Infrastructure`

</div>

---

## Previously

<div class="grid cards" markdown>

-   :material-shield-key:{ .lg .middle } &nbsp; __[Which of your admins still don't have MFA?](blog/posts/find-admins-without-mfa-graph.md)__

    ---

    Find privileged accounts with no second factor — the modern Microsoft Graph
    replacement for the dead MSOnline approach.

    `PowerShell` · `Identity`

-   :material-certificate-outline:{ .lg .middle } &nbsp; __[Never get surprised by an expired app secret again](blog/posts/report-expiring-app-secrets-graph.md)__

    ---

    Scan every Entra app registration for expiring secrets and certificates —
    including the SAML signing certs everyone forgets.

    `PowerShell` · `Identity`

-   :material-key-change:{ .lg .middle } &nbsp; __[Rotating the krbtgt password without breaking Kerberos](blog/posts/rotate-krbtgt-password-safely.md)__

    ---

    The account that signs every Kerberos ticket in your domain — how to rotate
    it safely, and the double-reset that breaks everything.

    `PowerShell` · `Identity`

-   :material-laptop-account:{ .lg .middle } &nbsp; __[Finding the machines that slipped through LAPS](blog/posts/find-missing-laps-powershell.md)__

    ---

    Report on every Windows machine without a LAPS-managed local admin password
    — covering both Windows LAPS and the legacy attribute.

    `PowerShell` · `Identity`

-   :material-lock-alert:{ .lg .middle } &nbsp; __[Catching account lockouts before the helpdesk ticket](blog/posts/alert-locked-out-accounts-powershell.md)__

    ---

    Find locked-out AD accounts with Search-ADAccount, then trace the source
    machine via event 4740.

    `PowerShell` · `Identity`

-   :material-account-supervisor:{ .lg .middle } &nbsp; __[Get an email the moment someone joins Domain Admins](blog/posts/monitor-privileged-groups-powershell.md)__

    ---

    Near-real-time privileged group monitoring with a PowerShell state-file diff
    — no SIEM required.

    `PowerShell` · `Identity`

-   :material-server-network:{ .lg .middle } &nbsp; __[A domain controller health report you'll actually read](blog/posts/dc-health-report-powershell.md)__

    ---

    A colour-coded HTML health report across the whole forest — ping, DNS,
    uptime, disk, services, and DCDiag.

    `PowerShell` · `Infrastructure`

-   :material-account-key:{ .lg .middle } &nbsp; __[Assigning Entra ID access packages with PowerShell](blog/posts/assign-entra-access-packages-powershell.md)__

    ---

    Grant Entitlement Management access packages programmatically — the
    admin-add assignment request, end to end.

    `PowerShell` · `Identity`

-   :material-account-search:{ .lg .middle } &nbsp; __[Auditing Entra ID users with PowerShell](blog/posts/audit-entra-users-powershell.md)__

    ---

    List every user, flag the guests and dormant accounts, and hand security a
    CSV — in a few lines of Microsoft Graph.

    `PowerShell` · `Identity`

-   :material-console-line:{ .lg .middle } &nbsp; __[Connecting to Microsoft Graph with PowerShell](blog/posts/connect-microsoft-graph-powershell.md)__

    ---

    The foundation for every Entra automation: install the SDK, connect with the
    right scopes, delegated vs app-only.

    `PowerShell` · `Identity`

-   :material-shield-account:{ .lg .middle } &nbsp; __[Skills Directory — Getting Started Guide](blog/posts/entra-nextjs-getting-started.md)__

    ---

    Zero to a working Entra-authenticated Next.js app on your laptop —
    prerequisites, localhost, and Microsoft sign-in end to end.

    `Identity` · `Infrastructure`

-   :material-robot-outline:{ .lg .middle } &nbsp; __[The Great Unbundling](blog/posts/the-great-unbundling.md)__

    ---

    Why AI agents are quietly dismantling the SaaS economy — and forcing the
    Mag 7 to pivot before the window closes.

    `AI`

</div>
