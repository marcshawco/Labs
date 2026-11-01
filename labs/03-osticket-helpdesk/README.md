# Lab 3: osTicket help desk

Stand up a real help desk from an empty Windows VM: web server, PHP runtime, database, then osTicket on top. Configure it the way an IT department would, with departments, roles, SLAs and agents. Then work two tickets end to end, playing both the end user and the technicians.

This is the closest lab in this repo to the actual day job. The install teaches the stack; the tickets teach the process.

| | |
|---|---|
| VM | `osticket-vm`, Windows 11 Pro 25H2, West US 2 |
| Web server | IIS with CGI enabled |
| Runtime | PHP 7.3.8, registered in IIS via PHP Manager |
| Database | MySQL 5.5.62, managed with HeidiSQL 12.3 |
| Application | osTicket v1.15.8 at `localhost/osTicket` |
| Help desk | "Marcus's Help Desk" |
| Tickets worked | #800185 (outage), #909979 (application fault) |

## The stack, and why each piece is there

osTicket is a PHP web application backed by MySQL. Nothing in it runs unless three separate things are true:

- IIS is installed and serving, with CGI enabled so it can hand requests to an interpreter
- PHP exists on disk and IIS knows where to find it
- MySQL is running and an empty database exists for osTicket to populate

Most osTicket installs fail at one of those three, so that is the order I checked things in.

![Creating the VM](images/01-create-osticket-vm.png)

![Installation files extracted on the desktop](images/02-installation-files-extracted.jpg)

## Web server

IIS goes on through Turn Windows features on or off. CGI lives several levels deep, under World Wide Web Services > Application Development Features, and osTicket will not run without it.

![Enabling IIS with CGI](images/03-enable-iis-cgi.jpg)

Before installing, `127.0.0.1` in the browser refused the connection, because nothing was listening. After, it serves the IIS default page. That one check confirms the machine is now a web server.

![IIS default page on the loopback address](images/04-iis-default-page.png)

<details>
<summary>Dependency installs (6 screenshots)</summary>

PHP Manager for IIS, which gives IIS a UI for registering and configuring PHP.

![Installing PHP Manager for IIS](images/05-install-php-manager.jpg)

URL Rewrite Module 2, which osTicket needs for its clean URLs.

![Installing URL Rewrite](images/06-install-url-rewrite.jpg)

A `C:\PHP` folder, then the PHP binaries extracted into it.

![Creating C:\PHP](images/07-create-php-folder.jpg)

![PHP binaries in place](images/08-php-files-extracted.jpg)

Visual C++ redistributable, which PHP links against.

![Installing the VC++ redistributable](images/09-install-vc-redist.jpg)

MySQL 5.5.62, typical setup, with root as both user and password. That credential choice is terrible practice and fine for an isolated lab VM with no inbound database port open. In production this would be a dedicated service account with a generated password and no remote root login.

![Installing MySQL](images/10-install-mysql.jpg)

</details>

Then IIS needs to be told PHP exists. Register the PHP executable through PHP Manager and restart the server.

![IIS Manager](images/11-iis-manager.jpg)

![Registering php-cgi.exe](images/12-register-php-executable.jpg)

osTicket's `upload` folder gets copied into `C:\inetpub\wwwroot` and renamed to `osTicket`. Exact casing, no spaces, because that string becomes part of the URL.

![Renaming upload to osTicket in wwwroot](images/13-rename-upload-to-osticket.jpg)

## Installer prerequisites

Browsing to the site gives osTicket's own prerequisite check, which is the most useful screen in the whole install.

![Installer showing missing PHP extensions](images/14-installer-missing-extensions.jpg)

Red crosses next to IMAP, Intl and OPcache. IMAP is what osTicket uses to fetch mail; Intl handles localisation; OPcache is a performance cache.

Rather than guess, I went back to PHP Manager, enabled those three extensions, and refreshed.

