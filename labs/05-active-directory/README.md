# Lab 5: Active Directory

Build a domain from nothing: a Windows Server domain controller, a Windows 11 client joined to it, a delegated admin account, 10,000 user accounts created by script, and an account lockout policy pushed by Group Policy. Then lock an account out on purpose and track it through Event Viewer.

If you work a help desk, this is the system you will spend most of your day in. Password resets, lockouts, group membership, "I can't get into my machine" all land here.

| | |
|---|---|
| Domain | `mydomain.com`, new forest, single domain |
| Domain controller | `dc-1`, Windows Server 2022 Datacenter, static 10.0.0.6 |
| Client | `client-1`, Windows 11 Pro 25H2, 10.0.0.5 |
| Virtual network | `Active-Directory-VNet`, subnet 10.0.0.0/24, East US |
| Roles | AD DS and DNS on the same host |
| Domain admin | `jane_admin`, in the Domain Admins group |
| Users created | 10,000, via PowerShell into the `_EMPLOYEES` OU |

Labs 6 and 7 run on this same domain.

## What Active Directory is for

The recognisable version: a university where one student ID signs you in to any of several thousand campus machines. The machines are joined to a domain, the accounts live centrally on a domain controller, and any joined machine can authenticate any domain account.

The alternative is a local account on every machine, which does not survive contact with more than about ten computers.

## Infrastructure

Two VMs in one virtual network. Server 2022 for the domain controller, Windows 11 for the client.

![Creating dc-1](images/01-create-dc-1-basics.png)

![dc-1 networking](images/02-create-dc-1-networking.png)

The first thing that has to be right is the domain controller's private IP. Azure hands out dynamic addresses by default, and a dynamic address can change when a VM is deallocated and started again.

![Setting the DC NIC to static allocation](images/03-dc-1-static-private-ip.png)

That matters because this server is about to become the DNS server for the whole domain, and I am about to hard-code its address into the client's network settings. A DNS server whose address can move is a DNS server that will break.

I also turned off the Windows Firewall on the domain controller for this lab so ICMP would answer during testing. On a production DC you would leave it on and open the specific ports instead.

## Pointing the client at the domain controller

By default an Azure VM inherits DNS from the virtual network, which resolves against Azure's own resolver.

![Client NIC inheriting DNS from the vnet](images/04-client-1-dns-inherited.png)

That resolver knows nothing about `mydomain.com`. If the client asks it where the domain controller for `mydomain.com` is, it will go looking at the real public `mydomain.com` and come back with nothing useful.

So the client's NIC gets a custom DNS server: the domain controller's static address.

![Client NIC with custom DNS](images/05-client-1-dns-custom.png)

Then restart the client and verify, rather than assume.

![Pinging the DC from the client](images/06-client-1-ping-dc-1.jpg)

![ipconfig /all confirming the DNS server](images/07-client-1-ipconfig-dns.jpg)

`ipconfig /all` shows the client at 10.0.0.5 with DNS pointed at 10.0.0.6. Ping replies confirm the two machines can reach each other across the subnet.

Those two checks are the entire reason the domain join later works first time. Nearly every failed domain join I have read about traces back to one of them.

## Promoting the domain controller

Install the AD DS role, then promote the server to a domain controller in a new forest named `mydomain.com`, with DNS installed alongside.

Once that completes and the server restarts, logging in changes. A local username on its own no longer works, because accounts now live in a domain and the login has to say which one: `mydomain.com\labuser`.

![Logging in with a domain-qualified account](images/08-rdp-dc-1-domain-login.png)

That prefix is the concept of logon context, and it is the thing to understand here. On a domain-joined machine an account can be local to that machine or it can live in the domain, and those are different accounts even if they share a name.

![Server Manager showing AD DS and DNS](images/09-server-manager-ad-ds-dns.png)

AD DS and DNS both running. DNS came along with the promotion, which is the standard pairing because domain members find the domain controller through DNS.

## Organisational units and a delegated admin

OUs for `_ADMINS`, `_EMPLOYEES` and later `_CLIENTS`. The leading underscore sorts them above the built-in containers, which makes a populated domain much easier to navigate.

