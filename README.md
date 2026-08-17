# Hybrid Identity & Endpoint Management Lab

This is a homelab project I built to get hands-on with the tools that come up constantly in helpdesk and IT support job postings: Active Directory, Entra ID, Intune, and Configuration Manager.

The setup mixes on-prem and cloud, which is honestly what most real environments look like these days. A Proxmox host runs the on-prem side (domain controller, ConfigMgr server, domain-joined clients), and a Microsoft 365 tenant handles the cloud side (Entra ID, Intune). A couple of client VMs live on VMware Workstation separately for the Autopilot pieces, more on why below.

## Why I built it this way

I originally started following a Microsoft deployment lab guide pretty much cover to cover, but partway through I stepped back and asked what would be useful for the job I'm going for as helpdesk/IT support. Some sections in the guide are genuinely great for that. Others are more security-engineering or infrastructure-architect territory, stuff a helpdesk person would support around, not build from scratch.

So I made some deliberate cuts, and I think the reasoning behind those cuts is worth as much as the stuff I built. I'd rather show I can scope a project sensibly than pad it out with things I didn't really need.

## Environment

**On-prem (Proxmox, host "mikelab"):**
- WIN-DC-01 – Domain Controller, domain MGEnterprise.local
- CM-01 – Configuration Manager site server (SQL Server, WSUS, ADK)
- MGE-WINCLT1 / MGE-WINCLT2 – domain-joined Windows 11 clients, ConfigMgr-managed

**Cloud (Microsoft 365 tenant "MGEnterprise"):**
- Entra ID, Entra Connect (hybrid identity sync)
- Intune (device management)
- Test users TU1/TU2

**Separate (VMware Workstation):**
- WIN-CLT3 / WIN-CLT4 – used for the Autopilot/cloud-native pieces. I built these on a different platform than the Proxmox clients mainly because I was running low on storage on the Proxmox box partway through, and it turned out to be a decent excuse to show I can work across more than one virtualization platform.

## Foundation: getting the on-prem side ready

Before any of the cloud/hybrid pieces could work, the on-prem side had to be solid: a healthy domain controller, a certificate authority, and a fully working ConfigMgr site server with domain-joined clients reporting in.

**Domain controller baseline**

![dcdiag output](screenshots/01-ad-baseline/01-dcdiag-verbose-output.png)
![Get-ADDomain output](screenshots/01-ad-baseline/02-get-addomain-output.png)

**Certificate services (AD CS)**

![Certification Authority console](screenshots/02-certificate-services/02-certification-authority-console.png)

**CM-01 build (SQL, WSUS, prerequisites)**

![WSUS console initialized](screenshots/03-cm01-build/06-wsus-console-initialized.png)
![SQL services running](screenshots/03-cm01-build/05-sql-services-running.png)

**ConfigMgr install and AD integration**

![Schema extension success](screenshots/04-configmgr-install/01-schema-extension-success.png)
![Site status all healthy](screenshots/04-configmgr-install/03-site-status-all-healthy.png)

**Domain-joined clients reporting in**

![Both clients ConfigMgr managed](screenshots/05-domain-clients/03-both-clients-configmgr-managed.png)

**Cloud tenant stood up**

![M365 products active](screenshots/06-cloud-tenant/01-m365-products-active.png)
![Sales group members](screenshots/06-cloud-tenant/04-sales-group-members.png)

## What's actually in here

### 1. Hybrid identity (Entra Connect)
Set up Entra Connect on the DC to sync on-prem AD accounts to the cloud tenant, then got both domain-joined clients showing as Hybrid Entra joined. This part had more troubleshooting than anything else in the project, see below.

![Entra Connect sync review](screenshots/07-hybrid-identity-sync/02-entra-connect-sync-review.png)
![Entra Connect configuration complete](screenshots/07-hybrid-identity-sync/03-entra-connect-configuration-complete.png)

### 2. Windows Autopilot
Enrolled two clients (CLT3/CLT4) through Autopilot: hardware hash export, Intune device import, deployment profile, Enrollment Status Page, the whole flow from a factory-reset-style state to a fully enrolled, Intune-managed device with the branded OOBE screen.

