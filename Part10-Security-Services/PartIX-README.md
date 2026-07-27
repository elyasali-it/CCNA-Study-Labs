# Part 9: Security Services

## Chapter 9: Security Architectures

I know the basic vocabulary security people use to talk about risk. A vulnerability is a weakness that can compromise something, like a door on a building. An exploit is the tool used against that weakness, like a pry bar. A threat is an actual person using that tool to break in, steal, or damage something. All three have to line up before something bad actually happens.

I understand why enterprise networks are hard to lock down. A closed system where every user and every resource is known and controlled would be secure, but no real company runs that way. Businesses connect to the internet, let employees bring their own laptops and phones, offer guest wireless, and connect to partner networks. Every one of those connections stretches the boundary of the network and gives an attacker more places to try something.

I can describe the common ways attackers go after a network:

- **Spoofing attacks** fake a source address (IP, MAC, or ARP) so traffic looks like it's coming from somewhere it isn't.
- **Denial-of-service (DoS) attacks** flood a server with fake connection requests until it can't serve real users anymore. A distributed denial-of-service (DDoS) attack does the same thing but spreads it across many infected "bot" machines at once.
- **Reflection attacks** send spoofed traffic to an innocent third party (the reflector) so that party's reply goes to the real target instead of back to the attacker.
- **Amplification attacks** are reflection attacks where the reply is much bigger than the original request, so the attacker gets more damage for less effort.
- **Man-in-the-middle attacks** let an attacker sit between two systems and quietly read or change the traffic passing between them, usually by poisoning ARP tables so both sides send data to the attacker's MAC address instead of each other's.
- **Reconnaissance attacks** are how an attacker scouts a target before attacking it — looking up domain and IP information, pinging ranges of addresses to find live hosts, and scanning ports to see what services are running.
- **Buffer overflow attacks** send more data than a system's memory buffer is built to hold, which can crash the system or let an attacker run their own code on it.

I can describe the main types of malware and the difference between them. A Trojan horse is malicious software hidden inside something that looks legitimate, and it needs a person to install it. A virus also needs a person to spread it (opening an infected file, for example), but it injects itself into other software once it's running. A worm is the most dangerous of the three because it spreads on its own, with no user interaction required, by exploiting vulnerabilities directly.

I also know the human side of security threats, since a lot of attacks target people instead of technology. Social engineering exploits human trust to get someone to hand over information or access. Phishing tries to trick people into clicking a malicious link or giving up credentials. Spear phishing is a phishing attack aimed at one specific person or a small group using research the attacker did on them first. Whaling is spear phishing aimed at high-profile targets like executives. Vishing does phishing over a phone call, and smishing does it over text message. Pharming redirects someone to a fake website even when they typed the real address correctly, usually by compromising DNS or a hosts file. A watering hole attack compromises a website that a specific group of people is known to visit, so the malware only affects the intended targets.

I know why passwords alone are considered a weak factor. They can be guessed, especially when someone uses a simple or reused password, and they can be cracked through dictionary attacks (trying words from a wordlist) or brute-force attacks (trying every possible combination). To make authentication stronger, an enterprise can add other factors: something you have (a certificate or a code sent to your phone) and something you are (a biometric like a fingerprint or facial scan). Combining more than one of these factor types is multifactor authentication.

I understand the AAA model that IT departments use to manage user access to network devices. Authentication answers who the user is. Authorization answers what that user is allowed to do. Accounting answers what the user actually did, usually recorded as log messages. Cisco's Identity Services Engine (ISE) implements AAA using two protocols: TACACS+, a Cisco protocol that encrypts the whole session and separates authentication from authorization, and RADIUS, a standards-based protocol that combines authentication and authorization but only encrypts the password, not the rest of the message.

I know a security program needs more than just technical controls. It needs user awareness so people know what to watch out for, user training so they know the company's actual policies, and physical access control so that only the right people can walk into a network closet or data center in the first place.

## Chapter 10: Securing Network Devices

I know how to protect the passwords stored in a Cisco IOS configuration. The old `service password-encryption` command scrambles clear-text passwords in the config so a shoulder-surfer can't read them directly, but that encryption is weak and easily reversed with tools found online — it only slows down a casual attacker, not a serious one.

I understand why `enable secret` is the better option compared to the older `enable password` command. The `enable secret` command stores a hash of the password instead of the password itself, using MD5 by default. Because it's a one-way hash, IOS never has to store or reveal the real password — it just re-hashes whatever the user types and compares the two hashes. If both hash values match, the user gets into enable mode.

I know IOS has moved on from plain MD5 for the enable secret. Types 8 and 9 use stronger hashing (SHA-256 and Scrypt), and I can configure any of them directly with the `algorithm-type` keyword. Only one enable secret command can exist on a device at a time — configuring a new one always replaces whichever type was there before. The same three algorithm types apply to local usernames using the `username secret` command instead of the older `username password` command, which stored things in clear text.

