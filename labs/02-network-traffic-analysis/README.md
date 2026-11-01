# Lab 2: Network traffic analysis

Two VMs on one virtual network, a packet capture running on one of them, and five protocols watched live while I generate the traffic myself. Then a firewall rule to block one of those protocols and prove it stopped.

This is the lab where the OSI model stops being a diagram and starts being rows in a table.

| | |
|---|---|
| Resource group | `RG-Network-Activites` |
| Virtual network | `lab2-vnet`, 172.16.0.0/16 |
| Windows VM | `windows-vm`, Windows 11 Pro 25H2, private 172.16.0.4 |
| Linux VM | `linux-vm`, Ubuntu Server 24.04 LTS, private 172.16.1.4 |
| VM size | Standard D2als_v7, 2 vCPU, 4 GB |
| Analyzer | Wireshark 4.6.3 with Npcap |
| Protocols | ICMP, SSH, DHCP, DNS, RDP |

I used 172.16.0.0/16 rather than the 10.0.0.0/16 the lab suggested. Both are RFC 1918 private ranges, and I wanted the private addresses in the capture to be visibly different from the ones in the Active Directory labs so I would not confuse the two sets of screenshots later.

## Build

Resource group first.

![Create resource group](images/01-create-rg-network-activities.png)

![Virtual machines blade, empty](images/02-vms-empty.png)

Creating the Windows VM into that group. Azure will create the virtual network alongside the first VM if you let it.

![Create VM, resource group selection](images/03-create-vm-resource-group.png)

Size selection. Two vCPUs is the floor for this to be usable over RDP. The monthly figure assumes the VM runs 24/7, which mine does not.

![VM size picker](images/04-vm-size-picker.png)

![Windows VM basics](images/05-windows-vm-basics.png)

On the networking tab Azure proposes a default virtual network name. I renamed it and set the address space myself.

![Windows VM networking tab](images/06-windows-vm-networking.png)

![Virtual network address space](images/07-vnet-address-space.png)

A /16 gives 65,536 addresses in the virtual network; the subnet inside it is a /24, so 256. Azure reserves five addresses per subnet, which is why the first usable host address is .4 rather than .1.

The review screen flags that RDP is open to the internet. For a lab that is the point, since I need to reach it from my Mac. In production that port would be restricted to known source ranges or fronted by a bastion.

![Review and create](images/08-windows-vm-review-create.png)

![Deployment in progress](images/09-deployment-in-progress.png)

Worth watching this screen. One "create a VM" click produces a network security group, a public IP, a network interface, a disk, and the VM. Each of those is a separate billable resource with its own lifecycle, which is exactly why deleting the resource group rather than the VM is the right cleanup.

![Deployment resources created](images/10-deployment-resources-created.png)

Then the Ubuntu VM into the same virtual network.

![Linux VM deployed](images/11-linux-vm-deployed.png)

![Both VMs running](images/12-both-vms-running.png)

## Remote access from macOS

I am on a Mac, so RDP goes through the Windows App rather than mstsc.

<details>
<summary>RDP client setup (4 screenshots)</summary>

![RDP client with no devices](images/13-rdp-client-empty.png)

![Add PC menu](images/14-rdp-add-pc-menu.png)

![Adding the Linux VM by public IP](images/15-rdp-add-linux-vm.png)

![Saved device](images/16-rdp-saved-device.png)

</details>

![Connecting](images/17-rdp-connecting.png)

![Adding the Windows VM](images/18-rdp-add-windows-vm.png)

![First login to the Windows VM](images/19-windows-vm-first-login.jpg)

Note what the public IP is doing here. My Mac reaches 20.83.211.130, the VM's public address. Inside the virtual network the VM knows itself as 172.16.0.4. Everything in the captures below is private addressing, because the two VMs talk to each other inside the virtual network and never leave it.

## Capture setup

Wireshark installed on the Windows VM, with Npcap, which is the driver that lets Wireshark see the network adapter at all.

![Installing Wireshark](images/20-wireshark-install.png)

Pick the Ethernet adapter, not the loopback one. The little activity graph next to each interface tells you which one is actually carrying traffic.

![Selecting the capture interface](images/21-wireshark-interface-select.jpg)

An unfiltered capture on an idle machine. Nothing is happening from my point of view and it has already logged over a thousand packets.

![Unfiltered capture](images/22-capture-unfiltered.png)

That volume is the whole reason display filters exist.

## ICMP

Filter set to `icmp`. Zero packets, because nothing is pinging anything yet.

![ICMP filter with no matches](images/23-filter-icmp-empty.png)

Then `ping 172.16.1.4` from the Windows VM to the Linux VM.

![Ping with matching ICMP echo request and reply](images/24-ping-echo-request-reply.png)

Four replies in PowerShell, eight packets in Wireshark. That difference is the thing to notice: PowerShell reports what came back, Wireshark shows both halves of each exchange. Echo request out, echo reply in, four times.

Then I opened one of those frames and matched it against `ipconfig /all` on the same machine.

![MAC address in the frame matching ipconfig output](images/25-mac-address-matches-frame.png)

The source MAC in the Ethernet II header is the same physical address `ipconfig /all` reports for the adapter. That is layer 2 and layer 3 of the OSI model lined up in one screen: Ethernet carries MAC addresses, IP carries the 172.16.x.x addresses, and the ICMP payload sits inside both.

## Blocking ICMP with a network security group

