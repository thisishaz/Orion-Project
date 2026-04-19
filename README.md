# Orion Project

This is the **Orion Project**. It is essentially a homelab and my attempt at building, managing, and maintaining a simulation of a real-deal enterprise/business network. From spinning up common IT infrastructure to conducting simulated cyberattacks and putting up proper security measures, I'm doing it all hands-on to actually understand cybersecurity (both defensive and offensive) and networking in practice. Such examples of what I would be doing is provisioning networks, setting up proper domain configs, adding deliberate vulnerabilities, setting up detection with SIEM, and eventually attacking my own setup to see what breaks.

**What it accomplishes**: A working simulation of a small-to-medium-sized company's network. This includes Active Directory domains, user workstations of different OSes, different servers, monitoring and security stacks, and other technologies you'd find in actual corporate environments. 

**Future Aims**: 
  - Shift different aspects of the homelab into Kubernetes clusters and docker containers for proper maintainability and replicability of the environment.
  - Shift the NAT Network-based environment to an isolated environment with a pfsense firewall between the host and the project for proper malware and virus analysis
  - Full automation of certain activities like email sending and receiving from example users, etc.
  - Full working environments with vulnerability snapshots for red/blue team playgrounds and simulations.
  - Conduct more simulated cyberattacks and proper log analysis for certain attacks (like malware, etc.)

**What is currently accomplished**: 
  - Full on-premise enterprise network has been successfully set up. This is includes:
    -  Microsoft DC Server (For Active Directory).
    -  VM Workstations simulating real-life user and worker accounts.
    -  VM Servers designed for specific purposes (Email server)
    -  VM Security Box used for monitoring logs, alerts and incident responses.
    -  Attacker VM used for simulating cyberattacks.
  
  - Vulnerable Environment completed and setup, to simulate how an attacker would gain initial access within the Orion Network, escalate their privilege, and exfiltrate data and persist within an already comporomised      system.


## What I've Learned So Far
- Microsoft Domain Controller and AD setup has taught me how to setup domains for a specific network, how to configure and establish DNS and DHCP routes, and the prerequisites to ensure a working and stable enterprise network.
- VM Setup has taught me how to provision VMs pertaining to their OSes, and how to setup the IP Addresses so each respective VM is properly connected to the domain network.
- Email Server simulation setup taught me how email servers properly work, how their different protocols function, and how docker containers are composed and setup.
- Wazuh Manager setup taught me how to properly look for logs pertaining to certain parameters, how to setup specific alerts/monitors and file integrity checks based on specific triggers (such as parameters), and how to setup actions once an alert is caused.
- Learned how attackers learn about a network through reconnaissance (using tools like nmap towards specific ports), how an attacker gains initial network through brute-force tactics (using tools like hydra, and email phishing), how an attacker uses inference to escalate their privilege within the network by moving from workstation to workstation and learning more about the network (using nxc or evil-winrm to gain access to windows workstations), how an attacker exfiltrates data using tools like scp, and how an attacker maintains privilege within the network (creating a reverse shell protocol that activates at specific times).

## Basic Lab Setup

### 1. Environment Setup
**Status: ✅ Done**


Provisioned all Orion VMs in VirtualBox 
  - NAT network 10.0.0.0/24, Static DNS from 10.0.0.100-200 (Temporarily, might change to DHCP)


Key VMs & Specs:
| VM Name | OS | CPU/RAM | Disk | IP | Role |
|---------|----|---------|------|----|------|
| orion-dc | Windows Server 2025 | 2/4GB | 50GB | 10.0.0.5 | Domain Controller (AD/DNS/DHCP) |
| orion-corp-svr | Ubuntu 24.04.01 Server | 1/2GB | 25GB | 10.0.0.8 | Corporate Server |
| orion-sec-box | Ubuntu 24.04.01 Desktop | 2/4GB | 80GB | 10.0.0.10 | Dedicated Security Server |
| orion-sec-work | Security Onion | 1/2GB | 55GB | 10.0.0.103 | Security Playground |
| orion-win-client | Windows 11 Enterprise (Workstation) | 2/4GB | 80GB | 10.0.0.100 | Windows Workstation |
| orion-linux-client | Ubuntu Desktop 24.04.01 Desktop | 1/2GB | 80GB | 10.0.0.101 | Linux Workstation |
| orion-attacker | Kali Linux | 1/2GB | 55GB | 10.0.0.50 | Attacker Environment |


**Steps**:
1. Created the NAT network "orion-network", allowing it to have a subnet mask of 24 and a default gateway of 10.0.0.1.
2. Grabbed and provisioned respective ISOs (MS Eval Center for Windows, Ubuntu ISOs, and other respective ISOs such as Security Onion, and Kali Linux OS) within Virtualbox.
4. Set static IPs/gateways in respective VMs so that it connects to the DC.
5. Ping/DNS tests across everything to ensure everything is properly configured and connected.

