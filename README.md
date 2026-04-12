# Orion Project

This is the **Orion Project**. It is essentially a homelab and my attempt at building, managing, and maintaining a simulation of a real-deal enterprise/business network. From spinning up common IT infrastructure to conducting simulated cyberattacks and putting up proper security measures, I'm doing it all hands-on to actually understand cybersecurity (both defensive and offensive) and networking in practice. Such examples of what I would be doing is provisioning networks, setting up proper domain configs, adding deliberate vulnerabilities, setting up detection with SIEM, and eventually attacking my own setup to see what breaks.

**What it accomplishes**: A working simulation of a small-to-medium-sized company's network. This includes Active Directory domains, user workstations of different OSes, different servers, monitoring and security stacks, and other technologies you'd find in actual corporate environments. 

**Future Aims**: 
  - Shift different aspects of the homelab into Kubernetes clusters and docker containers for proper maintainability and replicability of the environment.
  - Shift the NAT Network-based environment to an isolated environment with a pfsense firewall between the host and the project for proper malware and virus analysis
  - Full automation of certain activities like email sending and receiving from example users, etc.
  - Full working environments with vulnerability snapshots for red/blue team playgrounds and simulations.

**What is currently accomplished**:I've got the full enterprise network up: VMs, domains, Active Directory—all rock solid. Currently knee-deep in the vulnerable environment build: controlled weaknesses for attack sims, tuning Wazuh detections/alerts ahead of attacking my own Orion net. Every step below has my exact process, configs I used, problems I hit and fixed, and what clicked for me. Snapping screenshots as I go.

## What I've Learned So Far
- AD setup showed me DNS screwups kill the whole party—time sync too.
- Wazuh's great for vuln scans but Windows agents were being dumb til I tweaked policies.
- Building vulns: Gotta be surgical or detection tuning turns into a nightmare.



## Lab Progress

### 1. Environment Setup (Complete)
**Status: ✅ Done Weeks Ago**

Fired up all Orion VMs in VirtualBox (NAT network 10.0.0.0/24, DHCP 10.0.0.100-200).

Key VMs & Specs:
| VM Name | OS | CPU/RAM | Disk | IP | Role |
|---------|----|---------|------|----|------|
| Orion-DC | Windows Server 2025 | 2/4GB | 50GB | 10.0.0.5 | Domain Controller (AD/DNS/DHCP) |
| Orion-Admin | Windows 11 Ent | 2/4GB | 80GB | 10.0.0.8 | Admin Workstation |
| Orion-HR | Windows 11 Ent | 2/4GB | 80GB | 10.0.0.100 | User Workstation |
| Orion-SecBox | Ubuntu 22.04 | 2/4GB | 80GB | 10.0.0.10 | Security Server (Wazuh) |
| Orion-LinuxClient | Ubuntu Desktop 22.04 | 1/2GB | 80GB | 10.0.0.101 | Linux Workstation |

**Steps**:
1. Made NAT network "orion-nat".
2. Grabbed ISOs (MS Eval Center for Windows, Ubuntu ISOs).
3. Set static IPs/gateways in VM settings or netplan/ifconfig.
4. Ping/DNS tests across everything.

**Issues**: Win11 TPM crap—fixed with registry hack (bcdedit /set hypervisorlaunchtype auto).
**Learned**: Don't skimp DC resources or replication lags hard. Snapshots = sanity.

[image:1]

Orion network layout—segments keep it from being a flat hackfest.

### 2. Active Directory & Domain Setup (Complete)
**Status: ✅ Fully Functional**

Orion-DC handles AD DS + DNS only—kept it lean.

**Steps**:
1. Server Manager > Add Roles > AD DS + DNS. dcpromo time.
2. Domain: corp.orion.local. OUs for Users, Servers, Workstations.
3. Joined Admin/HR: DNS to 10.0.0.5, sysdm domain join (Administrator/@Deeboodah1!).
4. Linux join: realmd/sssd install, `realm join corp.orion.local -U Administrator`. DNS stub fix in /etc/systemd/resolved.conf.
5. Users: jane.orion (password123!), sec-admin, weak service accounts with SPNs.
6. GPOs: Lame password policy (8 chars min, no complexity—on purpose).