I know the rule IOS follows when both an enable password and an enable secret are configured: if both exist, the enable secret always wins and the enable password is ignored completely. If only one is configured, that's the one used. If neither is configured, console users go straight into enable mode with no password prompt, while Telnet and SSH users get rejected outright since there's no password to check against.

I understand what a traditional firewall actually does and how it differs from a router doing basic ACL filtering. Firewalls sit directly in the forwarding path and can match source and destination IPs and ports just like an ACL, but they go further — they can look at application-layer data, track connection state, and enforce security zones so that traffic is only allowed to initiate connections in the direction the company wants. A stateful firewall in particular remembers information about earlier packets in a flow, which lets it recognize abnormal patterns like a sudden flood of new connections that would signal a DoS attack, something a stateless router ACL can't catch on its own.

I know what security zones and a DMZ are for. Firewalls group interfaces into zones so a rule can apply to every interface in that zone at once, and the basic rule most companies use is that hosts in the inside zone can initiate connections to the outside zone, but not the other way around. A demilitarized zone (DMZ) is a separate zone for servers the public needs to reach, like a company website, so those servers stay isolated from the rest of the internal network even though the internet can talk to them.

I know how a traditional IPS works and how it's different from a firewall. Instead of an engineer defining rules by port number and zone, an IPS compares traffic against a database of exploit signatures supplied mostly by the vendor, and it can log, discard, or redirect packets that match a known attack pattern.

I understand what "next-generation" actually means for Cisco firewalls and IPS products, since it's more than a marketing label. A next-generation firewall (NGFW) still does the traditional job of stateful filtering, NAT, and VPN termination, but it adds Application Visibility and Control (AVC), which identifies the actual application in a flow instead of trusting the port number — this stops attackers who deliberately move known applications onto random ports to sneak past a traditional firewall. NGFWs can also run Advanced Malware Protection and URL filtering. A next-generation IPS (NGIPS) adds contextual awareness (knowing what OS and software each host is actually running so it can skip signatures that don't apply) and reputation-based filtering pulled from Cisco Talos threat intelligence.

## Chapter 11: Implementing Switch Port Security

I know port security is a Layer 2 tool that restricts which devices are allowed to send traffic through a specific switch port, based on the source MAC address of the frames coming in. It's meant for access ports where the network engineer already knows what device should be plugged in.

I understand the core logic every version of port security follows. Each port has a maximum number of allowed unique source MAC addresses. The switch keeps a running list of the MAC addresses it has seen on that port, and if a new address would push the count past the configured maximum, that's a violation. The switch then takes action based on the configured violation mode.

I know the three ways to define which MAC addresses are allowed:

- Statically define specific MAC addresses one at a time with `switchport port-security mac-address mac-address`.
- Let the switch learn them dynamically off the wire, up to the configured maximum.
- Use sticky learning with `switchport port-security mac-address sticky`, which tells the switch to learn addresses dynamically and automatically add them into the running configuration so I don't have to look them up by hand.

I know the basic steps to configure port security on an interface: set the port to access or trunk mode, enable port security with `switchport port-security`, and then optionally set the maximum MAC count, the violation mode, and any static or sticky MAC addresses.

I know the three violation modes and how differently they behave:

- **Shutdown** (the default) discards the offending traffic, sends log and SNMP messages, and puts the interface into an err-disabled state, which stops all traffic on the port — not just the violating frame — until someone manually recovers it with a `shutdown` then `no shutdown`, or until errdisable auto-recovery brings it back after a configured interval.
- **Restrict** discards the offending traffic and sends log and SNMP messages, but leaves the port up and forwarding good traffic.
- **Protect** only discards the offending traffic — no messages, no counter increment, and the port stays up. This one is the hardest to troubleshoot because nothing in the switch's behavior visibly flags that anything happened.

I know the commands I'd actually run to check port security status. `show port-security interface type number` gives full detail on one port — port status, violation mode, MAC counts, and the last MAC address and VLAN that triggered a violation. `show port-security` gives a shorter one-line-per-port summary across the whole switch. Once a port has port security enabled, its learned addresses stop showing up under `show mac address-table dynamic` — I have to use `show mac address-table secure` or `show mac address-table static` to see them instead.

## Chapter 12: DHCP Snooping and ARP Inspection

I understand DHCP Snooping as a Layer 2 filtering tool that watches DHCP traffic passing through a switch and decides which of it to trust. Every port in a VLAN using DHCP Snooping is either trusted or untrusted, and by default every port is untrusted. Ports connecting to an actual DHCP server, or toward the rest of the trusted network, need to be manually marked trusted with `ip dhcp snooping trust`.

I know the attack DHCP Snooping is built to stop: a rogue or spurious DHCP server. If an attacker plugs in their own laptop and starts answering DHCP requests, they can hand out a real IP address and subnet mask so the victim's connection still works, but set the default gateway to the attacker's own machine. From that point on, all of the victim's outbound traffic flows through the attacker first — a man-in-the-middle attack that started from nothing more than a bad DHCP reply.

