# Part VIII: IP Access Control Lists

## TCP and UDP: How Applications Share a Connection

Before I could really understand ACLs, I had to understand what TCP and UDP are actually doing at Layer 4. Layer 4 just means the transport layer, the part of the network stack that handles getting data between applications, not just between devices.

TCP (Transmission Control Protocol) is the reliable option. It does error recovery, meaning if a piece of data gets lost, TCP notices and resends it. It also does flow control using something called windowing, which is really just a way for the receiving computer to tell the sender "slow down, I can only handle this much right now." TCP also handles connection establishment and termination, which is the three-way handshake (SYN, SYN-ACK, ACK) I'd heard about before but never really understood the purpose of. It exists so both sides agree on port numbers and synchronize sequence numbers before any real data moves.

UDP (User Datagram Protocol) skips all of that. No error recovery, no windowing, no connection setup. It's faster and uses less overhead, which is why real-time stuff like VoIP and video use it. If a voice packet gets lost, retransmitting it late doesn't help anyone, so there's no point in TCP's overhead for that kind of traffic.

Both protocols share one important concept: multiplexing using port numbers. If a computer has three applications running at once, TCP and UDP use port numbers to know which packet belongs to which app. A socket is the combination of an IP address, a transport protocol, and a port number, and that combination is what makes a connection between two computers unique.

Port ranges matter for troubleshooting:
- Well-known ports: 0–1023 (assigned by IANA to standard services)
- Registered/user ports: 1024–49151
- Ephemeral (dynamic) ports: 49152–65535, used temporarily by client applications

I also learned the practical side of DNS here. DNS uses port 53, and depending on the situation it can use UDP or TCP. A client asks a DNS server for the IP address behind a hostname, and if that DNS server doesn't know the answer, it can act as a recursive resolver and go ask other DNS servers (root, then TLD, then the authoritative server) until it gets an answer back.

HTTP versions also showed up here in a way I hadn't thought about before. HTTP/1.0 and 1.1 run over TCP on port 80 (or 443 for HTTPS). HTTP/2 still uses TCP and the same ports, it just improves performance under the hood. HTTP/3 is different: it moves to UDP using a protocol called QUIC, and it uses port 443 for both encryption and transport. That's a real shift I need to remember when I'm looking at ACLs or firewall rules built around expected ports, since HTTP/3 traffic won't look like classic HTTP/HTTPS traffic on the wire.

## Standard IPv4 Access Control Lists

An ACL is a filter I can apply to a router interface that decides whether to let a packet through or discard it. Standard ACLs are the simplest type because they can only match one thing: the source IP address of a packet. That's it. No destination address, no port numbers, no protocol type.

The two decisions I have to make with any ACL are location and direction. Location means which interface I enable it on, direction means whether I apply it to traffic coming in (inbound) or going out (outbound) on that interface. Since standard ACLs can only match source address, the rule I picked up is to place them close to the destination, not close to the source. If I put a standard ACL too close to the source, I risk blocking traffic to more places than I meant to, since I can't narrow it down by where the traffic is headed.

ACLs use first-match logic. The router checks each line in order, and as soon as a packet matches a line, that line's action (permit or deny) is applied and the router stops checking further lines. If a packet doesn't match any line in the ACL, it hits an implicit deny at the very end, meaning every ACL ends with an invisible "deny everything else" unless I write my own permit statements to override that.

Wildcard masks are how I tell the router which parts of an address to check and which parts to ignore. This tripped me up until I boiled it down to two rules:
- A 0 in the wildcard mask means "compare this octet normally."
- A 255 in the wildcard mask means "ignore this octet, treat it as already matching."

So a wildcard mask of 0.0.0.255 checks the first three octets and ignores the last one, meaning it matches an entire /24 subnet. There's also a shortcut for matching a whole subnet: use the subnet number as the source, and find the wildcard mask by subtracting the subnet mask from 255.255.255.255.

