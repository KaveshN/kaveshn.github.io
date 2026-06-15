---
hide:
  - navigation
  - toc
---

# Kavesh Naidoo

Notes from the intersection of **identity, infrastructure, and AI** — Entra ID,
PowerShell, and the systems in between. I lead IT/infra and I'm learning to
build, so expect equal parts architecture and hands-on.

[Browse all posts →](blog/index.md){ .md-button .md-button--primary }

---

## Latest writing

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