![Creating the _ADMINS OU](images/10-create-ou-admins.png)

Then a user, Jane Doe, with the logon name `jane_admin`.

![Creating jane_admin](images/11-create-user-jane-admin.png)

Creating it inside `_ADMINS` and calling it admin does not make it one. What makes it a domain admin is membership of the built-in Domain Admins security group.

![Adding jane_admin to Domain Admins](images/12-add-jane-to-domain-admins.png)

This is the separation of duties pattern: the default `labuser` account stops being the thing I log in with, and a named account with explicit privilege takes over. In a real environment the same person would also hold a separate non-privileged account for everyday work.

Domain Admins is a serious group. Its members have authority over every account and every joined machine in the domain.

![Retrying the jane_admin login](images/13-jane-admin-login-retry.png)

First login attempt failed. Credential propagation after an account change is not instant, and retrying after a minute worked.

## Joining the client

On the client, System Properties, change membership from workgroup to domain.

![Changing client-1 to domain membership](images/14-client-1-join-domain.jpg)

The join needs an account with permission to add machines to the domain, so `jane_admin`.

![Domain join credentials](images/15-domain-join-credentials.jpg)

![Welcome to the domain](images/16-welcome-to-the-domain.jpg)

Because the client's DNS pointed at the domain controller, it located the domain on the first try. Pointing a client at a DNS server that does not host the zone produces "an Active Directory domain controller for the domain could not be contacted", which is the error I would otherwise have been debugging.

The computer object appears in AD, in the default Computers container.

![client-1 in the Computers container](images/17-client-1-in-computers-container.png)

I moved it into a `_CLIENTS` OU. Group Policy links to OUs, so where a computer sits determines what policy applies to it.

![client-1 moved to the _CLIENTS OU](images/18-client-1-moved-to-clients-ou.png)

Then enabled Remote Desktop on the client and added Domain Users to the allowed list, so ordinary accounts can sign in remotely rather than only administrators.

![Remote Desktop enabled](images/19-enable-remote-desktop.jpg)

![Adding domain groups to Remote Desktop users](images/20-add-domain-users-to-rdp.jpg)

![Remote Desktop users configured](images/21-rdp-users-configured.jpg)

Doing this per machine is fine for one client. For a fleet this is a Group Policy setting linked to the `_CLIENTS` OU, which is the same reason the OU exists.

## 10,000 users with PowerShell

Clicking through 10,000 New User dialogs is not a job anyone does. The script generates random names, builds a username from them, and calls `New-ADUser` with the target OU as the path.

![The user creation script in PowerShell ISE](images/22-create-users-script.png)

The script depends on the `_EMPLOYEES` OU existing and being spelled exactly right, because the distinguished name is built from that string.

First run failed.

![Access denied from New-ADUser](images/23-script-access-denied.jpg)

`New-ADUser : Access is denied`, repeated per account. The account was a domain admin, so the permission was there. The problem was the session: PowerShell ISE was running unelevated, so the administrative privilege in the token was not active. Running ISE as administrator fixed it.

![Script running elevated and creating accounts](images/24-script-running-elevated.png)

Worth separating those two ideas, because they look identical from the error message. Having a privilege and having it active in the current session are different things, and User Account Control is what sits between them.

![The _EMPLOYEES OU populated](images/25-employees-ou-populated.png)

A realistically sized directory, which makes the next part behave the way a real one would.

Then a sanity check: pick one of the generated accounts at random and sign in to the client with it.

![RDP as a generated user](images/26-rdp-as-generated-user.png)

![Generated user properties](images/27-generated-user-properties.png)

It worked because that account is a member of Domain Users by default, and Domain Users is on the client's Remote Desktop list.

![User profiles created on the client](images/28-user-profiles-on-client.jpg)

`C:\Users` on the client now holds a profile per account that has signed in. That first login is slow because Windows is building the profile, which is the real answer to "why did my first login take so long".

## Account lockout policy

The scenario to reproduce: someone fails their password enough times that the account locks, and a help desk unlocks it.

I failed the login about eleven times and the account did not lock.

![Failed credentials, account still unlocked](images/29-lockout-credential-failure.png)