There are three ways to write the same single-host match in a standard ACL: `10.1.1.1 0.0.0.0`, `host 10.1.1.1`, or just `10.1.1.1` by itself when no wildcard follows. All three tell the router "match this exact address, nothing else."

Implementing a standard ACL comes down to three steps: plan the location and direction, write the access-list commands (remembering the list is processed top to bottom with an implicit deny any at the end), then enable it on the interface using `ip access-group [number] in|out`. I also learned that adding the `log` keyword to an ACL line makes the router generate log messages whenever that line matches traffic, which is genuinely useful for troubleshooting because I can see exactly which line is catching which packets, and the router keeps a running match counter I can check with `show access-lists` or `show ip access-lists`.

## Named and Extended IP ACLs

Numbered ACLs work, but they're clunky to edit. Named ACLs solve that. Instead of `access-list 1 permit ...`, I use `ip access-list standard NAME` (or `extended NAME`), which drops me into ACL configuration mode where I type `permit` and `deny` as subcommands instead of global commands.

Named ACLs use sequence numbers automatically, starting at 10 and incrementing by 10 for each new line. That matters because it lets me insert or delete individual lines without rebuilding the whole ACL. To delete one line, I either retype that exact permit/deny/remark command with a `no` in front, or I use `no [sequence-number]` to remove just that line by its number. To insert a new line in a specific spot, I give the permit or deny command a sequence number that falls between two existing ones.

Numbered ACLs can also be edited in ACL mode now (`ip access-list standard [number]`), which is a newer capability. But the old-style global `access-list` commands still have real limits: I can't delete a single line with a global command (any `no access-list [number] ...` command deletes the entire ACL), and I can't insert a line anywhere except the end.

Extended ACLs are the more useful type in real networks because they can match protocol, source IP, destination IP, and for TCP/UDP, source and destination port numbers. Every field in an extended ACL line has to match the packet for that line to count as a match. That's an easy thing to get wrong: if I write a UDP rule but the packet actually has a TCP header, it doesn't match, even if the IP addresses are perfect.

The basic syntax pattern is: `permit|deny protocol source [wildcard] destination [wildcard] [port info]`. The protocol keyword is usually `ip`, `tcp`, `udp`, or `icmp`. For TCP and UDP, I can add `eq [port]` to match a specific port, or use operators like `gt`, `lt`, `ne`, and `range` to match port ranges. IOS actually swaps some numeric ports for keywords automatically in the running-config, so port 80 becomes `www`, port 23 becomes `telnet`, and so on, though it doesn't have a keyword for every port (443 stays numeric, for example).

One detail that matters for direction: if I'm filtering packets going toward a server, I check the destination port. If I'm filtering the return traffic coming from that server, the well-known port shows up as the source port instead. Getting that backwards is one of the more common ACL mistakes.

Standard ACLs support three ways to write a single address, extended ACLs only support two (`address wildcard-mask` or `host address`, no bare address with default wildcard). Also, for numbered ACLs, standard ACL numbers are 1–99 or 1300–1999, and extended ACL numbers are 100–199 or 2000–2699.

## Applied IP ACLs and Infrastructure Traffic

Writing an ACL that only handles obvious end-user traffic is the easy part. The harder part, and the part this section is really about, is making sure I don't accidentally break infrastructure protocols that keep the network functioning: DNS, DHCP, ICMP, routing protocols, and remote access like SSH.

For DNS, since it can use either UDP or TCP on port 53, a production ACL needs to account for both. I can go broad (permit all DNS traffic to any destination) or more locked down (permit DNS only to the specific IP addresses of my known DNS servers, then explicitly deny all other DNS traffic afterward as a second layer of control).

For ICMP, the practical reason I need it permitted at all is that ping relies on it. Ping sends ICMP Echo Request messages and expects ICMP Echo Reply messages back. If an ACL blocks all ICMP, ping breaks. There's a tradeoff here too: permitting all ICMP is simple but loose, while narrowing it down to only Echo and Echo Reply is tighter but can accidentally break traceroute, since traceroute depends on ICMP Time Exceeded messages that a narrow ACL might not have accounted for.

