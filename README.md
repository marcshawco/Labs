# IT infrastructure labs

Seven hands-on labs I built in Microsoft Azure covering cloud resources, network traffic analysis, a help desk ticketing system, VPNs, Active Directory, DNS, and Windows file share permissions.

Every screenshot here is from my own environment. I built each lab from an empty Azure subscription, broke a few things along the way, and wrote down how I fixed them.

**Start here if you only read one:** [Active Directory](labs/05-active-directory/) or [network traffic analysis](labs/02-network-traffic-analysis/).

---

## The labs

| # | Lab | What I built | Core skills |
|---|-----|--------------|-------------|
| 1 | [Azure fundamentals](labs/01-azure-fundamentals/) | Resource group, storage account, blob container, upload/edit/download round trip | Azure portal, resource groups, storage redundancy, tagging, cost cleanup |
| 2 | [Network traffic analysis](labs/02-network-traffic-analysis/) | Virtual network with a Windows and an Ubuntu VM, traffic captured in Wireshark | Wireshark, ICMP/SSH/DHCP/DNS/RDP, network security groups, OSI layers 2-4 |
| 3 | [osTicket help desk](labs/03-osticket-helpdesk/) | Full IIS + PHP + MySQL stack, osTicket installed from scratch, two tickets worked to closure | Web server config, MySQL, SLAs, ticket triage, RBAC, escalation |
| 4 | [VPN and geolocation](labs/04-vpn-geolocation/) | Three-hop comparison of public IP and geolocation across home, Azure, and a VPN exit | VPN clients, public vs private addressing, geolocation, tunneling |
| 5 | [Active Directory](labs/05-active-directory/) | Domain controller, domain-joined client, 10,000 users via PowerShell, lockout policy via GPO | AD DS, Group Policy, PowerShell automation, account lockouts, Event Viewer |
| 6 | [DNS](labs/06-dns/) | A records, CNAME records, and a live demonstration of client-side DNS caching | DNS Manager, A/CNAME records, resolution order, cache troubleshooting |
| 7 | [File shares and permissions](labs/07-file-shares-permissions/) | Four SMB shares with different permission levels, access tested as a standard user | SMB shares, NTFS permissions, security groups, nested group membership |

Labs 5, 6, and 7 run on the same domain, so read them in order if you want the full picture.

---

## Why this repo exists

I am moving into IT and I do not have years of ticket queue history to point at. What I do have is a lab environment I built myself and can explain line by line.

So each writeup answers three questions a hiring manager actually cares about:

1. What did you build, and does it work?
2. What broke, and how did you figure it out?
3. Can you explain why it works that way?

The third one is the point. Anyone can follow a checklist. The writeups spend most of their words on what the steps mean, which is the part that transfers to a real environment.

## What each writeup contains

Objective and environment up front, then the build in order with screenshots at the decision points, then a troubleshooting section where something actually went wrong, then a short list of what I can speak to in an interview. Longer labs keep the repetitive installer steps in a collapsed section so the page stays readable.

## Lab environment

| | |
|---|---|
| Cloud | Microsoft Azure, pay-as-you-go subscription |
| Regions | West US 2 (labs 1-4), East US (labs 5-7), Korea South (lab 4 only) |
| Server OS | Windows Server 2022 Datacenter |
| Client OS | Windows 11 Pro 25H2, Ubuntu Server 24.04 LTS |
| VM size | Standard D2als_v7 / D2ms_v6 (2 vCPU, 4-8 GB RAM) |
| Remote access | RDP from macOS via Windows App, SSH from the Windows VM |
| Domain | `mydomain.com`, single forest, single domain |

Resource groups were deleted after each standalone lab, and cost analysis checked, so the subscription stayed near zero between sessions. Labs 5 through 7 share one domain that stayed up across sessions with the VMs stopped in between.

## Tools used

Azure portal, Windows Admin Center / Server Manager, Active Directory Users and Computers, Group Policy Management Console, DNS Manager, Event Viewer, PowerShell and PowerShell ISE, Wireshark, IIS Manager, MySQL, HeidiSQL, osTicket, Proton VPN, Microsoft Remote Desktop.

## Skills by lab

| Skill | Labs |
|---|---|
| Azure resource and cost management | 1, 2, 3, 4, 5 |
| Virtual networks, subnets, NSGs | 2, 5 |
| Packet capture and protocol analysis | 2 |
| Windows Server administration | 3, 5, 6, 7 |
| Active Directory and Group Policy | 5, 6, 7 |
| DNS administration and troubleshooting | 2, 5, 6 |
| PowerShell scripting | 5, 6 |
| Identity, access control, permissions | 3, 5, 7 |
| Help desk workflow and ticket triage | 3 |
| Log review and security events | 5 |

## A note on provenance

These labs follow an IT course curriculum. The instructor supplied the lab objectives; the environment, the execution, the mistakes, and every screenshot and word in these writeups are mine. Where I diverged from the suggested setup (different regions, a different address space, extra verification steps) I say so in the lab.

---

Marcus Shaw · [marcus@marcusshaw.com](mailto:marcus@marcusshaw.com)