I know the rules DHCP Snooping actually enforces. Any DHCP message normally sent by a server (like OFFER or ACK) that shows up on an untrusted port gets dropped immediately, since a legitimate server should never be sitting behind an untrusted, client-facing port. Messages normally sent by clients (DISCOVER and REQUEST) get an extra check — the switch compares the source MAC in the Ethernet header against the client hardware address (chaddr) field inside the DHCP payload, and if they don't match, it drops the message, since that mismatch is a sign of an attacker trying to exhaust the DHCP pool using spoofed addresses. Messages that release or decline an address (RELEASE and DECLINE) get checked against the DHCP Snooping binding table, so an attacker can't sit on a different port and prematurely release someone else's active lease.

I know what the DHCP Snooping binding table actually is: a running record the switch builds automatically, one entry per legitimate DHCP lease it observes, storing the client's MAC address, leased IP, VLAN, and the interface the lease came in on. That table is what makes several other features possible, including DAI.

I know the two global commands required to turn DHCP Snooping on: `ip dhcp snooping` to enable the feature, and `ip dhcp snooping vlan vlan-list` to say which VLANs it should run on. Both commands are required together. I also know that if the switch isn't also acting as a DHCP relay agent, I need `no ip dhcp snooping information option` to stop the switch from inserting option 82 fields that would otherwise cause legitimate DHCP requests to get dropped. I can rate-limit how many DHCP messages a port accepts per second with `ip dhcp snooping limit rate number`, and pair that with `errdisable recovery cause dhcp-rate-limit` so a port that trips the limit can recover automatically instead of needing a manual shutdown/no shutdown.

I understand Dynamic ARP Inspection (DAI) as the ARP-side equivalent of DHCP Snooping, and I know it depends on the DHCP Snooping binding table to work correctly by default. On untrusted ports, DAI compares the sender MAC and sender IP fields inside every incoming ARP message against that binding table. If the pairing doesn't match a known legitimate lease, DAI discards the ARP message.

I know the attack DAI is built to stop: a gratuitous ARP. Normally ARP only runs as a request-and-reply exchange, but a host is also allowed to broadcast an unsolicited ARP reply — a gratuitous ARP — to announce its own MAC address without anyone asking. An attacker can abuse that by sending a gratuitous ARP that claims to own another host's IP address. Every device that receives it will update its ARP table to point that IP at the attacker's MAC instead, which quietly redirects traffic through the attacker — another route into a man-in-the-middle position, this time through ARP instead of DHCP.

I know the two required commands to enable DAI: `ip arp inspection vlan vlan-list` globally, and `ip arp inspection trust` on any interface that should be exempt from the checks, like the port going toward the router. I know DAI can also validate the Ethernet header itself with `ip arp inspection validate`, checking that the source MAC in the Ethernet header matches the sender MAC in the ARP message, that the destination MAC on a reply matches the ARP target MAC, and that neither the sender nor target IP address is something unexpected like 0.0.0.0. I can rate-limit ARP messages per port with `ip arp inspection limit rate number burst interval seconds`, which is one difference from DHCP Snooping — DAI supports a configurable burst interval, while DHCP Snooping doesn't.

I know the verification commands for this pair of features. `show ip dhcp snooping` and `show ip dhcp snooping binding` confirm what VLANs DHCP Snooping is running on and what's currently in the binding table. `show ip arp inspection` shows the configuration and violation counters for DAI, broken down by VLAN, including how many ARP messages were dropped and specifically how many were dropped because of a DHCP Snooping binding mismatch versus an ARP ACL mismatch.

## Troubleshooting Scenario

A junior engineer calls me because users on VLAN 11 keep losing connectivity to the file server for a few seconds at a time, and it's happening on and off throughout the day. My first move is `show port-security` across the access switch, since random intermittent connectivity loss is a classic symptom of port security kicking a port into shutdown mode. Nothing shows a violation, so port security isn't it.

Next I check `show ip dhcp snooping binding` and see the table is basically empty even though there are dozens of active hosts in VLAN 11. That tells me DHCP Snooping either isn't enabled on VLAN 11 or the uplink port toward the real DHCP server was never marked trusted, which means every legitimate DHCPOFFER and DHCPACK coming from the real server is getting silently dropped as if it came from a rogue server. I run `show ip dhcp snooping` and confirm the uplink port is still sitting at its default untrusted state.

Once I add `ip dhcp snooping trust` on that uplink interface, DHCP starts completing normally again and the binding table starts filling in. But I also know that half-finished DHCP failures like this can trigger a second problem: hosts falling back to old cached leases or stale ARP entries pointing at the wrong gateway, which would show up as a man-in-the-middle-style symptom even without an actual attacker involved. So before closing the ticket, I check `show ip arp inspection statistics` for VLAN 11 to make sure the DHCP Drops counter isn't still climbing now that the binding table is populated correctly, confirming the fix actually held and DAI isn't quietly discarding legitimate ARP traffic on top of it.