OSPF is worth calling out because of a router behavior I didn't expect: routers bypass outbound ACLs for locally generated packets, which includes OSPF hello messages the router itself originates. That means an outbound ACL won't filter OSPF traffic the router is sending. Inbound ACLs are what actually matter for OSPF, since incoming OSPF packets do get checked. The safe move is to just permit all OSPF traffic rather than trying to filter it, since there's rarely a good reason to deny it and a mistake here can break routing adjacencies.

DHCP has a quirk tied to the IP helper function. When a router uses `ip helper-address` to forward DHCP broadcasts to a real DHCP server, the packet's source and destination addresses get rewritten as part of that helper process, and that rewriting happens before any outbound ACL sees the packet, but the inbound ACL sees the packet before the rewrite. That means an inbound ACL on the interface facing the DHCP client has to match the original unhelped addresses (source 0.0.0.0, destination 255.255.255.255), not the addresses that show up after the helper does its job.

For SSH and Telnet, both use TCP, with SSH on port 22 and Telnet on port 23. Since either side of a connection could be the one initiating or responding, and I usually don't know in advance where every SSH server or client sits, the safer pattern is to check for both the source and destination port using the `range 22 23` syntax so I catch traffic in both directions rather than just traffic headed toward the well-known port.

There's also a completely separate way to protect the router or switch itself: a vty ACL, applied with the `access-class` command inside `line vty` configuration, rather than the `ip access-group` command used on regular interfaces. A vty ACL isn't tied to a specific interface, and IOS only applies vty ACL logic to packets destined for the router's own IP address on the SSH or Telnet ports, nothing else. Outbound vty ACLs exist too and work in a way I found counterintuitive: they filter based on the destination IP address the router itself is trying to connect out to (like when I use the router's own `ssh` or `telnet` command to reach another device), not based on the source of an incoming connection.

Two more differences show up specifically in IOS XE. First, ACL sequence number persistence: classic IOS automatically renumbers every ACE back to increments of 10 whenever the router reloads, while IOS XE keeps whatever numbering I set, by default, and calls that behavior ACL persistence. Second, IOS XE supports a "common ACL" feature that lets me enable two IPv4 ACLs on the same interface and direction at once, using `ip access-group common [name] [name] in|out`, with the common ACL processed first. Classic IOS is limited to one ACL per protocol, per direction, per interface, no exceptions.

## Troubleshooting Scenario

If I get a ticket saying a branch office suddenly can't resolve hostnames, can't get new DHCP leases, and OSPF just dropped between two routers, all after someone added an ACL to a WAN interface, here's how I'd work through it using what's in this Part.

First, I'd check location and direction on the ACL with `show ip interface [interface]` to confirm exactly which direction and interface the ACL is applied to, since a packet only gets filtered if the ACL is on the actual path that packet travels. Then I'd pull `show access-lists` or `show ip access-lists` to see the match counters per line, which tells me which lines are actually being hit and which ones are sitting at zero.

For the DNS failure, I'd check whether the ACL matches both UDP and TCP port 53, since DNS can use either depending on the query size and situation, and a rule that only covers one transport protocol would explain intermittent failures.

For the DHCP failure, I'd remember the helper-address ordering issue: if this is the interface doing `ip helper-address`, the inbound ACL needs to match the pre-helper addresses (source 0.0.0.0, destination 255.255.255.255), not the addresses that show up after the helper rewrites them, since the ACL evaluates the packet before that rewrite happens.

For OSPF, since routers bypass outbound ACLs for their own locally generated packets, I'd focus on the inbound direction and confirm there's an explicit `permit ospf any any` (or a more specific version matching the neighbor's IP) somewhere in the ACL rather than assuming an outbound permit would cover it.

And underneath all of that, I'd keep first-match logic in mind the whole time. If a broad deny statement got placed too early in the list, above the permits that were supposed to handle DNS, DHCP, and OSPF, the router would stop checking further lines the moment it hit that early deny, and none of the more specific rules below it would ever get a chance to match.