- Orion network layout ensures that everything is properly segmented and checked for, so visualization of said enterprise network is easy to understand and maintain.


### 2. Active Directory, Domain Setup and Workstation Connection
**Status: ✅ Done**

'orion-dc' handles AD, DNS, and DHCP. It serves as the backbone of the network as it handles authentication, authorization, name resolution and acts as the main gateway between the host system and the enterprise network (will be fully adapted and further isolated in the future).

**Steps**:
1. Prepare 'orion-dc' by setting Static IP of 10.0.0.5/24, default gateway of 10.0.0.1 (host NAT), and DNS temp 10.0.0.5.
2. AD DS Setup: Add 'Active Directory Domain Services' to the server manager and complete the setup properly. Run dcpromo and create a new 'forest' of 'corp.orion-dc.com'.
3. DNS Setup: Use the server manager and properly configure the 'DNS Server' properly. Ensure that DNS listens to all interfaces (no port forwarding yet).
4. DHCP Setup: Go to roles and add DHCP Server. Set scope to 10.0.0.0/24 (range 10.0.0.100-200, exclude 10.0.0.1-20), router 10.0.0.1, DNS 10.0.0.5. Activate scope.
5. Connect User Workstations:
    - **Windows**: Set DNS 10.0.0.5 only, reboot. Sysdm.cpl > Change Settings > Domain: corp.orion-dc.com. Reboot.
    - **Linux**: Set domain connection to non-native windows using Samba/Winbind configuration via Kerberos


### 3. Wazuh Deployment & Base Detection
**Status: ✅ Done**

Manager on Orion-SecBox (Ubuntu 10.0.0.10), agents deployed to all endpoints (DC, Admin, HR, LinuxClient). Central SIEM for logs, vulns, FIM.

**Steps**:
1. Prepare SecBox Server (setting IP to 10.0.0.10)
2. Add Wazuh Repo
3. Install Wazuh Manager (including the indexer and dashboard)
4. Ensure that wazuh automatically starts as the system is booted up through systemctl symlink creation. Ensure that access is given to the localhost
5. Deploy Agents:
    - **Windows**: Install the agent from the dashboard > agents section, follow the steps and run the final installation command as administrator.
    - **Ubuntu**: Same thing as windows, follow the installation guide within the wazuh dashboard > agents section.
6. Restart agents and wazuh manager to ensure that devices are properly connected for further configuration
7. Create base configs. This includes:
    - Policies for different groups (based on OS like windows and Linux)
    - Alerts for specific potential access points for an attacker (such as WinRM Logon, and ssd connection)
    - Set up triggers for specific parameters for those specific alerts.
    - Set up File Integrity Management on a dummy file acting as sensitive data within the dc server.
8. **Tune Rules** (for certain access points and workstation access):
    - Failed ssd attempts
    - WinRM Logon
    - File accessed (dummy data)


### 4. Creating Vulnerable Environment
**Status: ✅ Done**

Controlled vulnerabilities within our enterprise environment such as no alert detection for certain aspects, open ports within certain workstations. These are vulnerabilities that SHOULD NOT exist in a real workplace, however are being made vulnerable within this project to learn how attackers gain a foothold within networks and gain sensitive data.

#### 4.1 Weak Credentials & Accounts
**Steps**:
1. @password123! for all user accounts
2. Root accounts for certain workstations are made to be VERY weak and vulnerable.

**Learned**: Predictable and easy passwords must be avoided as brute-force tactics can be leveraged to gain initial access within the network (using tools like Hydra).

#### 4.2 Misconfigurations for Abuse
**Steps**:
1. Basic configurations for AD, non proper measures to counter potential access by attackers.
2. Shares wide open Administrator access on all accounts
3. Enabled RDP

**Learned**: Misconfigurations or basic configurations might result in a potential pathway for attackers to gain access into the network.

#### 4.3 Network Exposures
**Steps**:
1. No active strong firewall to accept incoming and outgoing data transfers.
2. Ports left open within both Windows and Linux workstations.

**Learned**: Exposures result in pathways for attacker to gain access into a network

### 5. End-to-end basic attack simulation "From Initial Access to Persistence" 
**Status: ✅ Done**

Conducted a simple simulation of how an attacker would gain access into the orion network and eventually gain privilege and persist within the network. 
**Steps** are as follows:
1. Reconnaissance + Initial Access
2. Privilege Escalation
3. Data Exfiltration
4. Persistence


## Future Ideas
- More simulations on cyberattacks, log analysis, and further configuration and networks setup that promotes better security across the orion network.
- Isolate the network with a pfsense firewall so host system is protected and is unaffected by any form of virus analysis
- Shift network into docker containers properly managed with k8s
- Shift on-premise network into a more cloud-based infrastructu

**Progress**: Net built, vulns planted, detections set. Red team time. Updated: April 11, 2026.