Which is correct behaviour, because no lockout policy was defined. Rather than move on, I went and configured one, since the undefined state was the actual finding.

Group Policy Management Console, Default Domain Policy, which is already linked at the domain root.

![GPMC with the Default Domain Policy](images/30-gpmc-default-domain-policy.png)

Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Account Lockout Policy. Everything Not Defined.

![Lockout policy undefined](images/31-lockout-policy-undefined.png)

![Setting lockout duration](images/32-set-lockout-duration.png)

![Lockout policy configured](images/33-lockout-policy-configured.png)

| Setting | Value | What it means |
|---|---|---|
| Account lockout threshold | 5 attempts | Failures before the account locks |
| Account lockout duration | 30 minutes | How long it stays locked without intervention |
| Reset counter after | 10 minutes | Idle time before the failure count returns to zero |

The reset counter is the setting people misread. Fail twice, wait ten minutes, and you have five attempts again rather than three. It stops a slow accumulation of unrelated typos over a week from locking someone out.

Policy refreshes on its own roughly every 90 minutes. To test now, force it on the client.

![gpupdate /force](images/34-gpupdate-force.png)

![gpresult confirming the policy applied](images/35-gpresult-policy-applied.png)

`gpresult /r` confirms the Default Domain Policy applied and when. That command is the first thing to run when a policy "isn't working", because it answers whether the machine ever received it.

Then the same test again.

![RDP refusing a locked account](images/36-rdp-account-locked-out.png)

Locked on the sixth attempt, and the client reports the reason directly: the account has been locked due to too many sign in attempts. Error 0xd07.

![Unlocking the account in ADUC](images/37-unlock-account-in-aduc.png)

Unlocking is one checkbox on the Account tab. Right click the user offers Reset Password, which can unlock at the same time.

![Signing in after the unlock](images/38-login-after-unlock.jpg)

I also disabled and re-enabled an account, which produces a different message ("your account has been disabled") and is the correct state for someone who has left. A disabled account is almost never something to re-enable without asking why it was disabled.

## Reading the logs

The failures are in the Security log, and they are on the client rather than the domain controller, because that is where the logon was attempted.

![Event Viewer showing 4625 audit failures](images/39-event-viewer-4625-failures.png)

Event ID 4625 is a failed logon. The run of audit failures is two or three seconds apart, then a gap of about a minute and ten seconds, then a success. That gap is me switching to the domain controller and unlocking the account.

Each event carries the account name, the domain, the failure reason and the source address.

Pattern is everything here. A handful of failures seconds apart from the machine the user sits at is somebody mistyping a password. Hundreds of failures from an address nobody recognises is something else, and that distinction is the starting point of security operations.

The client did not have access to its own Security log as a standard user, so I ran Event Viewer as `jane_admin` with Run as different user, which is the normal way to do this without logging the user out.

## What broke

**`New-ADUser : Access is denied` as a domain admin.** PowerShell ISE was not elevated. Privilege present in the token but not active.

**No lockout after eleven failed attempts.** No policy defined. I configured the Default Domain Policy, forced a refresh with `gpupdate /force`, confirmed with `gpresult /r`, and retested.

**The first `jane_admin` login failed.** Credential changes take a moment to propagate. Retrying worked.

## What I can speak to

- Installing AD DS and promoting a server to a domain controller in a new forest
- Why a domain controller needs a static address and why clients must use it for DNS
- Domain versus local logon context, and reading `DOMAIN\user`
- Creating a delegated domain admin, and why group membership is what grants the privilege
- Joining a client to a domain and diagnosing a failed join from DNS
- OUs as policy attachment points, not just folders
- Bulk account creation with PowerShell and `New-ADUser`
- The difference between holding a privilege and running elevated
- Account lockout threshold, duration and reset counter, and the behaviour each one produces
- Forcing and verifying Group Policy with `gpupdate` and `gpresult`
- Unlocking, resetting, disabling and enabling accounts, and when each is right
- Reading 4625 events and telling a mistyped password from a password attack

---

[Back to all labs](../../README.md) · Previous: [VPN and geolocation](../04-vpn-geolocation/) · Next: [DNS](../06-dns/)