![Autopilot devices assigned](screenshots/08-autopilot/01-autopilot-devices-assigned.png)
![Autopilot branded OOBE](screenshots/08-autopilot/02-autopilot-branded-oobe.png)

### 3. BitLocker (via Intune)
Deployed a BitLocker policy through Intune's disk encryption profile, encryption on by default, recovery keys escrowed to Entra ID so they're recoverable from the portal instead of stuck on a machine that might not boot.

![BitLocker recovery key in Entra ID](screenshots/09-bitlocker/01-clt3-recovery-key-entra.png)

### 4. Windows Defender Antivirus (on-prem, via ConfigMgr)
Configured Endpoint Protection through Configuration Manager rather than Intune for this one, on purpose, since a lot of real environments still manage AV on-prem even if everything else has moved to the cloud. Scoped a custom antimalware policy to a single test collection to prove targeting works, not just "it applied to everything."

![WDAV antimalware policy](screenshots/10-defender-av/01-wdav-antimalware-policy-config.png)
![Managed by administrator](screenshots/10-defender-av/02-clt1-managed-by-administrator.png)

### 5. Tenant Attach / Co-Management
This is the one I'm most glad I did. Enabled co-management so a single device is managed by both ConfigMgr and Intune at the same time, with specific workloads (compliance policy, Windows Update) handed off to Intune while ConfigMgr keeps the rest. A lot of companies are sitting in exactly this in-between state right now, mid-migration from on-prem to cloud, so understanding how it actually works (and what the workload numbers mean) felt worth the time.

![Co-management overview](screenshots/11-co-management/01-clt1-comanagement-overview.png)
![Co-management log compliant](screenshots/11-co-management/02-comanagementhandler-log-compliant.png)

## Troubleshooting I ran into

This is the part I think matters most, since anyone can follow steps in a guide. Here's what went sideways and how I worked through it.

**MSOnline PowerShell module doesn't exist anymore.** The lab guide has you install the MSOnline module to look up an AAD connector account. Turns out Microsoft retired that module in May 2025 and pulled it from the PowerShell Gallery entirely, so no combination of fixing NuGet, PowerShellGet, or TLS settings was ever going to make Install-Module MSOnline work again. I confirmed it wasn't a network or environment problem first (DNS resolution and connectivity to the Gallery both worked fine), then found the same information through the Entra Connect wizard's own "view current configuration" screen instead.

**Entra Connect sign-in kept failing with a vague error.** Hit "There was an issue looking up your account" during the Entra Connect setup wizard. Worked through it in order: fixed Internet Explorer's Enhanced Security Configuration (it was blocking the wizard's embedded sign-in control), then applied a TLS 1.2 registry fix on the domain controller. Ruled out DNS and IPv6 first by directly testing resolution and connectivity, so I wasn't just guessing at fixes.

**BitLocker "encrypted" but not actually protected.** One client showed 100% encrypted in manage-bde -status, but Protection Status said "Off" and there were no key protectors at all, meaning it wasn't actually protecting anything despite what the percentage suggested. TPM checked out fine, so it wasn't a hardware problem. The real cause: BitLocker refuses to attach a TPM protector if it detects a bootable CD/DVD device present in the VM at all, even with nothing mounted and the connect-on-boot option unchecked. The fix was removing the virtual CD/DVD device from the VM entirely, not just disconnecting it. This is a good example of a case where the error message told me what was wrong once I actually read it carefully, instead of assuming the TPM was broken and going down that path.

**ConfigMgr console showed a client as unhealthy when it wasn't.** A client had a warning icon in the console. Rather than assume the console was right, I went to the actual health evaluation log (CcmEval.log) on the client itself, and it showed every check passing. Turned out to be stale display data in the console, not a real problem. Worth noting because it's tempting to trust a management console's status icons at face value; the underlying logs are the real source of truth.

**Documentation doesn't always match the current portal.** Ran into this constantly: download URLs that no longer exist, endpoint.microsoft.com (now intune.microsoft.com), old Azure AD blade paths that have moved in the current Entra portal, deprecated policy profile types replaced by newer "Settings Catalog" versions. None of this is a knock on the guide, Microsoft's admin portals change fast, but it meant I couldn't just follow steps blindly. I had to verify what the current interface looked like before trusting any given instruction and treat the guide as a roadmap rather than a script.