A network security group is Azure's packet filter. It sits on the network interface and evaluates rules by priority, lowest number first.

![Linux VM inbound security rules](images/26-nsg-inbound-rules.png)

I added a rule denying ICMPv4 from any source to any destination at priority 290.

![Adding the deny ICMP rule](images/27-nsg-add-deny-icmp-rule.png)

Priority 290 matters. The default rules start at 65000, and my rule has to be evaluated before them to take effect. Destination port is a wildcard because ICMP has no port numbers, which is a detail that trips people up: ports are a TCP and UDP concept.

I left a continuous ping running (`ping 172.16.1.4 -t`) while the rule propagated.

![Pings timing out after the deny rule applies](images/28-ping-timeouts-after-deny.png)

The capture tells the clearer story. Before the rule: request, reply, request, reply. After: request, request, request, nothing coming back. The Linux VM is silently dropping the packets rather than refusing them, which is why the client reports a timeout instead of an error.

Deleting the rule restored replies, which confirms the rule caused the change rather than something else.

## SSH

Filter `ssh`, then `ssh labuser@172.16.1.4` from the Windows VM.

![SSH handshake in the capture](images/29-ssh-handshake-capture.png)

The handshake is visible in clear text: protocol version exchange, key exchange init, Diffie-Hellman, new keys. That is the negotiation, and it happens before encryption is in effect.

![SSH session established](images/30-ssh-session-established.png)

Once I am in, the prompt changes to `labuser@linux-vm` even though I am physically sitting in the Windows VM.

![Running commands over SSH with encrypted payloads](images/31-ssh-encrypted-payload.png)

Every keystroke generates a packet. I typed a single character and watched one frame appear. Expanding it shows "Encrypted packet" where the payload should be, so I can see that something was sent and how big it was, but not what it was.

The TCP layer is also readable: source port is high and ephemeral, destination port is 22. On the reply those two are swapped. `tcp.port == 22` as a filter produces the same result as `ssh`, which is a useful thing to know when a service runs on a non-standard port.

Contrast with Telnet, which sends the same keystrokes in clear text. That comparison is the entire reason SSH replaced it.

## DHCP

Filter on UDP ports 67 and 68, then forced a lease renewal.

![DHCP renewal captured](images/32-dhcp-renew-capture.png)

The four-step exchange is discover, offer, request, acknowledge. On a renewal you only see request and acknowledge, because the client already knows which server to talk to.

I learned something the hard way here. Running `ipconfig /release` on its own killed my RDP session immediately, because releasing the address took the network connection with it. Obvious in hindsight. The fix was to script release and renew together in a `.bat` file so the renew ran without needing a live connection, which also meant the full discover/offer/request/ack sequence got captured.

In a full discover the source address is 0.0.0.0, because the client has no address yet, and the destination is 255.255.255.255, a broadcast. It has to shout, because it does not yet know where the DHCP server is.

## DNS

Filter `dns`, then `nslookup disney.com`.

![DNS query and response captured](images/33-dns-nslookup-capture.png)

The query goes to the Azure-provided resolver at 168.63.129.16 and comes back with 130.211.198.204. The capture shows the A record lookups, the AAAA lookups for IPv6, and the SOA responses for names that do not exist.

Browsing to the raw IP returns a Disney 404 page rather than the homepage.

![Browsing the resolved IP](images/34-browse-resolved-ip.png)

That 404 is informative. The server answered, so DNS resolved correctly, but modern hosting serves many sites from one address and routes by the `Host` header. No hostname in the request, no idea which site I wanted.

DNS uses port 53 on both TCP and UDP. Client lookups use UDP; TCP comes in for zone transfers and oversized responses. `udp.port == 53 || tcp.port == 53` catches both.

## RDP

Filter `tcp.port == 3389`, which is the session I am sitting inside.

It floods instantly, even when I touch nothing. RDP streams the screen continuously, so the overhead is constant regardless of whether I am typing. SSH sends a packet per keystroke; RDP sends a picture many times per second. That is the cost of a GUI over the wire, and it is why a remote session feels slow on a bad link while SSH stays usable.

## What broke

**RDP rejected a password I had typed correctly.** Happened more than once across these labs. The fix is the Azure portal's own Reset password under Help on the VM blade, which pushes new credentials through the VM agent. It takes a minute or two to apply, so retrying immediately looks like it failed when it has not finished yet.

**`ipconfig /release` dropped my session.** Covered above. Scripting release and renew together solved it and produced a better capture than doing it by hand would have.

**The virtual network did not appear in the dropdown** when creating the second VM, even though the region matched. A hard refresh of the portal fixed it. Worth knowing that the portal caches, because the instinct is to assume you misconfigured something.

## What I can speak to

- Reading a packet capture and mapping fields to OSI layers 2, 3, and 4
- Wireshark display filters, by protocol name and by port
- Why ICMP has no port number and TCP and UDP do
- NSG rule evaluation by priority, and drop versus reject behaviour
- The DHCP four-step exchange and why discover is a broadcast from 0.0.0.0
- What is and is not visible in an encrypted SSH session
- Why DNS resolving correctly does not mean a site will load over its IP
- The bandwidth difference between a streamed GUI session and a terminal session

---

[Back to all labs](../../README.md) · Previous: [Azure fundamentals](../01-azure-fundamentals/) · Next: [osTicket help desk](../03-osticket-helpdesk/)