![Enabling PHP extensions](images/15-enable-php-extensions.jpg)

![Prerequisites passing](images/16-installer-prereqs-passing.jpg)

Green across the required items. This is the moment the stack is actually correct, and it is worth stopping at rather than pushing past.

## Database

HeidiSQL connects to MySQL on 127.0.0.1 as root.

![Installing HeidiSQL](images/17-install-heidisql.jpg)

MySQL was installed, but osTicket needs its own empty database inside it, named exactly `osTicket`.

![Creating the osTicket database](images/18-create-osticket-database.jpg)

Then the installer takes the admin account details and the database credentials together.

![Installer configuration](images/19-installer-admin-and-db-config.jpg)

![Install complete](images/20-install-complete.jpg)

Back in HeidiSQL, the database that was empty two minutes ago now holds the osTicket schema: `ost_ticket`, `ost_department`, `ost_staff`, `ost_email` and the rest.

![Tables created by the installer](images/21-database-tables-created.jpg)

That is the proof the application and database are actually wired together, rather than the web front end merely loading.

![Logging in as an agent](images/22-agent-login.jpg)

![System settings](images/23-system-settings.jpg)

![System logs](images/24-system-logs.jpg)

There are two separate front doors. `/scp/` is the staff control panel for agents and admins. The root URL is the support centre where end users raise tickets.

![End user support centre](images/25-end-user-support-center.jpg)

## Configuration

osTicket separates four concepts that are easy to confuse, so configuring them in order makes the model clear.

**Departments** control visibility. A ticket belongs to a department, and agents see tickets for departments they have access to. I added SysAdmins alongside the default Support.

![Default departments](images/26-departments-default.png)

![Adding the SysAdmins department](images/27-add-department-sysadmins.png)

**Roles** are permission sets. They answer "what can this person do to a ticket", separately from "which tickets can they see".

![Roles list](images/28-roles-list.png)

![Creating a Supreme Admin role](images/29-add-role-supreme-admin.png)

![Role permissions](images/30-role-permissions.png)

Access in osTicket is department plus role together. Jane Doe is SysAdmins with Supreme Admin, so she can do everything to SysAdmins tickets. John Doe started as Support with View only, which is deliberately restrictive and pays off later in the ticket walkthrough.

**Teams** cut across departments, for people from different departments who work the same system.

![Teams](images/31-teams.png)

<details>
<summary>Creating agents (4 screenshots)</summary>

![Agents before](images/32-agents-initial.png)

![Agent account details](images/33-add-agent-account.png)

![Agent department and role](images/34-add-agent-access.png)

![Agent team assignment](images/35-add-agent-teams.png)

</details>

![Agents configured](images/36-agents-configured.png)

**SLAs** are the clock. A grace period plus a schedule produces a due date, and the ticket is flagged overdue past it.

![Default SLA](images/37-sla-default.png)

![Creating the Sev-A plan](images/38-add-sla-sev-a.png)

| Plan | Grace period | Schedule |
|---|---|---|
| Sev-A | 1 hour | 24/7 |
| Sev-B | 4 hours | 24/7 |
| Sev-C | 8 hours | Business hours, Mon-Fri |

![SLA plans created](images/39-sla-plans-created.png)

The schedule is the part people miss. A Sev-C raised at 4pm Friday on a business hours schedule is not overdue until Monday afternoon, because the clock stops overnight and at weekends. Sev-A runs 24/7, so it keeps counting at 3am.

**Help topics** are the categories end users pick from, and they drive routing and default priority.

![Default help topics](images/40-help-topics-default.png)

![Adding a help topic](images/41-add-help-topic.png)

![Help topics configured](images/42-help-topics-configured.png)

I also unchecked the setting requiring user registration, so end users can raise a ticket without an account. That matches how most internal help desks work.

## Ticket #800185: online banking outage

The scenario: Karen manages online banking at a bank. She comes back from lunch and cannot log in. Neither can any teller at her branch.