**Issues**: Linux Kerberos bombed from systemd-resolved—forced DC DNS only. Time skew blocked joins (ntpd).
**Learned**: AD lives on DNS, test SRV records (nslookup _ldap._tcp.corp.orion.local). Linux joins suck in prod too.

Screenshots: [AD Users](screenshots/ad-users.png) | [Domain Join](screenshots/linux-join.png)

### 3. Wazuh Deployment & Base Detection (Complete)
**Status: ✅ Agents Everywhere**

Manager on SecBox, agents on all.

**Steps**:
1. Ubuntu: GPG key curl, repo add, `apt install wazuh-manager`.
2. Services up, dashboard at https://10.0.0.10:5601.
3. Agents: Win MSI from dashboard, Linux apt wazuh-agent, `/var/ossec/bin/agent-auth -m 10.0.0.10`.
4. Policies: Vuln detect on (syscheck, rootkit), Win inventory (NVD).
5. Rules tuned for failed logons (5716), service installs.

**Issues**: Win vuln alerts empty—agent restart + policy sync fixed.
**Learned**: Auto CVE matching on packages. Need custom AD decoders next.

[image:2]

Wazuh vulns page—old .NET already flagged.

### 4. Creating Vulnerable Environment
**Status: ✅ Core Weaknesses In**

Controlled vulns/misconfigs for attack chains: foothold → lateral → domain dom. Firewalls off for now.

#### 4.1 Weak Credentials & Accounts
**Steps**:
1. password123! everywhere (jane, hr-user, svc-sql).
2. Local admins same pw workstations.
3. svc-roast SPN: `setspn -s MSSQLSvc/orion-admin.corp.orion.local:1433 svc-roast`.

**Learned**: Reuse = free wins for attackers. BloodHound paths go nuts. Wazuh auth logs catch it.

#### 4.2 Unpatched Software & Services
**Steps**:
1. Win VMs no updates, SMBv1 on (`Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol`).
2. Old IIS on DC (no TLS), EternalBlue tester.
3. Linux: Vulnerable Apache 2.4, SSH root login yes.

**Learned**: Wazuh spots 'em (CVE-2017-0144). Prod: Patch automation or die.

#### 4.3 Misconfigurations for Abuse
**Steps**:
1. AD: Unconstrained delegation svc-roast, Domain Admins overperms.
2. Shares wide open C$/ADMIN$ (icacls /grant Everyone:F).
3. RDP/WinRM everywhere, no NLA.

**Learned**: Kerberoast/PtH/delegation abuse gold. PowerView enum first.

#### 4.4 Network Exposures
**Steps**:
1. Host firewalls off (netsh advfirewall set allprofiles state off).
2. Ports open: 445 SMB, 3389 RDP, 5985 WinRM.
3. DVWA vuln webapp in DMZ VM.

**Learned**: No segments = easy pivots. Need router ACLs later.

Screenshots: [BloodHound Paths](screenshots/bloodhound.png) | [Open Shares](screenshots/smb-enum.png)

**Overall**: Vulns stack—creds + unpatched = full chains. Expect MS17-010 → hashes → DCSync.

### 5. Detection & Alerts Tuning for Attacks
**Status: ⏳ Tuning Now**

Wazuh prepped for red phase, custom rules for TTPs.

**Steps**:
1. Sigma rules: Kerberoast (4769 RC4), ASREPRoast, PsExec (4688).
2. AD audit GPO: Logon/Logoff, Object Access on.
3. WinEvent forward: Security, Sysmon later.
4. Dashboards: ATT&CK maps, baselines.
5. Test `Invoke-Kerberoast`—alert rule 51003 fired.

**Issues**: Legit logon noise—ossec.conf whitelist.
**Learned**: Thresholds key (>10 TGT/min = roast). Sysmon+Wazuh = EDR vibes.

## Tech Stack
- Hypervisor: VirtualBox
- SIEM: Wazuh/ELK
- Vuln Apps: DVWA, weak services
- Attacks: BloodHound, CrackMapExec (soon)

## Future Ideas
- Attack runs: Phish → exploit → AD ownage.
- K8s cloud shift for hybrid.
- [ideas.md](ideas.md) has more.

**Progress**: Net built, vulns planted, detections set. Red team time. Updated: April 11, 2026.