She raises it herself and, like most users, picks the wrong category. She chose Feedback, which defaults to Low priority.

![End user raising the ticket](images/43-ticket1-end-user-submit.png)

John picks it up. With View only access he can read the ticket and nothing else. No status change, no priority change, no assignment.

![Read-only agent view](images/44-ticket1-read-only-agent.png)

He can still post an internal note. Internal notes are agent-only; replies go to the user. Mixing those up is how a frustrated internal comment reaches a customer.

![Posting an internal note](images/45-ticket1-internal-note.png)

I then switched to the admin account and raised John to All Access, which is the same permission change a team lead makes when someone's scope widens.

![Granting the agent All Access](images/46-grant-agent-all-access.png)

Now the triage, which is the part of the job this lab is really teaching.

John calls Karen and confirms every teller system at the branch is down during business hours. That is a business-blocking outage, not feedback. Priority goes Low to Emergency, with the justification in the note.

![Updating priority](images/47-ticket1-update-priority.png)

SLA goes from the 18 hour default to Sev-A, one hour, 24/7.

![Updating the SLA](images/48-ticket1-update-sla.png)

Help topic goes from Feedback to Report a Problem / Business Critical Outage.

![Updating the help topic](images/49-ticket1-update-help-topic.png)

Then the escalation, with a reason attached.

![Escalation reply](images/50-ticket1-escalation-reply.png)

Every one of those changes is logged with who, when, and the before and after value.

![Ticket audit trail](images/51-ticket1-audit-trail.png)

That audit trail is why ticketing systems exist. Six months later, nobody remembers whether the ticket was always Sev-A or whether someone upgraded it forty minutes in. The ticket does.

Assigned to Jane, transferred to SysAdmins.

![Assigned and transferred](images/52-ticket1-assigned-and-transferred.png)

Worth noting what happens next: John loses access to the ticket entirely once it moves to SysAdmins, because his access is Support. That is the department model doing its job, and it is exactly the thing an end user runs into when they reply to the original technician and get silence.

Jane picks it up with the full history already in front of her. She works it, confirms the fix with Karen, and closes it.

![Ticket closed](images/53-ticket1-closed.png)

## Ticket #909979: Adobe Reader

The second ticket is deliberately vague, because real ones are. Ken writes "some people in the account department cant use adobe reader". No numbers, no error message, no screenshot.

Nothing in that sentence tells you how urgent it is, so the first action is not troubleshooting. It is calling the user.

![Full ticket thread](images/54-ticket2-full-thread.png)

What the call produced: 10 of 12 people affected, the icon hangs on double click with no error, and an internal audit is running that needs Reader. That turns an unrankable ticket into a High priority Sev-B with a documented justification.

Then the diagnosis. Ten people losing the same application overnight is not ten coincidences, it is a change. Instead of reinstalling Reader on ten machines, John contacts the Desktop Admins group and finds a Reader package was pushed the previous night. The resolution is to wait for the corrected package or pull it manually from the software catalog.

Ken confirms it works, John closes the ticket.

The useful contrast between the two tickets: #800185 needed speed and escalation, #909979 needed questions and a theory about what changed. Both get written down the same way.

## What I can speak to

- Installing and configuring IIS, PHP and MySQL, and diagnosing which layer is broken
- Reading an application's own prerequisite check instead of guessing at dependencies
- The difference between departments (visibility) and roles (permissions), and why both are needed
- How SLA grace periods interact with schedules, and why that changes a due date
- Triaging a miscategorised ticket: call the user, set priority against business impact, document the reason
- Escalating with a written handover, so the next person does not restart the investigation
- Internal notes versus replies, and why that distinction matters
- Looking for a recent change when many users break at once

---

[Back to all labs](../../README.md) · Previous: [Network traffic analysis](../02-network-traffic-analysis/) · Next: [VPN and geolocation](../04-vpn-geolocation/)
